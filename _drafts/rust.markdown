---
layout: post
title: Rust bare metal on ARM microcontroller
categories: Programming
tags: rust
---
Recently at the CVRA we decided to rewrite one important external library writen in C++.
We wanted to rewrite in C, but since Rust recently hit beta I wanted to see if it was feasible to use it for our application.
To do this, I decided to write a little demo application using Rust on a Texas Instruments Tiva Launchpad dev board.

# Install native rustc
Nothing special here, just don't use the beta version, as we will use some unstable features.
I installed latest Rust nightly on OSX via the following commands
{% highlight bash %}
brew tap cheba/rust-nightly
brew install rust-nightly
{% endhighlight %}

# Download rust target description
This file is needed to tell Rust/LLVM about our Cortex M4 CPU, it's pointer size and so on.
We will use the one from the Zync project:

{% highlight bash %}
wget "https://raw.githubusercontent.com/hackndev/zinc/master/support/target-specs/thumbv7em-none-eabi.json" -O thumbv7em-none-eabi
{% endhighlight %}

# Building libcore
Libcore provides the most fundamental data types and functions supported by Rust.
You can read more about it in [the documentation](https://doc.rust-lang.org/core/index.html).
Since it is really the foundation of the compiler's result, they must have the same version, down to the commit.
You can check which version of `rustc` you have on your computer by running `rustc --version -v` and looking for the commit-hash field.

First download the correct version of Rust's source code:

{% highlight bash %}
git clone https://github.com/rust-lang/rust
cd rust
git checkout $HASH
cd ..
{% endhighlight %}

Then we can build libcore using the target description we used before.

{% highlight bash %}
mkdir libcore-thumbv7m
rustc -C opt-level=2 -Z no-landing-pads --target thumbv7m-none-eabi -g rust/src/libcore/lib.rs --out-dir libcore-thumbv7m
{% endhighlight %}

# Provide needed runtime
For this little example I will reuse the runtime I build for my Tivaware template before.
We will only need a few more functions needed by the Rust runtime, mostly for panic functions.

Put the following in `runtime.rs`:

{% highlight rust %}
#![no_std]
#![crate_type="staticlib"]
#![feature(lang_items)]

extern crate core;

#[lang="stack_exhausted"] extern fn stack_exhausted() {}
#[lang="eh_personality"] extern fn eh_personality() {}
#[lang="panic_fmt"]
pub fn panic_fmt(_fmt: &core::fmt::Arguments, _file_line: &(&'static str, usize)) -> !
{
    loop { }
}

#[no_mangle]
pub unsafe fn __aeabi_unwind_cpp_pr0() -> ()
{
    loop {}
}
{% endhighlight %}

You can now try compiling this first Rust file for ARM by running the following commands:

{% highlight bash %}
rustc -C opt-level=2 -Z no-landing-pads --target thumbv7em-none-eabi -g --emit obj -L libcore-thumbv7m -o runtime.o runtime.rs

file runtime.o # Check that the file was compiled for ARM
{% endhighlight %}

We can now add Rust support to the Makefile.
This is just adding a rule to make `.o` object file from `.rs` source using the Rust compiler with the above flags.
You can see how I did it in [my commit](https://github.com/antoinealb/rust-demo-cortex-m4/commit/033a80ea998267cca27eac75cfd0b2bac132febd).

*Note:* We don't need to compile each of our Rust file on it's own.
All I had to do was build `main.rs` and "include" the other files with `mod` directives.
This was a bit surprising at first and caused me quite some problems.

# Porting our main function to Rust
My objective was reimplementing the "Hello, world" of embedded systems: blinking a LED.
Basically this requires three things:

1. Clock setup.
2. General Purpose Input / Output setup.
3. GPIO writing in a main loop.

All of this is usually done using either direct register access or a library.
As I said I am very interested by Rust's compatibility with C, so I decided to use Texas Instruments' Tivaware, which is a very basic library to deal with low level hardware settings.

I only wrote bindings for what I used, since I probably won't use Tivaware for my more "real" projects (we don't use Texas Instruments chips, but STM32).

## Sysctl binding
The `SysCtl` subsystem as its name implies is related to system control, like clock, peripherals and interrupts.
For this project I only need two Tivaware functions, `SysCtlClockSet` to configure system oscillator and `SysCtlPeripheralEnable` to enable the GPIO port on which my LED is connected.
I will also need a few constants.

My particular board has an external crystal at 16 Mhz.
I will use the PLL to rise it up to 400 Mhz, then divide it by 5 to have 80 Mhz, which is the max speed of this particular microcontroller.

From my previous article on Tivaware, I knew that this required the following constants: `SYSCTL_SYSDIV_2_5`, `SYSCTL_USE_PLL`, `SYSCTL_XTAL_16MHZ` and `SYSCTL_OSC_MAIN`.
We can get the value of those constants by opening Tivaware's `sysctl.h`.
Since we are here, we will also lookup the value of `SYSCTL_PERIPH_GPIOF`, which we will need to turn on the LED.
We can put them as public constants in `sysctl.rs`:
{% highlight rust %}
/* Sysctl.rs */
pub const SYSCTL_SYSDIV_2_5       : u32 = 0xC1000000;  // Processor clock is pll / 2.5
pub const SYSCTL_USE_PLL          : u32 = 0x00000000;  // System clock is the PLL clock
pub const SYSCTL_XTAL_16MHZ       : u32 = 0x00000540;  // External crystal is 16 MHz
pub const SYSCTL_OSC_MAIN         : u32 = 0x00000000;  // Osc source is main osc
pub const SYSCTL_PERIPH_GPIOF     : u32 = 0xf0000805;  // GPIO F
{% endhighlight %}

The functions binding are not very complicated.
You just declare them as `extern` and you can then call them from unsafe blocks.
I will also write safe wrappers around them for ease of use.
Since Rust functions are module private by default, we don't have to worry about someone directly using the C functions incorrectly.

{% highlight rust %}
// sysctl.rs
extern {
    fn SysCtlClockSet(config: u32);
    fn SysCtlPeripheralEnable(peripheral: u32);
}

pub fn clock_set(config: u32)
{
    unsafe {
        SysCtlClockSet(config);
    }
}

pub fn peripheral_enable(peripheral: u32)
{
    unsafe {
        SysCtlPeripheralEnable(peripheral);
    }
}
{% endhighlight %}

## GPIO bindings
I applied the same process as before, but later reworked it to have a bit more type safety.
By using `u32` directly like in the sysctl module, we gain little to nothing over C, so I decided that for the GPIO driver I will use custom types for ports and pins.
I will just put the different constants in enumerations and use those enumerations in my public API.
Here is the code:
{% highlight rust %}
// gpio.rs
#![allow(dead_code)]

pub enum Port {
    PortF = 0x40025000,
}

pub enum Pin {
    Pin0 = (1 << 0),
    Pin1 = (1 << 1),
    Pin2 = (1 << 2),
    Pin3 = (1 << 3),
    Pin4 = (1 << 4),
    Pin5 = (1 << 5),
    Pin6 = (1 << 6),
    Pin7 = (1 << 7),
}

extern {
    fn GPIOPinTypeGPIOOutput(base: *const u32, mask: u32);
    fn GPIOPinWrite(base: *const u32, mask: u32, value: u32);
}

pub fn make_output(port: Port, pin: Pin) {
    let mask = pin as u32;
    let base = port as u32;
    unsafe {
        GPIOPinTypeGPIOOutput(base as *const u32, mask);
    }
}

pub fn write(port: Port, pin: Pin, value: bool) {
    let base = port as u32;
    let shifted_val = pin as u32;
    unsafe {
        if value {
            GPIOPinWrite(base as *const u32, shifted_val, shifted_val);
        } else {
            GPIOPinWrite(base as *const u32, shifted_val, 0);
        }
    }
}
{% endhighlight %}

# Putting it together
First we write a simple LED driver to hide the hardware details from main.
This driver needs to configure a given pin as output, enable the port on which the led is connected and provide functions to set the LED state.

{% highlight rust %}
// led_driver.rs
use sysctl;
use gpio;
use gpio::Pin;

pub const RED: Pin = Pin::Pin1;
pub const BLUE: Pin = Pin::Pin2;

pub fn led_init() {
    sysctl::peripheral_enable(sysctl::SYSCTL_PERIPH_GPIOF);
    gpio::make_output(gpio::Port::PortF, RED);
    gpio::make_output(gpio::Port::PortF, BLUE);
}

pub fn set_red(state: bool) {
    gpio::write(gpio::Port::PortF, RED, state);
}

pub fn set_blue(state: bool) {
    gpio::write(gpio::Port::PortF, BLUE, state);
}
{% endhighlight %}

And finally the main, which just a loop doing busy wait.
I had to add a few statements to remove the standard library and allow bare metal Rust (mostly taken from my references).

{% highlight rust %}
#![feature(no_std)]
#![feature(core)]
#![feature(lang_items)]
#![no_std]

#![crate_type="staticlib"]

extern crate core;

mod runtime;
mod sysctl;
mod led_driver;
mod gpio;

fn clock_init() {
    let clock_config = sysctl::SYSCTL_SYSDIV_2_5 + sysctl::SYSCTL_USE_PLL +
                       sysctl::SYSCTL_XTAL_16MHZ + sysctl::SYSCTL_OSC_MAIN;
    sysctl::clock_set(clock_config);
}


#[no_mangle] pub fn main()
{
    clock_init();
    led_driver::led_init();

    loop {
        let mut i = 0;
        while i < 1000000 {
            i += 1;
            led_driver::set_red(false);
        }

        i = 0;

        while i < 1000000 {
            i += 1;
            led_driver::set_red(true);
        }
    }
}
{% endhighlight %}

Just run `make all load` and the LED will start blinking! The complete project is available on [Github](https://github.com/antoinealb/rust-demo-cortex-m4).

# What's next ?
This was just a basic experiment with Rust and ARM microcontrollers.
I started working on interrupts and bindings to ChibiOS.
I also would like to design safer APIs instead of simply translating the code to Rust.

# Sources:
* http://spin.atomicobject.com/2015/02/20/rust-language-c-embedded/
* https://doc.rust-lang.org/core/
* [My previous post about Stellaris](http://antoinealb.net/programming/2014/04/21/stellaris-linux.html)
* StackOverflow, as always
