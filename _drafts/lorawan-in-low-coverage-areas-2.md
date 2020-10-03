---
title: "LoraWAN in low coverage areas"
date: 2020-10-02T15:00:00-00:00
categories:
  - blog
  - Arduino
  - Hardware
  - LoraWAN
  - IoT
tags:
  - ESP32
  - TTN
  - SPIFFS
---

In the UK - and many other parts of the world - much of the LoraWAN infrastructure is being provided for free by generous members of the public (although more and more companies are also starting to see the benefits of low-cost wide area networks). That being said, LoraWAN coverage in the UK is still in the early stages of growth, and therefore ensuring a reliable data stream for your IoT application can be a considerable challenge.

I took on this challenge for a recent project, in which we were assigned the task of monitoring pollution (PM10 and PM2.5 specifically) on a large industrial site. The site had no existing LoraWAN infrastructure, so we had to install our own gateway, and the cellular backhaul we used to connect the gateway to the internet also suffered from connection dropouts.

This guide outlines the steps we went through to overcome these issues.

# Hardware

## Gateway
We initially made the mistake of using an entry level gateway for our application - [The Things Network Gateway](https://www.thethingsnetwork.org/docs/gateways/gateway). While the device is cheap and integrates seamlessly with TTN's own software backend (which is free), there really is no alternative for an industrial grade gateway, especially if want to communicate with sensors over 1km away. We eventually went for the [Kerlink Wirnet iStation](https://www.thethingsnetwork.org/docs/gateways/kerlink/istation/).

It is definitely worth assigning a good proportion of your project budget to the gateway(s), as this is really the linchpin of the entire network. Thankfully, one of the benefits of LoraWAN sensors is that they are typically very cheap compared to other types of sensors, so apportioning the budget this way makes sense in the grand scheme of things.

## Sensors
The majority of LoraWAN sensors boil down to 4 components - sensor, battery, MCU and LoraWAN transceiver. We ended up combining the MCU and transceiver components into a single PCB to simplify assembly of the device. The PCB also included clips for attaching the batteries, and a charging circuit so we could hook up a solar panel. We used a [Sensirion SPS30](https://www.sensirion.com/en/environmental-sensors/particulate-matter-sensors-pm25/) for our pollution sensor.

For the MCU we used an ESP32, as it can be programmed easily using the Arduino IDE, and has features such as deep-sleep which allow it to conserve battery life. The LoraWAN transceiver we chose - the RFM95 - is an all-in-one module which made it easy to add to the PCB. We also included a RTC module to facilitate precise timekeeping on the device (as the ESP32's internal RTC tends to drift over time)

# Software

## ESP32 firmware
On the most basic level, the job of the MCU is to take sensor values and send them over the air via the LoraWAN transceiver. To achieve this, the MCU must be able to communicate with both of these peripherals. We used [MCCI's arduino-lmic](https://github.com/mcci-catena/arduino-lmic) library to talk with our RFM95 module. This library gives you a full LoraWAN stack (a lot of Arduino libraries only do basic Lora) with full support for bidirectional data transfer - so you can send data to your sensors as well as receive data.








While going through some design improvements for our rover, I decided to implement a simpler method of turning the rover on and off. Here is what we ultimately want to achieve:

* Our roslaunch file should run automatically once the host PC boots up (launching ROS normally requires use of the command line)
* A single toggle switch should control the on/off state of both the PC and the motor controllers
* The PC should be given ample time to shutdown properly and not just have it’s ‘plug pulled’ when the toggle switch is opened

**By the way:** We are using a Jetson Xavier NX developer kit as our PC (with ROS Kinetic), and a pair of ODrives for our motor controllers. These instructions should still be relevant for similar hardware configurations, although the software steps may be quite different for a non-Linux setup.
{: .notice--info}

# ros_upstart

ros_upstart is a ROS package which configures a Linux daemon (background) service to run your chosen roslaunch file. The service starts automatically once the OS is booted, but can also be started and stopped manually just like any other daemon. [Here is the guide I used to configure ros_upstart.](https://roboticsbackend.com/make-ros-launch-start-on-boot-with-robot_upstart/){:target="_blank"}

One thing that might be important for you to consider are the permissions granted to the service. By default the service is run without a user specified, which may cause problems if your ROS nodes need to access hardware interfaces e.g. GPIO, UART. In Linux, use of these interfaces is typically controlled using groups. A group is given access to a particular interface via a udev rule, and users who belong to that group are able to use the interface.

The solution is to specify a user parameter – who is a member of all the necessary groups - while configuring ros_upstart:

``` 
rosrun robot_upstart install my_robot_bringup/launch/my_robot.launch --job my_robot_ros --symlink --user patrick
```

This ensures the target roslaunch file runs with the permissions of user "patrick". I also had to modify one of the files generated by ros_upstart (my_robot_ros-start in /usr/sbin) to get everything working, but I’d only recommend this if the prior steps are insufficient:

```
# Replace this line:
setuidgid patrick roslaunch $LAUNCH_FILENAME & PID=$!

# With this one:
envuidgid patrick roslaunch $LAUNCH_FILENAME & PID=$!
```

You can use the command `rosnode list` to verify that all your nodes have loaded successfully by the service.

# Switch detection

While ros_upstart takes care of the start-up procedure, we still need a way of turning the system off. As one of our primary aims is to control everything using a single toggle switch, we need some way of detecting the state of this switch and running the `shutdown` command when it is in the off state. Running this command ensures all services (including ROS) are stopped in a controlled manner, and protects against filesystem corruption.

In our rover the toggle switch is used to connect / disconnect the high side of a 24V battery, so the first obstacle we face is translating a 24V signal into a 3.3V signal compatible with the TTL level of our PC’s GPIO pins. We achieve this using an optocoupler ([a IS2801-1 from Isocom Components to be exact](https://datasheet.lcsc.com/szlcsc/Isocom-Components-IS2801-1_C89879.pdf){:target="_blank"}, although most optocouplers should be OK).

Optocouplers have the additional benefit of galvanic isolation, therefore there is no need for the battery negative and signal negative to be tied together, as would be the case for a normal level translation circuit. This, along with the fact we are using an isolated DC/DC converter to power our PC from the 24V battery, keeps high currents from the motor controllers away from the PC’s ground connections.

Detecting the switch state and initiating shut down can be achieved in a few lines of code within a ROS node:

```python
#!/usr/bin/env python

import Jetson.GPIO as GPIO
import rospy
import os

# initialise GPIO (switch signal connected to pin 37)
GPIO.setmode(GPIO.BOARD)
GPIO.setup(37, GPIO.IN)

# how many seconds the switch must be in the 'off' position before calling shutdown
shutdown_count = 3

def main():

	# node setup
	rospy.init_node('shutdown_input')
	r = rospy.Rate(1)

	counter = 0

	# ensure this node closes down if ROS exits
	while not rospy.is_shutdown():

		# switch in 'off' position
		if GPIO.input(37):

			counter += 1

		# switch in 'on' position
		else:

			counter = 0

		if counter >= shutdown_count:
			
			os.system("shutdown now -h")

		r.sleep()

	# release GPIO on exit
	GPIO.cleanup()

if __name__ == '__main__':

	main()						

```

# Delay off timer

There is one extra thing we need to ensure the shutdown function can work effectively. While the toggle switch is sufficient to power off the motor controllers and signal the PC to shutdown, there needs to a way to keep the PC powered while it is shutting down, as a toggle switch by it’s nature causes a break in the circuit. This can be achieved using a ‘delay off’ timer relay. The complete circuit, including the aforementioned optocoupler is shown below:

![](/assets/images/off_relay_circuit.png)

The delay timer here keeps the coil energised for a fixed interval once the switch has been opened (about 30 seconds in our case), allowing current to flow through the relay contacts and thus keep the DC/DC converter powered while the shutdown is completed. After this delay the coil de-energises and the DC/DC converter is switched off, preventing anything from draining the 24V battery while the rover is switched off. 

# Finishing up

With everything wired up and working, it’s just a case of adding the shutdown node to the list of nodes that are launched on startup via ros_upstart. It’s definitely worth testing this node independently before adding it to your roslaunch file, as you may end up with the PC shutting itself down instantly on startup, which could be an annoying problem to fix.

