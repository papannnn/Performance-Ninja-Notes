# CPU Microarchitecture

## Instruction Set Architecture (ISA)

Instruction Set Architecture is a contract between software and hardware.

ISA defines the rules of communication, so software need to follow the rules to make the software runs correctly.

## Pipelining

Pipelining is a technique that makes CPU run fast.

From Computer Architecture book, CPU pipelining has 5 stages:

1. Instruction Fetch (IF)
2. Instruction decode (ID)
3. Execute (EXE)
4. Memory access (MEM)
5. Write back (WB)

```
                 Clock cycle
Instruction      1    2    3    4    5    6    7    8    9
─────────────────────────────────────────────────────────────
Instruction x    IF   ID   EXE  MEM  WB
Instruction x+1       IF   ID   EXE  MEM  WB
Instruction x+2            IF   ID   EXE  MEM  WB
Instruction x+3                 IF   ID   EXE  MEM  WB
Instruction x+4                      IF   ID   EXE  MEM  WB
```

In this diagram, shows the best case scenario when doing CPU pipelining.

Throughput is determined on how much number of instructions complete per unit of time.

Latency is determined on total time for instruction X finished from start to finish.

The time required for a stage move to another stage is determined on the clock cycle.

The value of clock cycle is determined on the slowest stage in the pipeline.

### Pipeline Hazard

In pipelining, there's some thing that prevent pipelining can work ideally, it's called Pipeline Hazard.

Pipeline Hazard divided into 3 categories.

#### Structural Hazard

Caused by resource conflict. For example 2 instructions are competing same resource.

For example there's only one machine that can handle addition instruction, but there's 2 instruction want that machine, that means only 1 instruction can be served, another instruction must wait (stall).

#### Data Hazard

Caused by data dependencies, it also divided into 3 categories:

- **Read After Write (RAW)**

Assuming and instruction like this

```
R1 = R0 ADD 1
R2 = R1 ADD 2
```

Because CPU pipeline execute it like this

```
Cycle       1    2    3    4    5    6
-----------------------------------------
I1          IF   ID   EXE  MEM  WB
I2               IF   ID   EXE  MEM  WB
```

That means there's no way R2 get the new value on R1 yet. Because R1 still not yet finished writing, so one approach is to stall the pipeline


```
Cycle       1    2    3    4    5    6    7
-----------------------------------------------
I1          IF   ID   EXE  MEM  WB
I2               IF   ID   --   --   EXE  MEM  WB
```

Another approach is to do bypassing, instead of waiting the result being written from ALU to R1, why not just use the value on ALU directly? This will improve the speed of pipelining.

- **Write After Read (WAR)**

First of all, to fill more context. In modern CPU there's called Out of Order execution, that means it doesn't guarantee instruction will be executed from line 5, 6, 7, 8, etc by order.

As long the result is same, CPU can execute it out of order 8, 5, 6, 7.

Now assume the instruction is like this

```
I1: R1 = R0 ADD 1
I2: R0 = R2 ADD 2
```

Assuming because of some kind of hazard, `I1` got stalled

```
Cycle       1    2    3    4    5    6
------------------------------------------
I1          IF   ID   --   --   EXE(reads R0)  WB(R1)
I2               IF   ID   EXE  WB(writes R0)
```

This is unacceptable, if this cycle happens, it will return different value

There's some ways to fix this, the simple one is by doing stalling.

```
Cycle       1    2    3    4    5    6    7    8
--------------------------------------------------------
I1          IF   ID   --   --   EXE  MEM  WB
I2               IF   ID   --   --   --   EXE   MEM  WB(writes R0)
```

But stalling will sacrifice the performance since CPU do nothing at stall.

Another approach is to do register renaming, instead of waiting for the read to finish, hardware will use another register to store old value.

```
R101 = R100 ADD 1
R103 = R102 ADD 2
```

- Write After Write (WAW)

When more than 1 instruction writing to the same address, but if Out of Order execution happen, it may writing a wrong value to that address

```
I1: R1 = R0 ADD 1
I2: R2 = R1 SUB R3
I3: R1 = R0 MUL 3
```

As you can see, R1 is being written in `I1` and `I3`, what if the `I3` finished first then `I1` finished last? It will causing wrong data written to that address.

We can handle it by stalling the `I3`

```
Cycle:      1    2    3    4    5    6    7    8    9
------------------------------------------------------------
I1: R1=R0+1      IF   ID   EXE  MEM  WB
I2: R2=R1-R3          IF   ID   --   --   EXE  MEM  WB
I3: R1=R0*3                IF   --   --   ID   EXE  MEM  WB
```

And we can handle it by doing register renaming

```
R101 = R100 ADD 1
R102 = R101 SUB R103
R104 = R100 MUL 3
```

#### Control Hazard

It's caused by program flow, for example the `if` `else` `jump`.

The instruction already go inside the pipelining, but because the flow is changing because of the `if` `else` `jump`, that means we don't do that branch anymore.

Some techniques to handle this including `Dynamic Branch Prediction`,  `Speculative Execution`.

## Exploiting Instruction Level Parallelism (ILP)

### Out of Order (OOO) Execution

Most modern CPU support out of order execution, that means sequential instruction can enter execution stage in any order as long it doesn't conflict with the dependency and the availability of the resource.

CPU with OOO execution still need to give same result like if the instruction execution is executed orderly.

The reason we use OOO execution is to avoid underutilization of CPU resources due to stall caused by dependencies.

```
Instruction       1    2    3    4    5    6    7    8    9   10
---------------------------------------------------------------------
Instruction x     IF   ID   EXE  MEM  WB
Instruction x+1        IF   ID        EXE  MEM  WB
Instruction x+2             IF   ID   EXE  MEM        WB
Instruction x+3                  IF   ID        EXE  MEM       WB
```

Assuming `x+1` can't be executed at cycle time 4 due to some conflict. If this not using OOO execution, instruction `x+2` will do execution at cycle no 7 because of stall.

Even though the execution can be out of order, the retirement of instruction still be in order, as you can see the write back is done in order and `x+2` need to stall to wait `x+1` finish doing write back.

The process of reordering instruction is called *instruction scheduling*, instruction scheduling can be done in compile time (static scheduling) or runtime (dynamic scheduing).

```
I1: LOAD R1, [A]    ; slow
I2: ADD R2, R1, R3  ; depends on I1
I3: MUL R4, R5, R6  ; independent
```

```
Program order:
I1 → I2 → I3

Execution order:
I1 → I3 → I2
```

### Static scheduling

Most of the modern CPU is not using Static Scheduling anymore, basically the compiler is responsible to think which instruction should go first and making sure the instruction didn't get any hazard.

```
Compiler
   ↓
decides instruction timing
   ↓
CPU executes the schedule
```

### Dynamic scheduling

Two most important algorithm in Dynamic Scheduling is Scoreboarding & Tomasulo Algorithm.

```
                 Runtime
                    │
                    ▼
             ┌─────────────┐
             │   CPU sees  │
             │ instructions│
             └──────┬──────┘
                    │
          ┌─────────┴─────────┐
          │                   │
      Can execute?         Dependency?
          │                   │
         YES                  WAIT
          │                   │
          ▼                   │
       Execute ◄──────────────┘
```

### Superscalar Engines

Most of modern CPU are superscalar, that means it can issue more than one instruction in any given cycle.

The maximum number of instructions that can be issued during same cycle is called *Issue Width*.

```
+-----------------+----+----+-----+-----+-----+----+
|                 |         Clock cycle            |
|   Instruction   +----+----+-----+-----+-----+----+
|                 |  1 |  2 |  3  |  4  |  5  |  6 |
+-----------------+----+----+-----+-----+-----+----+
| Instruction x   | IF | ID | EXE | MEM | WB  |    |
+-----------------+----+----+-----+-----+-----+----+
| Instruction x+1 | IF | ID | EXE | MEM | WB  |    |
+-----------------+----+----+-----+-----+-----+----+
| Instruction x+2 |    | IF | ID  | EXE | MEM | WB |
+-----------------+----+----+-----+-----+-----+----+
| Instruction x+3 |    | IF | ID  | EXE | MEM | WB |
+-----------------+----+----+-----+-----+-----+----+
```

In this example, the issue width is 2. That's why in clock cycle 1, there's 2 `IF` running at the same time.

### Speculative Execution

Control hazard can cause significant performance in pipelining because it it need to wait to see which branch is getting resolved, that means it will having a stall.

One technique to avoid this is to use *branch prediction*.

Basically, CPU predicts the likely target of branch and start executing that branch without knowing is the branch got resolved or not.

```c++
if (a < b)
    foo();
else
    bar();
```

Assuming the code is like this above.

```
+------------------+----+----+-----+-----+-----+-----+-----+-----+----+
|                  |                   Clock cycle                  |
|   Instruction    +----+----+-----+-----+-----+-----+-----+-----+----+
|                  |  1 |  2 |  3  |  4  |  5  |  6  |  7  |  8  |  9 |
+------------------+----+----+-----+-----+-----+-----+-----+-----+----+
| BRANCH (a<b)     | IF | ID | EXE | MEM | WB  |     |     |     |    |
+------------------+----+----+-----+-----+-----+-----+-----+-----+----+
| CALL foo         |    |    |     | IF  | ID  | EXE | MEM | WB  |    |
+------------------+----+----+-----+-----+-----+-----+-----+-----+----+
| // Instr from foo|    |    |     |     | IF  | ID  | EXE | MEM | WB |
+------------------+----+----+-----+-----+-----+-----+-----+-----+----+
```

If not using branch prediction, the CPU need to stall the pipeline first to find out which branch the instruction is going to.

```
+------------------+----+-----+-----+-----+-----+-----+-----+----+----+
|                  |                   Clock cycle                  |
|   Instruction    +----+-----+-----+-----+-----+-----+-----+----+----+
|                  |  1 |  2  |  3  |  4  |  5  |  6  |  7  |  8 |  9 |
+------------------+----+-----+-----+-----+-----+-----+-----+----+----+
| BRANCH (a<b)     | IF | ID  | EXE | MEM | WB  |     |     |    |    |
+------------------+----+-----+-----+-----+-----+-----+-----+----+----+
| CALL foo         |    | IF* | ID* | EXE | MEM | WB  |     |    |    |
+------------------+----+-----+-----+-----+-----+-----+-----+----+----+
| // Instr from foo|    |     | IF* | ID  | EXE | MEM | WB  |    |    |
+------------------+----+-----+-----+-----+-----+-----+-----+----+----+
```

Now if using branch prediction, CPU can "guess" `foo` instruction will be called and the stall can be avoided.

If the prediction is correct, it will saves a lot of cycles. But sometimes prediction can be wrong, instead it should call `bar` function.

That means, CPU need to throw away the `foo` related instruction on pipeline, this is called `branch misprediction penalty`

### Branch Prediction

Correct prediction will saves a lot of cycles, but wrong prediction will causing costly penalties.

There are 3 type of branching:

#### Unconditional jumps and direct calls

This is easiest the predict since there's no another branch to choose

#### Conditional branches

Two potential outcomes, taken or not taken. Like in `if-else` statement or in `loop`

#### Indirect calls and jumps

Can be generated in `switch` statement, function pointer, or virtual function call.

#

Most prediction algorithm is based on the past outcomes of the branch.

```
BPU (Branch Prediction Unit)
│
├── Direction Predictor
│     └── Taken / Not Taken
│
├── BTB
│     └── Target Address
│
└── RAS
      └── Return Address
```

Inside Branch Prediction Unit (BPU), there's a hardware data structure called Branch Target Buffer (BTB).

```
BTB
┌─────────────┬──────────────┐
│ Branch PC   │ Target Addr  │
├─────────────┼──────────────┤
│ 0x1000      │ 0x5000       │
│ 0x2000      │ 0x8000       │
│ 0x3000      │ 0x3500       │
└─────────────┴──────────────┘
```

BTB's purpose is to cache the target addresses for every branch, so assuming there's branching happen in address `0x1000`, after some "searching", the branching will go to address `0x5000`, BTB will "take a note" of that pair.

Next time, when Prediction algorithm want to check where's the `0x1000` will go branch, it just need to check on BTB cache instead of "searching" it again.

Every cycle, BPU will try to query target address from BTB. BPU can actually just "search" from instruction decoding, but it need to finish the decoding stage first, making the pipeline stall.

For conditional branches, CPU need to predict is the branch is taken or not, is the branch predicted to be taken, it will directly go to BTB and get the target address, if not, it will do fall through (searching from instruction decoding).

All prediction try to exploit 2 important principles:

- Temporal Corelation (Local Corelation)

If previously the branch got taken, then there's high chance the same branch will be taken again.

- Spatial Corelation (Global Corelation)

If branch A got taken, then after branch A there's branch B got taken, then after branch B, there's branch C, then there's high chance branch C got taken also.

#

Another trick is by doing hydrid prediction. If the branch always do 99.9% taken, then we don't need to use BTB because it's only polluting it's data structure.

### SIMD Multiprocessors

This is another technique to improve performance, Single Instruction Multiple Data (SIMD). 

It operates many data elements in single cycle. Operation on vector or matrices is usually a good place because it's usually can be processed using same instruction.

```c++
double *a, *b, *c;
for (int i = 0; i < N; ++i) {
    c[i] = a[i] + b[i];
}
```

```
  SCALAR MODE                    SIMD MODE

  [ a1 ]                    [ a1 | a2 | a3 | a4 | a5 | a6 | a7 | a8 ]
     +                                        +
  [ b1 ]                    [ b1 | b2 | b3 | b4 | b5 | b6 | b7 | b8 ]
     =                                        =
  [ c1 ]                    [ c1 | c2 | c3 | c4 | c5 | c6 | c7 | c8 ]

  1 op = 1 result           1 op = 8 results

  Register width used:
  128-bit  [ a1 | a2 ]                         (2 lanes)
  256-bit  [ a1 | a2 | a3 | a4 ]                (4 lanes)
  512-bit  [ a1 | a2 | a3 | a4 | a5 | a6 | a7 | a8 ]  (8 lanes)
```

Let's compare scalar mode vs SIMD mode, scalar mode only can handle `ax + bx` double data type in one cycle, but if we're using SIMD code that can handle 256 bit vectors, it can do 4 double data type in one go.

Assuming the data size is 5, and we're using CPU that support SIMD code that can handle 256 bit vector, that means in one go, it can only handle 4 data. We can process the first 4 data, but the last element should be processed individually. This is called loop remainder.

Loop remainder means portion of loop that less than the SIMD width, it need scalar code to handle this.

Another solution is to use loop masking, to enable / disable the SIMD lanes based on condition.

Some use cases for SIMD:

- String processing: finding characters, validating UTF-8, parsing JSON and
CSV
- Hashing, random generation, cryptography(AES);
- Columnar databases (bit packing, filtering, joins);
- Sorting built-in types (VQSort, QuickSelect);
- Machine Learning and Artificial Intelligence (speeding up PyTorch, TensorFlow).

### Exploiting Thread-Level Parallelism

To improve more performance, we can exploit thread level parallelism, Thread-Level Parallelism divided into 3 categories:

#### Multicore Systems

The idea of Multicore Processor is to make your computer can run multiple process at the same time, for example listening to music while you do browse internet.

But this is not a free performance, each core generates heat, that means if there's 10 cores that running 10 computer process. It will create 10x more heat in your computer. In some situation, multicore processor also reduces clock speed.

It's harder to add more cores because each cores shares resources with another cores, they shares resources using memory bus for example, when a lot of cores using memory bus, memory bus will become a bottleneck.

#### Simultaneous Multithreading

Another approach is to do Simultaneous Multithreading (SMT), people also call this hyperthreading.

SMT basically running multiple software in same core.

The multiple software can be anything, can be another thread in same process, can be another process also.

```
     Non-SMT                      SMT2
   issue slots                 issue slots
  ┌───┬───┬───┬───┐          ┌───┬───┬───┬───┐
1 │███│   │   │   │        1 │███│▒▒▒│   │   │
  ├───┼───┼───┼───┤          ├───┼───┼───┼───┤
2 │███│███│   │   │        2 │███│███│▒▒▒│▒▒▒│
  ├───┼───┼───┼───┤          ├───┼───┼───┼───┤
3 │   │   │   │   │        3 │▒▒▒│▒▒▒│▒▒▒│   │
  ├───┼───┼───┼───┤          ├───┼───┼───┼───┤
..│███│███│███│   │       .. │███│███│███│   │
  ├───┼───┼───┼───┤          ├───┼───┼───┼───┤
  │   │   │   │   │          │▒▒▒│   │   │   │
  ├───┼───┼───┼───┤          ├───┼───┼───┼───┤
  │███│   │   │   │          │███│▒▒▒│▒▒▒│   │
  ├───┼───┼───┼───┤          ├───┼───┼───┼───┤
  │███│███│███│███│          │███│███│███│███│
  ├───┼───┼───┼───┤          ├───┼───┼───┼───┤
  │███│███│   │   │          │███│███│▒▒▒│▒▒▒│
  └───┴───┴───┴───┘          └───┴───┴───┴───┘
   cycles ↓                   cycles ↓

  Legend:
  ███  = thread 1
  ▒▒▒  = thread 2
  (blank) = unused slot
```

The reasoning of SMT because in superscalar system, the CPU core has multiple execution unit (ALU, load / store unit, FPUs, etc). That means, if we're only doing single threaded application, there's high chance some execution unit being idle, wasting some performance there.

Although the two program running in the same core, they are completely separated with each other, having each different context to maintain the correctness of the program running.

SMT give a burden to developer due to unpredictable nature, because of that this topic is not a priority.

#### Hybrid Architectures

In Hybrid Architecture, in one processor usually have more than 1 type of core, for example the big core and small core. Big core can handle heavier task, small core is more energy friendly.

### Memory Hierarchy

CPU memory hierarchy is build with these 2 fundamentals:

- Temporal Locality

If you access this data, most likely in the future you're gonna access this again, so CPU will try to cache this data so you can get the data more faster in the future.

- Spatial Locality

If you access this data, most likely in the future you're gonna access data around this data. So CPU will try to cache data around this data.

#### Cache Hierarchy

Cache is an organized small, fast storage block that lives near execution unit. Because it's lives near execution unit, the retrieval can be fast. Not like RAM that lives far away from execution unit.

The bigger the cache, the slower it can be accessed.

Cache are organized as a block with defined size called cache line, usually it's 64B, but other processor like Apple has 128B for the L2 cache. L1 cache usually have size of 32KB to 128KB. Mid level cache usually has 1MB or above.

#### Placement of Data within the Cache

Address that being used to access data in memory is being used as a key to access the cache.

- Direct Mapped Caches

In direct-mapped caches, a block address can only appear in one location in the cache

Number of Blocks in the Cache = Cache Size / Cache Block Size

Direct mapped location = (block address) mod (Number of Blocks in the Cache )

Direct mapped caches is simple to make and fast access since you just need to mod the block address and check that location, but it has high miss rate because another data with the same mod value can overwrite the previous data in cache.

- Fully Associative Cache

There's another approach, using fully associative cache, basically block address can be placed in any location in cache, but need high hardware complexity and impractical.


- Set Associative Mapping

An intermediate approach is to do set-associative mapping. In the cache, blocks are organized as sets, each set contains 2, 4, 6, 8 or 16 blocks.

In a set, the address can be placed anywhere.

Number of Sets in the Cache = Number of Blocks in the Cache / Number of Blocks per Set (associativity)

Set (m-way) associative location = (block address) mod (Number of Sets in the Cache)

#

Let's have an example, let's assume we have L1 cache that having:

- a size of 32KB
- cache line 64B
- 64 sets
- associativity of 8 blocks.

That means, we have:

- 32KB / 64B = 512 Lines

A new cache line can only be inserted in appropriate set in one of the 8 block available.

Let's have another example:

For Apple M1 processor. L1 data cache each core has:

- a size of 128KB
- cache line 64B
- 256 sets
- associativity of 8 blocks

For L2 data cache in Apple:

- a size of 12MB
- associativity of 12 blocks
- cache line 128B

With this info, we can get the amount of set

- Set = Cache Size (12MB) / (Associativity (12) * Line Size(128)) = 12,582,912 / 1,536 = 8,192

#### Finding Data in the Cache

```
+--------------------------------------------------------+-------------+
|                     Block Address                      |   Block     |
|                                                        |   offset    |
+---------------------------+----------------------------+-------------+
|            Tag            |            Index           |             |
+---------------------------+----------------------------+-------------+
```

Here's an example on how the data got fetched in cache.

Assuming:

- Address is `0x123403232`
- Set 8192
- Associativity of 12
- Cache line of 128B

Calculate:

- Address bit = `000100100011010000000011001000110010` 36 bit
- Because cache line is 128B = log(128) = 7 bit for offset
- Because set 8192 = log(8192) = 13 bit for index
- Tag bit = 36 bit - 7 - 13 = 16 bit

Split into fields

```
| TAG (16 bits)       | INDEX (13 bits)      | OFFSET (7 bits) |
| 0001001000110100    | 0000001100100        | 0110010         |
| = 0x1234            | = 100 (decimal)      | = 50 (decimal)  |
```

Intepret the field:

Index = 100, means it will go to set no 100.

Offset = 50, means once we get the block, we go to offset 50 on that cache line

Because in 1 set, there's 12 block, it will parallel check all 12 block, check is any block has the tag `0x1234`.

If yes, read byte on offset 50

If no, fetch data from memory or L3 cache, put the data into set no 100, put the tag as `0x1234`

#### Managing Misses

If there's cache misses:

- Direct mapped cache

Because direct mapped cache can only go into single location, that means the previous entry will be overwritten with the new data.

- Set Associative Cache

Because cache block can be put anywhere as long in the right set, replacement algorithm is required, usually we use LRU policy, another way is choose randomly.

#### Managing Writes

CPU use two basic mechanism to handle write in cache:

- Write Through Cache

If CPU want to write something on address `0x1234`, it will write to L1 cache, then L2, then L3, then main memory.

- Write Back Cache

If CPU want to write something address `0x1234`, it will write to L1 cache only, but set dirty bit on L1, to tell this need to write into another cache / memory.

When that `0x1234` need to be evicted in L1, look at the dirty bit first, if dirty bit is true, that means CPU need to write to L2 first, set as dirty bit true, then evict L1. And so on if CPU want to evict L2.

#

Cache misses in write operation can be handled with two way:

- Write Allocate Cache

When CPU want to write something but the data is not even in the cache, it will traverse the data to the below hierarchy until go to main memory, then fill the data to all cache. Then update the data in the cache and put dirty bit on L1

```
CPU writes X
   │
   └──→ L1: MISS
           │
           └──→ L2: MISS
                   │
                   └──→ L3: MISS
                           │
                           └──→ Fetch block from Main Memory
                                     │
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                      ▼
        Allocate in L3         Allocate in L2         Allocate in L1
        (block placed here)   (block placed here)    (block placed here,
                                                        then CPU's write
                                                        applied, dirty=1)
```

What if the data found on L3?

```
CPU writes X
   │
   └──→ L1: MISS
           │
           └──→ L2: MISS
                   │
                   └──→ L3: HIT!  (data found here, no memory access needed)
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
        Allocate in L2            Allocate in L1
        (copy placed here)       (copy placed here,
                                   write applied, dirty=1)
```

- No write allocate policy

If data that CPU want to write is not on cache, just write it directly on main memory.

#

Most design usually choose write-back cache with write allocate policy.

Write through caches typically use no write allocate policy

#### Other Cache Optimization Techniques

Average Access Latency = Hit Time + Miss Rate * Miss Penalty

Tldr, cache miss will make pipeline stall, we want to minimize cache miss as much as possible.

Miss rate is dependent to cache architecture (block size, associativity) and software.

#### Hardware and Software Prefetching

One way to avoid cache misses is to prefetch the data and let the data go into the cache first before we start the application.

Most CPU has implicit prefetching, and developer can also do manual software prefetching to complement.

Hardware prefetching look at the behavior on the program and do the prefetching when it see some pattern for cache misses.

But hardware prefetching is very limited, that's why developer also need to do software prefetching.

### Virtual Memory

Virtual memory basically a way to represent a physical memory to a program in computer.

It's also provide a protection so the program didn't access memory that doesn't intended, for example accessing another program memory.

To help manage the physical memory, it divided into pages.

Pages will be saved in page table with the translation of their physical address.

```
                        Virtual address (64 bit)
        +----------------------------+----------------------+
        |   virtual page number      |     page offset      |
        |         (52 bits)          |      (12 bits)       |
        +----------------------------+----------------------+
                     |                          |
                     |                          |
                     v                          |
             +---------------+                  |
             |               |                  |
             |  Page table   |                  |
             |               |                  |
             +---------------+                  |
                     |                          |
                     | Physical address         |
                     |      (64 bit)            |
                     v                          v
             +---------------------------------------+
             |                                       |
             |             Main memory               |
             |                                       |
             +---------------------------------------+
```

Failure to query the physical address from page table is called page fault, this is because the page is invalid or the translation is not in main memory.

#### Translation Lookaside Buffer (TLB)

Searching the physical address through page table can be an expensive process, to minimize the long process, we use TLB.

TLB is like a cache for Virtual Address to Physical Address.

#### Huge Page

Having huge page that means reducing the TLB pressure because less caching needed since we didn't fetch a lot of Virtual Memory, but having huge page means higher chance to have memory fragmentation.

## Question and Exercises

- Describe pipelining, out-of-order, and speculative execution

Assume pipelining like a laundry system.

There's washing state, rinse state, drying state, ironing state, folding state.

When there's 4 person want to do laundry system, person 2 doesn't need to wait for person 1 to finish the folding state first in order to join the laundry system. Person 2 just need to wait until person 1 finished the washing state and go to rinse state.

This method will make sure no one wasting their time.

Out of Order execution means, instruction in pipeline can be executed not in order, as long the final result still the same like in order execution. This help to improve the performance of pipeline incase some instruction got stalled due to hazard.

Speculative Execution is a way to prevent Control Hazard by guessing which branch will go into pipeline, instead of waiting the branch got resolved in execution phase.

- How does register renaming help to speed up execution?

Because of Data Hazard, especially in Write after Read (WAR) and Read after Read (RAR). Especially in OOO execution, there's a chance the output of the instruction can be wrong if we're not doing stalling in pipeline.

But we don't want to do stall in pipeline because stalling means sacrificing the performance.

- Describe spatial and temporal locality

Spatial: If I access this, there's a chance I will access data around here in the future

Temporal: If I access this, there's a chance I will access this data again in the future.

- What is the size of the cache line in the majority of modern processors?

64B

- Name the components that constitute the CPU frontend and backend

Idk, skipping this

- What is the organization of the 4-level page table?

Idk, skipping this

- What is a page fault?

When you can't access the translation of Virtual address to Physical Address, usually the translation is being put in the hard disk because of swapping happened.

- What is the default page size in x86 and ARM architectures?

4KB (?)

- What role does the TLB (Translation Lookaside Buffer) play?

To cache the Virtual Address to Physical address mapping, since querying the Physical address through the page table is more heavier computation.
