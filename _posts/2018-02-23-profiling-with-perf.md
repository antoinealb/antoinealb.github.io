---
layout: post
title: Measuring program performance with perf(1)
categories: Programming
---

In my new internship I am working on optimizing some software.
But before starting making changes to the source code, I need to make sure that I am optimizing important parts.
Imagine for a moment that I spend days making one part of a program go 10x faster (!!), only to realize later that it is responsible for 1% of the total run time.
How much would I have saved?
The answer is not much: the new run time would be 99% + (1% / 10) = 99.1% of the original run time.
In other words, only 1.01 times faster.
Not so impressing now, is it?

This post will introduce one method to measure how the program is spending its time.
This is known as *profiling* the code.
I learnt a lot of techniques from my colleagues, but also a lot comes from [Chandler Carruth's talk at CppCon 2015](https://www.youtube.com/watch?v=nXaxk27zwlk).
Go watch it if you want more information than what I expose here.

## Sample program to profile

For those experiments we will need a small program to profile.
I wrote a simple program (see below) that computes the Fibonacci sequence and displays it.
I know that this is not optimal in any way, but we will see how.

```cpp
#include <vector>
#include <iostream>

#include <benchmark/benchmark.h>

static void Benchmark_Fibonacci(benchmark::State& state) {
    const int N = 200;
    for (auto _ : state) {
        std::vector<int> v;

        v.push_back(1);
        v.push_back(2);

        for (int i = 1; i < N; i++) {
            v.push_back(v[i] + v[i - 1]);
        }

        benchmark::DoNotOptimize(&v);
        benchmark::ClobberMemory();
    }
}
BENCHMARK(Benchmark_Fibonacci);

BENCHMARK_MAIN();
```

We can now build this program using `g++ benchmark.cpp -o fibo && ./benchmark`.
No output is produced, but the suite is computed, so it is time to measure what takes some time.

Now that this is done, we can run the program with the perf utility (found in `linux-tools`).
We will run it with the `-g` option to record call graph information:
We need to run it as root because the access to performance counter are disabled by default on Linux.
Normal users can be given access to this, but this is out of the scope of this article.

```bash
$ sudo perf record -g ./benchmark
$ sudo perf report -g 'graph,0.5,caller'
```

You should now be presented with an interactive window in which you can see how much time your program spent in each function.

One nice trick in the code above is the use of the functions `DoNotOptimize` and `ClobberMemory`.
Those twp functions basically tell the compiler to ignore the fact that the results of the computation are not used anywhere.
Otherwise, most optimizers will delete the code of our benchmark, as it does not create side effect.

For those of you who are curious, you can implement it yourself using the following code:

```cpp
void DoNotOptimize(void *p)
{
    volatile asm("" : : "g"(p) : "memory");
}

void ClobberMemory(void)
{
    volatile asm("" : : : "memory");
}
```

Basically those two lines of code are used to create fake zones where the results of the benchmark *might* be used.
The compiler is also disallowed to optimize those parts by the use of the `volatile` keyword.
For more detail on this, see the talk I linked above or GCC's documentation on inline assembly.

There is a lot to be said on how to interpret perf's output and optimize your code, but I just wanted to share what I learnt last week.
The rest will be for another post!

