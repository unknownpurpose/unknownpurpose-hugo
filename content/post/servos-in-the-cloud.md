---
title: "Servos in the Cloud"
description: "Dynamixel, meet Docker"
date: 2017-11-01T09:55:31Z
---
If you followed along with our [introduction](/post/cloud-robotics-with-ros-and-docker) to Docker, ROS, and overlay networking you'll have a couple of machines (at least) in a Swarm, and an overlay network called `botnet` which is all ready to go.

For the next part, we're going to hook up a Beaglebone Black with four Dynamixels attached via a USB2AX controller, then control it remotely. Whilst this is pretty specific requirements-wise, these instructions will also generalise fairly well to other platforms – Raspberry Pi for instance – with only a bit of tinkering. So…

First up, clone and prepare the tentacle-os repository on each of your Swarm-enabled machines:

```
git clone http://github.com/unknownpurpose/tentacle-os
cd tentacle-os
git submodule update --init
chmod +x rep.sh
```

This repo contains two dockerfiles – one for the Beaglebone and another for desktop linux.

To build and launch `roscore`, the ROS central comms server, run the following on a Swarm-enabled desktop linux machine (or via docker-machine on a Virtualbox instance if you're running on Mac or Windows):

```
docker build -f deploy/desktop/Dockerfile -t fhtagn:base .
docker run -it --rm -d \
    --net botnet \
    --name master \
    fhtagn:base \
    roscore
```

To then build and launch TentacleOS packages on the Swarm-enabled Beaglebone, run the following:

```
docker build -f deploy/beaglebone/Dockerfile -t fhtagn:base .
docker run -d \
    --net botnet \
    --device=/dev/ttyACM0 \
    --name servo \
    --env ROS_MASTER_URI=http://master:11311 \
    fhtagn:base \
    roslaunch tentacle_control tentacle_controller.launch
```

The main things to note here is that we're passing through a reference to the USB2AX device, accessible at /dev/ttyACM0 by default, and setting ROS_MASTER_URI to point at our recently launched roscore.

From anywhere on our Swarm network we can now control the servo remotely, e.g.

```
docker run -it --rm -d \
    --net botnet \
    --env ROS_MASTER_URI=http://master:11311 \
    fhtagn:base \
    rostopic pub -1 /tilt_controller/command std_msgs/Float64 -- 1.5
```
