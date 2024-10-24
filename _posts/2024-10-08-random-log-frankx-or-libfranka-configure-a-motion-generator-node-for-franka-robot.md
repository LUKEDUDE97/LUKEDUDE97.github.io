---
layout: post
title: 'FrankX or Libfranka Configure A Motion Generator Node for Franka
  Robot'
date: 2024-10-08 09:49 +0200

categories: [Eins-Project, Engineering Log]
tags: [ROS, Franka]

math: true
mermaid: true

post_description: Implementing a ros node to acquire the distal joint status in real-time based on libfranka, which includes a defined motion generator callback in control loop and smoonthing on commands.
---

## Get Robot State

---

I will start with walking through how to access robot status in real-time based on the interface provided by FrankX and libfranka. 

### [FrankX](https://github.com/pantor/frankx.git)

The original version of this package does not provide a implementation on acquiring the state data while robot moving. You have to manually add the changes from [Pull requests](https://github.com/pantor/frankx/pull/44) to your lib, which has instantiated a `franka::RobotState asynchronous_state`. Get real-time robot state information by `robot.get_state(read_once=False)` while drive it by `move_async`. There’s a general limitation (far less than 1000Hz) of sampling rate of this method, it can’t reach the actual control looping frequency from robot, which is reasonable. So maybe it is only useful for just some entry application or without a strict demand on executing frequency. 

> ⛔ By the way, wired thing would happen when you try to print out the state information on your terminal after `get_state(read_once=Flase)`, the obtained data would be totally random in this situation. You should change the `read_once` to `True` to get proper output, yet with a drastic decline on sampling rate.
{: .prompt-warning }

### [Libfranka](https://github.com/frankaemika/libfranka.git)

The real-time commands on robot motion can only be generated and sent by Libfranka in its control loop. The motion generation module provided by FrankX is only a one-time thing, mostly used to drive the robot move along certain path points. 

![Fig.1](/localdata/assets/EinsProject/LibfrankaInterfaceScematic.png)
_Fig. 1. Libfranka Realtime interfaces: motion generators and controllers_

Realtime commands are UDP based and require a 1 kHz connection to Control. There are two types of real-time interfaces: a. **Motion generators**, which define a robot motion in joint or Cartesian space; b. **Controllers**, which define the torques to be sent to the robot joints. Details instruction to generate real-time commands from libfranka control loop is in [here](https://frankaemika.github.io/docs/libfranka.html#errors). I have no idea what does it actually mean here by external or internal, but fortunately, I don’t have to really figure it our temporarily. Just follow the instruction below:

```cpp
try {
  franka::Robot robot("<fci-ip>"); // build connect and instantiate the robot
```

Build a real-time loop (motion generator or controller) by directly defined a callback function in `robot.control()`, which receives the robot state in real-time and return the specific types of commands interface.

```cpp
robot.control(
     [=, &time](const franka::RobotState&, franka::Duration period) -> franka::JointVelocities {
       time += period.toSec();

       double cycle = std::floor(std::pow(-1.0, (time - std::fmod(time, time_max)) / time_max));
       double omega = cycle * omega_max / 2.0 * (1.0 - std::cos(2.0 * M_PI / time_max * time));

       franka::JointVelocities velocities = {{0.0, 0.0, 0.0, omega, omega, omega, omega}};

       if (time >= 2 * time_max) {
         std::cout << std::endl << "Finished motion, shutting down example" << std::endl;
         return franka::MotionFinished(velocities);
       }
       return velocities;
     });
```

This registered `control` method of the `franka::Robot` class will then run the control loop automatically by executing the callback function at a 1 kHz frequency. All control loops are finished once the `motion_finished` flag of a real-time command is set to `true`. As in this example, it uses the joint velocity motion generator interface, as it returns a `franka::JointVelocities` object. Note that if you use only a motion generator, the default controller is the internal joint impedance controller. We don’t actually have to care about this, according the demands so far from the [previous page](https://www.notion.so/Tactile-Servoing-or-Active-Perception-No-Matter-What-I-Need-Maintain-An-Optimal-Contact-24b2ffa82c354a36b0a9e7a205f75a50?pvs=21), the only thing left to do is to build another `CartesianVelocities` based motion generator in control loop. 

## Prerequisite

---

Before getting into the details, you need to first prepare the computer to connect with the Franka robot. I will skip some details here.

1. Of course you should first build the FrankX and libfranka from source and install them
2. Set up the [real-time kernel](https://frankaemika.github.io/docs/installation_linux.html) for running the libfranka and Franka robot. 
3. Connect to the Franka robot based on its ip address (sometimes the end-effector may not initialized correctly on the first connection, if so, you need to restart the end-effector itself from the control panel `192.168.178.12` in settings)

## Motion Generation Node

---

The biggest problem encountered here is to set constraints on your desired commands and further smooth them, otherwise you will constantly encounter the **`cartesian motion generator joint acceleration discontinuity`** or other similar errors. The detailed specifications on this issue is listed [here](https://frankaemika.github.io/docs/control_parameters.html#control-parameters-specifications). Basically, it prevent the robot from a sudden acceleration on their joints. The motion will be aborted accordingly unless the necessary conditions are fulfilled along the trajectory. 

Considering we are building a `CartesianVelocities` based control, the necessary conditions for us are as followed:

$$
\begin{equation}
\begin{array}{c}
    T \text{ is a proper transformation matrix} \\[1pt]
    -\dot{p}_{\text{max}} < \dot{p}_{c} < \dot{p}_{\text{max}} \quad \text{(Cartesian velocity)} \\[1pt]
    -\ddot{p}_{\text{max}} < \ddot{p}_{c} < \ddot{p}_{\text{max}} \quad \text{(Cartesian acceleration)} \\[1pt]
    -\dddot{p}_{\text{max}} < \dddot{p}_{c} < \dddot{p}_{\text{max}} \quad \text{(Cartesian jerk)}
\end{array}
\end{equation}
$$

Transform the desired orientation target into rotary speed before that, and conduct a speed limit on three elements here.

```cpp
    // dP limit
    target_v_x = std::max(-TRANSLATION_VEL_LIMIT, std::min(target_v_x, TRANSLATION_VEL_LIMIT));
    target_v_y = std::max(-TRANSLATION_VEL_LIMIT, std::min(target_v_y, TRANSLATION_VEL_LIMIT));
    target_w_z = std::max(-ROTATION_VEL_LIMIT, std::min(target_w_z, ROTATION_VEL_LIMIT));
```

Calculate the target acceleration based on the target derivatives. Further limit and ramp the accelerations (equals to limit the jerk on acceleration) on three elements. 

```cpp
    // Compute target ddP
    double target_a_x = (target_v_x - robot_state.O_dP_EE_c[0]) / dt;
    double target_a_y = (target_v_y - robot_state.O_dP_EE_c[1]) / dt;
    double target_a_wz = (target_w_z - robot_state.O_dP_EE_c[5]) / dt;

    // ddP limit
    target_a_x = std::max(-TRANSLATION_ACC_LIMIT, std::min(target_a_x, TRANSLATION_ACC_LIMIT));
    target_a_y = std::max(-TRANSLATION_ACC_LIMIT, std::min(target_a_y, TRANSLATION_ACC_LIMIT));
    target_a_wz = std::max(-ROTATION_ACC_LIMIT, std::min(target_a_wz, ROTATION_ACC_LIMIT));

    // Ramp ddP based on dddP limits
    double a_x = ramp(target_a_x, robot_state.O_ddP_EE_c[0], TRANSLATION_JERK_LIMIT, dt);
    double a_y = ramp(target_a_y, robot_state.O_ddP_EE_c[1], TRANSLATION_JERK_LIMIT, dt);
    double a_wz = ramp(target_a_wz, robot_state.O_ddP_EE_c[5], ROTATION_JERK_LIMIT, dt);
```

The ramp function is as followed.

```cpp
double ramp(double target, double previous, double max_rate, double dt)
{
    double diff = target - previous;
    double max_change = max_rate * dt;
    if (fabs(diff) > max_change)
    {
        target = previous + max_change * (diff > 0 ? 1 : -1);
    }
    return target;
}
```

Update the actual derivates based on ramped acceleration, the generated cartesian motion here is finally acceptable for the robot, which is smoothed on its derivative, acceleration and jerk.

```cpp
    double v_x = robot_state.O_dP_EE_c[0] + a_x * dt;
    double v_y = robot_state.O_dP_EE_c[1] + a_y * dt;
    double w_z = robot_state.O_dP_EE_c[5] + a_wz * dt;
```

The numerical value on Limits in the Cartesian space are as follows:

![Fig2](/localdata/assets/EinsProject/LibfrankaNumericalRequisite.png)

We further turn them down accordingly to ensure a smoothed movement. 

```cpp
#define TRANSLATION_VEL_LIMIT 1.7           // dp.Translation - (m/s)
#define TRANSLATION_ACC_LIMIT 13.0 * 0.1    // ddp.Translation - (m/s^2) : factor 10
#define TRANSLATION_JERK_LIMIT 6500.0 * 0.1 // dddp.Translation - (m/s^3) : factor 10
#define ROTATION_VEL_LIMIT 2.5              // dp.Rotation - (rad/s)
#define ROTATION_ACC_LIMIT 25.0 * 0.1       // ddp.Rotation - (rad/s^2) : factor 10
#define ROTATION_JERK_LIMIT 12500.0 * 0.1   // dddp.Rotation - (rad/s^3) : factor 10
```

The entire implementation `motion_generator.cpp` is as followed:

```cpp
#include <cmath>
#include <iostream>
#include <mutex>
#include <thread>
#include <franka/exception.h>
#include <franka/robot.h>
#include "examples_common.h"
#include <ros/ros.h>
#include <whisker_customed_msg/FrankaCtrl.h>
#include <geometry_msgs/TwistStamped.h>
#include <Eigen/Dense>

#define TOTAL_VEL 0.006 // Ensure consistency

#define TRANSLATION_VEL_LIMIT 1.7           // dp.Translation - (m/s)
#define TRANSLATION_ACC_LIMIT 13.0 * 0.1    // ddp.Translation - (m/s^2) : factor 10
#define TRANSLATION_JERK_LIMIT 6500.0 * 0.1 // dddp.Translation - (m/s^3) : factor 10
#define ROTATION_VEL_LIMIT 2.5              // dp.Rotation - (rad/s)
#define ROTATION_ACC_LIMIT 25.0 * 0.1       // ddp.Rotation - (rad/s^2) : factor 10
#define ROTATION_JERK_LIMIT 12500.0 * 0.1   // dddp.Rotation - (rad/s^3) : factor 10

struct ControlCommands
{
    double x_velocity = 0.0;
    double y_velocity = -TOTAL_VEL;
    double rotation = M_PI; // Target rotation position on z-axis
    std::mutex mtx;
};

ControlCommands control_commands;
std::array<double, 16> initial_pose;
double accumulated_val = control_commands.rotation;

void controlCommandCallback(const whisker_customed_msg::FrankaCtrl::ConstPtr &msg)
{
    std::lock_guard<std::mutex> lock(control_commands.mtx);
    control_commands.x_velocity = msg->xvel;
    control_commands.y_velocity = msg->yvel;
    control_commands.rotation = static_cast<double>(msg->orientation); // DYX : float32 -> double
    ROS_INFO("!!! Received control command: x_velocity=%f, y_velocity=%f, rotation=%f", msg->xvel, msg->yvel, msg->orientation);
}

double ramp(double target, double previous, double max_rate, double dt)
{
    double diff = target - previous;
    double max_change = max_rate * dt;
    if (fabs(diff) > max_change)
    {
        target = previous + max_change * (diff > 0 ? 1 : -1);
    }
    return target;
}

Eigen::Vector3d rotationMatrixToEulerAngles(const Eigen::Matrix3d &R)
{
    Eigen::Vector3d euler;
    euler[0] = atan2(R(2, 1), R(2, 2));                                  // Roll
    euler[1] = atan2(-R(2, 0), sqrt(pow(R(2, 1), 2) + pow(R(2, 2), 2))); // Pitch
    euler[2] = atan2(R(1, 0), R(0, 0));                                  // Yaw
    return euler;
}

geometry_msgs::TwistStamped matrixToTwistStamped(const std::array<double, 16> &O_T_EE)
{
    Eigen::Matrix3d rotation_matrix;
    rotation_matrix << O_T_EE[0], O_T_EE[1], O_T_EE[2],
        O_T_EE[4], O_T_EE[5], O_T_EE[6],
        O_T_EE[8], O_T_EE[9], O_T_EE[10];

    Eigen::Vector3d euler_angles = rotationMatrixToEulerAngles(rotation_matrix);

    geometry_msgs::TwistStamped twist;
    twist.twist.linear.x = O_T_EE[12];
    twist.twist.linear.y = O_T_EE[13];
    twist.twist.linear.z = O_T_EE[14];

    twist.twist.angular.x = euler_angles[0];
    twist.twist.angular.y = euler_angles[1];
    twist.twist.angular.z = euler_angles[2];

    return twist;
}

void wrapDetection(double cur, double pre)
{
    if (!std::isnan(pre))
    {
        if (pre > 0 && cur < 0 && std::abs(pre - cur) > 3) // pi -> -pi
        {
            accumulated_val += (cur + 2 * M_PI) - pre;
        }
        else if (pre < 0 && cur > 0 && std::abs(pre - cur) > 3) // -pi -> pi
        {
            accumulated_val += (cur - 2 * M_PI) - pre;
        }
        else // 0 -> -0 and -0 -> 0 and all the other situations
        {
            accumulated_val += cur - pre;
        }
    }
}

int main(int argc, char **argv)
{
    ros::init(argc, argv, "robot_ctrl");
    ros::NodeHandle nh;

    if (argc != 2)
    {
        ROS_ERROR("Usage: %s <robot-hostname>", argv[0]);
        return -1;
    }

    ros::Subscriber control_command_sub = nh.subscribe("/Franka_Ctrl", 10, controlCommandCallback);
    ros::Publisher robot_state_pub = nh.advertise<geometry_msgs::TwistStamped>("/FrankaEE_State", 10);

    // Use AsyncSpinner for non-blocking callback handling, the spin loop and control loop were separate
    ros::AsyncSpinner spinner(1);
    spinner.start();

    try
    {
        franka::Robot robot(argv[1]);
        setDefaultBehavior(robot);
        setDefaultBehavior(robot);
        std::array<double, 7> q_default = {{0, -M_PI_4, 0, -3 * M_PI_4, 0, M_PI_2, M_PI_4}}; // Franka Robot Default Pose
        // std::array<double, 7> q_initPosi_MPI = {{0.21412348725398378, -0.057754063327873625, -0.00023695136063781077, -2.2539809572387735, -0.000522136062031758, 2.1969546655019125, -2.142070478597319}}; // Bottle
        // std::array<double, 7> q_initPosi_MPI = {{0.027985085379659082, 0.3900154336046925, -3.026799439030516e-05, -1.8750191500898001, 0.0006431883205659688, 2.262886618296305, 0.812911443180508}}; // 160 circle 

        std::array<double, 7> q_initPosi_MPI = {{0.212508466987283, 0.2168878125684303, 0.000354176827846183, -1.7287890095961722, 0.0003577190608942624, 1.9441698141098267, 0.9973949831209064}}; // 160 rectangle
        
        ROS_WARN("This example will move the robot! Please make sure to have the user stop button at hand!");
        ROS_INFO("Press Enter to continue..."); 
        std::cin.ignore();
        MotionGenerator motion_generator_default(0.5, q_default);
        robot.control(motion_generator_default);
        MotionGenerator motion_generator_initPosi(0.5, q_initPosi_MPI);
        robot.control(motion_generator_initPosi);
        ROS_INFO("Finished moving to initial joint configuration.");

        robot.setJointImpedance({{3000, 3000, 3000, 2500, 2500, 2000, 2000}});

        std::array<double, 7> lower_torque_thresholds_nominal{{25.0, 25.0, 22.0, 20.0, 19.0, 17.0, 14.0}};
        std::array<double, 7> upper_torque_thresholds_nominal{{35.0, 35.0, 32.0, 30.0, 29.0, 27.0, 24.0}};
        std::array<double, 7> lower_torque_thresholds_acceleration{{25.0, 25.0, 22.0, 20.0, 19.0, 17.0, 14.0}};
        std::array<double, 7> upper_torque_thresholds_acceleration{{35.0, 35.0, 32.0, 30.0, 29.0, 27.0, 24.0}};
        std::array<double, 6> lower_force_thresholds_nominal{{30.0, 30.0, 30.0, 25.0, 25.0, 25.0}};
        std::array<double, 6> upper_force_thresholds_nominal{{40.0, 40.0, 40.0, 35.0, 35.0, 35.0}};
        std::array<double, 6> lower_force_thresholds_acceleration{{30.0, 30.0, 30.0, 25.0, 25.0, 25.0}};
        std::array<double, 6> upper_force_thresholds_acceleration{{40.0, 40.0, 40.0, 35.0, 35.0, 35.0}};
        robot.setCollisionBehavior(
            lower_torque_thresholds_acceleration, upper_torque_thresholds_acceleration,
            lower_torque_thresholds_nominal, upper_torque_thresholds_nominal,
            lower_force_thresholds_acceleration, upper_force_thresholds_acceleration,
            lower_force_thresholds_nominal, upper_force_thresholds_nominal);

        double time_max = 150.0;
        double time = 0.0;
        double previous_yaw = control_commands.rotation;
        robot.control([=, &time, &robot_state_pub, &previous_yaw](const franka::RobotState &robot_state, franka::Duration period) -> franka::CartesianVelocities
                      {
    time += period.toSec();
    double dt = period.toSec();
    if (time == 0.0) {
        initial_pose = robot_state.O_T_EE_c;
    }
    ROS_INFO("-----");
    ROS_INFO("At time : %f; Perception control commands : %f, %f, %f.", time, control_commands.x_velocity, control_commands.y_velocity, control_commands.rotation);

    geometry_msgs::TwistStamped ee_state = matrixToTwistStamped(robot_state.O_T_EE);
    ee_state.header.stamp = ros::Time::now();
    robot_state_pub.publish(ee_state);
    ROS_INFO("At time : %f; Robot cartesian pose : %f, %f, %f.", time, ee_state.twist.linear.x, ee_state.twist.linear.y, ee_state.twist.angular.z);

    // Retrieve linear velocity
    double target_v_x;
    double target_v_y;
    {
        std::lock_guard<std::mutex> lock(control_commands.mtx);
        target_v_x = control_commands.x_velocity;
        target_v_y = control_commands.y_velocity;
    }

    // Retrieve target rotation and transform into angular velocity 
    double current_yaw = atan2(robot_state.O_T_EE_c[1], robot_state.O_T_EE_c[0]);
    wrapDetection(current_yaw, previous_yaw);
    previous_yaw = current_yaw;
    double target_yaw;
    {
        std::lock_guard<std::mutex> lock(control_commands.mtx);
        target_yaw = control_commands.rotation;
    } 
    ROS_INFO("Current yaw: %f, accumulated yaw: %f", current_yaw, accumulated_val);
    double angular_error = target_yaw - accumulated_val;
    double target_w_z = angular_error * 1.0;

    // dP limit
    target_v_x = std::max(-TRANSLATION_VEL_LIMIT, std::min(target_v_x, TRANSLATION_VEL_LIMIT));
    target_v_y = std::max(-TRANSLATION_VEL_LIMIT, std::min(target_v_y, TRANSLATION_VEL_LIMIT));
    target_w_z = std::max(-ROTATION_VEL_LIMIT, std::min(target_w_z, ROTATION_VEL_LIMIT));

    // Compute target ddP
    double target_a_x = (target_v_x - robot_state.O_dP_EE_c[0]) / dt;
    double target_a_y = (target_v_y - robot_state.O_dP_EE_c[1]) / dt;
    double target_a_wz = (target_w_z - robot_state.O_dP_EE_c[5]) / dt;

    // ddP limit
    target_a_x = std::max(-TRANSLATION_ACC_LIMIT, std::min(target_a_x, TRANSLATION_ACC_LIMIT));
    target_a_y = std::max(-TRANSLATION_ACC_LIMIT, std::min(target_a_y, TRANSLATION_ACC_LIMIT));
    target_a_wz = std::max(-ROTATION_ACC_LIMIT, std::min(target_a_wz, ROTATION_ACC_LIMIT));

    // Ramp ddP based on dddP limits
    double a_x = ramp(target_a_x, robot_state.O_ddP_EE_c[0], TRANSLATION_JERK_LIMIT, dt);
    double a_y = ramp(target_a_y, robot_state.O_ddP_EE_c[1], TRANSLATION_JERK_LIMIT, dt);
    double a_wz = ramp(target_a_wz, robot_state.O_ddP_EE_c[5], ROTATION_JERK_LIMIT, dt);

    // Update dP based on ramped ddP
    double v_x = robot_state.O_dP_EE_c[0] + a_x * dt;
    double v_y = robot_state.O_dP_EE_c[1] + a_y * dt;
    double w_z = robot_state.O_dP_EE_c[5] + a_wz * dt;
    ROS_INFO("At time : %f; Final control commands input : %f, %f, %f.", time, v_x, v_y, w_z);

    franka::CartesianVelocities output = {{v_x, v_y, 0.0, 0.0, 0.0, w_z}};
    if (time >= time_max) {
        ROS_INFO("Finished motion, shutting down example");
        return franka::MotionFinished(output);
    }
    return output; });
    }
    catch (const franka::Exception &e)
    {
        ROS_ERROR("Franka exception: %s", e.what());
        try
        {
            franka::Robot robot(argv[1]);
            ROS_WARN("Attempting automatic error recovery...");
            robot.automaticErrorRecovery();
            ROS_INFO("Automatic error recovery successful.");
        }
        catch (const franka::Exception &recovery_exception)
        {
            ROS_ERROR("Automatic error recovery failed: %s", recovery_exception.what());
            return -1;
        }
    }

    spinner.stop();
    return 0;
}
```

Run this ros node by `rosrun franka_whisker robot_control 192.168.178.12` start to receive the desired commands from `ActivePerception4Whisker` node. The entire project is [here](https://github.com/LUKEDUDE97/ActivatePerception4WhiskerTipEXP.git).
