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
