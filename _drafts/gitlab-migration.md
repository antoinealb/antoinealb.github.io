---
layout: post
title: Migrating Gitlab server
categories:
    - System Administration
---

Ubuntu 12.04 is getting near its end of life date, which means my current Gitlab server won't be receiving any updates after April 2017.
I also want to use Gitlab CI to run continuous integration builds of my projects.
I plan to use the Docker runner for this, which prevents me from doing it on my current VPS (it uses OpenVZ, which means I cannot change the kernel to use a Docker-enabled one).
In the process I will also migrate away from the source installation of Gitlab to use the Omnibus packages, which are the recommended way of installation now.

#Picking up a VPS and an OS
For this project I decided to go with a Kimsufi server, having a good experience with them at Wise Robotics.
I picked a relatively small server, which should be enough for my needs.

For the Linux distribution I had to choose between Ubuntu LTS (14.04 or 16.04), Debian (7 or 8) or CentOS (6 or 7).
I decided to stay with an Ubuntu, as that was what I was familiar with and I wanted my server to be as maintenance free as possible.
I went with an Ubuntu 14.04 (Trusty Tahr) as it will still be supported for 3 years (long enough for me) and is more battle tested.

I ordered the server, asked for a Trusty install and went on to perform basic configuration...

#First steps: security lockdown
*Note:* I am not an infosec professional.
    I am only trying to replicate what I think the good practices are.
    Take my words with a grain of salt.

The first step I take on any server I am charged with setting up are a set of basic configuration tweaks intended to protect me against basic attacks:

1. Create a user account and allow it to use `sudo` to perform administrative tasks.
2. Disable root login via SSH.
3. Setup SSH public key based authentication and disable password-based login.
4. Add a firewall to whitelist incoming traffic to ports 22 (SSH), 80 (HTTP) and 443 (HTTPS).
5. Turn on automated security updates

##Setting up a user account

To create an administrator user called `antoine`, use the following commands as root:

{% highlight bash %}
adduser antoine
# -a means append, -G means groups
usermod -aG sudo antoine

# Checks that the user can do administrative tasks
su antoine
sudo whoami # should print "root"
{% endhighlight %}

The following steps will assume that you are logged in using your newly created account.

## Disabling SSH root login

Open `/etc/ssh/sshd_config`, search for the following line:

{% highlight config %}
PermitRootLogin yes
{% endhighlight %}

and replace it by:

{% highlight config %}
PermitRootLogin no
{% endhighlight %}

## Using public key authentication

First if you don't have an SSH key, generate one with `ssh-keygen -t rsa`.
Setting up a passphrase is a good idea, as it protects you in case your key gets compromised.

You can then copy the **public** key file to your server:

{% highlight bash %}
# On the server
mkdir -p ~/.ssh/

# On your local machine (replace the server name with yours)
scp ~/.ssh/id_rsa.pub antoine@git.antoinealb.net:~/.ssh/authorized_keys
{% endhighlight %}

Restart sshd for your changes to take effect: `sudo initctl restart ssh`.
You should be able to log out and log back in without providing your password.

If that worked you can now disable password logins.
Open `/etc/ssh/sshd_config` and find the following line:

{% highlight config %}
PasswordAuthentication yes
{% endhighlight %}

and change it to:

{% highlight config %}
PasswordAuthentication no
{% endhighlight %}

Restart `sshd` again, and you are good to go!

##Firewall configuration

For this section I will use `nmap` to scan the server for open ports.
This is not mandatory but I think it is a good way to ensure your firewall config is correct.
Scan your server using `nmap git.antoinealb.net`.
You should get SSH plus a few extra ports opened.

To filter traffic I will `iptables` directly.
I know that there are some tools to ease the setup like `ufw`, but my rules are so simple that I prefer to avoiding adding a moving part to my setup.
Here are the rules I implemented:

1. All inbound traffic is denied by default.
2. All outbound traffic is authorized by default.
3. All inbound traffic on loopback traffic is authorized.
    This is required to connect to Redis or PostgreSQL locally for example.
3. ICMP (ping), SSH, HTTP and HTTPS inbound traffic is authorized.

Put the following in `/etc/network/if-pre-up.d/firewall` and make it executable (`sudo chmod +x /etc/network/if-pre-up.d/firewall`).

{% highlight bash %}
#!/bin/sh

# Drop all input by default
iptables -P INPUT DROP

# Authorize all output
iptables -P OUTPUT ACCEPT

# Allow ICMP (ping) traffic
iptables -A INPUT -p icmp -j ACCEPT

# Allow all TCP connections from our side to enter
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow all input on loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow SSH traffic
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP traffic
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Allow HTTPs traffic
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
{% endhighlight %}

Reboot your server and run NMap again to be sure that your server is correctly protected.

**Note:** Apparently my ISP modifies my traffic so that the port 25 of the server is always opened.
I don't know why they do this, but it is not apparent when I scan my server from somewhere else, so be aware that it can happen to you too.

# Updating the old server
Gitlab can only be migrated from server to server using a backup restore to an identical version of Gitlab.
Therefore we need to update the old install of Gitlab to the Omnibus version we plan to install.
Fortunately for us, this is a [well documented and tested process](https://gitlab.com/gitlab-org/gitlab-ce/tree/master/doc/update).
Don't forget to make a backup at the beginning in case things go south.

# Installing Gitlab

## Installing Postfix

First of all we need to install the Postfix mail server.
It will be used by Gitlab to send email to the users, for notifications, password recovery, etc.
It is really important that your firewall denies connections to Postfix from the outside world, otherwise your email will be marked as spam and refused by users' ISPs.
This happened to me before, and it is practically impossible to revert once it happens.

To install Postfix, run `sudo apt-get install postfix` and select `Internet Site` when asked what type of installation you want.

## Installing Gitlab

First of all, add the Gitlab repository to your server:

{% highlight bash %}
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
{% endhighlight %}

Then install Gitlab Community Edition itself:

{% highlight bash %}
sudo apt-get install gitlab-ce
{% endhighlight %}

You can then start Gitlab by running `sudo gitlab-ctl reconfigure`.

**TODO:** configure gitlab properly and add let's encrypt

