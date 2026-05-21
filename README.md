# fp_bridge

Custom ROS 1 ↔ ROS 2 bridge based on [`ros2/ros1_bridge`](https://github.com/ros2/ros1_bridge), extended to support [`fp_core_msgs`](https://github.com/fp-robotics/fp_core_msgs) custom message types.

The workspace layout is:

```
fp_bridge/
├── ros1_ws/        # ROS 1 workspace (fp_core_msgs)
├── ros2_ws/        # ROS 2 workspace (ros2_fp_core_msgs)
└── bridge_ws/      # bridge workspace (ros1_bridge with patches)
```

---

## Installation

### Prerequisites
Blah Blah Blah wsl, ubuntu, ros2 humble, ros1 from ros-core-dev
https://docs.ros.org/en/humble/How-To-Guides/Using-ros1_bridge-Jammy-upstream.html
https://docs.ros.org/en/humble/Installation/Alternatives/Ubuntu-Development-Setup.html
-> force ros packages
-> remove ros2 catkin to download ros1


clone this repro

## Build

Build ros2 if not done before
Build the three workspaces in order: ROS 1 messages → ROS 2 messages → bridge with the right sources.

### 1. Build the ROS 1 workspace

```bash
source /opt/ros/noetic/setup.bash
cd ros1_ws
catkin_make
cd ..
```

### 2. Build the ROS 2 workspace

```bash
source ~/ros2_humble/setup.bash
cd ros2_ws
colcon build
cd ..
```

### 3. Build the bridge

Source **both** ROS environments before building so the bridge can discover the custom message types:

```bash
source /opt/ros/noetic/setup.bash
source ~/fp_bridge/ros1_ws/devel/setup.bash
source ~/ros2_humble/setup.bash
source ~/fp_bridge/ros2_ws/install/setup.bash

cd bridge_ws
colcon build --cmake-force-configure
cd ..
```

> **Note:** `--cmake-force-configure` is required to force the bridge to regenerate factories for the custom types.

---

## Run

Open **three terminals** and source the appropriate environments in each one.

### Terminal 1 – ROS 1 core

connect to the roscore on the robot

### Terminal 2 – Bridge

```bash
source /opt/ros/noetic/setup.bash
source ros1_ws/devel/setup.bash
source /opt/ros/humble/setup.bash
source ros2_ws/install/setup.bash
source bridge_ws/install/setup.bash

ros2 run ros1_bridge dynamic_bridge --print-pairs
# or to start the bridge:
ros2 run ros1_bridge dynamic_bridge
```

Use `--bridge-all-2to1-topics` if you need all ROS 2 topics visible on the ROS 1 side regardless of active subscribers:

```bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-2to1-topics
```
