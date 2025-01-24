This is a modified version of the official *stretch_nav2* package so that all the navigation functionality can be deployed on an external computer.

# Deploy on Stretch

It is recommended to put the robot in the stow position before starting any mapping or navigation. Make sure to power up Stretch and run it untethered, connect it to the same network as the external computer and run:

1. Free up Stretch processes:
```bash
stretch_free_robot_process.py
```
2. Launch Stretch main driver:
```bash
# Terminal 1: Start the Stretch Driver Node
ros2 launch stretch_core stretch_driver.launch.py mode:=navigation broadcast_odom_tf:=True
```
3. Launch LiDAR:
```bash
# Terminal 2: Start lidar.
ros2 launch stretch_core rplidar.launch.py
```

# Deploy on external computer

The *stretch_nav2_offloaded* package provides the standard ROS 2 navigation stack (Nav2) with its launch files. This package utilizes slam_toolbox and Nav2 to drive Stretch around a mapped space.

Make sure to have ROS2 humble installed and create a workspace directory (I use *ament_ws* but you can use another name):

```bash
source /opt/ros/humble/setup.bash
mkdir -p ~/ament_ws/src
cd ~/ament_ws/src/
```

Also, make sure to have nav2 installed:

```bash
sudo apt install ros-humble-navigation2
sudo apt install ros-humble-nav2-bringup
```

Finally, clone this repository using https or ssh and then build the ROS2 package (remember to source your workspace afterwards):

```bash
# git clone https or ssh
cd ~/ament_ws
colcon build
source install/setup.bash
```

## Mapping

The first step is to map the space that the robot will navigate in. The `offline_mapping.launch.py` will enable you to do this. In this setup, the robot will be teleoperated through a joystick (default) or keyboard (check below). If you are using the joystick, make sure to connect the controller provided by Stretch to the computer. By default, */dev/input/js0* is used, but this can be changed in *launch/teleop_twist.launch.py* with whatever ID your device uses. You can check this using `lsusb` in a terminal.

Next, run:

```bash
ros2 launch stretch_nav2_offloaded offline_mapping.launch.py
```

Rviz will show the robot and the map that is being constructed. With the terminal open, use the joystick (see instructions below for using a keyboard) to teleoperate the robot around. Avoid sharp turns and revisit previously visited spots to form loop closures. In Rviz, once you see a map that has reconstructed the space well enough, open a new terminal and run the following commands to create a `stretch_user/` directory (if you don't have it) and an environmental variable.

```bash
mkdir ~/stretch_user
export HELLO_FLEET_PATH=~/stretch_user
```

Then we save the map to it:

```bash
mkdir ${HELLO_FLEET_PATH}/maps
ros2 run nav2_map_server map_saver_cli -f ${HELLO_FLEET_PATH}/maps/<map_name>
```

**NOTE**: The `<map_name>` does not include an extension. The map_saver node will save two files as `<map_name>.pgm` and `<map_name>.yaml`.

**Tip**: For a quick sanity check, you can inspect the saved map using a pre-installed tool called Eye of Gnome (eog) by running the following command:

```bash
eog ${HELLO_FLEET_PATH}/maps/<map_name>.pgm
```

## Navigation

Next, with `<map_name>.yaml`, we can navigate the robot around the mapped space. Run:

```bash
ros2 launch stretch_nav2_offloaded navigation.launch.py map:=${HELLO_FLEET_PATH}/maps/<map_name>.yaml
```

Note: you might need to run ```export HELLO_FLEET_PATH=~/stretch_user``` again if you cannot find the map.

A new RViz window should pop up with a `Startup` button in a menu at the bottom left of the window. Press the `Startup` button (if it's not already pressed by default) to kick-start all navigation related lifecycle nodes. Rviz will show the robot in the previously mapped space, however, it's likely that the robot's location on the map does not match the robot's location in the real space. To correct this, from the top bar of Rviz, use `2D Pose Estimate` to lay an arrow down roughly where the robot is located in the real space. This gives an initial estimate of the robot's location to AMCL, the localization package. AMCL will better localize the robot once we pass the robot a `2D Nav Goal`.

In the top bar of Rviz, use `2D Nav Goal` to lay down an arrow where you'd like the robot to navigate. In the terminal, you'll see Nav2 go through the planning phases and then navigate the robot to the goal. If planning fails, the robot will begin a recovery behavior - spinning around 180 degrees in place or backing up.

**Tip**: If navigation fails or the robot becomes unresponsive to subsequent goals through RViz, you can still teleoperate the robot using the Xbox controller.

**Note**: The nav2 parameters (e.g., controller and costmap frequency, robot safety radius...) can be changed in the *config/nav2_params.yaml*.

### Teleop using a Joystick Controller

The launch files expose the launch argument "teleop_type". By default, this argument is set to "joystick", which launches joystick teleop in the terminal with the xbox controller that ships with Stretch RE1. The xbox controller utilizes a dead man's switch safety feature to avoid unintended movement of the robot. This is the switch located on the front left side of the controller marked "LB". Keep this switch pressed and translate or rotate the base using the joystick located on the right side of the xbox controller.

If the xbox controller is not available, the following commands will launch mapping and navigation, respectively, with keyboard teleop:

```bash
ros2 launch stretch_nav2_offloaded offline_mapping.launch.py teleop_type:=keyboard
```
or
```bash
ros2 launch stretch_nav2_offloaded navigation.launch.py teleop_type:=keyboard map:=${HELLO_FLEET_PATH}/maps/<map_name>.yaml
```

---

**Maintainer**: [Victor Nan Fernandez-Ayala](mailto:victor@animum.ai)
**License**: Copyright Animum AB 2025
