---
layout: post
title: Why the CVRA should switch to an über repository
author: Antoine Albertelli
categories:
    - System administration
---

# Description of the setup
At the CVRA our robotics stack has really jumped in complexity in 2015.
We switched from a centralized system running one binary to a network of independent systems running different code.
Currently there is 3 software types:

* The motor control boards.
    The code is written in C/C++ and communicates with other systems via CAN.
    It is pretty self-contained, only receiving simple commands from the other nodes.
    It has a lot of external reuse value since it provides basic, isolated functionality over a standard bus.

* The master board.
    This board is responsible for coordinating the various motor control boards to create complex motions.
    The software is written in C/C++ too, it communicates via CAN too but it also talks to the PC via Ethernet.
    It is not really self-contained as it has a lot of interactions with the application, but it might be reusable for similar robots.

* The application on the PC.
    This part is written in C++ and Python.
    It is made of a lot of different microservices communicating over an existing TCP middleware (ROS).
    This part will probably receive the most changes this year since most of the robot "brain" is here.
    It has almost no external re-use value, given that it is custom made for our problem.

Currently the code is scattered in a lot of different repositories: according to my count, running the whole robot stack requires 32 repositories (including two that are more of operation / tooling related).
I have grouped those into some categories:

1. The top level repositories contain what could be described as one "project".
    In this group we have `cvra/motor-control-firmware`, `cvra/master-firmware`, `cvra/beacons`, etc.

2. The external vendor repositories, such as our RTOS (`cvra/ChibiOS`), our IP stack (`cvra/lwIP`), etc.
    Those have generally been forked under our organization to apply custom patches if needed.

3. Custom made software libaries that are shared between different projects.
    This is currently almost only made of C code shared between the master board firmware and the motor control board firmware, such as our configuration tree implementation (`cvra/parameter`), the cyclic redundancy code (`cvra/crc`), etc.

4. Custom software parts used only by one project.
    This is mostly Python code, although some of the C repositories go in this category too.
    For example of that last category look at `cvra/korra-the-koordinator` or `cvra/odometry`.

## problems
* Hard to deploy
* Hard to run because no centralized system, everybody does stuff their own way.
* Hard to refactor or remove unused parts
* Impossible to partially implement a feature to quickly prototype an idea.
    Waterfall anyone ?
* Hard to do CI of the entire system when changing one part.
    Because how do you know that something changed ?

two different sets of problems ? Deploy and develop

# Python subsystems additional problems
* Implicit dependencies

# C subsystems (motor board & master)
* Easier to deploy: copy the binary to embedded PC and flash it to microcontrollers.
* Easier to build because `packager.py`.
* Workflow mostly OK so we won't talk about it today.

# proposed solution: Put all the PC shit in one repository
* Easy to deploy: `git push robot master`.
* Deploy is fun again: very few external dependencies
    - Still got some deps, but less.
* Global testing becomes feasible.
    - Including on Travis CI
* Reduces turnaround time for system wide changes.
* Also makes it easier to review changes to the whole system


# Con: Harder to open source / reuse
Totally true. However:

two kinds of components:

1. The kind that might be reuesed by other projects.
2. The kind that won't.

The main robot application (Eurobot specific) clearly falls in the second category.
Some of the auxilliary libraries fall into the first (serial datagrams, operating systems, crc, etc.)
The motor boards, for example are also pretty standalone.

Why bother with making the second category easy to reuse ?
Make it easy to *use* first.

# Peudo-con: It will make code more tightly coupled
* Might be true
* Counterargument: easier to refactor, easier to review
* Cultural problems require cultural solutions: code review, etc.

# Current approach: design by interface / contract
* Totally better in theory.
* Doesn't take time into account
* Stuff doesn't work until both sides of the contract / interfaces are written
* Assumes contract is respected: not always the case
* Huge bias to like system engineering in the team: promotes paralysis by over-analysis.

# Y U NO DOCKER
* Define docker
* Docker is only a part of a solution
* Could replace virtualenvs to solve dependency leaks
* Also solves (in a very ugly way) the deployment problem: prepare the docker container, ship it to the robot and run it there.
    Will take a ton of time.
* Doesn't solve other the main aspects of the problem
* You are essentially deploying a big binary of your application so things like architecture compatibility between build machine and deploy target are now a problem.
* Might bring its own set of problems
* Definitely something to keep in mind.
    Plus I think it is cool tech.
