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