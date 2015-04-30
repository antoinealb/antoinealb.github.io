# Install native rustc
On OSX, using Homebrew: `brew install rust`

# Download rust target description
This file is needed to tell Rust/LLVM about our Cortex M4 CPU, it's pointer size and so on.
We will use the one from the Zync project:

{% highlight bash %}
wget "https://github.com/hackndev/zinc/blob/master/support/target-specs/thumbv7em-none-eabi.json" -O thumbv7em-none-eabi
{% endhighlight %}

# Building libcore
Libcore provides the most fundamental data types and functions supported by Rust.
You can read more about it in [the documentation](https://doc.rust-lang.org/core/index.html).
Since it is really the foundation of the compiler's result, they must have the same version, down to the commit.
I used Rust beta, but you can check which version of `rustc` you have on your computer by running `rustc --version -v` and looking for the commit-hash field.

First download the correct version of Rust's source code:

{% highlight bash %}
git clone https://github.com/rust-lang/rust
cd rust
git checkout 1.0.0-beta # replace the tag by your commit hash if necessary
cd ..
{% endhighlight %}

Then we can build libcore using the target description we used before.

{% highlight bash %}
mkdir libcore-thumbv7m
rustc -C opt-level=2 -Z no-landing-pads --target thumbv7m-none-eabi -g rust/src/libcore/lib.rs --out-dir libcore-thumbv7m
{% endhighlight %}

Sources:
* http://spin.atomicobject.com/2015/02/20/rust-language-c-embedded/
