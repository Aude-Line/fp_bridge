cmake_minimum_required(VERSION 2.8.3)
project(fp_core_msgs)

# find message_generation library
find_package(catkin REQUIRED
  geometry_msgs
  std_msgs
  message_generation
)

# generate messages in the 'msg' folder
add_message_files(
  FILES
  JointAngles.msg
  OrientationNautical.msg
  Pose.msg
)

# generate services in the 'srv' folder
add_service_files(
  FILES
  Calibrate.srv
  DigitalInputOutput.srv
  RequestData.srv
  Initialize.srv
  ForwardKinematics.srv
  InverseKinematics.srv
  ListInt.srv
  SetInt.srv
  MoveJoint.srv
  MoveTool.srv
  SetString.srv
  SimplePath.srv
  TestScript.srv
  GetStatus.srv
  GetListFloat.srv
  GetListBool.srv
  GetGridPoints.srv
  GetPose.srv
  IsMotorMoving.srv
)

# generate added messages and services
generate_messages(
  DEPENDENCIES
  std_msgs
  geometry_msgs
)

catkin_package(
  CATKIN_DEPENDS 
    message_runtime 
    std_msgs 
    geometry_msgs
)