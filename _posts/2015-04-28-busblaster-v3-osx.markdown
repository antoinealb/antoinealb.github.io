---
layout: post
title: Using BusBlaster v3/v4 on OSX
categories:
    - System administration
---

I recently got myself a Macbook Pro as my main laptop.
The migration was fine except for one thing: My Dangerous Prototype BusBlaster (3rd generation) was not usable.
Since a friend's 2nd generation one was working flawlessly, I supposed there must be a way to do it.

I finally found the solution, and to avoid forgetting I am posting it here.
To use a 3rd gen BusBlaster with OSX Yosemite, you must unload the default FTDI driver:

{% highlight bash %}
sudo kextunload -bundle com.apple.driver.AppleUSBFTDI
{% endhighlight %}

*Note:* This means your FTDI based serial adapter won't work anymore until you reboot or reload the module.
You should be able to reload the driver by doing `sudo kextload -bundle com.apple.driver.AppleUSBFTDI`, but I did not test this.

