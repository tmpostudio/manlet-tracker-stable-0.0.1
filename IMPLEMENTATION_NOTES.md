# Pushup Tracker v0.1 - Implementation Notes

## Overview
This document explains the key technical decisions, solutions to problems encountered, and important implementation details for the Pushup Tracker. Read this before making modifications to understand the "why" behind the code.

---

## Critical Technical Decisions

### 1. Why TensorFlow MoveNet (Not MediaPipe)
**Decision**: Use TensorFlow.js MoveNet SINGLEPOSE_LIGHTNING model
**Reason**: 
- Simpler coordinate mapping than MediaPipe
- More reliable keypoint detection in our testing
- Better performance on the V001 prototype
- Direct keypoint access without complex wrappers

**Trade-off**: MediaPipe has more features, but added unnecessary complexity

### 2. Keypoint-to-Canvas Alignment (CRITICAL!)
**Problem**: Skeleton joints didn't align with video feed - they were offset and incorrectly scaled.

**Root Cause**: When using `object-fit: cover` on video, the browser crops/scales the video to fill the container. The canvas keypoints use raw video coordinates (0-1 normalized), but need to be mapped to the *visible* portion of the video after cropping.

**Solution**: Calculate video scaling with offset
```javascript
function calculateVideoScale() {
    const videoAspect = video.videoWidth / video.videoHeight;
    const containerAspect = canvas.width / canvas.height;
    
    if (videoAspect > containerAspect) {
        // Video is wider - crop left/right
        renderHeight = canvas.height;
        renderWidth = video.videoWidth * (canvas.height / video.videoHeight);
        offsetX = (canvas.width - renderWidth) / 2;
        offsetY = 0;
    } else {
        // Video is taller - crop top/bottom
        renderWidth = canvas.width;
        renderHeight = video.videoHeight * (canvas.width / video.videoWidth);
        offsetX = 0;
        offsetY = (canvas.height - renderHeight) / 2;
    }
    
    return { offsetX, offsetY, scaleX: renderWidth / video.videoWidth, scaleY: renderHeight / video.videoHeight };
}
```

Then apply when drawing:
```javascript
ctx.arc(
    offsetX + keypoint.x * scaleX,  // NOT just keypoint.x * scaleX
    offsetY + keypoint.y * scaleY,
    6, 0, 2 * Math.PI
);
```

**Why This Matters**: Without this, skeleton joints appear offset from the actual body parts in the video. This is the #1 issue that breaks the visual feedback.

---

## Anti-Cheat Constraint Design

### 3. Why Shoulder Width as Universal Reference
**Decision**: Use shoulder width as the measuring unit for all distance-based constraints

**Reason**:
- Scales automatically with body size (tall person vs short person)
- Scales automatically with camera distance (close vs far)
- Always visible in front-facing pushup view
- Eliminates need for hard-coded pixel values

**Example**: "Wrist-to-hip distance must be ≤ 1.5x shoulder width"
- For a person with 40cm shoulders: max distance = 60cm
- For a person with 50cm shoulders: max distance = 75cm
- Camera 2m away or 5m away: ratio stays the same

**Alternative Considered**: Fixed pixel distances → rejected because they break at different camera distances

### 4. Wrist-to-Hip Distance (Not Wrist-to-Ankle)
**Decision**: Measure from wrists to hips to detect plank position

**Evolution**:
1. First attempt: Knee-ankle alignment → didn't work well, legs often cut off in frame
2. Second attempt: Wrist-to-ankle distance → feet detection unreliable
3. **Final**: Wrist-to-hip distance → hips always visible, reliable detection

**Implementation**:
```javascript
// Check BOTH sides independently
const leftDistance = calculateDistance(leftWrist, leftHip);
const rightDistance = calculateDistance(rightWrist, rightHip);

// Both must be ≤ 1.5x shoulder width
if (leftDistance > threshold || rightDistance > threshold) {
    return { valid: false };
}
```

**Why Both Sides**: Prevents one-arm pushups or asymmetric positioning

### 5. Standing Detection (Shoulder-Hip Vertical Alignment)
**Problem**: User could stand up, raise hands to shoulders, and trigger rep counts

**Solution**: Check vertical distance between shoulders and hips
```javascript
const shoulderHipVerticalDiff = Math.abs(avgHipY - avgShoulderY);

// In plank: shoulders and hips at similar Y (horizontal body)
// Standing: hips much lower than shoulders (vertical body)
if (shoulderHipVerticalDiff > shoulderWidth * 1.5) {
    return 'Standing detected';
}
```

**Why 1.5x multiplier**: Allows for slight body angles in plank without false positives

### 6. Wrist Orientation Check
**Problem**: User standing with hands raised could pass other checks

**Solution**: Wrists must be *below* shoulders in screen space
```javascript
// Y increases downward in screen coordinates
if (avgWristY < avgShoulderY) {
    return 'Wrong orientation';
}
```

**Simple but effective**: Catches inverted/standing positions immediately

### 7. Wrist Symmetry (±15cm tolerance)
**Purpose**: Prevent one-arm pushups or heavily asymmetric form

**Implementation**: Compare vertical distance of each wrist from its shoulder
```javascript
const leftWristHeight = Math.abs(leftWrist.y - leftShoulder.y);
const rightWristHeight = Math.abs(rightWrist.y - rightShoulder.y);

// Convert to cm using shoulder width reference
const heightDiff = Math.abs(leftWristHeight - rightWristHeight) * cmPerUnit;

return heightDiff <= 15; // 15cm tolerance
```

**Why 15cm**: Allows natural slight asymmetry but prevents gaming the system

---

## State Machine & Rep Counting

### 8. Three-State System
```
none → down → up → down → up (rep counted on down→up transition)
```

**States**:
- `none`: Initial state or invalid form
- `down`: Elbows bent < 90°
- `up`: Elbows extended > 160°

**Rep Counted When**:
1. User is in `down` state
2. All constraints pass (plank position, wrist symmetry, orientation, not standing)
3. Elbow angle exceeds 160°
4. Hold time requirement met (if enabled)

**Why This Order**: Down→Up transition ensures full range of motion

### 9. Hold Time (Optional Feature)
**Purpose**: Prevent "bouncing" to game the system
**Implementation**: Track time spent in each position
```javascript
if (currentState === 'down') {
    const timeAtBottom = currentTime - lastBottomTime;
    if (timeAtBottom >= holdTime) {
        // Transition allowed
    }
}
```

**Default**: 0ms (disabled) - Can be adjusted via toggle during testing

---

## UI/UX Decisions

### 10. Feedback Debouncing (100ms)
**Problem**: Feedback messages jittered rapidly when user was on threshold of passing/failing

**Solution**: Debounce feedback state changes
```javascript
const FEEDBACK_DEBOUNCE_MS = 100;
```

**Why 100ms**: 
- Fast enough to feel responsive
- Slow enough to prevent jitter
- Sweet spot found through testing

**Note**: Debug panel updates immediately without debounce for real-time monitoring

### 11. Mobile-First Design
**Decision**: Max width 430px (iPhone 14 Pro Max), centered on desktop

**Why**: Primary use case is mobile phone propped up while doing pushups

**Implementation**:
```css
#root {
    max-width: 430px;
    max-height: 932px;
}
```

### 12. Mirror Mode Always On
**Decision**: Video is always mirrored (`transform: scaleX(-1)`)

**Why**: Natural mirror experience - when you raise your right hand, the right side of the video moves

**Critical**: Canvas must also be mirrored to keep skeleton aligned

---

## MoveNet Keypoint Indices
For reference, the keypoint indices used:

```javascript
0: nose
1: left_eye
2: right_eye
3: left_ear
4: right_ear
5: left_shoulder   // Used for constraints
6: right_shoulder  // Used for constraints
7: left_elbow      // Used for angle calculation
8: right_elbow     // Used for angle calculation
9: left_wrist      // Used for constraints
10: right_wrist    // Used for constraints
11: left_hip       // Used for plank detection
12: right_hip      // Used for plank detection
13: left_knee
14: right_knee
15: left_ankle
16: right_ankle
```

---

## Configuration Values & Tuning

### Current Thresholds
```javascript
ELBOW_ANGLE_DOWN: 90°        // Bent elbow threshold
ELBOW_ANGLE_UP: 160°         // Extended elbow threshold
MIN_CONFIDENCE: 0.3          // Keypoint confidence threshold
WRIST_SYMMETRY_CM: 15        // Max wrist height difference
PLANK_SHOULDER_WIDTH_MULTIPLIER: 1.5  // Wrist-to-hip max distance
FEEDBACK_DEBOUNCE_MS: 100    // Feedback update delay
SESSION_DURATION: 60         // Session length in seconds
```

### How to Tune
If constraints are too strict/loose:

**Making plank detection more lenient**:
```javascript
PLANK_SHOULDER_WIDTH_MULTIPLIER: 1.5 → 2.0
```

**Making wrist symmetry more strict**:
```javascript
WRIST_SYMMETRY_CM: 15 → 10
```

**Adjusting ROM requirements**:
```javascript
ELBOW_ANGLE_UP: 160° → 150°  // Less extension required
ELBOW_ANGLE_DOWN: 90° → 100° // Less bend required
```

---

## Known Limitations

### 1. Hip Keypoint Occlusion
**Issue**: Baggy clothing or certain body positions can hide hip keypoints
**Impact**: Plank detection may fail
**Workaround**: Ensure hips are visible, wear form-fitting clothing
**Future**: Could add fallback to torso length or other measurements

### 2. Lighting Sensitivity
**Issue**: Poor lighting reduces keypoint confidence
**Impact**: Constraints may not detect body properly
**Recommendation**: Good lighting required for reliable detection

### 3. Camera Angle Dependency
**Issue**: Side views or angled views don't work well
**Requirement**: Front-facing view with full body visible
**Why**: Constraints designed specifically for front view

### 4. Frame Rate Performance
**Issue**: On slower devices, pose detection may lag
**Impact**: Rep counting might be delayed
**Note**: MoveNet LIGHTNING model chosen for speed

---

## Testing & Debugging

### Debug Panel
Toggle to show real-time constraint values:
- Left/Right/Average elbow angles
- Plank detection status with ratios
- Wrist symmetry status
- Current state machine state

**How to Use**:
1. Click TOGGLE button to show/hide
2. Watch values while doing pushups
3. Adjust thresholds based on observed values

### Common Debug Scenarios

**Skeleton offset from body**:
- Check `calculateVideoScale()` is being called
- Verify `offsetX/offsetY` being applied to keypoint drawing
- Ensure canvas and video both have `scaleX(-1)` for mirror mode

**Plank detection always failing**:
- Check debug panel for actual ratios (e.g., "L:2.3x R:2.4x")
- If values consistently above threshold, increase `PLANK_SHOULDER_WIDTH_MULTIPLIER`
- Verify hip keypoints being detected (confidence > 0.3)

**Reps not counting**:
- Check which constraint is failing in debug panel
- Verify user is actually in proper form
- Check elbow angles match expected thresholds

---

## Future Enhancement Ideas

### Potential Improvements
1. **Adaptive thresholds**: Learn user's body proportions over time
2. **Side view support**: Different constraint set for side-facing camera
3. **Rep quality scoring**: Grade each rep (A, B, C, D, F)
4. **Form coaching**: Specific feedback on what to improve
5. **Historical tracking**: Save sessions, show progress over time
6. **Multi-person detection**: Support workout partners
7. **Voice feedback**: Audio cues instead of just visual

### Architecture for Extensions
- Constraint system is modular - easy to add new checks
- State machine can be extended with more states
- Configuration values centralized for easy tuning
- Debug panel template ready for additional metrics

---

## File Structure

```
pushup-tracker-stable-v0.1/
├── pushup-tracker-stable-v0.1.html  // Main application (single file)
├── CONSTRAINTS.txt                   // User-facing constraint documentation
├── IMPLEMENTATION_NOTES.md           // This file - developer documentation
└── README.md                         // Setup and usage guide
```

---

## Development Workflow

### Making Changes
1. Read CONSTRAINTS.txt to understand current rules
2. Read this file (IMPLEMENTATION_NOTES.md) to understand implementation
3. Test changes with debug panel enabled
4. Update configuration values if needed
5. Document any new constraints or major changes

### Adding New Constraints
1. Define the constraint logic (what body position to check)
2. Choose appropriate reference measurement (shoulder width, etc.)
3. Add to `analyzePushUp()` function
4. Add debug output to show constraint value
5. Test extensively at different camera distances/body sizes
6. Update CONSTRAINTS.txt with user-facing description
7. Update this file with technical reasoning

---

## Common Pitfalls to Avoid

### ❌ Don't Use Fixed Pixel Values
```javascript
// BAD - breaks at different camera distances
if (distance > 200) { /* ... */ }

// GOOD - scales with body size
if (distance > shoulderWidth * 1.5) { /* ... */ }
```

### ❌ Don't Forget Coordinate Offsets
```javascript
// BAD - skeleton won't align
ctx.arc(keypoint.x * scaleX, keypoint.y * scaleY, ...);

// GOOD - accounts for video cropping
ctx.arc(offsetX + keypoint.x * scaleX, offsetY + keypoint.y * scaleY, ...);
```

### ❌ Don't Mirror Only Video or Only Canvas
```javascript
// BOTH must have scaleX(-1) or neither
video { transform: scaleX(-1); }
canvas { transform: scaleX(-1); }
```

### ❌ Don't Check Constraints Before Confidence
```javascript
// BAD - may use invalid keypoints
const angle = calculateAngle(shoulder, elbow, wrist);

// GOOD - check confidence first
if (shoulder.score < MIN_CONFIDENCE || ...) return;
const angle = calculateAngle(shoulder, elbow, wrist);
```

---

## Performance Considerations

### Frame Rate
- MoveNet LIGHTNING chosen for speed (vs THUNDER which is slower but more accurate)
- Runs at ~30fps on modern devices
- Canvas drawing is relatively cheap
- Pose estimation is the bottleneck

### Memory
- Single model loaded once
- No video recording/storage
- Minimal state tracking
- Should run indefinitely without memory leaks

### Battery
- Continuous pose detection is CPU-intensive
- 60-second sessions keep battery drain reasonable
- Consider adding "rest mode" between sessions

---

## Version History Context

### Why V0.1 Works Better Than Later Attempts
**Issue**: Later prototypes (after V001) had video stretching and poor keypoint alignment

**Root Causes**:
1. Switched from TensorFlow MoveNet to MediaPipe (unnecessary complexity)
2. Lost the proper coordinate offset calculation
3. Changed canvas sizing approach

**Solution**: Return to V001 approach:
- TensorFlow MoveNet
- Simple coordinate scaling with offset
- Clear separation of video and canvas coordinate systems

**Lesson**: When something works, document *why* it works before changing it

---

## Questions for Future Developers

If you're modifying this code, ask yourself:

1. **Does my change account for different camera distances?**
   - Use proportional measurements (shoulder width)

2. **Does my change work for different body sizes?**
   - Test with tall/short people

3. **Did I update the debug panel?**
   - New constraints should show their values

4. **Did I test at the threshold boundaries?**
   - Test right at 1.5x shoulder width, exactly 15cm asymmetry, etc.

5. **Does the skeleton still align with the video?**
   - After any coordinate or scaling changes

---

## Contact & Contributions

This is a prototype. Areas needing work:
- More robust hip detection
- Better handling of occluded keypoints
- Adaptive threshold learning
- Side-view camera support

When making changes, please update both:
- CONSTRAINTS.txt (user perspective)
- This file (developer perspective)

---

*Last Updated: November 2025*
*Version: 0.1*
