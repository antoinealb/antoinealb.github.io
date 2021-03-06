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

* The master board.
    This board is responsible for coordinating the various motor control boards to create complex motions.
    The software is written in C/C++ too, it communicates via CAN too but it also talks to the PC via Ethernet.

* The application on the PC.
    This part is written in C++ and Python.
    It is made of a lot of different microservices communicating over an existing TCP middleware (ROS).
    This part will probably receive the most changes this year since most of the robot "brain" is here.

Currently the code is scattered in a lot of different repositories: according to my count, running the whole robot stack requires 32 repositories (including two that are more of operation / tooling related).
I have grouped those into some categories:

1. The top level repositories contain what could be described as one "project".
    In this group we have `cvra/motor-control-firmware`, `cvra/master-firmware`, `cvra/beacons`, etc.
    This is the group that integrates other repositories as submodules.

2. The external vendor repositories, such as our RTOS (`cvra/ChibiOS`), our IP stack (`cvra/lwIP`), etc.
    Those have generally been forked under our organization to apply custom patches if needed.

3. Custom made software libaries that are shared between different projects.
    This is currently almost only made of C code shared between the master board firmware and the motor control board firmware, such as our configuration tree implementation (`cvra/parameter`), the cyclic redundancy code (`cvra/crc`), etc.

4. Custom software parts used only by one project.
    This is mostly Python code, although some of the C repositories go in this category too.
    For example of that last category look at `cvra/korra-the-koordinator` or `cvra/odometry`.

I believe this huge amount of repositories creates a lot of different problems that reduce the development speed and quality.
Those problems can be divided between two categories that I will name Dev and Ops to reuse a modern taxonomy.

In the Dev category lies everything that slows down the creation of new functionalities because you cannot easily make changes spanning several repositories.
This creates an incentive to develop code in isolation, leading to integration hell.
It is also harder to integrate partial implementations of a functionality.

Scattered code is hard to refactor: function extraction, for example, might span several repositories.
Unused module will also slowly creep into the codebase, fading away in their repositories, with no one to delete them.

On the other hand the Ops problems are basically those which makes hard to bring changes from the programmer's laptop to the robot.
In this category having several repositories make code transfer via Git harder because you have to synchronize all dependencies first (and Git submodules suck at this).
Continuous integration also becomes harder since you can't know easily which part depends on what and test them together.


## problem ideas
The Python repositories also bring the additional problems of dependencies management.
All developers must use virtual environments to isolate their code from the system packages to avoid having a setup that no one can reproduce manually.
This problem can be reduced by using virtualenvs (easy) or Docker (much harder).


# Solution

As an experiment I suggest merging all the repositories containing software intended to run on the PC (with ROS) as a single repo.
External dependencies or custom stable libraries would still be installed via virtualenv (automated with a script).

This would allow some other features to be implemented:
* Travis CI integration to easily test pull requests.
* Easy deployment using `git push` although this is not the only way to do it (see Docker section).
* System-wide changes possible for easy experimentation.

I suggest not to merge together the firmware repositories written in C for now.
Currently the development workflow suffers much less from the problems explained above than the Python one.
Repository explosion is pretty limited and most submodules are either vendor code or shared libraries.
Build and testing complexity is already handled by our own custom tool called `packager.py` which can generate Makefiles from a dependency tree.
Deployment workflow is also much simpler and already solved: application binary is built on dev laptop, copied to the robot then flashed.
Finally, merging only code running on the embedded PC brings limits the risks associated with this kind of methodology changes.

## Counter arguments

###Con: Harder to open source / reuse*
Totally true. However:

two kinds of components:

1. The kind that might be reuesed by other projects.
2. The kind that won't.

The main robot application (Eurobot specific) clearly falls in the second category.
Some of the auxilliary libraries fall into the first (serial datagrams, operating systems, crc, etc.)
The motor boards, for example are also pretty standalone.

Why bother with making the second category easy to reuse ?
Make it easy to *use* first.

###Peudo-con: It will make code more tightly coupled

* Might be true
* Counterargument: easier to refactor, easier to review
* Cultural problems require cultural solutions: code review, etc.

###Current approach: design by interface / contract

* Totally better in theory.
* Doesn't take time into account
* Stuff doesn't work until both sides of the contract / interfaces are written
* Assumes contract is respected: not always the case
* Huge bias to like system engineering in the team: promotes paralysis by over-analysis.


# Deployment workflow

*TODO*

## Virtualenv deploys

*TODO*

A more lightweight approach is to use Git to copy the code to the robot and create virtual environment there to hold all pythons dependencies.
This approach leaks more dependencies than the Docker one but is also much simpler to put together.


## What about Docker?
If some of you don't know what Docker is, let me quote the official Docker website:

> Docker containers wrap up a piece of software in a complete filesystem that contains everything it needs to run: code, runtime, system tools, system libraries – anything you can install on a server.
> This guarantees that it will always run the same, regardless of the environment it is running in.

Basically it is a kind of lightweight virtual machines called containers, based on a Linux technology called LXC.
Although it requires Linux to run, you can develop on any platform using a small virtual machine called boot2docker.
For a wider introduction to Docker, I suggest you watch the talk [Demistifying Docker by Andrew T. Baker](https://www.youtube.com/watch?v=GVVtR_hrdKI).

In our situation Docker could be used like this: 1. The application is packaged on the developper laptop along with all its dependencies. 2. The tests are run in the packaged app and deployement aborts if the tests fail. 3. The packaged application is exported and copied to the robot. 4. The packaged application is run on the robot.
This looks a lot like our workflow for the microcontrollers programs except that instead of binaries we copy a container image.

I think Docker is a really interesting technology and that I definitely need to take some time to put it in a serious project.
However it only solves our operation problems and not in a very lightweight way: a simple docker image with python weights around 50M (with a stripped down image).

* Could replace virtualenvs to solve dependency leaks
* You are essentially deploying a big binary of your application so things like architecture compatibility between build machine and deploy target are now a problem.

Finally, Docker is a new technology and no team member is familiar with it.


