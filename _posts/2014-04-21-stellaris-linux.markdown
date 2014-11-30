---
title: Working with the TI Tiva Launchpad on LinuRobotics x
layout: post
categories: Programming
tags: c arm linux
---


I recently received a Texas Instruments Tiva Launchpad board from another team in the Eurobot contest (thank you Ifrelo ;)) when I told them I was interested in ARM dev.
Once I had this board, I looked around for compilers / IDEs, but I quickly realized that most of them were closed source or Windows only.
Since I really like to work on my Linux box, I decided to take some time to work on it.

# Building an ARM bare metal toolchain
To work on a microcontroller we need to build a bare metal toolchain which will produce binary that will run without any operating system.
To do that, we can use the script "Summon arm toolchain".
Since the original project is not developped anymore, we will use a fork.

First we need to download it :

{% highlight bash %}
git clone "https://github.com/mchck/summon-arm-toolchain.git"
cd summon-arm-toolchain/
{% endhighlight %}

Then we will compile it.
I explicitely disabled OpenOCD compilation since I plan to use L4MFtools at first.

{% highlight bash %}
./summon-arm-toolchain OOCD_EN=0
{% endhighlight %}

Once summon-arm-toolchain succeeds, you should have a working compiler in `~/sat/`.
Check that it worked properly:
{% highlight bash %}
~/sat/bin/arm-none-eabi-gcc --version
{% endhighlight %}

# Build lm4flash
lm4tools is a set of tools by Fabio Utzig to interact with the Tiva Launchpad.
We are only interested in the lm4flash program which allows to send binaries on the board.
I decided to go with this instead of OpenOCD for now because it seemed easier to use.

Note for Arch Linux users:
you will find a working PKGBUILD in the AUR but for the others, you simply need a few commands to build it. Don't forget to add it to your path.

{% highlight bash %}
git clone "https://github.com/utzig/lm4tools/"
cd lm4tools/lm4flash
make
{% endhighlight %}

# Getting the template and building the first app.
I made a Stellaris template based on the original work of Mauro Scomparin.
It was mostly a matter of fixing a few includes.
I then tweaked it to my own needs.
It is available on Github : [https://github.com/antoinealb/tivaware-template](antoinealb/tivaware-template).
If you find a bug or a suggestion, you are more than welcome to open an issue or a pull request !
Building the project and sending on the board is really easy :

{% highlight bash %}
git clone --recursive "https://github.com/antoinealb/tivaware-template"
cd tivaware-template/
make
sudo make load
{% endhighlight %}

You need to do a recursive clone because I included Texas Instrument's Tivaware libraries as a submodule of the repository.
It will also prompt you for a password when you try to download it on the board because you don't have access to the port by default.
I will later explain how to add a udev rule to allow standard users to access the board.
Now your board should happily blink:

![Blink demo]({{ site.url }}/assets/media/stellaris_blink.gif)


If this is the case, congrats and have fun with ARM Microcontrollers !
Otherwise, don't hesitate to shoot me an email or an issue and we will try to fix it !

# Sources
* [http://jeremyherbert.net/get/stm32f4_getting_started](Getting Started with the STM32F4 and GCC)
* [http://kernelhacks.blogspot.ch/2012/11/the-complete-tutorial-for-stellaris.html](The complete tutorial for Stellaris LaunchPad development with GNU/Linux)



