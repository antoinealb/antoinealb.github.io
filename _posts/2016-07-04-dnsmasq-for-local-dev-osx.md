---
layout: post
title: Using Dnsmasq for VM testing
---

Today I was trying a new configuration option for Gitlab and I wanted to make sure I did not make any mistakes before trying the site live.
Therefore I decided to deploy the configuration in a virtual machine and then copy it on the live server.

To access the page served by the VM two conditions needed to be met:

1. The HTTP port should be forwarded to be available on the host.
    [Vagrant][vagrant] makes it extremely easy, take a look at this tool if you don't know it already.
2. The virtual machine should be accessible under the domain given to Gitlab's web server.
    Simply typing the IP address might not work because a single web server might be configured to serve different sites depending on the domain.
    This part is usually done by hacking `/etc/hosts` to resolve the chosen name to `127.0.0.1` but I decided to try using Dnsmasq for this after reading [a blog post about it][dnsmasq-blog].

## What is Dnsmasq ?

Among other things, Dnsmasq can act as a DNS caching server.
This can be used to increase resolution speed, or, in our case, to inject fake DNS records into our network.
Specifically, I will use to resolve all `.dev` domains to my own machine.

## Installing Dnsmasq on OSX
Using Homebrew installing Dnsmasq is relatively easy: `brew install dnsmasq`.
I decided to track the configuration in my [dotfiles repository][dotfiles] and to symlink the actual file to the versioned one:

{% highlight bash %}
cp /usr/local/opt/dnsmasq/dnsmasq.conf.example ~/dotfiles/dnsmasq.conf
sudo ln -s ~/dotfiles/dnsmasq.conf /usr/local/etc/dnsmasq.conf
{% endhighlight %}

## Configuring the DNS server
*Note:* you can find the latest version of the file [on Github][dnsmasq-conf].

The first step is to be a good netizen by disabling the forwarding of ill-formed domain names.
I don't know if this is strictly required but the Dnsmasq example guide suggests enabling it.
We will also only allow requests coming from localhost for security reasons.

{% highlight conf %}
# Never forward plain names (without a dot or domain part)
domain-needed
# Never forward addresses in the non-routed address spaces.
bogus-priv

# Only allow localhost requests
listen-address=127.0.0.1
{% endhighlight %}

Then we will add the redirection rules.
We only have one, i.e. sending all `.dev` to `127.0.0.1`.

{% highlight conf %}
# Add domains which you want to force to an IP address here.
# Sends all domain ending in .dev to localhost
address=/dev/127.0.0.1
{% endhighlight %}

After restarting Dnsmasq (`sudo brew services restart dnsmasq`), you should be able to test your config by running `dig foo.dev @localhost`.
If you get an answer mapping to 127.0.0.1, everything is working!

## Sending DNS requests to Dnsmasq
This step if pretty straightforward: Open System Preferences, go to "Network", then click "Advanced..." and add `127.0.0.1` to your servers under the "DNS" tab.
Don't forget to click apply once you left the advanced settings!
Now you should be able to `ping foo.dev`, however normal websites won't be recognized anymore.

## Adding upstream servers
The [original guide][dnsmasq-blog] chooses to send *only* the `.dev` queries to Dnsmasq, but I preferred to add upstream servers to my config.
Upstream DNS servers are queried when Dnsmasq doesn't know how to resolve a given query.
Enabling them means that DNS queries will be cached, making them slightly faster.
It also means that you don't have to fumble with OSX files such as `/etc/resolver/`, which is forbidden by El Capitan (I haven't updated my laptop to El Capitan though).

To enable upstream servers add the following to your `dnsmasq.conf`.
I chose to use two different DNS providers for reliability reasons (Google DNS and OpenDNS).

{% highlight conf %}
# Upstream DNS servers
server=8.8.8.8
server=8.8.4.4
server=208.67.222.222
server=208.67.220.220
{% endhighlight %}

Restart Dnsmasq and voilà!
You are now able to ping google.com and foo.dev should resolve to localhost.
You can now configure your virtual server as foo.dev and it should correctly answer, but that is left as an exercise to the reader ;)

[vagrant]: https://www.vagrantup.com/
[dnsmasq-blog]: https://passingcuriosity.com/2013/dnsmasq-dev-osx/
[dnsmasq-conf]: https://github.com/antoinealb/dotfiles/blob/master/dnsmasq.conf
[dotfiles]: https://github.com/antoinealb/dotfiles
