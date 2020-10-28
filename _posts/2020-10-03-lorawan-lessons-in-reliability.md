---
title: "LoRaWAN - lessons in reliability"
date: 2020-10-03T15:00:00-00:00
categories:
  - blog
  - Hardware
  - LoRaWAN
  - IoT
tags:
  - Kerlink
  - TTN
---

In the UK - and many other parts of the world - much of the LoRaWAN infrastructure is being provided for free by generous members of the public (although more and more companies are also starting to see the benefits of low-cost wide area networks). LoRaWAN coverage in the UK is still in the early stages of growth, and therefore ensuring a reliable data stream for your IoT application can be a considerable challenge.

I took on this challenge for a recent project, in which we were assigned the task of monitoring pollution (PM10 and PM2.5 specifically) on a large industrial site. The site had no existing LoRaWAN infrastructure, so we had to install our own gateway, and the cellular backhaul we used to connect the gateway to the internet also suffered from connection dropouts.

Here are some valuable lessons we learned while setting up a LoRaWAN sensor network on this site. Hopefully some of what you read here will be useful for your next sensor project.

# 1. Don't skimp on the gateway

We initially made the mistake of using an entry level gateway for our installation - [The Things Network Gateway](https://www.thethingsnetwork.org/docs/gateways/gateway). While the device is cheap and integrates seamlessly with TTN's own software backend (which is free), there really is no alternative for an industrial grade gateway, especially if you want to communicate with sensors over kilometre-scale distances. We eventually swapped our initial gateway for a [Kerlink Wirnet iStation](https://www.thethingsnetwork.org/docs/gateways/kerlink/istation/), and that saved us quite a few connectivity headaches.

A significant part of your project's budget should go towards the gateway(s), as this is really the linchpin of the entire network. Thankfully, one of the benefits of LoRaWAN sensors is that they are typically very cheap compared to other types of sensors, so apportioning the budget this way makes sense in the grand scheme of things.

# 2. Ensure you can reset the gateway remotely

There's a reason the old adage of 'have you tried turning it off and on again' is still in common parlance to this day - it actually works. Once you've setup your gateway and left it in a working state, you can pretty much guarantee it will eventually run into some problem. If - like us - you are monitoring a site which is halfway across the country from where you normally live and work, you need some way of reseting the gateway remotely. You could always ask someone working on the site to do it for you, but this is not ideal.

We encountered this problem with both our Things Network Gateway, and the Kerlink Wirnet iStation we used to replace it. Kerlink offer their own software backend called Wanesy, which allows you to SSH in to the gateway and perform a software reset, something which isn't possible on TTN. With TTN your only option is a hardware reset, which can be achieved using a GSM switch on the gateway's power input.

# 3. Store data on sensors if possible

You should always assume there will be times when your sensors are unable to communicate with your gateways. This is especially true if your sensors are mobile, but even for static sensors there are many ways signal dropout can occur. In these situations it is helpful to store data onboard the sensor so an attempt can be made later to resend it. If you are using a prebuilt sensor then such functionality might be off limits, but if you are building custom sensors then it can as simple as making use of some non-volatile memory. In our case we used ESP32 based sensors which have onboard persistant memory in the form of SPIFFS. 

Given LoRaWAN's inherent limitations on duty cycle and bandwidth, you may found it difficult to stockpile and send a high volume of data. However, if you optimise the structure of each sample at the byte level, then it is possible to pack 10's or 100's of samples into a single message. To achieve such optimisation you will probably have to forego sending messages in JSON format, as this adds a lot of byte overhead. A message in raw byte format will be a bit trickier to decode once it's received, but this is worth it if you want to limit the number of gaps in your data.

# Conclusion

Can you think of any other things you would add to this list? Please let me know in the comments below.










