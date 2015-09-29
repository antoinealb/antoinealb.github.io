---
layout: post
title: Why the CVRA should switch to an über repository
categories:
    - System administration
---

# Description of the setup
* One system, two communication protocols, 3 completely different softwares
* Requirement: build must work without internet access.
    Prior download of dependencies is acceptable though.


# The problems
* Implicit dependencies
* Hard to deploy
* Hard to run
* Impossible to do CI of the entire system

# proposed solution: Put all the PC shit in one repository
* Easy to deploy: `git push robot master`.
* Deploy is fun again: very few external dependencies
    - Still got some deps, but less.
* Global testing becomes feasible.
    - Including on Travis CI



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
