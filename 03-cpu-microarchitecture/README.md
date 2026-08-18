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