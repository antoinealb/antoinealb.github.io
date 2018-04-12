---
layout: post
title: Test your external dependencies
---

In the unit testing crowd your often hear the mantra that you should not test your external dependencies with unit tests and that they are only for integration testing.
In a way this makes sense: the developper of the library probably took some time to test the library on their own, so why redo the work?
However, I like to write a few unit tests when I am integrating a new library into a project.
Most of the time I even commit them to the source repository and add them to the CI build.

For sure it takes much longer to integrate the library now.
You have to write some tests, and you may have to code some mock objects as well.
However, I find this investment in time well worth it, for two main reasons:

First, it gives you a better understanding of the API of your library.
You can clearly see how the library works, which expectations it has.
With mock objects you can also inject faults and see how the code reacts.
In fact, I had the idea for this blog as I was integrating a library I used before in a new project.
It took me so long to debug a weird issue, but when writing a test, it quickly became clear that I just forgot a step in the initialization of the library.

Second, it can be used as a future reference (or cheat sheet) by you and your team.
This is why I add the tests to the repository: to make sure they will be available when I forget how to do a given operation.

Finally, once your test suite is in place, it becomes easy to prototype new interactions with the dependency.
Just write tests for this new use case and see if it works.
This is also useful if you ever have to fix a bug in the library: you can try and reproduce it in your familiar testing environment.

Happy testing!
