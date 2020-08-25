---
layout: post
title: "Introducing Prometheus"
description: "Drone Simulation and Monitoring Application"
tags: [cpp, drone, opengl, prometheus]
---

In my last post, I introduced a hobby drone project I have been working on. In
that post I laid out the scope and goals of the project as whole, without going
into many details about any one component. This post will start discussing the
first major portion of the project:
[Prometheus](https://github.com/jdtaylor7/prometheus). Prometheus is a desktop
application made primarily to streamline the process of building the drone. In
its final iteration, it will be a multifaceted tool which can:

1. Simulate a drone system
2. Refine a drone system’s control algorithm
3. Monitor a running drone system in real-time

As of the tool’s most recent v0.2.0 release, Prometheus is capable of the third
point above. While being provided with a stream of real-time data (position and
orientation in the time domain), Prometheus visually represents the state of the
drone in a 3D environment. A short demo of the tool is below.

{% include image.html path="prometheus_demo.gif" path-detail="prometheus_demo.gif" alt="prometheus demo" %}

As mentioned on the Github page, this demo shows Prometheus receiving data from
a “drone” (just an Arduino in this case) and plotting that data while also using
it to position the drone model visually. The data in question is just randomized
noise data about (0, 0) in the zx-plane, 2 on the y-axis, and (0, 0, 0) for
roll/pitch/yaw.

### Technologies

Prometheus is written in C++17 on top of raw OpenGL. Some additional third party
libraries support the graphics side of the codebase, which can be found in the
`third_party` directory of the repository. CMake is the build system of choice.

At the moment the tool is supported on Arch/Manjaro Linux, and support for
Ubuntu is planned. A continuous integration pipeline is also on the way, which
will support unit tests via Google Test.

### Dependencies

As mentioned, a number of dependencies are included in the project, mostly in
support of the graphical functionality. Of note are GLFW for windowing,
DearImgui for user interface, and Libserial for USB support.

### Interface

The 3D environment is fully navigable, allowing the user to view the model from
any angle.


### Graphics

Cover this briefly

### IO

Mention telemetry pipeline and ground station
