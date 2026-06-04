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
Follow the instructions in the quick start guide (in `PRob3_usefulDocumentation`) to set up the Ethernet communication to the robot.
You can also access the myP interface as another way to access the robot functions and sensor values. Note that the ROS functions are a subset of the script functions available in myP, with a smaller set of parameters. Be careful of the following:
- If the robot needs to calibrate and a myP window is open, a popup will appear in that window and the robot will block the calibration until it is validated.
- If the robot is connected on either ROS or myP, it cannot connect to the other one.
- If you want to use the hold/release function, use the myP interface instead of ROS, there is a known problem with these functions in ROS in the myP 1.4.4 version.

Note: it appears that if the robot is connected via myP and the ESP via ROS, the micro-ROS agent disconnects whenever a command is run through the myP window. This should be tested more thoroughly if you need to use both simultaneously.

## Installation

### Prerequisites

This project is developed in WSL2 on Ubuntu 22.04.

Key modifications made to the standard setup:
- ROS 2 Humble built with upstream patches to support `bool`, less stable than a standard install as no ROS 1 version is fully supported on Ubuntu 22.04 and ROS 2 needs to be built from sources.
- The bridge was modified to support custom class tables used in the fp library (example: `Joints[]`).
- The `fp` library was modified to generate readable ROS 1 messages and services, and a ROS 2 side was implemented with custom mapping rules.
- Custom mapping rules require recompiling the bridge (which was also necessary due to the above modifications of the bridge).

Reference guides:
- [Using ros1_bridge on Jammy upstream](https://docs.ros.org/en/humble/How-To-Guides/Using-ros1_bridge-Jammy-upstream.html)
- [ROS 2 Humble — Ubuntu development setup](https://docs.ros.org/en/humble/Installation/Alternatives/Ubuntu-Development-Setup.html)
- [ros1_bridge bool patch (PR #446)](https://github.com/ros2/ros1_bridge/pull/446/changes/8e422c4c644526c018fee107790400eb297ff9a2#diff-1552bca16b8f3f0de4bf527fd4661c21d93318fdfd404d1af86613fad56cabc9R362-R407)

## Install build tools (required for C++)

```bash
sudo apt update
sudo apt install -y build-essential g++ cmake git
```

## Install and build ROS 1 workspace

### 1) Install ROS 1 core packages on Ubuntu 22.04 (Jammy)

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

## Install and build ROS 2 workspace
> **Note:** These steps worked for me, but the installation is not robust and may not work on every machine. Rule of thumb: build everything needed for ROS 1 before touching ROS 2, and when downloading ROS 2, remove all conflicting ROS 1 libraries. This approach works since we do not need to run anything natively in ROS 1, we only need to connect to the robot node and generate the bridge mapping table.

### 1) Download ROS 2 Humble from sources, compatible with ROS 1

Install the required libraries:
```bash
# Fix broken deps first
sudo apt --fix-broken install

# Locale & base tools
sudo apt install -y locales software-properties-common build-essential cmake git wget python3-pip python3-setuptools

# Python / ROS build tools
sudo apt install -y python3-rosdep2   # note: python3-rosdep didn't work, used python3-rosdep2
sudo apt install -y python3-flake8 python3-flake8-blind-except python3-flake8-builtins \
  python3-flake8-class-newline python3-flake8-comprehensions python3-flake8-deprecated \
  python3-flake8-docstrings python3-flake8-import-order python3-flake8-quotes \
  python3-pytest python3-pytest-cov python3-pytest-repeat python3-pytest-rerunfailures

# colcon & vcs (already present via pip/apt)
python3 -m pip install -U colcon-common-extensions vcstool
```

Download and set up the ROS 2 Humble workspace:
```bash
# 1. Set locale
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

# 2. Enable universe repo
sudo add-apt-repository universe

# 3. Create workspace and clone all sources
mkdir -p ~/ros2_humble/src
cd ~/ros2_humble
export PATH="$HOME/.local/bin:$PATH"
vcs import --input https://raw.githubusercontent.com/ros2/ros2/humble/ros2.repos src

# 4. Init and update rosdep
sudo rosdep init || true
rosdep update

# 5. Install dependencies (skip keys that are built from source and already installed with pip), some other dependencies may make this fail, try to continue the steps
cd ~/ros2_humble
rosdep install --from-paths src --ignore-src -y \
  --skip-keys "fastcdr rti-connext-dds-6.0.1 urdfdom_headers python3-catkin-pkg-modules python3-vcstool" \
  --os=ubuntu:jammy
```

### 2) Build ROS 2 Humble

Open a new WSL terminal and run:

```bash
cd ~/ros2_humble
colcon build --symlink-install --packages-skip-build-finished
```

The build takes around 2 hours. If it stops or gets stuck (some libraries require more memory than WSL allocates by default), use the sequential single-job fallback:

```bash
MAKEFLAGS="-j1" colcon build --symlink-install --packages-skip-build-finished --executor sequential
```

### 3) Build `ros2_ws`

In a new shell:

```bash
cd ~/semesterProject_LMTS/fp_bridge/ros2_ws

source ~/ros2_humble/install/setup.bash

colcon build
```

## Build the bridge

Open a clean shell. Before building the bridge, install the ROS 1 Python modules it imports:

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

Even if you do not have access to the ROS 1 robot, you can still inspect the bridge pair list and filter it to `fp_core_msgs` to check if the bridge compilation was successful:

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

### ROS 1 validation

Use a dedicated terminal to validate ROS 1 connectivity to the robot before running the bridge:

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

### Run the bridge

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