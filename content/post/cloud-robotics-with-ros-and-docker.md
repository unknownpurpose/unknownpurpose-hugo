---
title: "Cloud Robotics With ROS and Docker"
date: 2017-11-01T09:52:01Z

---
Here at the institute we're busy building our own take on cloud robotics, about which you can read more [here]. For this we need an approach that scales well to multiple subsystems, with differing OS requirements – from remote data processing on GPU accelerated hardware, to embedded motor control. To this end we've settled on a combination of the open-source Robot Operating System ROS, and an overlay network (enabled using Docker Swarm) to link each subcomponent into a single 'distributed robot'.

(If you plan to follow along, this assumes you're running on a Mac. However, all of the below should also work on Windows and Linux with minimal/no modification.)

## Creating the Docker swarm and overlay network
We first want to create a new Docker VM on our local machine, on which to run the Docker Swarm manager, ROS Core (in a Docker process), and set up the overlay network. We are assuming Virtualbox as the VM Management software.

(Something to note at this stage: if you're on Mac or Windows you can't just use your primary machine as the swarm manager because Docker for [Mac|Windows] doesn't yet allow that, throwing inexplicable errors instead.)

So with that in mind…

```
docker-machine create --driver virtualbox bot
docker-machine stop bot
```

Our docker-machine can't access the local network directly by default, so you now need to open the Virtualbox GUI and tweak the settings on your newly created VM. Add a new 'bridged' network adapter, so the VM will have an IP address on your local network when you start it up again. Then:

```
docker-machine start bot
eval $(docker-machine env bot)
```

For the next step – setting up our Docker Swarm – we need an IP address for our VM which is accessible by all the machines joining our Swarm. Use `docker-machine ssh bot ifconfig` to find the IP address on the local network, then  `docker swarm init --advertise-addr <reachable IP>`. This will return you a `docker swarm join <...>` command which you can then run on elsewhere on your network. The final part of the Swarm setup is the creation of an /attachable/ overlay sub-network, to which we can connect our ROS-enabled Dockers:

```
docker network create \
  --driver overlay \
  --subnet 10.0.9.0/24 \
  --attachable \
botnet
```

### Meanwhile elsewhere on the network...
Connect another Docker-enabled machine to the Swarm. (We're using a Beaglebone Black.) Copy-paste the `docker swarm join` command from earlier. If you've lost it somewhere, or need to hook up another computer to the swarm, simply run `docker swarm join-token worker` to get another one.

Congratulations you now have a functioning overlay network!

### Starting up ROS
For the next steps we need a ROS-enabled base image. You could use the ros-tutorials image, from the 'ROS meets Docker' tutorial here: https://hub.docker.com/_/ros/ and pick up on their 'Deployment example' over there, swapping out their local network 'foo' for the overlay network 'botnet' you just created.  

One thing to note is that if you're planning on creating a ros-enabled RasPi or Beaglebone Black, these tutorials will /almost/ work, but you need to base your ros-tutorials image off an ARM-enabled "ROS meets Debian" image instead, e.g. https://hub.docker.com/r/pablogn/rpi-ros-core-indigo/.

Alternatively… if you're feeling adventurous (and have access to a set of Dynamixels, plus a USB2AX) check out our introduction to Dynamixel + Docker + Beaglebone Black.

### Take home points
The main take-home here is that we now have a Docker Swarm manager running, and an attachable overlay network up to which we can hook new Docker instances, in such a way that they behave as if they are all on the same network even though they may be running in completely different places. Here we set up our Swarm manager locally via Docker Machine, then made it accessible across the local network. However we could have set up our Swarm manager remotely (e.g. AWS) if we chose to. You could also make your local Swarm manager available remotely by enabling port-forwarding on your router and routing traffic on the relevant ports (see here: [Getting started with swarm mode | Docker Documentation](https://docs.docker.com/engine/swarm/swarm-tutorial/#open-protocols-and-ports-between-the-hosts))
