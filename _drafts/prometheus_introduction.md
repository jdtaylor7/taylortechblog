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

{% include image.html path="prometheus_demo.gif" path-detail="prometheus_demo.gif" alt="prometheus demo gif" %}

As mentioned on the Github page, this demo shows Prometheus receiving data from
a “drone” (just an Arduino in this case) and plotting that data while also using
it to position the drone model visually. The data in question is just randomized
noise data about (0, 0) in the zx-plane, 2 on the y-axis, and (0, 0, 0) for
roll/pitch/yaw.

### Technologies

Prometheus is written in C++17 on top of raw OpenGL. Some additional third party
libraries support the graphics side of the codebase, which can be found in the
`third_party` directory of the repository. CMake is the build system of choice.

At the moment the tool is supported on Arch Linux and Ubuntu Focal (20.04 LTS).
A CircleCI pipeline tests compatibility with these systems via associated
Docker images.

### Dependencies

As mentioned, a number of dependencies are included in the project, mostly in
support of the graphical functionality. Of note are GLFW for windowing,
DearImgui for user interface, and Libserial for USB support.

All dependencies that need to be installed manually are detailed in the Github
repository's [README](https://github.com/jdtaylor7/prometheus#dependencies).

As a side note, I also have a dependency on a small bounded buffer library that
I wrote recently. It provides a variety of interfaces, is thread-safe, and is
written in C++17. If this sounds interesting to you, the repository is located
[here](https://github.com/jdtaylor7/bounded_buffer). Parts
[1](https://www.taylortechblog.com/posts/cpp-bounded-buffer-1) and
[2](https://www.taylortechblog.com/posts/cpp-bounded-buffer-2) of the blog post
I wrote about it are available as well.

### Interface

The 3D environment is fully navigable, allowing the user to view the model and
room from any angle. There are also a set of interfaces along the left and right
sides of the screen.

{% include image.html path="prometheus_screenshot.png" path-detail="prometheus_screenshot.png" alt="prometheus screenshot" %}

The panels along the left side of the screen display info to the user.

* FPS: Number of frames per second, locked at 60fps
* Drone data: The current position and orientation values being read from the
connected drone system, along with plots of those values over time
* Camera data: Position and orientation data for the camera, mostly for debugging
and repositioning purposes

The panels along the right are for interacting with Prometheus.

* Application mode: Choosing how to run the tool. Currently there are just two
modes, telemetry and edit
    * Telemetry mode: Scan for, connect to, and read from serial devices
    * Edit mode: Move the camera around the scene
* Controls: Changes depending on the application mode. In telemetry mode, allows
you to see available devices, connect to one, and ensure you are reading from
one

The interface will grow in scope as simulation capabilities and other features
are added to Prometheus.

### Graphics

I value visual representation of data quite highly. If I can hold the drone in
my hand, rotate it, and see that movement matched in a graphical environment, I
can much more easily check the validity of the data pipeline, the fidelity of
the sensors, the sensor fusion control algorithm, and more. Thus, this is a core
requirement of Prometheus.

To achieve this, I decided to implement the graphical part of the tool in
OpenGL. Since I had not touched OpenGL before, I leaned heavily on
[LearnOpenGL](https://learnopengl.com/), which is an excellent tutorial provided
by Joey de Vries. If you are interested in learning OpenGL yourself, I would
highly recommend it as a resource.

Since a certain level of graphical fidelity also lends an air of quality and
professionalism to a piece of software, I chose to implement not just basic
shape and texture rendering, but also model loading and rudimentary lighting
(the small blue cube is a light source, if that wasn't clear). I am quite
satisfied with the result of these OpenGL endeavors.

### IO

As mentioned in the previous section, Prometheus must be able to receive data
from a drone system. I have chosen to implement this communication interface in
hardware as a custom-designed transmitter/receiver pair of PCBs. The transmitter
is located on the drone itself, connected to the drone's flight computer. The
receiver (the ground station) connects to a desktop via USB.

For simplicity's sake, the ground station imitates a serial device over USB, in
the same way that Arduino boards do. Whenever this board receives a telemetry
packet from the drone, it will forward that data on to Prometheus via the
desktop.

In the demo at the beginning of this post, the data streaming in is being
generated from an Arduino UNO. For testing purposes, UNO boards can easily
substitute the custom ground station hardware that I am building.

Serial communication was handled with the excellent
[LibSerial](https://github.com/crayzeewulf/libserial/) library. While the
connections provided by the library are synchronous, Prometheus has no real need
for asynchronous reads/writes so this works just fine for our purposes.

### Other Interesting Features

While not really core to the project, some of these smaller features may be
interesting to some. I may write separate posts about some of them in the
future.

* Logging
* Compatibility testing with CircleCI and Docker
* Creating high-quality GIFs in Linux

### Conclusion

Prometheus is officially up and running and can be used to debug control
algorithms, which was my goal for this release of the tool. Development and
testing of control algorithms can now begin in earnest.

As always, suggestions and feedback are welcome!
