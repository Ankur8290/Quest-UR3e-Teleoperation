# Quest-UR3e Teleoperation Framework

Mixed Reality Teleoperation of a Universal Robots UR3e
using Meta Quest 3/3s.

Video Link https://youtu.be/hq68BAeUli4
# Quest–UR3e Mixed Reality Teleoperation Framework

A research-oriented mixed-reality teleoperation framework for controlling a **Universal Robots UR3e** using a **Meta Quest 3** controller.

The system uses the Meta Quest 3 as a spatial interaction interface. The operator sees the real environment through mixed-reality passthrough, while the tracked controller provides 6-DoF position and orientation information. Controller data is transmitted over Wi-Fi to a robot-side computer, where it is calibrated, transformed into the robot coordinate frame, safety-checked, and used as the basis for Cartesian robot control.

The framework is designed as a modular research platform that can later be extended with **FMG-based gripper control, tactile sensing, haptic feedback, and other human–robot interaction modalities**.

---

## Overview

Traditional robot teleoperation interfaces often rely on joysticks, teach pendants, keyboards, or specialized motion-capture systems. These interfaces can make spatial robot manipulation unintuitive because the operator must translate hand movements into robot motion through an intermediate control interface.

This project explores a more natural interaction paradigm:

> **Move your hand/controller in space → the robot follows the corresponding motion.**

A Meta Quest 3 provides:

* 6-DoF controller tracking
* Mixed-reality passthrough
* Low-latency spatial tracking
* A familiar wireless interface
* Programmable input buttons and triggers

The UR3e provides the physical robotic platform.

The two systems communicate through a robot-side PC.

---

## System Architecture

```text
                         META QUEST 3
                              │
                              │
                    Controller Tracking
                              │
                              ▼
                         Unity / XR
                              │
                    Position + Rotation
                              │
                              ▼
                       UDP over Wi-Fi
                              │
                              ▼
                    ┌──────────────────┐
                    │   Robot PC       │
                    │                  │
                    │ UDP Receiver     │
                    │       │          │
                    │       ▼          │
                    │ Calibration      │
                    │       │          │
                    │       ▼          │
                    │ Coordinate       │
                    │ Mapping          │
                    │       │          │
                    │       ▼          │
                    │ Safety Layer     │
                    │       │          │
                    │       ▼          │
                    │ Robot Controller │
                    └────────┬─────────┘
                             │
                           RTDE
                             │
                             ▼
                           UR3e
```

The architecture intentionally separates the **operator interface** from the **robot-control computer**.

The Quest does not communicate directly with the robot.

Instead:

```text
Quest
  │
  │ Wi-Fi / UDP
  ▼
Robot PC
  │
  │ Ethernet / RTDE
  ▼
UR3e
```

This allows the robot-side computer to perform calibration, coordinate transformation, filtering, safety checks, and robot control independently of the Unity application.

---

# Key Features

## 1. Mixed Reality Passthrough

The Quest 3 runs the teleoperation interface in mixed reality.

The operator can see the physical environment while simultaneously viewing the telemetry dashboard.

The dashboard can display:

* Controller position
* Controller orientation
* Trigger state
* A/B button state
* Teleoperation state
* Robot TCP position
* Connection state
* FPS
* Latest system event

This makes the system usable as a spatial robot-control interface rather than a conventional VR-only application.

---

## 2. 6-DoF Controller Tracking

The Unity application obtains the right-hand controller pose.

The controller provides:

```text
Position

X
Y
Z
```

and:

```text
Orientation

Quaternion
qx
qy
qz
qw
```

The position is expressed in Unity's coordinate system.

The orientation is transmitted as a quaternion to avoid unnecessary Euler-angle conversion during communication.

---

## 3. Trigger-Based Motion Enable

The controller trigger is used as an additional motion-enable condition.

The system distinguishes between:

```text
Teleoperation disabled
```

and:

```text
Teleoperation enabled
+
Trigger held
```

This provides a clutch-like interaction mechanism.

The operator can therefore release the trigger without changing the calibrated reference position.

---

## 4. Controller Buttons

The controller buttons provide control commands rather than gripper control.

Current assignments include:

### A Button

Calibration request.

```text
A pressed
    ↓
Calibration Requested
```

### B Button

Teleoperation toggle.

```text
B pressed
    ↓
Teleoperation Enabled / Paused
```

The gripper is intentionally not controlled through the Quest controller because a separate **FMG-based interface** is planned for gripper control.

---

# 5. Typed UDP Communication

The Quest communicates with the robot PC using UDP.

Three packet types are currently defined:

```text
pose
heartbeat
calibration
```

This is preferable to transmitting unstructured data because the robot-side software can explicitly identify the purpose of each packet.

---

## Pose Packet

A pose packet contains:

```json
{
    "version": 1,
    "type": "pose",
    "timestamp": 48.59,
    "px": -0.265,
    "py": 0.720,
    "pz": -0.049,
    "qx": -0.581,
    "qy": -0.228,
    "qz": -0.431,
    "qw": 0.652,
    "trigger": 1.0,
    "teleoperationEnabled": true
}
```

The pose packet is transmitted continuously while teleoperation is active and the trigger is held.

---

## Heartbeat Packet

Heartbeat packets allow the robot-side system to determine whether the Quest application is still communicating.

Conceptually:

```text
Quest
  │
  ├── heartbeat
  ├── heartbeat
  ├── heartbeat
  └── heartbeat
```

The robot-side application can use heartbeat/packet timing as part of its communication watchdog.

---

## Calibration Packet

Pressing A generates:

```json
{
    "version": 1,
    "type": "calibration"
}
```

The robot-side application uses the latest controller pose and current robot TCP pose to establish a common reference.

---

# 6. Relative Pose Calibration

Absolute controller coordinates should not directly determine robot coordinates.

Instead, the system uses relative motion.

At calibration:

```text
Controller Home = C₀

Robot Home = R₀
```

During operation:

```text
Controller Current = C
```

Controller displacement:

```text
ΔC = C - C₀
```

The displacement is scaled and transformed into the robot coordinate frame.

Conceptually:

```text
Robot Target

= Robot Home
+ CoordinateTransform(ΔController)
```

This prevents the robot from jumping when teleoperation begins.

The operator can therefore place the controller in a comfortable reference position and use movements relative to that position.

---

# 7. Coordinate Transformation

Unity and UR3e use different coordinate-frame conventions.

The current prototype uses a measured mapping between the controller displacement and robot displacement.

For the current physical setup:

```text
Unity +X
   ↓
Robot -Y

Unity +Y
   ↓
Robot +Z

Unity +Z
   ↓
Robot +X
```

Therefore:

```text
Robot ΔX = Unity ΔZ

Robot ΔY = -Unity ΔX

Robot ΔZ = Unity ΔY
```

The transformation is applied after calibration.

The important design principle is that the transformation is applied to **relative motion**, rather than treating the raw Quest coordinates as absolute robot coordinates.

---

# 8. Motion Scaling

Controller movement does not necessarily need to correspond one-to-one with robot movement.

A scaling factor is therefore applied:

```text
Scaled displacement

= Controller displacement × Motion Scale
```

For example:

```text
Controller movement = 10 cm

Motion Scale = 0.2

Robot movement = 2 cm
```

This allows large hand movements to produce smaller, more controllable robot movements.

---

# 9. Robot Communication

The robot-side computer communicates with the UR3e through Ethernet using RTDE.

The current hardware network is:

```text
Robot PC
192.168.31.1

      │
      │ Ethernet
      │
      ▼

UR3e
169.254.88.226
```

The Quest communicates separately with the robot PC over Wi-Fi.

The robot PC's current Wi-Fi address is:

```text
192.168.0.137
```

Therefore:

```text
Quest
   │
   │ Wi-Fi
   ▼
192.168.0.137
Robot PC
   │
   │ Ethernet
   ▼
169.254.88.226
UR3e
```

The IP addresses are configuration values and should not be hard-coded throughout the application.

---

# 10. Robot-Side Software Architecture

The Python application is organized into independent modules.

```text
Python/
│
├── main.py
├── config.py
├── network.py
├── calibration.py
├── coordinate_mapper.py
├── robot_controller.py
└── safety.py
```

## `main.py`

Coordinates the complete application.

Conceptually:

```text
Network
   ↓
Calibration
   ↓
Coordinate Mapping
   ↓
Safety
   ↓
Robot Controller
```

---

## `config.py`

Contains configurable parameters such as:

* Robot IP
* UDP port
* Control frequency
* Motion scale
* Robot-control parameters
* Workspace limits

Centralizing configuration makes the system easier to deploy on another computer.

---

## `network.py`

Responsible for:

* UDP socket initialization
* Packet reception
* JSON decoding
* Packet validation
* Maintaining the latest controller state

The network layer does not directly control the robot.

---

## `calibration.py`

Responsible for maintaining:

```text
Controller Home
Robot Home
Calibration State
```

The calibration module converts the initial controller/robot relationship into a reusable reference.

---

## `coordinate_mapper.py`

Contains the mathematical transformation between:

```text
Quest Controller Frame
```

and:

```text
UR3e Robot Frame
```

Keeping this logic separate makes it possible to test the transformation independently from the robot.

---

## `robot_controller.py`

Contains the robot communication layer.

Its responsibility is to provide a clean interface between the application and the UR3e's RTDE interface.

---

## `safety.py`

Provides the safety-validation layer between the calculated target and robot-control interface.

Potential checks include:

* Workspace boundaries
* Maximum Cartesian displacement
* Maximum velocity
* Packet timeout
* Teleoperation state
* Trigger state
* Calibration state
* Invalid packet detection
* Robot connection state

The safety layer should prevent invalid or stale controller data from becoming robot commands.

---

# 11. Control State Machine

The system can be viewed as a simple state machine.

```text
                 ┌──────────────┐
                 │    START     │
                 └──────┬───────┘
                        │
                        ▼
                ┌───────────────┐
                │ WAIT FOR DATA │
                └───────┬───────┘
                        │
                        ▼
                ┌───────────────┐
                │   CONNECTED   │
                └───────┬───────┘
                        │
                  A / Calibration
                        │
                        ▼
                ┌───────────────┐
                │   CALIBRATED  │
                └───────┬───────┘
                        │
               B / Enable + Trigger
                        │
                        ▼
                ┌───────────────┐
                │  TELEOP ACTIVE│
                └───────┬───────┘
                        │
                 Safety failure
                        │
                        ▼
                ┌───────────────┐
                │     STOP      │
                └───────────────┘
```

This makes the operating logic explicit rather than relying on individual scripts to determine whether motion is allowed.

---

# 12. Complete Data Flow

The complete system operates as follows:

### Step 1 — Quest initialization

Unity starts the XR environment and initializes controller tracking.

### Step 2 — Controller tracking

The right-hand controller pose is continuously obtained.

### Step 3 — Dashboard update

The operator can see:

```text
Position
Rotation
Trigger
A/B state
Teleoperation state
FPS
Robot state
```

### Step 4 — UDP transmission

Unity transmits the controller state to the robot PC.

### Step 5 — Packet reception

Python receives the UDP packet and updates the latest controller state.

### Step 6 — Calibration

The operator presses A.

The robot-side application stores:

```text
Controller Home
Robot TCP Home
```

### Step 7 — Relative motion

The controller displacement is calculated:

```text
ΔC = C - C₀
```

### Step 8 — Coordinate transformation

The displacement is converted into the UR3e coordinate frame.

### Step 9 — Motion scaling

The transformed displacement is multiplied by the configured motion scale.

### Step 10 — Safety validation

The resulting target is checked against configured safety constraints.

### Step 11 — Robot control

The validated target is passed to the robot-control layer.

---

# Hardware Setup

## Required Hardware

* Meta Quest 3
* Universal Robots UR3e
* Robot-connected Windows PC
* Wi-Fi router/access point
* Ethernet cable
* UR3e controller

Optional future hardware:

* FMG band
* Custom gripper
* Tactile sensor
* Haptic feedback device

---

# Network Setup

The recommended network arrangement is:

```text
                  Wi-Fi Router
                 192.168.0.1
                    /     \
                   /       \
                  /         \
             Quest 3      Robot PC
                         192.168.0.137
                              │
                              │ Ethernet
                              │
                         UR3e Controller
                         169.254.88.226
```

The Quest and robot PC must be able to communicate over the same Wi-Fi network.

The robot PC must separately be able to communicate with the UR3e over its robot Ethernet connection.

---

# Unity Setup

The Unity application was developed using:

```text
Unity 2022.3.62f3 LTS
```

The project uses the Meta XR ecosystem for Quest deployment and mixed-reality functionality.

The Quest application provides:

* Passthrough
* Controller tracking
* Controller input
* Dashboard
* UDP communication

---

# Python Environment

Create a Python virtual environment:

```bash
python -m venv .venv
```

Activate it on Windows:

```powershell
.venv\Scripts\activate
```

Install the required dependencies:

```bash
pip install -r requirements.txt
```

The main robot communication dependency is:

```text
ur-rtde
```

---

# Configuration

Before running the system, configure:

```text
Quest → Robot PC IP

Robot PC → UR3e IP

UDP Port

Motion Scale

Control Frequency

Workspace Limits
```

Do not hard-code environment-specific addresses throughout the source code.

---

# Running the System

## 1. Start the robot

Power on the UR3e and verify that the robot controller is operational.

## 2. Connect the Robot PC

Verify communication with the UR3e.

For example:

```bash
ping 169.254.88.226
```

## 3. Start the Python application

Run:

```bash
python main.py
```

## 4. Start the Quest application

Launch the teleoperation APK on the Meta Quest 3.

Verify that the dashboard is updating.

## 5. Verify network communication

The Python application should receive:

```text
heartbeat
pose
calibration
```

packets.

## 6. Calibrate

Hold the controller at the desired reference position and press:

```text
A
```

The system records the controller and robot reference poses.

## 7. Enable teleoperation

Use the configured controller button to enable teleoperation.

The trigger acts as the motion clutch.

---

# Controller Mapping

| Input                     | Function                     |
| ------------------------- | ---------------------------- |
| Right Controller Position | End-effector translation     |
| Right Controller Rotation | End-effector orientation     |
| Trigger                   | Motion clutch                |
| A                         | Calibration                  |
| B                         | Teleoperation enable/disable |
| FMG Band                  | Future gripper control       |

The FMG interface is intentionally kept separate from the Quest input system so that gripper control can evolve independently.

---

# Current Limitations

This is a research prototype rather than a production teleoperation system.

Current limitations include:

* Position mapping is currently calibrated for the experimental setup.
* Coordinate transformation assumes the current physical arrangement.
* Network latency depends on the Wi-Fi environment.
* Controller tracking quality depends on Quest tracking conditions.
* Orientation control requires further validation and tuning.
* No force-feedback channel is currently implemented.
* The gripper is not yet controlled through the Quest.
* Tactile feedback integration is planned.
* The system currently focuses on Cartesian teleoperation.

---

# Future Work

Several extensions are planned.

## FMG-Based Gripper Control

An FMG band can provide hand/forearm muscle information for controlling the robotic gripper.

```text
FMG Band
   ↓
Signal Processing
   ↓
Gripper State
   ↓
UR3e Gripper
```

This separates arm teleoperation from hand/gripper control.

---

## Tactile Sensing

A custom tactile sensor can be integrated into the gripper.

The sensor can provide information about:

* Contact
* Pressure
* Pulse-related signals
* Grasp interaction

The existing signal-processing models can be connected to the teleoperation framework.

---

## Haptic Feedback

A future version can provide haptic information to the operator.

```text
Tactile Sensor
      ↓
Signal Processing
      ↓
Haptic Encoding
      ↓
Operator
```

This would transform the system from visual teleoperation into a multimodal teleoperation platform.

---

## Orientation Teleoperation

The current framework transmits controller orientation as a quaternion.

Future work will refine the mapping between controller orientation and robot tool orientation.

---

## Adaptive Motion Scaling

Motion scaling could be dynamically adjusted based on:

* Workspace location
* Robot velocity
* Task phase
* Operator input
* Object proximity

---

## Multi-Modal Teleoperation

The long-term architecture can combine:

```text
Quest 3
   │
   ├── Arm Pose
   │
   └── Operator Interface
           │
           ▼
        Robot
           ▲
           │
   ┌───────┴────────┐
   │                │
FMG Band       Tactile Sensor
   │                │
Gripper          Feedback
```

---

# Research Motivation

The project investigates whether modern consumer mixed-reality hardware can provide an accessible and intuitive interface for robotic teleoperation without requiring dedicated optical motion-capture infrastructure.

The use of the Meta Quest 3 provides:

* Commodity hardware
* 6-DoF tracking
* Mixed-reality visualization
* Wireless operation
* Programmable interaction

while the modular robot-side architecture allows additional sensing and control modalities to be integrated without redesigning the complete system.

---

# Reproducibility

The project is intended to make the complete software pipeline reproducible.

A researcher should be able to:

1. Clone the repository.
2. Install Unity dependencies.
3. Configure the Quest application.
4. Configure the robot-side Python environment.
5. Configure network addresses.
6. Connect the UR3e.
7. Launch the Quest application.
8. Start the Python application.
9. Calibrate the system.
10. Test the teleoperation pipeline.

Environment-specific parameters should be stored in configuration files rather than embedded throughout the code.

---

# Project Status

| Component                       | Status          |
| ------------------------------- | --------------- |
| Quest 3 integration             | ✅ Working       |
| Mixed Reality passthrough       | ✅ Working       |
| Controller position tracking    | ✅ Working       |
| Controller orientation tracking | ✅ Working       |
| Trigger input                   | ✅ Working       |
| A/B buttons                     | ✅ Working       |
| Dashboard                       | ✅ Working       |
| UDP communication               | ✅ Working       |
| Typed packets                   | ✅ Working       |
| Heartbeat                       | ✅ Working       |
| Calibration                     | ✅ Working       |
| Relative motion mapping         | ✅ Working       |
| Unity → UR coordinate mapping   | ✅ Prototype     |
| UR3e RTDE communication         | ✅ Working       |
| Cartesian target generation     | ✅ Working       |
| Live teleoperation              | 🔬 Experimental |

---

# Contributing

Contributions are welcome.

Potential areas include:

* Improved coordinate calibration
* Alternative robot platforms
* ROS/ROS2 integration
* Network optimization
* Motion filtering
* Haptic feedback
* Tactile sensing
* Gripper interfaces
* Visualization
* Human–robot interaction experiments

Please open an issue before implementing major architectural changes.

---

# Safety Notice

This repository is intended for **research and experimental use**.

It interfaces with a physical industrial robot. Users are responsible for ensuring that the robot is operated in accordance with the manufacturer's safety requirements and the safety procedures of their laboratory.

Before operating the system:

* Verify the robot workspace.
* Verify the robot's safety configuration.
* Verify the coordinate transformation.
* Verify calibration.
* Test with conservative motion parameters.
* Ensure the emergency stop is accessible.
* Keep personnel outside the robot's hazardous workspace.
* Never rely solely on software safety checks for physical safety.

The software should be treated as an experimental research platform and not as a certified industrial safety system.

---

# Citation

If you use this project in academic research, please cite:

```bibtex
@software{quest_ur3e_teleoperation,
  author = {Ankur Shrivastava},
  title = {Quest-UR3e Mixed Reality Teleoperation Framework},
  year = {2026},
  url = {https://github.com/Ankur8290/Quest-UR3e-Teleoperation}
}
```

---

# License

MIT License

---

# Acknowledgements

This project uses and builds upon technologies from:

* Meta Quest / Meta XR
* Unity
* Universal Robots
* RTDE / ur-rtde
* OpenXR

Please refer to the respective projects and documentation for their licensing and attribution requirements.

---

# Contact

For questions, research collaboration, or technical discussion, please open a GitHub issue or contact the project author.

**Project:** Quest–UR3e Mixed Reality Teleoperation
**Platform:** Meta Quest 3 + Universal Robots UR3e
**Purpose:** Research in mixed-reality human–robot teleoperation
