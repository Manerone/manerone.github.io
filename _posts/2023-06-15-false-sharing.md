---
layout: post
title: Cache False Sharing
subtitle: The perils of cache sharing
gh-repo: manerone/HardwareEffects
gh-badge: [star, fork, follow]
tags: [performance, cache]
comments: false
author: Matheus Nerone
readtime: true
---

In this post I want to talk a bit about the phenomena of false sharing of cache lines. I think it will be easier to understand if we walk through it as if we are telling a story.

## Filter and count

Imagine you are developing a system, a database or something else. At some point, you need to filter several vectors and maintain a counter of the times the filter condition was met.

For example, in a database this could be some sort of aggregation across multiple columns. 
Let's set up some basic structures we will use to solve this task:

* **SharedState:** where you keep the state of your computation. In our example here, we have four counters for four different filters.

{% highlight c++ linenos %}
/// Shared state used to keep track of all
/// the different counters
struct SharedState {
    std::atomic<int64_t> counter0;
    std::atomic<int64_t> counter1;
    std::atomic<int64_t> counter2;
    std::atomic<int64_t> counter3;
};
{% endhighlight %}

{: .box-note} Why do we use `std::atomic`? The cpu seems to be smart enough to avoid false sharing if we only have `int64_t`, so atomic is necessary otherwise the example doesn't work.

* **Table:** represents the data that will be filtered. To make coding the example easier, the constructor creates a table with four columns, and 1.000.000 rows.
{% highlight c++ linenos %}
/// Table with columns that will be used to count
struct Table {
    std::vector<std::vector<int64_t>> columns;

    Table(){
        columns.resize(4);
        for(auto& col : columns){
            col.resize(1000000);
            for(size_t i = 0; i < col.size(); i++){
                col[i] = i;
            }
        }
    }
};
{% endhighlight %}

* **Filter Method:** used to count the number of time the filter condition was met.

{% highlight c++ linenos %}
/// Method used to apply the filters and count the occurrence in a column
void filter_and_count(std::vector<int64_t> &column, std::atomic<int64_t> &counter, std::function<bool(int64_t)> &&filter){
    for(const auto value : column){
        if(filter(value)){
            counter++;
        }
    }
}
{% endhighlight %}

## Single thread

As a good developer, the first thing we try is to make the system work. So, you come with an awesome solution where we scan each column using its respective filters, and add to the counter. Something in the lines of:
{% highlight c++ linenos %}
/// Using a single thread, filter and count the occurrences
static void BM_SINGLE_THREAD(benchmark::State& gb){
    Table table;
    while (gb.KeepRunning()) { // Ignore this while loop for now
        SharedState shared_state;

        filter_and_count(table.columns[0], shared_state.counter0, [](int64_t v){
            return v % 2 == 0;
        });

        filter_and_count(table.columns[1], shared_state.counter1, [](int64_t v){
            return v % 3 == 0;
        });

        filter_and_count(table.columns[2], shared_state.counter2, [](int64_t v){
            return v % 4 == 0;
        });

        filter_and_count(table.columns[3], shared_state.counter3, [](int64_t v){
            return v % 5 == 0;
        });
    }
}
{% endhighlight %}

The code seems fine, every column goes through the process of filter  and counting. What rests now is to execute benchmark the solution.

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
BM_SINGLE_THREAD/real_time          8.69 ms         8.68 ms           80
```

## Multi-threaded

Being the avid developer you are, you notice that making `BM_SINGLE_THREAD` execute in parallel is quite simple and straight-forward.
Each column is completely independent from the others, and there is one counter for each column. In theory, you can just plug a thread on each call and call it a day. Something like this:

{% highlight c++ linenos %}
static void BM_NAIVE_THREAD(benchmark::State& gb){
    Table table;
    while (gb.KeepRunning()) {
        SharedState shared_state;

        std::thread t1([&](){
            filter_and_count(table.columns[0], shared_state.counter0, [](int64_t v){
                return v % 2 == 0;
            });
        });

        std::thread t2([&](){
            filter_and_count(table.columns[1], shared_state.counter1, [](int64_t v){
                return v % 3 == 0;
            });
        });

        std::thread t3([&](){
            filter_and_count(table.columns[2], shared_state.counter2, [](int64_t v){
                return v % 4 == 0;
            });
        });

        std::thread t4([&](){
            filter_and_count(table.columns[3], shared_state.counter3, [](int64_t v){
                return v % 5 == 0;
            });
        });

        t1.join();
        t2.join();
        t3.join();
        t4.join();
    }
}
{% endhighlight %}

All is good, life is beautiful, you can just compile, execute the benchmark. And **IT GOT SLOWER**, almost twice as slow as the single thread version. What in the world has happened?

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
BM_SINGLE_THREAD/real_time          8.69 ms         8.68 ms           80
BM_NAIVE_THREAD/real_time           14.0 ms        0.059 ms           50
```

## The perils of false sharing

What you just observed is the phenomena called False Sharing, which can happen in the system caches. Let's look at the memory for a bit. In your machine, you have different kinds of memory. Usually it goes something like:

* Disk: the largest amount of memory you have, but super slow to access.
* RAM: quite less memory than the disk, but much faster to access.
* Caches: a lot less memory than RAM, but miles faster to access.

The cache is usually divided into three levels where you have less memory but fast access. However, for this example you don't need to think about the levels. The cache also has something called a *cache line*. Each *cache line* has 64 bytes (512 bits) of size, and is read entirely by the CPU on every access.

If we look at our `SharedState` structure, we have four integers, each with 64 bits (8 bytes). When the CPU asks to read one of the counters, the whole of `SharedState` will be brought into the CPU registers (plus some trash at the end, which we don't mind). As depicted in the figure below:

![CPU-cache]({{ '/assets/img/mermaid-diagram-2024-06-15-171155.png' | relative_url }})

The problem here is that each counter is also atomic. That means the hardware implements special operations on them. When counting, as we did in our example, these operations are something like `read-modify-write`. This means, for the duration of the whole operation that cache line is "locked". So, when multiple threads try to access the same cache line, there is some waiting involved. As you saw in the benchmarks, that is quite catastrophic.

![Thread-wait]({{ '/assets/img/mermaid-diagram-2024-06-15-170848.png' | relative_url }})

## Solution 01 - Local Variables

So, now we understand why the initial multi-threaded version got worse performance. What can we do to avoid this situation? The first solution is to use local variables. Instead of counting directly on the shared state, it is possible to count on a variable that is local to the thread, and then move the result to the shared state. Something like:

{% highlight c++ linenos %}
static void BM_LOCAL_VARIABLES(benchmark::State& state){
    Table table;
    while (state.KeepRunning()) {
        SharedState shared_state;

        std::thread t1([&](){
            std::atomic<int64_t> local;
            filter_and_count(table.columns[0], local,
                             [](int64_t v) { return v % 2 == 0; });
            shared_state.counter0 = local.load();
        });

        std::thread t2([&](){
            std::atomic<int64_t> local;
            filter_and_count(table.columns[1], local,
                             [](int64_t v) { return v % 3 == 0; });
            shared_state.counter1 = local.load();
        });

        std::thread t3([&]() {
          std::atomic<int64_t> local;
          filter_and_count(table.columns[2], local,
                           [](int64_t v) { return v % 4 == 0; });
          shared_state.counter2 = local.load();
        });

        std::thread t4([&](){
            std::atomic<int64_t> local;
            filter_and_count(table.columns[3], local, [](int64_t v){
                return v % 5 == 0;
            });
             shared_state.counter3 = local.load();
        });

        t1.join();
        t2.join();
        t3.join();
        t4.join();
    }
}
{% endhighlight %}

## Solution 02 - Alignment

It is also possible to modify the `SharedState` structure to force its variables to be aligned to a power of two.

> Aligning a variable to a number means its address will always be divisible by that number. For example, aligning a variable to 16, means every time that variable is created, it will be on an address divisible by 16.

This can be done using the special construct `alignas(n)`. In our case, since the cache line is 64 bytes, and we would to avoid false sharing. It seems to make sense to align every counter to 64. For this, we must create a new structure:

{% highlight c++ linenos %}
struct AlignedSharedState {
    alignas(64) std::atomic<int64_t> counter0;
    alignas(64) std::atomic<int64_t> counter1;
    alignas(64) std::atomic<int64_t> counter2;
    alignas(64) std::atomic<int64_t> counter3;
};
{% endhighlight %}

Then, we can reuse the same code as the first multi-threaded implementation, just with the new structure.

{% highlight c++ linenos %}
static void BM_ALIGNED_COUNTERS(benchmark::State& state){
    Table table;
    while (state.KeepRunning()) {
        AlignedSharedState shared_state;

        std::thread t1([&](){
            filter_and_count(table.columns[0], shared_state.counter0, [](int64_t v){
                return v % 2 == 0;
            });
        });

        std::thread t2([&](){
            filter_and_count(table.columns[1], shared_state.counter1, [](int64_t v){
                return v % 3 == 0;
            });
        });

        std::thread t3([&](){
            filter_and_count(table.columns[2], shared_state.counter2, [](int64_t v){
                return v % 4 == 0;
            });
        });

        std::thread t4([&](){
            filter_and_count(table.columns[3], shared_state.counter3, [](int64_t v){
                return v % 5 == 0;
            });
        });

        t1.join();
        t2.join();
        t3.join();
        t4.join();
    }
}
{% endhighlight %}

When executing, if you ask to print the addresses of the counters you can see that they are indeed separated by 64 bytes. An example on my own machine:
```
0x16d805b80 --> counter0
0x16d805bc0 --> counter1
0x16d805c00 --> counter2
0x16d805c40 --> counter3
```

## Results

When we execute the benchmarks again, we can observe that the local variables, and the aligned version, are twice as fast as the single threaded version. I am not completely sure that the local variables version will always avoid false sharing. It could be that you are unlucky and the variables end up in the same cache line, since there are no alignment guarantees.

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
BM_SINGLE_THREAD/real_time          8.69 ms         8.68 ms           80
BM_NAIVE_THREAD/real_time           14.0 ms        0.059 ms           50
BM_LOCAL_VARIABLES/real_time        4.52 ms        0.053 ms          155
BM_ALIGNED_COUNTERS/real_time       4.31 ms        0.055 ms          161
```

### References and Thanks

Many thanks to CoffeeBeforeArch [post](https://coffeebeforearch.github.io/2019/12/28/false-sharing-tutorial.html) for the explanation and examples.

Also thanks to the talk from Fedor Pikus on CppCon 2017 ([link](https://youtu.be/ZQFzMfHIxng?si=n2j7P782lwJ9jimn)).


## Source Code

Source code is [here](https://github.com/Manerone/HardwareEffects/tree/main/cache_false_sharing).