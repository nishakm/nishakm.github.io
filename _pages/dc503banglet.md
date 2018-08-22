---
layout: page
title: The DC503 Banglet for DefCon26
permalink: engineering/dc503banglet/
subtitle: An overview of the DC503 party badge as seen at DefCon 2018
---

## Some Background

Hi! My name is Nisha, and I made a party bangle for my friend, [Miki](https://twitter.com/thedawgcr8), to take with her to DefCon25. It was my first fully-formed electronics project and it posed some interesting challenges due to its unusual form factor. You can read about my experiences with that project [here]({{site.baseurl}}/engineering/defconbangle).

Soon after DefCon25, I was approached by [r00tkillah](https://twitter.com/r00tkillah) to make over a 100 of something similar for the DC503 party at DefCon26. The plan was to combine the power of the [BMD-300 SoC by Rigado](https://www.rigado.com/products/modules/bmd-300/) used in the [Wagon Badge](https://hackaday.com/2017/08/04/all-the-hardware-badges-of-def-con-25/#jp-carousel-267568) from the previous year with my Neopixel bangle form factor. We would call it "The Banglet" and it was going to be awesome.

## Features

In passive mode, the banglet's LEDs light up when detecting nearby Bluetooth devices. The number of LEDs that are lit correspond to the number of BT devices detected and their colors are based on each device's mac address.

![banglet in scan mode](https://pbs.twimg.com/media/DjT3Kg5VsAAS8bW.jpg)

The number of LEDs on the banglet is 12...but you can find the mac addresses of all of the detected devices by querying it. Yup, it has a shell :)

<img src="{{site.baseurl}}/assets/img/banglet_shell_500.png">

You can talk to the banglet using a Bluetooth to UART phone app (There's one called Bluetooth Terminal and another called Serial Bluetooth Terminal on the Android store. I am not an Apple user so I don't know of any equivalent for iPhones). We used Adafruit's [BlueFruit LE Connect](https://play.google.com/store/apps/details?id=com.adafruit.bluefruit.le.connect) app, mostly because we could control the terminating character at the end of every sent char array.

Here is a quick demo of how to connect to the banglet once you've installed the app.
<iframe width="560" height="315" src="https://www.youtube.com/embed/ctZMv4urpgQ" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

You can unlock different modes based on how many devices the banglet has detected...or you can read the code and find the different modes. 

Check out the [github project](https://github.com/pdxbadgers/2018-banglet) for code, schematic and board files and 3D design files.

## Development Platform

We started with the [Adafruit Bluefruit Feather](https://learn.adafruit.com/bluefruit-nrf52-feather-learning-guide/introduction) for everything from circuit design to the programmable firmware. The learning guide comes with a bunch of examples which we used as a starting point for enabling the Bluetooth LE features on the banglet. I pretty much copied pieces of the schematic used in the Feather and just replaced the module with the BMD-300. We wanted something small and non-cumbersome, so we chose to route out only one data pin. This made the final board small enough be somewhat inconspicuous.

The board is powered by a small 3V LiPo battery so it isn't too heavy and doesn't need an external battery pack to work. We also made it so you could charge the battery via USB. That may need some fiddling with the board to get it out of its (admittedly sloppily carved out) slot so a USB cable can be plugged in.

## 3D Design

The banglet's 3D design is based on an Adafruit project showcasing a [neopixel bracelet](https://learn.adafruit.com/neopixel-bracelet/overview), especially the idea to use the neodymium magnets to make the ends fasten together. There were a few restrictions we needed to work around though. The board we were using is bigger and thicker than the trinket so we needed to find some way of placing it horizontally rather than vertically. We chose to wedge it in and secure it using heat shrink. We also ended up wedging the battery that way. Some of the banglets don't have a hinge as the plastic is flexible, so the ends can just be pulled apart to get it out of your wrist.

We tried making smaller sized banglets but the slot for the board scaled down as well. The one on the demo video was one where I carved out space for the board on one of the smaller ones so I could wear it without it falling off. I do have tiny wrists :)

## Programming

We relied heavily on Adafruit's Arduino libraries for flashing the programmable soft device on the BMD-300 and program it. If you happen to have one of these you can actually reprogram it with your own features. There is some sample code on the github to get you started. The sketch that is on the banglet is called 'banglet.ino' but all the others work too. Simply put, the banglet is always scanning for devices regardless of whether you switch it to any of the blinky light modes. You can always ask it for a list of devices or the number of devices through the phone app.


Questions? You can tweet at me @nishakmr or @dc503 if you would like more information or help if you are hacking the banglet for yourself. This was a challenging (and expensive) badge to pull off and was certainly a brutal initiation for me into #badgelife. The DC503 gang are ridiculously talented though, and I am so glad I got to work with them on this project.
