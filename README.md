# ROS 2 Kortex – RSS Project - Group 12
### *Controlling a Virtual Kinova Gen3 Arm Using Move-IT Servo with IMU Sensor Data*

---

## ⚠️ Dockerfile Changes
This repository provides a **Dockerized ROS 2 environment** for working with the Kinova Gen3 robotic arm **entirely in simulation**. But recent **picknik_controller** lib changes, it fails to compile properly for **ROS-Humble**.  
The following changes were added to Dockerfile so that it compiles successfully:

```bash
# Build the colcon_ws
WORKDIR /colcon_ws/src/picknik_controllers/
RUN git checkout humble
WORKDIR /colcon_ws/
RUN source /opt/ros/humble/setup.bash && \
    colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release --parallel-workers 3 --symlink-install
```

The project uses custom `moveit.rviz` config file to display extra information like the base_link and end_effector_link co-ordinate systems along with TwistedStamp component which visualizes the twist commands in the view. This custom config is set to be copied from `/overlay_ws/src/` to `/colcon_ws/src/ros2_kortex/kortex_moveit_config/kinova_gen3_7dof_robotiq_2f_85_moveit_config/config/` after all the installation is completed.

```bash
# Last operation
COPY /overlay_ws/src/moveit.rviz /colcon_ws/src/ros2_kortex/kortex_moveit_config/kinova_gen3_7dof_robotiq_2f_85_moveit_config/config/
```

---

## 📦 Installation Instructions

### 1. Clone the Repository

Each project group has its own repository named:

**RSS_WS26_Project_Group_12**

```bash
git clone --recurse-submodules https://git-ce.rwth-aachen.de/wzl-mq-ms/forschung-lehre/robotic-sensor-systems/rss_ws26_project_group_12.git

cd rss_ws26_project_group_12
```

If you forgot `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

---

### 2. Build the Docker Image

Run from the root folder:

```bash
bash docker_build.sh
```

This builds the image:

```
ros2-kortex:latest
```

---

### 3. Run the Docker Container

```bash
cd docker_run
bash docker_run.sh
```

This opens a ready-to-use ROS 2 Humble environment.

---

# 🤖 Running the Simulated Kinova Robot

### Source the environment
Inside the container:

```bash
source /opt/ros/humble/setup.bash
source /colcon_ws/install/setup.bash
```

---

# 🔷 Project Explanation

This project uses the following sensors:

1. **Rotary Encoder Sensor**
    - The rotary knob is used to switch between **3 Control Modes** -
        > TCP **Translational** Control (w.r.t. `base_link`).
        > TCP **Orientation** Control (w.r.t. `end_effector_link`).
        > **Gripper** open/close control.
    - The button is used in 2 modes -
        > Single press toggles between master **RUN** and master **OFF**.
        > Long press (> 1 sec) triggers **Homing** for the robotic arm. [Useful in near singularity conditions].

2. **IMU MPU6050**
    - The IMU sensor is used to calculate **Roll, Pitch, Yaw [RPY]** angles.
    - These angles are used by the control node as basis to calculate velocities **vx, vy and vz** respectively.
    - Based on **Control Mode** these velocities are used to generate **Twist Commands** for the moveit_servo controller.
    - The yaw is used to control opening and closing of gripper.

3. **IR sensor**
    - The IR sensor is used as an **Emergency Stop**.
    - No commands from IMU are accepted when the proximity of iR sensor is triggered.

---

# 🔷 Nodes and Packages Developed for Control

1. **arm_servoing:** 
    - This is the main controller node that accepts the sensor data from the topics `/wifi/imu` and `/wifi/enc`.
    - The encoder data is then used to define control modes. The IMU **RPY** data is used to calculate **vx, vy & vz**.
    - For Control mode: 0, the **vx, vy & vz** data is published to the linear argument of topic `/servo_node/delta_twist_cmds` for translation. In control mode 1: the data is passed to the angular argument of the same topic to control orientation.
    - The **vx** data is use to give position command to the action `/robotiq_gripper_controller/gripper_cmd` to open and close the gripper.

2. **moveit_servo:** 
    - [moveit2 package](https://github.com/moveit/moveit2.git) was cloned from the movit github repo and built in `overlay_ws` to enable `/moveit_servo` and it's complimentary services and topics.

3. **servo_rviz:**
    - The `gen3_servo_config.yaml` file contains the important parameters for the servo controller. It defines parameters like move_group_name, singularity_threshold, planning_frame, etc for our kenova gen3 robotic arm.
    - The `servo_rviz.launch.py` launches the rviz sim and all the nodes required to make the control pipeline work.

4. **wifi_sensor_bridge:**
    - This node accepts the sensor data from esp32 on udp port 5000 and pusblishes it to ROS topics `/wifi/imu` and `/wifi/enc`.

---

# 🔷 Launching servo_riz controller

The project controller can be launched using the following command:
```bash
ros2 launch servo_rviz servo_rviz.launch.py bind_ip:=X.X.X.X port:=5000
```

Arguments:
- `bind_ip:=X.X.X.X`         [IP Port: Ip to create the UDP socket]
- `port:=5000`               [Port: Port for the udp socket]
- `launch_ui:=true\false`    [Default: true]

`launch_ui` option can be used to launch a basic PyQt5 GUI Application. The application displays IMU sensor data, directional velocities and control modes.

![plot](./gui.png)

---

# 📘 End of README
