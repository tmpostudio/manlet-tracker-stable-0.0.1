# Pushup Tracker v0.1

A computer vision-based pushup counter with anti-cheat mechanisms for accurate rep counting.

## Features

- **Real-time pose detection** using TensorFlow.js MoveNet
- **Anti-cheat constraints** to prevent gaming the system
- **60-second timed sessions**
- **Debug panel** for monitoring detection accuracy
- **Mobile-optimized** with responsive design
- **Mirror mode** for natural viewing experience

## Quick Start

1. Download `pushup-tracker-stable-v0.1.html`
2. Open in Chrome or Safari (requires camera permissions)
3. Position yourself in front-facing view (camera shows full body)
4. Click START and begin your pushup session

## Browser Requirements

- Chrome/Safari (desktop or mobile)
- Camera access permissions
- JavaScript enabled

## Technical Details

See `CONSTRAINTS.txt` for detailed information about the anti-cheat mechanisms and implementation.

## Debug Mode

Click TOGGLE button to show/hide the debug panel which displays:
- Left/Right elbow angles
- Average elbow angle
- Plank detection status with ratios
- Wrist symmetry status
- Current state (none/up/down)

## Configuration

Key thresholds can be adjusted in the CONFIG object:
- `ELBOW_ANGLE_DOWN`: 90° (bent arm threshold)
- `ELBOW_ANGLE_UP`: 160° (extended arm threshold)
- `PLANK_SHOULDER_WIDTH_MULTIPLIER`: 1.5 (wrist-to-hip distance threshold)
- `WRIST_SYMMETRY_CM`: 15cm tolerance for asymmetry

## Known Limitations

- Requires front-facing camera view
- Works best with good lighting
- Full body must be visible in frame
- Hip keypoints must be detected (may fail with loose clothing)

## License

MIT
