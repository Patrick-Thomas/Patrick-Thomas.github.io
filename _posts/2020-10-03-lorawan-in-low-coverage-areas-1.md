---
title: "LoRaWAN in low coverage areas - Part 1: Hardware"
date: 2020-10-03T15:00:00-00:00
categories:
  - blog
  - Hardware
  - LoRaWAN
  - IoT
tags:
  - ESP32
  - RFM95
  - TTN
---

In the UK - and many other parts of the world - much of the LoRaWAN infrastructure is being provided for free by generous members of the public (although more and more companies are also starting to see the benefits of low-cost wide area networks). LoRaWAN coverage in the UK is still in the early stages of growth, and therefore ensuring a reliable data stream for your IoT application can be a considerable challenge.

I took on this challenge for a recent project, in which we were assigned the task of monitoring pollution (PM10 and PM2.5 specifically) on a large industrial site. The site had no existing LoRaWAN infrastructure, so we had to install our own gateway, and the cellular backhaul we used to connect the gateway to the internet also suffered from connection dropouts.

This multi-part guide outlines the steps we took to build a reliable LoRaWAN sensor network on this site. Hopefully some of what you read here will be useful for your next sensor project.

# Hardware

## Gateway
We initially made the mistake of using an entry level gateway for our application - [The Things Network Gateway](https://www.thethingsnetwork.org/docs/gateways/gateway). While the device is cheap and integrates seamlessly with TTN's own software backend (which is free), there really is no alternative for an industrial grade gateway, especially if want to communicate with sensors over kilometre-scale distances. We eventually swapped our initial gateway for a [Kerlink Wirnet iStation](https://www.thethingsnetwork.org/docs/gateways/kerlink/istation/), and that has saved us quite a few connectivity headaches.

It is definitely worth assigning a good proportion of your project budget to the gateway(s), as this is really the linchpin of the entire network. Thankfully, one of the benefits of LoRaWAN sensors is that they are typically very cheap compared to other types of sensors, so apportioning the budget this way makes sense in the grand scheme of things.

## Sensors
The majority of LoRaWAN sensors boil down to 4 components - sensor, battery, MCU and LoRaWAN transceiver. We ended up combining the MCU and transceiver components onto a single PCB to simplify assembly of the device. The PCB also included clips for attaching the batteries, and a charging circuit so we could hook up a solar panel. We used a [Sensirion SPS30](https://www.sensirion.com/en/environmental-sensors/particulate-matter-sensors-pm25/) for our pollution sensor.

For the MCU we used an ESP32, as it can be programmed easily using the Arduino IDE, and has features such as deep-sleep which is essential for conserving battery life. The LoRaWAN transceiver we chose - the RFM95 - is an all-in-one module which made it easy to add to the PCB. We also included a RTC module to facilitate precise timekeeping on the device (as the ESP32's internal RTC tends to drift over time).

# Coming up in Part 2: Firmware
I will go through the process of writing firmware for the sensor using the Arduino IDE.






