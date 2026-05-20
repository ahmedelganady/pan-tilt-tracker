# pan-tilt-tracker

Pan-tilt camera that visually tracks an object in real-time. ROS2 Jazzy, OpenCV, Raspberry Pi 5.

> **Status:** scaffolding (20 May 2026). Active development through 7 June 2026.

## Demo

_Coming soon._

## What this is and why it exists

A pan-tilt camera rig that visually tracks an object in real-time. Two milestones:

- **v1** (target 1 June 2026): tracks a yellow padel ball using HSV colour thresholding.
- **v2** (target 7 June 2026): tracks an arbitrary user-selected object via OpenCV CSRT, by swapping the `tracker_node` only — no other node changes.

This is intentionally the smallest realistic system that touches every layer of a modern robotics stack — perception, estimation, control, actuation, middleware — and demonstrates them working together. The padel-ball detector is intentionally trivial so the engineering focus stays on the integration loop.

See [`docs/rfp.md`](docs/rfp.md) for the full project specification.

## Key results

_Pending v1 acceptance. See [`docs/results.md`](docs/results.md) when populated._

## Hardware

Raspberry Pi 5, Pi Camera Module 3, 2× MG90S servos, PCA9685 PWM driver, separate 5 V servo supply.

Full BOM and wiring in [`docs/hardware.md`](docs/hardware.md).

## Architecture

Four ROS2 nodes — `camera_node`, `tracker_node`, `controller_node`, `servo_node` — communicating via topics. PI control on pixel error, mapped to angular error via camera FOV.

Full design in [`docs/architecture.md`](docs/architecture.md).

## Quickstart

_To be written once v1 lands. Will be tested step-by-step against NFR-8 (a second person can bring the system up within 60 minutes from a clean clone)._

## Repo layout

```
pan-tilt-tracker/
├── docs/                       # design, hardware, calibration, results, report
├── ros2_ws/src/pan_tilt_tracker # ROS2 package: nodes, launch, config, tests
├── scripts/                    # standalone helpers (HSV tuner, servo sweep, etc.)
└── media/                      # demo videos, architecture diagram, rig photos
```

## What I learned / what I'd do differently

_To be written. See [`docs/report.md`](docs/report.md) for the full reflective report._

## License

MIT. See [`LICENSE`](LICENSE).

## Acknowledgements

To be added with the report. Notable upstream sources: official ROS2 documentation, Adrian Rosebrock's ball-tracking tutorials, Brian Douglas's PID Tech Talks, Justin Johnson's EECS 498-007 lectures.
