cmake_minimum_required(VERSION 3.5)
project(ros2_fp_core_msgs)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(geometry_msgs REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/JointAngles.msg"
  "msg/OrientationNautical.msg"
  "msg/Pose.msg"
  "srv/Calibrate.srv"
  "srv/DigitalInputOutput.srv"
  "srv/ForwardKinematics.srv"
  "srv/GetGridPoints.srv"
  "srv/GetListBool.srv"
  "srv/GetListFloat.srv"
  "srv/GetPose.srv"
  "srv/GetStatus.srv"
  "srv/Initialize.srv"
  "srv/InverseKinematics.srv"
  "srv/IsMotorMoving.srv"
  "srv/ListInt.srv"
  "srv/MoveJoint.srv"
  "srv/MoveTool.srv"
  "srv/RequestData.srv"
  "srv/SetInt.srv"
  "srv/SetString.srv"
  "srv/SimplePath.srv"
  "srv/TestScript.srv"
  DEPENDENCIES geometry_msgs
)

ament_export_dependencies(rosidl_default_runtime)

install(FILES
  mapping_rules.yaml
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
