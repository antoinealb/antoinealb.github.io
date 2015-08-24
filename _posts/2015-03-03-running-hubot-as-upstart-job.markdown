---
title: Running Github's Hubot as Upstart job
layout: post
categories:
    - System administration
tags: linux
---
In this post I just want to explain how to run [Github's Hubot](https://hubot.github.com/) automatically using Ubuntu.

You should read my previous [article](http://antoinealb.net/system/administration/2015/03/03/installing-lets-chat-on-ubuntu-precise.html) as I won't repeat the whole process about user creation, Node.js install, etc. here.

Hubot is really easy to install.
The only hard part is launching it automatically at boot or after a crash.
I used to do this manually but you can have the same result by putting the following in `/etc/init/hubot.conf`:

{% highlight nginx %}
description "Simple Hubot instance"
author "Antoine Albertelli"

start on (filesystem and net-device-up IFACE=lo)
stop on runlevel [!2345]

setuid node
setgid node

# Those are settings specific to my adapter, configure your environment accordingly
env HUBOT_LCB_TOKEN=APITOKEN
env HUBOT_LCB_ROOMS=room1,room2

# Path to hubot installation
chdir /home/node/zivibot

exec bin/hubot --adapter lets-chat --name "zivibot"
{% endhighlight %}




