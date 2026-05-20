# RFP: Pan-Tilt Camera Tracker

**Version:** 0.1 (draft)
**Date issued:** 20 May 2026
**v1 acceptance target:** 1 June 2026
**v2 acceptance target:** 7 June 2026
**Project codename:** `pan-tilt-tracker`

---

## 1. Executive Summary

A pan-tilt camera rig that visually tracks an object in real-time, built on Raspberry Pi 5 with ROS2 Jazzy, controlled via PI feedback on pixel error, and developed remotely from a Mac. Two milestones: **v1** tracks a yellow padel ball using HSV colour thresholding; **v2** tracks an arbitrary user-selected object using a classical correlation tracker (OpenCV CSRT). The architecture is fixed across both versions — only the `tracker_node` implementation changes. Deliverables include a public GitHub repository, a system report, validation results, and a demo video, all of portfolio quality suitable for a PhD application.

## 2. Project Framing

This is not really a "build a tracking camera" project. It is the smallest realistic system that touches every layer of a modern robotics stack — perception, estimation, control, actuation, middleware, and engineering practice — and demonstrates them working together. The padel-ball detector is intentionally trivial so that the engineering focus stays on the integration loop. The v1/v2 split exists to prove the architecture is modular: swapping the perception stage must not perturb the control stage. That is the lesson ROS2 is supposed to teach, made concrete.

The project also serves as the **portfolio template** for everything that follows this summer. Repo layout, README conventions, validation discipline, and reporting style established here are intended to be reused on subsequent projects with minimal modification.

## 3. Objectives

**Capability objectives**

- **C1:** A physical pan-tilt rig that visually tracks a yellow padel ball on the bench in real-time, smoothly, without manual intervention, under reasonable indoor lighting at 0.5–2 m range.
- **C2:** The same rig tracks an arbitrary user-initialised object, using only a `tracker_node` swap with no changes to camera, controller, or servo nodes.

**Learning objectives**

- **L1:** Working fluency with ROS2 Jazzy fundamentals: nodes, topics, messages, parameters, launch files, packages, `colcon` builds.
- **L2:** A functional Linux + SSH + VS Code Remote-SSH development workflow, fast enough to be the default for all future robotics work.
- **L3:** Hands-on PID intuition built from first principles, not blind tuning — able to articulate why each gain was chosen.
- **L4:** End-to-end systems thinking: identifying latency budgets, tracing failures across layers, understanding where each subsystem's limits bite.
- **L5:** Comfort with hardware bring-up: I2C, PWM, separate power rails, the discipline of testing each component in isolation before integration.

**Artefact objectives**

- **A1:** A public GitHub repository that a stranger can read in five minutes and understand what was built, why, and how.
- **A2:** A demo video that demonstrates the capability claims in §6 without narration.
- **A3:** A validation results document containing measured numbers for every non-functional requirement.
- **A4:** A reflective engineering report of ~1500–2500 words, written to a standard a PhD reviewer would find credible.

## 4. Scope

**In scope for v1:**

- Single object: yellow padel ball (high-saturation yellow, ~6.5 cm diameter).
- Single object at a time in the frame.
- Indoor, controlled lighting (overhead artificial or daylight, no direct backlight).
- Target distance 0.5–2 m from camera.
- Bench operation only (rig stationary on a desk/tripod).
- 2-DOF tracking: pan and tilt.

**In scope for v2:**

- All of the above, but with the target initialised by a user-drawn bounding box in the first frame.
- Tested with at least three distinct objects of different colours and textures.
- Same lighting and distance envelope as v1.

**Out of scope (both v1 and v2):**

- Multi-object tracking, object handoff, or identity-stable tracking through crossings.
- Outdoor operation or operation under direct sunlight / strong backlight.
- Tracking at distances >2 m or <0.3 m.
- 3D pose estimation, depth estimation, or stereo vision.
- Learned object detectors (YOLO, MobileNet, etc.).
- Mobile base or vehicle integration.
- Gazebo / Isaac simulation (everything runs on real hardware).
- ROS2 Navigation, MoveIt, or any other ROS2 capability beyond core pub/sub, parameters, and launch.
- IMU, encoders, or any sensor beyond the Pi camera.
- Velocity or feedforward control (P or PI only).

**Explicitly deferred (potential v3, post-7-June, no commitment):**

- Class-conditional learned detection (e.g. YOLOv8n) with runtime class selection.
- Multi-object tracking with re-identification.
- AprilTag / fiducial-based pose tracking.
- Predictive tracking with a Kalman filter or particle filter.

## 5. Functional Requirements

Each requirement is a testable shall-statement. They apply to v1 unless v2 is noted.

- **FR-1:** The system shall capture frames from the Pi Camera Module 3 at a rate of at least 20 frames per second at a resolution of at least 640×480.
- **FR-2:** The system shall publish raw camera frames as ROS2 `sensor_msgs/Image` messages on a documented topic.
- **FR-3 (v1):** The system shall detect a yellow padel ball in each frame and publish its centroid as a `geometry_msgs/PointStamped` on a documented topic.
- **FR-3 (v2):** The system shall track a user-initialised bounding box in each frame using OpenCV CSRT and publish the centroid of the tracked region on the same topic as v1.
- **FR-4:** The controller node shall compute desired pan and tilt servo angles from the published target centroid using a PI control law on pixel error, and publish the commanded angles on a documented topic.
- **FR-5:** The servo node shall command the PCA9685 via I2C to drive the pan and tilt servos to the commanded angles.
- **FR-6:** The system shall provide a configurable HSV threshold range loaded from a YAML config file (v1) and configurable PI gains loaded from a YAML config file (both versions).
- **FR-7:** The system shall be launched from a single `ros2 launch` command that brings up all four nodes with parameters loaded from config files.
- **FR-8:** The system shall handle target loss gracefully: when no target is detected for >0.5 s, servos shall hold their current position (no oscillation, no drift).
- **FR-9:** The system shall recover automatically when a previously lost target re-enters the field of view.
- **FR-10:** The system shall enforce servo angle limits in software, preventing the rig from commanding angles outside its physical safe range.
- **FR-11 (v2 only):** The system shall provide a means for the user to initialise the tracker by selecting a bounding box in the first frame (interactive ROI selection acceptable).

## 6. Non-Functional Requirements

Numbers below are targets. Validation tests in §11 measure against them.

- **NFR-1 — End-to-end latency:** Median latency from physical event to corresponding servo command shall be ≤100 ms; 95th percentile ≤150 ms.
- **NFR-2 — Tracker frame rate:** The detector/tracker shall process frames at ≥15 fps sustained over 60 s of continuous operation.
- **NFR-3 — Control loop rate:** The controller shall run at ≥30 Hz.
- **NFR-4 — Tracking accuracy under slow motion:** With the target moving laterally at ~30°/s, centroid shall stay within ±50 px of frame centre on ≥80% of frames over a 15 s window (640×480 frame, so ±50 px ≈ ±7° of FOV).
- **NFR-5 — Tracking accuracy under medium motion:** With the target moving at ~60°/s, centroid shall stay within ±100 px of frame centre on ≥70% of frames over a 10 s window.
- **NFR-6 — Smoothness:** Under steady-state tracking, servo command derivative shall not exhibit high-frequency oscillation visible in a time-series plot; no audible servo chatter.
- **NFR-7 — Reliability:** The system shall run for ≥5 minutes continuously without crashing, memory growth, or performance degradation.
- **NFR-8 — Reproducibility:** A second person following the README shall be able to clone the repo, install dependencies, and bring up the system within 60 minutes assuming hardware is already assembled.

## 7. Constraints

**Hardware constraints (fixed):**

- Compute: Raspberry Pi 5, 4 GB or 8 GB.
- Camera: Raspberry Pi Camera Module 3 (standard or wide).
- Actuators: 2× MG90S metal-gear servos.
- PWM driver: Adafruit PCA9685 16-channel I2C board.
- Power: Separate 5 V / ≥3 A supply for servos. The Pi's 5 V rail must not power the servos directly.
- Mount: 3D-printed or pre-assembled pan-tilt bracket sized for MG90S.

**Software constraints (fixed):**

- Pi OS: Ubuntu Server 24.04 LTS (64-bit) for ROS2 Jazzy compatibility.
- ROS2 distribution: Jazzy Jalisco (current LTS).
- Language: Python 3 (no C++ required for this project).
- CV: OpenCV 4.x via `python3-opencv`.
- Camera library: `picamera2`.
- Servo library: `adafruit-circuitpython-pca9685` and `adafruit-circuitpython-servokit`.
- Version control: Git, hosted on GitHub, public repository.

**Development environment constraints:**

- Primary dev machine: macOS.
- Dev workflow: SSH + VS Code Remote-SSH to the Pi. No native ROS2 on macOS.
- Code lives on the Pi (or syncs there); all builds and runs happen on the Pi.

**Time constraints:**

- v1 acceptance target: 1 June 2026.
- v2 acceptance target: 7 June 2026.
- Hard project termination: 7 June 2026. If v2 is not complete, it is abandoned, not extended.

**Budget constraint:**

- Hardware budget ceiling: £200 (UK). See BOM in §9.

## 8. System Architecture

Four ROS2 nodes, each with a single responsibility, communicating via topics:

```
┌──────────────┐    /camera/image_raw    ┌──────────────┐
│ camera_node  │ ──────────────────────▶ │ tracker_node │
└──────────────┘                         └──────┬───────┘
                                                │ /target/point
                                                ▼
                                         ┌──────────────┐
                                         │controller_   │
                                         │    node      │
                                         └──────┬───────┘
                                                │ /servo/angles
                                                ▼
                                         ┌──────────────┐
                                         │  servo_node  │
                                         └──────┬───────┘
                                                │ I2C
                                                ▼
                                            PCA9685
                                                │
                                            ┌───┴───┐
                                          pan   tilt
                                         servo  servo
```

**Topic contracts:**

| Topic | Type | Publisher | Rate |
|---|---|---|---|
| `/camera/image_raw` | `sensor_msgs/Image` | `camera_node` | ≥20 Hz |
| `/target/point` | `geometry_msgs/PointStamped` | `tracker_node` | tied to camera rate |
| `/target/found` | `std_msgs/Bool` | `tracker_node` | tied to camera rate |
| `/servo/angles` | `std_msgs/Float32MultiArray` (pan, tilt) | `controller_node` | ≥30 Hz |

**Control law:**

PI control on pixel error in image space, mapped to angular error via FOV. Convention: image origin top-left, x right, y down. Target centroid `(u, v)`, frame centre `(u₀, v₀)`. Pixel errors `e_pan_px = u₀ − u`, `e_tilt_px = v − v₀` (signs chosen so positive error means servo should increase its angle). Map to angular error using camera FOV (Pi Camera Module 3 standard: ~66° horizontal, ~41° vertical):

```
e_angle = e_pixel × FOV / frame_dim
```

This makes Kp dimensionally honest (degrees-per-degree-of-error) and means the same gains transfer if the resolution changes.

Pan and tilt loops independent. Each loop:

```
angle_cmd = angle_current + Kp · e_angle + Ki · ∫e_angle dt
```

with integral windup clamped to ±servo angle limits. Kd is optionally available but defaults to 0 for v1. Gains live in a YAML config and are tuned empirically.

**Coordinate frames and conventions** are documented in `docs/calibration.md`. Servo angle zero is "looking straight ahead", positive pan = camera turns left (toward positive x in world), positive tilt = camera looks up.

## 9. Bill of Materials (UK)

Estimated prices in GBP, May 2026. Verify at order time.

| Item | Qty | Est. price | Notes / Supplier |
|---|---|---|---|
| Raspberry Pi 5, 4 GB | 1 | £55–65 | Pi Hut / Pimoroni |
| Active cooler for Pi 5 | 1 | £5–8 | Recommended — Pi 5 throttles under sustained CV load |
| MicroSD card, 32 GB, A2 | 1 | £8–12 | SanDisk Extreme or Samsung Evo Plus |
| Pi 5 official 27 W USB-C PSU | 1 | £12–15 | Pi-specific; do not substitute generic |
| Pi Camera Module 3 (standard) | 1 | £25–30 | Pi Hut |
| Camera ribbon cable (Pi 5 uses smaller connector) | 1 | £3–5 | Pi 5 uses different cable than Pi 4 — verify |
| MG90S metal-gear servo | 2 | £4–6 each | Amazon, generic |
| Pan-tilt servo bracket kit | 1 | £8–12 | Amazon — search "MG90S pan tilt bracket" |
| Adafruit PCA9685 (or clone) | 1 | £10–15 | Pimoroni for genuine Adafruit, Amazon for clones |
| 5 V / 4 A DC power supply with barrel jack | 1 | £8–12 | For servos only, never share with Pi |
| Barrel jack to terminal block adapter | 1 | £2–4 | Amazon |
| Breadboard, jumper wires, basic kit | 1 | £10–15 | Amazon |
| **Total estimate** | | **£155–200** | Within budget ceiling |

Order 20–21 May to give 2–3 days delivery margin.

## 10. Development Workflow

This is genuinely new territory and worth specifying explicitly.

**Pi setup (one-time, ~half a day):**

1. Flash Ubuntu Server 24.04 LTS 64-bit to SD card using Raspberry Pi Imager. Pre-configure: hostname (`pantilt`), username, SSH enabled, Wi-Fi credentials, locale.
2. Boot, SSH from Mac: `ssh user@pantilt.local`.
3. System update, install essentials: `git`, `python3-pip`, `i2c-tools`, `vim`.
4. Enable I2C, enable camera in `/boot/firmware/config.txt`.
5. Install `picamera2`: `sudo apt install python3-picamera2`.
6. Install ROS2 Jazzy following official Ubuntu install guide.
7. Source ROS2 in `.bashrc`: `source /opt/ros/jazzy/setup.bash`.
8. Verify: `ros2 run demo_nodes_py talker` in one terminal, `ros2 run demo_nodes_py listener` in another.

**Mac setup:**

1. VS Code with Remote-SSH extension.
2. SSH config entry for the Pi in `~/.ssh/config` for one-click connection.
3. SSH key (not password) auth to the Pi.
4. Git config aligned between Mac and Pi (same name, email).

**Daily workflow:**

- VS Code Remote-SSH into Pi. Edit code on the Pi's filesystem. Terminal in VS Code runs on the Pi.
- Builds, tests, and runs all happen on the Pi.
- Camera output viewed via either `rqt_image_view` over X-forwarding (slow but real-time) or by saving frames to disk and `scp`ing them down (fast, for debugging single-frame issues).
- Recording demo videos: use the Pi camera directly via a small script that captures and saves, or point a phone at the rig.

This workflow doc gets its own `docs/workflow.md` in the repo, so future-you (and anyone replicating) doesn't have to rediscover it.

## 11. Validation Test Plan

Each test below has a fixed setup, a clear pass criterion, and a recorded artefact (video clip, plot, or numeric measurement). All results go into `docs/results.md` in the table format shown in §12.

### v1 validation tests

| ID | Name | Setup | Pass criterion |
|---|---|---|---|
| VT-1 | Static hold | Ball at frame centre, 1 m, stationary | Servo commands stable; pan/tilt command range <2° over 30 s |
| VT-2 | Off-centre acquisition | Ball stationary at ~25° pan offset, 1 m | System pans to centre within 2 s; centroid within ±25 px of centre at steady state |
| VT-3 | Slow lateral tracking | Ball moves hand-walked left-to-right at ~30°/s, 1 m | Centroid within ±50 px of frame centre for ≥80% of frames over 15 s |
| VT-4 | Medium tracking | Ball moves at ~60°/s | Centroid within ±100 px for ≥70% of frames over 10 s |
| VT-5 | Occlusion | Ball tracked, hand occludes for ~1 s, then released | Tracking resumes without manual intervention; servos hold during occlusion |
| VT-6 | Re-entry | Ball leaves FOV from one edge, re-enters from same edge | System reacquires within 1 s of ball being back in FOV |
| VT-7 | Latency | Ball jerked sharply; record with phone at 60 fps showing both ball and servo | Median latency ≤100 ms, 95th percentile ≤150 ms, n ≥ 20 events |
| VT-8 | Smoothness | 30 s of steady tracking | Time-series plot of servo commands shows no high-frequency oscillation; no audible chatter |
| VT-9 | Reliability | 5 minutes of mixed motion | No crashes; tracker frame rate ≥15 fps at end of run |

### v2 validation tests

Same VT-1 through VT-9 but with bounding-box-initialised CSRT tracker on three distinct objects: (a) coffee mug (matte, single colour), (b) book (rectangular, high-contrast features), (c) a moving hand or person. Pass criteria are softened by ~50% on accuracy metrics (e.g. VT-3 becomes ±75 px instead of ±50 px) because classical trackers are less precise than HSV on a known colour.

A v2-specific test:

| ID | Name | Setup | Pass criterion |
|---|---|---|---|
| VT-10 | Architecture swap | Run v1, kill `tracker_node`, launch v2 `tracker_node` with no other changes | System resumes tracking with new tracker; no changes to camera, controller, or servo nodes |

VT-10 is the most important v2 test. It's the proof the architecture worked.

## 12. Deliverables

### 12.1 GitHub repository

Repository name: `pan-tilt-tracker` (or chosen). Public. MIT license (recommended).

Required structure:

```
pan-tilt-tracker/
├── README.md
├── LICENSE
├── .gitignore
├── pyproject.toml          # or requirements.txt
├── docs/
│   ├── architecture.md     # system design narrative
│   ├── architecture.png    # the diagram
│   ├── hardware.md         # BOM, wiring diagram, photos
│   ├── calibration.md      # camera FOV, servo limits, HSV thresholds, conventions
│   ├── workflow.md         # dev setup, daily workflow
│   ├── results.md          # validation test results table
│   └── report.md           # the reflective engineering report
├── ros2_ws/
│   └── src/pan_tilt_tracker/
│       ├── package.xml
│       ├── setup.py
│       ├── pan_tilt_tracker/
│       │   ├── __init__.py
│       │   ├── camera_node.py
│       │   ├── tracker_hsv_node.py
│       │   ├── tracker_csrt_node.py
│       │   ├── controller_node.py
│       │   └── servo_node.py
│       ├── launch/
│       │   ├── v1_hsv.launch.py
│       │   └── v2_csrt.launch.py
│       ├── config/
│       │   ├── tracker_hsv.yaml
│       │   ├── tracker_csrt.yaml
│       │   ├── controller.yaml
│       │   └── camera.yaml
│       └── test/
│           ├── test_controller.py     # unit tests for control math
│           └── test_geometry.py       # unit tests for pixel→angle mapping
├── scripts/
│   ├── calibrate_hsv.py    # interactive HSV tuner
│   ├── test_servos.py      # standalone servo sweep, no ROS2
│   ├── test_camera.py      # standalone camera grab, no ROS2
│   ├── measure_latency.py  # automated latency measurement
│   └── plot_results.py     # generate results plots
└── media/
    ├── demo_v1.mp4
    ├── demo_v2.mp4         # if v2 lands
    ├── architecture.png
    └── hardware/
        ├── rig_front.jpg
        ├── rig_side.jpg
        └── wiring.jpg
```

Commit discipline: meaningful commit messages, small focused commits where reasonable, no commits of `WIP` or `stuff` to `main`. Branches for v2 work, merged via PR (to yourself, for the practice).

### 12.2 Top-level README

Required sections, in order:

1. **Title + one-line tagline.** "Pan-tilt camera that visually tracks an object in real-time. ROS2, OpenCV, Pi 5."
2. **Demo.** Embedded GIF or thumbnail linking to video. Must be above the fold.
3. **What this is and why it exists.** 2–3 paragraphs. Honest about it being a learning project.
4. **Key results.** Headline numbers from validation: tracking accuracy, latency, frame rate.
5. **Hardware.** Photo + link to `docs/hardware.md` for full BOM.
6. **Architecture overview.** Diagram + 1-paragraph description, link to `docs/architecture.md` for depth.
7. **Quickstart.** Cloning, dependencies, flashing, launching. Tested step-by-step. NFR-8 lives here.
8. **Repo layout.** Brief tree with one-line descriptions.
9. **What I learned / what I'd do differently.** 3–5 bullets, candid. Links to `docs/report.md`.
10. **License + acknowledgements.** Credit Adrian Rosebrock's tutorial, Brian Douglas's PID videos, etc.

Tone: technical, plain, no hype, no emojis, no "🚀 awesome project".

### 12.3 Architecture document

`docs/architecture.md`. Contains:

- The diagram (also embedded in README).
- A node-by-node walkthrough: each node's responsibility, input topics, output topics, parameters, key implementation notes.
- Topic contract table (rate, type, semantics).
- Coordinate frame conventions: image axes, servo angle conventions, sign choices.
- Control law derivation: pixel error → angular error using FOV, then PI in angle space.
- Latency budget: where each millisecond goes, measured vs predicted.

### 12.4 Hardware document

`docs/hardware.md`. Contains:

- BOM with supplier links and prices paid (not just estimates).
- Wiring diagram — hand-drawn and photographed, or made in Fritzing.
- Photos of the assembled rig from front, side, and above.
- Photo of the wiring with labels.
- Notes on what went wrong during assembly and how it was fixed (this is portfolio gold — show you debug hardware too).

### 12.5 Calibration document

`docs/calibration.md`. Contains:

- Camera FOV (measured if possible, not just spec sheet).
- Servo angle limits per axis (mechanical and software).
- HSV threshold ranges chosen for the ball, with photos of the calibration scene.
- Frame size used and rationale.
- Anything else a future-you would need to re-tune.

### 12.6 Workflow document

`docs/workflow.md`. The actual dev setup, written so you can re-bootstrap on a new Pi in an afternoon. Mac SSH config, VS Code Remote-SSH setup, ROS2 source-in-bashrc, troubleshooting notes.

### 12.7 Results document

`docs/results.md`. One table per version (v1, v2), one row per validation test, columns: Test ID, Name, Pass/Fail, Measured value, Target value, Notes/observations, Video clip filename.

Plus plots where appropriate:

- Latency histogram (from VT-7).
- Servo command time series for VT-8.
- Tracking error time series for VT-3 and VT-4.

Plots generated by `scripts/plot_results.py` from raw data CSVs also committed to the repo.

### 12.8 Reflective engineering report

`docs/report.md`. 1500–2500 words. Structure:

1. **Abstract** (3–4 sentences): what was built, what was achieved, headline numbers.
2. **Problem framing** (~200 words): what the project really is, why this scope.
3. **System design** (~400 words): architecture rationale, why four nodes and not three or five, why PI not PID, why HSV for v1 and CSRT for v2.
4. **Implementation notes** (~500 words): the three or four hardest moments. What was confusing, what was tried that didn't work, what fixed it. This is where the writing earns its keep.
5. **Validation results** (~200 words): summary of headline numbers, link to `results.md` for full table.
6. **Reflection** (~300 words): what was learned, what would be done differently, what limitations remain.
7. **Future work** (~200 words): concrete v3 ideas, what each would need.

Tone: first-person, technical, candid. Aim for the register of a good lab notebook or engineering blog post, not a marketing piece. Anything that sounds like "leveraged cutting-edge technologies" gets cut.

### 12.9 Demo video

60–90 seconds. Format: MP4, 1080p or 720p, landscape.

Required content:

- 0:00–0:05: Title card with project name.
- 0:05–0:15: Static shot of the rig, no narration.
- 0:15–0:45: v1 demo. Shows VT-2 (off-centre acquisition), VT-3 (slow tracking), VT-5 (occlusion recovery). Camera shows both the rig and a screen showing the tracker overlay if possible (split-screen optional).
- 0:45–1:15: v2 demo (if v2 landed). Shows bounding-box initialisation, then tracking of two or three different objects.
- 1:15–1:30: End card with GitHub URL.

No music, no voiceover required for v1. Captions for context optional.

Hosted: YouTube unlisted, linked from README. Also committed to `media/` in repo (compressed; aim for <50 MB).

### 12.10 Tests

At minimum:

- `test_controller.py`: unit tests for the PI control math. Given known errors and gains, assert correct output angles. No hardware needed.
- `test_geometry.py`: unit tests for pixel-to-angle mapping using FOV.

These are non-negotiable as portfolio signal — show you write tests for the parts that can be tested. Hardware integration is tested via the validation tests in §11, not unit tests.

Run via `pytest` in the package directory. Mention in README quickstart.

## 13. Risks and Mitigations

Ordered by likelihood × impact.

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | Ubuntu 24.04 + Pi 5 + camera driver friction | High | High | Allocate full day for OS+camera bring-up alone. Fallback: Raspberry Pi OS Bookworm with manual ROS2 Jazzy install via Robostack. Test camera with `libcamera-hello` before anything else. |
| 2 | Servo brown-out from shared Pi power | Medium | High (Pi crashes mid-debug) | Separate 5 V supply from day one. Never plug servos into Pi 5 V pin even "just to test". |
| 3 | HSV thresholds break under different lighting | High | Medium | Build `calibrate_hsv.py` interactive tuner early. Test under the actual lighting where demo will be recorded. Document chosen thresholds in `docs/calibration.md`. |
| 4 | PID tuning rabbit hole | Medium | Medium | Start with P-only, then add I if steady-state error is visible. Skip D. Time-box tuning to one afternoon. |
| 5 | SSH/VS Code Remote-SSH workflow friction | Medium | Medium | Invest in setup on day one. Don't try to develop via raw `vim` over SSH. |
| 6 | Pi 5 camera connector incompatibility | Medium | Low | Pi 5 uses smaller camera connector than Pi 4. Order the correct cable. Verify before assembly. |
| 7 | I2C address conflict / wiring error on PCA9685 | Low | Medium | Use `i2cdetect -y 1` to verify before writing code. PCA9685 default is 0x40. |
| 8 | Padel work crowds out pan-tilt time late May | High | Medium | Front-load 20–28 May aggressively. Accept light touch 29 May – 3 June. Use 5–7 June post-presentation. |
| 9 | Latency too high to meet NFR-1 | Medium | Low | If latency is the bottleneck, reduce resolution to 320×240 — usually halves the budget. Document the trade-off. |
| 10 | v2 doesn't land by 7 June | Medium | Low (planned for) | Terminate, write up v1 only, note v2 as future work. No emotional cost. |

## 14. Fallback Ladder

If timeline slips, drop in this order:

1. **First cut: v2.** v1 alone is a complete portfolio project. v2 is the cherry, not the cake.
2. **Second cut: VT-4 and VT-9.** Medium-speed tracking and long-duration reliability are nice-to-haves over slow tracking and basic correctness.
3. **Third cut: unit tests.** Reluctantly. Better to have a working demo and no unit tests than no demo.
4. **Fourth cut: latency measurement (VT-7).** Hardest test to set up, can be reported qualitatively.

Never cut: VT-1, VT-2, VT-3, VT-5, demo video, README, architecture doc.

## 15. Timeline

Reference dates only — slip is permitted within the 7 June ceiling.

| Window | Pan-tilt activity | Other goals dominant |
|---|---|---|
| 20–21 May (Wed–Thu) | Order hardware. Begin Pi OS install. Begin ROS2 install. | Padel kickoff, theory morning |
| 22–24 May (Fri–Sun) | Hardware arrives. Stage 1 (ROS2 hello world) complete. Begin Stage 2 (hardware bring-up). | Padel demo running, theory |
| 25–28 May (Mon–Thu) | Stage 2 complete. Stage 3 (camera + tracker nodes). Stage 4 (controller + servo nodes). | Padel test set, theory |
| 29–31 May (Fri–Sun) | Stage 5 (integration + tuning). v1 acceptance pass — run validation tests VT-1 through VT-9. | Padel slides drafted |
| 1 June (Mon) | **v1 deliverables: README, docs, results, demo video. v1 shipped.** | Padel rehearsal |
| 2–3 June (Tue–Wed) | Light touch only. Maybe start v2 `tracker_csrt_node` skeleton. | Padel rehearsals |
| 4 June (Thu) | Nothing. | **Padel presentation** |
| 5–6 June (Fri–Sat) | v2 push: CSRT tracker node, launch file, validation tests, demo clips. | Recovery + light theory |
| 7 June (Sun) | **v2 deliverables shipped or v2 cleanly abandoned with note in report.** Project closed. | Studentship prep ramping |

## 16. Definition of Done

**v1 is done when, and only when, all of the following are true:**

- [ ] All v1 functional requirements (FR-1 through FR-10, FR-3 v1 form) verified.
- [ ] All v1 non-functional requirements (NFR-1 through NFR-8) measured and recorded.
- [ ] Validation tests VT-1 through VT-9 executed; pass/fail recorded; results in `docs/results.md`.
- [ ] Demo video recorded, edited, uploaded, embedded in README.
- [ ] README complete with all 10 required sections.
- [ ] All required `docs/` files present and substantive.
- [ ] Unit tests pass.
- [ ] Repo public on GitHub.
- [ ] Reflective report draft complete (can be polished after v2).
- [ ] Someone other than you can follow the quickstart and bring up the system (or, failing that, you've reviewed it line-by-line as if you were them).

**v2 is done when:**

- [ ] FR-3 (v2 form) and FR-11 verified.
- [ ] Validation tests VT-1 through VT-9 re-run with CSRT tracker on three objects; results recorded.
- [ ] VT-10 (architecture swap) executed and recorded.
- [ ] v2 demo segment added to demo video.
- [ ] README updated to reflect v2.
- [ ] Report's "system design" section updated to reflect the swap demonstration.

**v2 is acceptably abandoned when:**

- [ ] A short section in `docs/report.md` explains what was attempted, where it stopped, and what would be needed to complete it.

## 17. Open Questions

Things to resolve, ideally before ordering hardware:

1. **Pi camera version:** standard (66° FOV) or wide (102° FOV)? Recommendation: standard.
2. **License:** MIT (default) or Apache-2.0? Recommendation: MIT.
3. **GitHub repo name + visibility:** confirm `pan-tilt-tracker`, public.
4. **Workspace location on Pi:** `~/ros2_ws/` conventional. Confirm.
5. **Pan-tilt mount:** 3D-printed or pre-assembled kit? Recommendation: pre-assembled kit unless 3D printer already accessible.
