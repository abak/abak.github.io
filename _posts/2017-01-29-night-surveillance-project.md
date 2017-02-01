---
layout: post
title: Night Surveillance
---

Time to start working on something new. The project I had in mind is a night 
surveillance system. This post describes the basic setup, required feature and 
optional features, along with useful links and ressources.

---

## Goals

The primary goal is to get a live video feed from inside a darkroom. This feed should be accessible from a device connected to the same LAN as the device. The secondary goals are as follows:

* get an audio feed in the same conditions
* have a notification system that could run a specific action or set of actions when:
  
  * motion is detected in the room
  * sound above a certain threshold is detected

* Get live feedback on the temperature and humidity in the room
* have a notification or run a specific action when temperature/humidity go below/above a set threshol

The primary goal would already provide some value, while the other ones are optional. In particular,
 getting a notification on my phone, or getting my [Jarvis](https://github.com/alexylem/jarvis)
  instance to tell me when things are happening are cool but not necessary as a first step 

## Hardware

After some minimal amount of digging, I settled on the following hardware:

* Raspberry Pi 3 model B: because the thing is cheap, available, widespread, and a super low footprint
 is not critical, so a Pi Zero is not necessary here.
* Raspberry Pi Camera NoIR V2: this thing is supposed to run in the dark so the lack of an IR cut 
filter is nice, the fact that colors will be wrong is not a concern.
* DHT22: this is a numerical sensor for both temprature and humidity. It provides its readings as a
 numerical signal, which is convenient, require minimal circuitry, and is super cheap while havingi
  a good enough range and accuracy.
* IR Leds: Preliminary testing of the camera rig showed that despite the absence of a IR cut filter,
in normal conditions, there is not enough light to get a clear image. Additional purchase of a couple IR Leds will be required to actually get a clear image at all times.

## Software

### [RPi_Cam_Web_Interface](https://github.com/silvanmelchior/RPi_Cam_Web_Interface)

This is an opensource component that already does most of the camera interfacing that will be needed:

* Video feed is served as a MJPG feed on a local webserver
* all camera controls are accessible from a web interface
* rudimentary motion detection is implemented in a native binary that can be queries on demand, and
 execute scripts on positive detection

In the future I may need to rewrite part of this for performance reasons, but I doubt it will the case.
A preliminary test showed that the latency is minimal, and the quality is good enough.

### DHT22 polling

Polling this sensor and serving the results over a small webserver will need to be done. I believe this
 will be a reasonably small. This component will need to:
 
 * poll the sensor repeatedly, with a configurable time period
 * log the measuremnet in a local or remote log file
 * serve the most current measurements, last N, or last X hours over a REST API
 
 THis sound like something easy enough for me to try out in Rust.
 
### Message sending
 
A number of services exist that can do the sort of things that i'm interested in:

* [Twilio](https://www.twilio.com/) for SMS sending
* [Instapush](https://instapush.im/) for push notification sending
* plain old curl'ing to send messages to another machine in the same network 


### Web Dashboard

Eventually a web dashboard sumarizing all of this data will be nice to have, this is something for which
 i have no idea so far, mostly because a simple HTML5 + Js page can functionally fit the bill, even if it ends up being ugly.


