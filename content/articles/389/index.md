---
title: "Wireless motion sensor"
date: '2020-05-03'
lastmod: '2022-06-27'
tags: ['Electronics', 'Projects', 'Arduino']
---

This is a wireless motion sensor that runs on rechargeable batteries. It can be deployed anywhere and sends a signal using an NRF24L01 wireless module whenever it detects motion.

![](5e78598c5e7fee69fd6e5745.jpg)

The device is composed of the following parts

* Arduino pro mini
* NRF24L01+ wireless transceiver
* HC-SR501 PIR motion sensor
* 14500 Lithium-ion battery
* LED + resistor
* Power switch
* 5.5mm x 2.1mm DC socket
* 2P JST-XH connector for battery charging

Those are laid out as follows

![](5eaf7dabe7dfb22ab52b55fb.png)

Here, simple protoboard has been used for the assembly.

![](5e796f9aad38483634f8697f.jpg)

The source code for the firmware running on the Arduino is available on [GitHub](https://github.com/maximemoreillon/wireless_motion_sensor).