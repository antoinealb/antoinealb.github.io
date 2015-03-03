---
title: Installing Let's chat on Ubuntu 12.04
layout: post
categories: System administration
tags: linux web
---
Recently we started using IRC to communicate between colleagues.
IRC was fine, except that the user experience isn't so good compared to modern chat clients.
You lack notifications, inline images and it is not specially user friendly.
A nice alternative targeting entreprises was [https://campfirenow.com/](Campfire), but sadly it is a closed source product.
I forgot about it for a while, then stumbled upon [http://sdelements.github.io/lets-chat/](Let's chat), which is described by its authors as

> Let's Chat is a persistent messaging application that runs on Node.js and MongoDB. It's designed to be easily deployable and fits well with small, intimate teams.
>
> It's free (MIT licensed) and ships with killer features such as LDAP/Kerberos authentication, a REST-like API and XMPP support.
>
> Let's Chat is a side-project of the development team at Security Compass. (A real life 10% time project!)

Plus it is supported (well sort of) by Hubot, which is our toy of choice.
So after toying with it using the Vagrant image available on the website, I decided to start a new VPS and install it for a real world test.

# Installing dependencies
## NodeJS
Installing Node.js is really easy, as there is a PPA (Personal Package Archive) available.
We will also need the `build-essential` package, since some NPM modules need to be compiled.
{% highlight bash %}
# Adds the Node.js PPA
curl -sL https://deb.nodesource.com/setup | sudo bash -
sudo apt-get install nodejs build-essential
{% endhighlight %}

## MongoDB
There is a package available:
{% highlight bash %}
sudo apt-get install mongodb-org
{% endhighlight %}

## Python
Python 2.7 is already installed with Ubuntu 12.04, there is no need to install anything.

# Installing Let's Chat
For security reason, we will run Let's Chat as its own user, with restricted privileges.
First create this user:
{% highlight bash %}
adduser node
{% endhighlight %}

Then as the `node` user, run the following commands to install Let's Chat:

{% highlight bash %}
https://github.com/sdelements/lets-chat
cd lets-chat
npm install
{% endhighlight %}

## Configuration
Let's chat settings are stored in `settings.yaml`.
There is a sample available and here is mine, slightly adapted from it:

{% highlight yaml %}
# See defaults.yml for all available options

env: production

http:
  enable: true
  host: '0.0.0.0'
  port: 5000

# Allow any file upload
files:
  enable: true
  restrictTypes: true

auth:
  local:
    salt: SALTSARESECRET

secrets:
  cookie: ITISASALTTOO
{% endhighlight %}

You can now run `npm start` and point your browser on your server (port 5000 for now, we will fix it in the next step).

# Serving the site on port 80
Since we are running the web application with a non priviledged user, as you should be, we cannot bind to port 80.
Recent Linux kernels can change this using a per-program authorization
Since our program is interpreted by the `node` executable, it would mean any Node.js application could bind to lower ports, which is not ideal.

The solution that I choose is to run the server on a non priviledged port and redirect traffic coming on port 80 to this port using Linux's builtin `iptables`.

To do this, simply run the following commands as root:
{% highlight bash %}
iptables -A PREROUTING -t nat -p tcp --dport 80 -j REDIRECT --to-port 5000
iptables -t nat -I OUTPUT -p tcp -d 127.0.0.1 --dport 80 -j REDIRECT --to-ports 5000
{% endhighlight %}

# Running the site on server boot
Running the site with `npm start` is great, but it would be better if it is started automatically at boot.
To do that we can use Ubuntu's Upstart init system.
It allows us to write job descriptions and when they should be run.
We can also specify a user and a directory to run the script from.

Put the following in `/etc/init/lets-chat.conf`:
{% highlight conf %}
description "Let's chat application server"
author "Antoine Albertelli"

# Taken from nginx job definition
start on (filesystem and net-device-up IFACE=lo)
stop on runlevel [!2345]

setuid node
setgid node

chdir /home/node/lets-chat

exec npm start
{% endhighlight %}

We also need to make our iptables rules persistent on every reboot.
To do this put the `iptables` commands above in `/etc/rc.local`, before the last line.

Tadaa ! Now you can run `sudo initctl start lets-chat` and try to connect to your host on port 80.
Your service will be started automatically on every boot.


# TODO
* Support for HTTPS (using self signed certificate).
* Add automatic respawn.


# References
* Let's chat's README
* Node.js website
* MongoDB website
* iptables redirection: http://www.catonmat.net/blog/unprivileged-programs-privileged-ports/
* Ubuntu's Upstart cookbook: http://upstart.ubuntu.com/cookbook/

