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
You can see how I did it in [my commit](https://github.com/antoinealb/rust-demo-cortex-m4/commit/033a80ea998267cca27eac75cfd0b2bac132febd).


# Porting our main function to Rust



Sources:
* http://spin.atomicobject.com/2015/02/20/rust-language-c-embedded/
