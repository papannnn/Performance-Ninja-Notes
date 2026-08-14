# Measuring Performance

## Noise in Modern Systems

There are many features in hardware and software that are designed to increase
performance, but not all of them have deterministic behavior.

Such as Dynamic Frequency Scaling (DFS), basically it's a turbo mode for CPU.

But DFS can't last long, since if running long, it will overheat the CPU. It makes the CPU run faster tho.

The problem is, when you benchmark the program but the DFS is currently running, that means you will see the performance is increasingly fast.

But the moment you do benchmark again, coincidentally the DFS not running anymore. Making the result of benchmark can't be reproducable.

Or maybe when you try to benchmark `git status`, that command will try to read many files in disk. 

First time you benchmark it, it will be slow.

But the second time of benchmark happens, it's running faster than before because first run will generate a disk cache to improve performance. Making the benchmark feels inconsistent.

It's impossible to have 1:1 result with every benchmark, but we can control program behavior so the program can behave what we want to help reproduce the same result.

## Continuous Benchmarking

When you maintain the features of the application, you sometimes adding more features, bug fixing, adding more observability like logging, etc.

Sometimes, those updates are killing performance if we don't keep an eye for them.

If we didn't check it often when adding features, there's a chance new features will degrades the current performance.

There's no exact best practice how to approach this, but typical CI performance tracking system will do like this:

1. Set up a system under test.
2. Run a benchmark suite.
3. Report the results.
4. Determine if performance has changed.
5. Alert on unexpected changes in performance.
6. Visualize the results for a human to analyze

## Manual Performance Testing

Sometimes, benchmarking using automated system is too complicated to do, because of that, we prefer to do local performance evaluation.

We typically measure the performance impact of our code change by:

1. measuring
the baseline performance
2. measuring the performance of the modified program
3. comparing them with each other

When you measure a program, don't just measure it 1x. because there's a chance we met the noise that we're talking about in the above.

Try to measure it several times, and get it's performance distribution, aggregate them and compare both baseline vs modified.

## Terms for the performance measurements

### Mean

Sum all of the value in dataset, divided by amount of dataset.

### Median

Middle value of dataset when value is sorted

### 25th percentile

Divide the lowest 25% of the data from highest 75%

### 75th percentile

Divide the lowest 75% of the data from highest 25%

### Outlier

Data point that differs significantly from other samples in dataset. Can be caused by error

### Min & Max

Most extreme data point that not considered as outlier

## Software and Hardware Timers

To benchmark execution time of application, engineer usually use two different timers

### System-wide high resolution timer

Called epoch, can be retrieved by OS system call.

```c++
#include <cstdint>
#include <chrono>
// returns elapsed time in nanoseconds
uint64_t timeWithChrono() {
    using namespace std::chrono;
    auto start = steady_clock::now();
    // run something
    auto end = steady_clock::now();
    uint64_t delta = duration_cast<nanoseconds>(end - start).count();
    return delta;
}
```

### Time Stamp Counter (TSC)

This is hardware timer, suitable for measuring short events from nanosecond up to 1 min.

```c++
#include <x86intrin.h>
#include <cstdint>
// returns the number of elapsed reference clocks
uint64_t timeWithTSC() {
    uint64_t start = __rdtsc();
    // run something
    return __rdtsc() - start;
}
```

Tl;dr, if short measurement, use TSC, if long, use System timer. Using system timer has an overhead.

## Microbenchmark

Small test to quickly see the hypothesis.

When writing benchmark, make sure the compiler didn't remove your code that you want to check. Sometimes compiler optimization makes the code got removed because compiler thinks it's not used.

```c++
// foo DOES NOT benchmark string creation
void foo() {
    for (int i = 0; i < 1000; i++)
        std::string s("hi");
}
```

One popular way to solve this problem is to add some kind of `DoNotOptimize` helper function

```c++
// foo benchmarks string creation
void foo() {
    for (int i = 0; i < 1000; i++) {
        std::string s("hi");
        DoNotOptimize(s);
    }
}
```

## Active Benchmarking

In summary, if you only benchmark something on the face level only, it's called passive benchmarking. To make it more extra miles, we can do active benchmarking.

By ensuring proper configuration, ran extensive testing, looked one level deeper, and collecting as many entries as possible to support it's conclusion is some way to do active benchmarking.

## Questions and Exercises

### Is it always safe to take a mean of a series of measurements to determine the running time of a program? What are the pitfalls?

I don't think only using mean as a measurement of running time is good enough. Because what if there's some outlier that disturb the Min & Max? It will ruin the value of the mean. 

I think we should check the outlier + noise first, then we can check the mean and other metrics after that.

### Suppose you’ve identified a performance bug that you’re now trying to fix in your development environment. How would you reduce the noise in the system to have more pronounced benchmarking results?

I think we need to benchmark more than 1 time, we need to do it several times, and every benchmark we need to reason why the benchmark is running faster / slower vs previous one.

Maybe it's running faster because now it's using the cache from previous run.

Maybe it's running slower because it's the first time running the benchmark, making it need to prepare the cache.

### Is it OK to track the overall performance of a program with function-level unit tests?

No, function level unit test is not enough to check overall performance, especially unit test framework usually have some kind of overhead that makes the program runs slower.

### Does your organization have a performance regression system in place? If yes, can it be improved? If not, think about the strategy of installing one. Take into consideration: what is changing and what isn’t (source code, compiler, hardware configuration, etc.), how often a change occurs, what is the measurement variance, what is the running time of the benchmark suite, and how many iterations you can run.

To be honest, I can't answer for this one, sorry.
