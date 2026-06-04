# fp_bridge

Custom ROS 1 ↔ ROS 2 bridge based on [`ros2/ros1_bridge`](https://github.com/ros2/ros1_bridge), extended to support [`fp_core_msgs`](https://github.com/fp-robotics/fp_core_msgs) custom message types.

The workspace layout is:

```text
fp_bridge/
├── ros1_ws/        # ROS 1 workspace (fp_core_msgs)
├── ros2_ws/        # ROS 2 workspace (ros2_fp_core_msgs)
└── bridge_ws/      # bridge workspace (ros1_bridge with patches)
```

## Robot network setup
Follow the instuctions in the quick start guide to setup the ethernet communication to the robot.
Then you can also access the myP interface to get an other way to access to the robot fonctions and sensor values. Just be careful as
- if the robot needs to calibrate and a myP windows is open, a popup will appear in this windows and the robot will bloc the calibration until it's validated
- it the robot is connected on either ros or myP, it can not connect to the other one
- if you want to use hold/release function, use the myP interface instead of ros, there is a know problem with these function in ros in the myp 1.4.4 version.
For more info on the fonctions, dont hesitate to read the documentation in the PRob3_usefulDocuemntation. Just know that the ros functions are a subset of the script fonction available in myP, with a less diverse set of parameters.

Also it seems like, if the robot is connected via myP and the esp via ROS, the microros agent disconnect whenever a command is run through the myP windows. To be tested more throughfully if you need to use it.

## Installation

### Prerequisites

This project is developed in WSL2 on Ubuntu 22.04.

humble with upstream to be able to use bool patch -> less stable
the bridge code was also modified to be able to use personal class tables (Joints[])

The fp library was modified to generate readable ROS1 msg and srv, and a ROS2 side was implemented, with custom mapping rules

with custom mapping rules the bridge needs to be recompiled, but as I modified it it needed to be recompiled anyway

Reference guides:
https://docs.ros.org/en/humble/How-To-Guides/Using-ros1_bridge-Jammy-upstream.html
https://docs.ros.org/en/humble/Installation/Alternatives/Ubuntu-Development-Setup.html

TODO clone this repro if not done already with submodules

## Install build tools (required for C++)

```bash
sudo apt update
sudo apt install -y build-essential g++ cmake git
```

## Install and build ROS1 workspace

### 1) Install ROS1 core packages on Ubuntu 22.04 (Jammy)

```bash
sudo apt update
sudo apt install -y ros-core-dev
```

Note: on Jammy with this setup, `/opt/ros/noetic` is not present. Do not run `source /opt/ros/noetic/setup.bash`.

### 2) Build `ros1_ws`

```bash
cd ~/semesterProject_LMTS/fp_bridge/ros1_ws
catkin_make
```

## Install and build ROS2

### 1) remove conflicting ROS1 files
TODO

Expected result: CMake config completes and `make` runs without `No CMAKE_CXX_COMPILER could be found`.

### 2) Download ROS2 humble from sources, compatiblel with ros1

TODO: copy log file

### 3) Build ROS2 humble

open a new wsl terminal and type

```bash
cd ~/ros2_humble
colcon build --symlink-install --packages-skip-build-finished
```

as the build is really long (around 2h on my computer) and some library takes too much memory for WSL, if the build stops or is stuck try with this to build the problematics libraries

```bash
MAKEFLAGS="-j1" colcon build --symlink-install --packages-skip-build-finished --executor sequential
```

### 4) Build `ros2_ws`

in a new shell

```bash
cd ~/semesterProject_LMTS/fp_bridge/ros2_ws

source ~/ros2_humble/install/setup.bash

colcon build
```

## Build the bridge

in a clean shell
Before building the bridge, install the ROS1 Python modules it imports:

```bash
sudo apt update
sudo apt install -y python3-rosmsg python3-roslib python3-rospkg python3-catkin-pkg python3-genpy
```

Then build the bridge:

```bash
cd ~/semesterProject_LMTS/fp_bridge/bridge_ws

source ~/semesterProject_LMTS/fp_bridge/ros1_ws/devel/setup.bash

source ~/ros2_humble/install/setup.bash

source /home/fleur/semesterProject_LMTS/fp_bridge/ros2_ws/install/setup.bash

MAKEFLAGS="-j1" colcon build --packages-select ros1_bridge --cmake-force-configure --event-handlers console_direct+
```

## Offline check for `fp_core_msgs`

Even if you do not have access to the ROS 1 robot, you can still inspect the bridge pair list and filter it to `fp_core_msgs` to check if the bridge compilation was sucessful:

```bash
source ~/semesterProject_LMTS/fp_bridge/ros1_ws/devel/setup.bash
source ~/ros2_humble/install/local_setup.bash
source ~/semesterProject_LMTS/fp_bridge/ros2_ws/install/local_setup.bash
source ~/semesterProject_LMTS/fp_bridge/bridge_ws/install/local_setup.bash

ros2 run ros1_bridge dynamic_bridge --print-pairs | grep fp_core_msgs
```

This only shows supported message pairs. It does not require the bridge to connect to the robot.

## Robot Connection Check

For ROS1/bridge communication with the robot master, set `ROS_MASTER_URI` and, if needed, `ROS_IP` to an IP reachable by the robot.

In WSL, `ROS_IP` can change between sessions; re-check with `hostname -I`.

Get your available IPs:

```bash
# Quick list (space-separated)
hostname -I

# Detailed list by interface (recommended)
ip -4 addr show
```

How to choose which IP to use for `ROS_IP`:
- Choose your WSL IPv4 on the same subnet as the robot master (`10.0.0.203`), so usually `10.0.0.x`.
- Do not use loopback (`127.0.0.1`) or Docker-only interfaces.
- If multiple `10.0.0.x` addresses exist, test reachability and pick the one that can reach the robot:

```bash
ping -c 3 10.0.0.203
```

Example:

```bash
export ROS_MASTER_URI=http://10.0.0.203:11311
export ROS_IP=10.0.0.42
```

### ROS1 validation

Use a dedicated terminal to validate ROS1 connectivity to the robot before running the bridge:

```bash
source ~/semesterProject_LMTS/fp_bridge/ros1_ws/devel/setup.bash
export ROS_MASTER_URI=http://10.0.0.203:11311
export ROS_IP=<yourIP>
```

Then print ROS1 topics and services:

```bash
rostopic list
rosservice list
```

### Run steps for bridge

Open a new terminal and source the appropriate environments.

```bash
source ~/semesterProject_LMTS/fp_bridge/ros1_ws/devel/setup.bash
source ~/ros2_humble/install/setup.bash
source ~/semesterProject_LMTS/fp_bridge/ros2_ws/install/setup.bash
source ~/semesterProject_LMTS/fp_bridge/bridge_ws/install/setup.bash

ros2 run ros1_bridge dynamic_bridge
```

Use `--bridge-all-2to1-topics` if you need all ROS 2 topics visible on the ROS 1 side regardless of active subscribers:

```bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-2to1-topics
```