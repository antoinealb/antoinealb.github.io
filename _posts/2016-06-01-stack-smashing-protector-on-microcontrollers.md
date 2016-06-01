---
layout: post
title: Using GCC's Stack Smashing Protector on microcontrollers
categories: Programming
tags: c arm
---

Writing your code in C means manual memory management means a lot of bug types: Double free, use after free, stack overflow, etc.
Those bugs can be especially hard to debug because they will cause erratic behavior but might not trigger an error condition immediately.

I recently added a memory protection unit (MPU) driver that I used to detect NULL pointer dereference.
This (combined with other patches) significantly increased the stability of our platform, but we still had occasional issues.

In order to debug those, I wanted to add more memory hardening to the system.
I started with the stack smashing detection, since it was the easiest.

# What is stack smashing ?

To quote Wikipedia:

> In computer security and programming, a buffer overflow, or buffer overrun, is an anomaly where a program, while writing data to a buffer, overruns the buffer's boundary and overwrites adjacent memory locations.
> This is a special case of the violation of memory safety.

Stack smashing is a class of buffer overflow which occurs on stack-allocated buffers.
It can be used by an attacker to gain code execution by modifying a function return address.
That is not a concern on our robot, but it can be an issue if you are developing an Internet-connected product.

Consider the following code:

{% highlight c %}
void my_buggy_function(const char *user_provided_message)
{
    char buffer[10];
    strcpy(buffer, user_provided_message);
}
{% endhighlight %}

In this overly simplificated example, if the user provided message is longer than nine characters (plus terminating zero), then the copy will overflow from the buffer into following variables.
An attacker could use this to override the function return address, gaining code execution.

## How does Stack Smashing Protection work ?
Stack Smashing Protection (SSP) tries to prevent most of those bugs by adding an extra variable (called a canary) in every function.
On function entry this canary is set to a value and on function exit the canary's value is checked.
If it has changed during function execution it means the stack has been smashed and a callback is fired.

Of course SSP cannot detect every buffer overflow but it is still better than nothing for debugging.
It also effectively closes a whole class of security flaws if correctly implemented.
However, it has a (small) runtime cost, which might be a problem depending on your requirements.

# Enabling SSP
Turning on SSP with GCC is quite easy: Just add `-fstack-protector-all` to your CFLAGS.
You might also be interested in `-fstack-protector` and `-fstack-protector-strong` which use some heuristics to exclude some functions from being checked.

So let's build and see how it goes:

{% highlight bash %}
arm-none-eabi/bin/ld: cannot find -lssp_nonshared
arm-none-eabi/bin/ld: cannot find -lssp
{% endhighlight %}

Apparently some libraries are missing.

## Adding missing libraries
A bit of Googling teaches me that I should be able to circumvent that problem by linking against empty static libraries instead.
First, ask GCC to look for libraries in the current folder by adding `-L .` to your `LDFLAGS`.
Then, create empty `libssp.a` and `libssp_nonshared.a` using the following commands:

{% highlight bash %}
arm-none-eabi-ar rcs libssp.a
arm-none-eabi-ar rcs libssp_nonshared.a
{% endhighlight %}

Now, rebuild the project and GCC should complain about missing references to `__stack_chk_guard` and `__stack_chk_fail`.

## Writing the Stack Smashing protector callback
To work correctly SSP requires two symbols to be defined:

* `__stack_chk_guard` which contains the initial value of the stack protector, and,
* `__stack_chk_fail` which is called when a stack smashing is detected.
    This function should never return.

Here is a minimal implementation for ChibiOS but adapting it to your platform should be trivial.
Just be careful to adapt `STACK_CHK_GUARD` to the word width of your architecture (the example is for 32 bits).

{% highlight c %}
uintptr_t __stack_chk_guard = 0xdeadbeef;

void __stack_chk_fail(void)
{
    chSysHalt("Stack smashing detected");
}
{% endhighlight %}

If you want to make your system harder to exploit I recommend setting `__stack_chk_guard` to a random value on every boot using the hardware random number generator found in recent microcontrollers.
Otherwise an attacker might be able to find the value of your stack canary and exploit this knowledge to circumvent this protection.

# Testing the protection
We are now able to check if the stack smashing protection works correctly by running the following function:

{% highlight c %}
void foo(void)
{
    char buffer[2];
    strcpy(buffer, "hello, I am smashing your stack!");
}
{% endhighlight %}

If everything goes well your panic handler should be called on function exit (check that with a debugger).
If it doesn't, try to reduce optimization level or check your console output for any warning/error messages.

# Conclusion & Future work
SSP is one of the numerous tool you can use to make your code more secure.
It has the advantage of being easy to apply to your whole codebase at once since it does not require any change to your source code.

I plan to add other other debugging features in the following weeks.
I have an idea on how to use the MPU to prevent thread stack overflows (different from the buffer stack overflow we explored today).
I also would like to implement heap corruption detection but I don't know how I will do this yet.
If anybody has done this before I would be glad to hear about it.

