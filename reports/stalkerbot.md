#### Cosi 119a - Autonomous Robotics
#### Team: Stalkerbot
#### Team members: Danbing Chen, Maxwell Hunsinger
#### Final Report

## Introduction
### Background
In the recent years, the pace of development of autonomous robotics has quickened, due to the increase in demand for robots which perform actions in the absence of human control and minimal human surveillance. The robots are through the years more capable of performing an ever-various range of tasks, and one such task, which has become more prevalence in the capabilities of modern robots, is the ability to follow a person. Robots which are capable of following people are widely deployed and used for caretaking and assistance purposes.

Although the premise of a person-follow algorithm is simple, in practice the algorithm must consider several complexities and challenges, some of which depend on the general purpose of such said robots, and some of which are due to the inherent complication of the practice of robotics.

### Overview
Stalkerbot is an algorithm which detects and follows the targeted individual until the robot has reached a certain distance within the target, which then it will stop. Although the algorithm has the capability to detect and avoid obstacles, it does not operate well in a spatially crowded environment where the room for robotic maneuver is limited. The algorithm works well only in an environment with enough light exposure; hence it will not function properly or at all in the darkness.

One of the most basic and perhaps fundamental challenge of such algorithm is the recognition of the targeted person. The ability of the algorithm to quickly and accurately identify and focus on the target is crucial, as it is closely linked to most of the other parts of the algorithm, which may include path finding, obstacle avoidance, motion detect, etc.

## Algorithm Design
### External Libraries and Dependencies
Although there are sophisticated methods of recognizing the potential target, including 3D Point Clouds and convoluted computer vision algorithms, our project will only be using a rather simple library in rospy called the Aruco Detect, which is a fiducial marker detection library which can detect and identify specific markers using a camera and return the pose and transformation of the fiducial. We chose to use the library because it is simple to implement and does not involve sophisticated and heavy computation such as machine learning, with, of course, the downside of the algorithm’s having dependency on fiducial marker as well as the library.

We will be using SLAM for local mapping of the robot, which takes in lidar data and generate the map of its immediate surroundings. We will also be using ros_navigation package, or more specifically, the move_base module in the package to facilitate the path finding algorithm of Stalkerbot. It is important to note that these two packages make up most of the computational consumption of the algorithm, as the rest of the generic scripts written in Python are standard, light-weight ROS nodes.
### Hardware Specification
Sensors are the foundation of every robotics application, they are to bridge the gap between the digital world and the real world, they give perception to robots and allow them to respond to the changes in the real world, therefore it is not surprising that the other defining characteristics of a path finding algorithm is its sensors.

The sensors which we use for our algorithm are the lidar and the raspberry-pi camera. Camera will be used to detect the target via the aforementioned Aruco Detect library. We will support the detection algorithm with sensor data from the lidar which is equipped on the Turtlebot. These data will be used to map the surroundings of the robot in real time and play an important part in the path planning algorithm, which will enable the robot to detect and avoid obstacles on the path to the target.

On the Turtlebot3, the camera will be installed in the front at an inclination of about 35 degrees upwards and the lidar is installed at around 10cm above the ground level. These premises are important factors in the consideration of the design of the algorithm, and they often ease or pose limitations on the design space of the algorithm. In addition, the Turtlebot runs on Raspberry-pi 3 board. The board is adequate for running simple programming scripts; however, they are not fit to run computationally heavy script such as Aruco Detect or SLAM, therefore, these programs have to be run remotely and the data will have to be communicated to the robots wirelessly.
### Algorithm
![](https://i.imgur.com/nOomheX.png)
The very first design choice which we must consider is this: Where does the robot go? The question might sound simple, or even silly, because the goal of the algorithm is to follow a person, so the target of the robot must be the person which it follows. However, it is important to remember that the robot does not reach the target, instead it follows the target up to a certain distance behind the target, therefore a second likely choice for which where the robot should aim to move to is the point which is at said distance behind the target.

The two different approaches translate to two very different behaviors of the robots, for example, a direct following approach reacts minimally to the target’s sudden change in rotation, whereas the indirect following approach reacts more drastically to it. It should be said that under different situations one might be better than the other. If the target simply rotates but does not turn, for example, then by using the direct approach the robot would stand still, however, if the target intends to rotate, then the indirect approach helps more adequately to better prepare for the change of course preemptively than the direct approach.

We have chosen the direct approach in the development of Stalkerbot, because even though the indirect approach is often the more advantageous approach in many situations, it has a severe drawback that is associated with how the camera is mounted on the robot. For example, when the target turns, the robot must move side-ways, and since its wheels are not omnidirectional, but bidirectional, the robot must turn to move to the appropriate spot, and in term lose sight of the target in the process, because the robot can only detect what is in front of it, since the camera is mounted directly ahead of the robot, and no additional camera is mounted on the robot. The continuous update of the state of the target is crucial in the algorithm, and the lack of the information on the target is unjustifiable, hence the indirect approach is not considered.

Fiducial markers can be identified by their unique barcode-like patterns, and each is assigned an identification number. We only want to identify specific fiducial markers. The reason which we want to do this is this: there are other projects, including the campus_rover_3, who are using the same library as we do, and we do not wish to confuse the robots with the fiducials from other projects. All the accepted fiducials are stored as their identification number in an array in the configuration file. The node fiducial_filter filters out unwanted fiducial markers and republishes those that match against the fiducial markers whose identification number is in the configuration file.

It is important to ensure that the robot follows the target in a smooth manner. However, the detection capability with the camera which is mounted on the robot is not perfect. Although Aruco Detect can detect the fiducial markers, it is awfully inconsistent, and the detection rate drops significantly with distance, with the marker becoming almost impossible to detect at any distance farther than 2.5 meters. At normal range, it is expected that the algorithm will miss certain frames. To ensure the smoothness of the robot’s motor movement, a buffer time is introduced to the follow algorithm to take the momentary loss of information on the state of the target into account. If the robot is unable to detect any fiducial marker within this buffer duration, it will keep moving as if the target is still at the place when it is last detected, however when the robot is still unable to detect any markers after that grace period, the robot will stop. We have found that 200 milliseconds to be an appropriate time, with which the robot will run smoothly even if the detection algorithm does not.

These algorithms are contained in the advanced_follow node, which is also where all of the motion algorithms lie. This node is responsible for how the robot reacts to the change in the sensor data, therefore, this node can be seen as the core of the person-following algorithm. The details of this node will be explained further later in this section.

In order to compare the time elapsed since the robot has last recognized a fiducial marker and the current time, we have created a ROS node called fiducial_interval, which measures the time since the last fiducial marker is detected. Note that when the algorithm first starts, and no fiducial is detected, the node will publish nothing. Only when the first fiducial marker is detected, then the node will be activated and start to publish the duration.

Aruco Detect publishes the information on the fiducial markers which the robot detects, however, to extract and filter the important data (pose, mainly), and to filter the unwanted data (timestamp, id, fiducial size, error rate, etc), we have created a custom node called location_pub, which subscribes to fiducial_filter and publishes only the important information which will be useful for the Stalkerbot algorithm.

One of the biggest components of Stalkerbot is its communication with the ros_tf package. The package is a powerful library that aligns the locations and orientations of different objects that have coordinates on different coordinate frames. In Stalkerbot, we feed information of the relationship between two frames into ros_tf and we also extract those data from it. We require this because it is otherwise difficult to understand and calculate the pose of an object from, for example, the robot, when the data is gathered by the camera, which uses a different coordinate system, or frame, as the robot.

The node location_pub_tf publishes to ros_tf the pose of an object as perceived by another object. In the node, two different relationships are transmitted to ros_tf. Firstly, the node broadcasts a static pose of the camera from the robot. The pose is static because the camera is mounted on the robot and the pose between them will not change during the execution and runtime of Stalkerbot. Because it is a static broadcast, the pose is sent to ros_tf once, at the beginning of the algorithm, via a special broadcast object that specifies this broadcast is static. Furthermore, the node will, at a certain frequency, listen to the topic /stalkerbot/fiducial/transform and broadcast the pose of the target with respective to the frame of the camera to ros_tf. Note that, just like the node fiducial_interval, nothing will be broadcasted before the detection of the first fiducial marker.

Like mentioned before, not only do we broadcast the coordinates to ros_tf, we also request them from it as well. In fact, the only place where we request for any coordinates between two objects is in the advanced_follow node. The advanced follow node uses, additionally, the move_base module from ros_nav, however, to send a goal to the module requires the algorithm to have the coordinate frame between the frame map and target. Therefore, at a certain frequency, the node requests the pose between map and target from ros_tf.
When the robot reaches the destination and the target is not in motion for a certain number of seconds, the follow algorithm will shut itself down. The detection of motion is contained in the motion_detect node. This is an algorithm just as complicated as the follow algorithm. It listens to the ros_tf transform between the robot and the target and calculate the distance from the pose of the current frame to the last known frame. However, most of the logic issued in this node is to enable the algorithm to recognize motion and the lack of motion in the presence of sensory noise. It uses average mean and a fluctuating distance threshold level to determine whether the object is moving at any given moment.

The node also implements a lingering algorithm which has a counter which counts down when the rest of the algorithm returns false and reverts the response to positive. When the counter reaches zero, it is no longer able to revert any negative response. The counter is reset when a fiducial marker is detected. This is again a method to combat the inconsistency of Aruco Detect, you can think of the counter as the minimal number of times the algorithm will return true. However, during the development stage, this method is not used anymore. The counter is specified to 0 according to the configuration file.
Similar to fiducial_filter, it also has its corresponding interval node, motion_interval, which returns the time since the last time when the target is in motion.

Lastly, Stalkerbot also implements a special node called sigint_catcher, in which the node only activates when the program is terminated via KeyboardInterrupt. The only purpose of this node is to send an empty cmd_vel to roscore, thereby stopping the robot.

## Story of the Project

The inspiration for this project is a crowdfunding project that was initiated by a company called "Cowarobot", which aims to create the world's first auto-following suitcase/luggage. This project is a scaled-down version of this project. Initially, we had great ambitions for the project; we had plans to use facial recognition and machine learning to recognize the target which the robot follows, however, it is soon clear to us that given our expertise, the time which we have, and the scale of the problem, we have to scale down the ambition of the project as well. At the end of the project proposal period, we have made a very conservative model for our project – A robot follows a person wearing the fiducial marker in an open, obstacle-free environment, and we are glad to say that we have achieved this very basic goal and some more.

Over the course of the few weeks which we spent working on Stalkerbot, there arose numerous challenges, most of them were the result of our lack of deep understanding of ROS and robotic hardware. The two biggest challenge were the following:
The inconsistency of the camera and Aruco Detect was our first big obstacle. We followed the lab instruction on how to enable the camera on the robot, however, for our project, a custom exposure mode is needed to improve the performance of the camera, and it took us some time to figure out how the camera should exactly be correctly configurated. However, over the course of the entire development of Stalkerbot, Aruco Detect sometimes cannot pick up on the fiducial markers. Sometimes, it is unable to detect any markers altogether at certain specific camera settings.

The second challenge is the ros_tf library, and in general how orientation and pose function in the 3D world. At the beginning of the algorithm development, we did not have to deal with poses and orientations, but when we had to deal with them, we realized the poses and orientations we thought were aligned were all over the place. It turned out, that the axes used by Aruco Detect and ros_tf were different. The x axis in ros_tf is the y axis in Aruco Detect, for example. In addition, communicating with ros_tf requires a good understanding of the package as well as spatial orientation. There are many pitfalls, such as mixing the target frame and the child frame, mixing up the map frame and the odom frame.

Over the course of the development, Danbing worked on the basic follow algorithm, fiducial_filter, fiducial_interval, motion_detect, detect_interval, and sigint_catcher. Max worked on location_pub, move_base_follow and the integration of move_base_follow to make advanced_follow.