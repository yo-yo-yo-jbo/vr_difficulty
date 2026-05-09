# Why vulnerability discovery is mathematically difficult
With the recent news of folks finding vulnerabilities [left](https://en.wikipedia.org/wiki/Copy_Fail) and [right](https://chromereleases.googleblog.com/2026/05/stable-channel-update-for-desktop.html) using LLMs, some folks hope that we'd be able to find every single one vulnerability.  
Today, I hope to shatter that idea. This post will be slightly more mathematical than usual, but it doesn't require much background.

## Vulnerabilities and computability
In this section we will discuss the computability aspect of vulnerability discovery.

### Background - on programs and Turing Machines
In the real world, we use programming languages to express ideas. Those, in turn, eventually run code that runs on your CPU (via compilation, interpretation etc.) - each programming language might express similar ideas but run different instructions.  
Moreover, there are different CPUs and instruction sets, so comparing all different languages and architectures is quite difficult - especially when we want to discuss types of vulnerabilities and runtimes.  
The frmaework used in universities is called a **Turing Machine**, and it's a computation model that represents any "reasonable" programmable computer.

Without getting into the rigorous definition, that machine is defined as:
1. A strip of **tape**, divided into **cells**, that streches from index 1 (leftmost) to infinity. That represents the working memory of the machine. We further define a set of **symbols** - think of those as the supported character set of the tape.
2. A **head** that can read or write symbols on the tape cells, and move the tape left or right (one cell at a time).
3. A **state register**, which saves the current "state" of the machine. There are only a finite number of states for a Turing Machine. Initially, that state register has a special value - the "start stage", which is also a part of the Turing Machine definition.
4. A finite **transition table** - which for every state and symbol currently pointed by the **head**, tells the machine what symbol to write to the current cell, what new state the **state register** should be assigned to, and where to move the head (left, right or stay where it is).
5. A set of states that are considered to be **final states**. When the machine transitions to such a state (using the **transition table**) we say that the machine **halts**.

Initially, the **tape** is given some **input** (from our set of allows **symbols**). When the machine halts, everything written on the tape to the left of the **head** is considered the machine's **output**.

This simplistic model is equivalent to any programming language, and is a convenient model for proofs about runtime complexity (since it's programming-language structure independent and architecture independent).  
One of the most important properties of Turing Machines is that they can run other Turing Machines that they get as inputs - that makes them **Turing Complete**.  
One other important aspect is that all programs can be represented by some natural number - just like your day-to-day programs can be represented as text files on disk, and those are just bytes that could be represented by (commonly huge) numbers. We will refer to this property as the **encoding** of the Turing Machine.

Note: throughout this blogpost I will be using the terms "Turing Machine", "Program" and "Algorithm" interchangably, with the understanding that all of those models are equivalent.

### Background - the Halting Problem
People who just started to code might think that coding is very powerful, in a sense that it can calculate every concievable function, given enough time and resources (remember that we have an infinitely long **tape**!).  
That is false - for those of you who are familiar with the concept of [Mathematical Cardinality](https://en.wikipedia.org/wiki/Cardinality) it'd be easy - the set of possible Turing Machines is of $\aleph_0$, but the set of functions from $\mathbb{N}$ to $\mathbb{N}$ is $2^{\aleph_0}$, so "most" functions are not computable.  
However, even if you are not familiar with those concepts, there is a famous problem that was proven to be **undecidable** - the **Halting Problem**.  
There are several variations of the **Halting Problem**, but we'll describe the easiest to understand.  
Our programming challenge is this: we are given as input **another Turing Machine**, and are supposed to indicate whether that Turing Machine halts (stops) or gets stuck in an infinite loop, on the empty input.  
The Halting Problem is known to be undecidable, i.e. there's no Turing Machine that returns `false` or `true` for every input, while not getting stuck in an infinite loop itself.  
The proof for that is quite simple, and done by contradiction:
1. We assume such a magical algorithm exists - $Alg\(M)$, which always stops and returns `true` if $M$ halts on the empty input, and `false` if it does not halt.
2. We construct a new Turing Machine $Q\(M\)$ which does the following:
  1. Runs $Alg\(M\)$.
  2. If the result was `false` - exits immediately.
  3. If the result was `true` - gets stuck in an infinite loop on purpose.
3. Now we have the encoding of $Q$, we run $Q\(Q\)$ and ask ourselves - does it halt or not?
  1. If it halts then $Alg\(Q\)$ must have returned `false`, so we have a contradiction between what really happened and the result of $Alg$.
  2. If it doesn't halt then $Alg\(Q\)$ must have returned `true`, and again we have a contradiction.
4. Conclusion: $Alg$ cannot exist, and the **Halting Problem is undecidable**.

The cool thing about the **Halting Problem** is that it can be used as an anchor to prove other problems are undecidable.  
If we can show that solving some computational problem solves the Halting Problem, then that computational problem, by extension, is also undecidable. We call that technique **reduction** and it's used very commonly in theoretical computer science.

### Defining vulnerability classes with Turing Machines
Now that we understand what Turing Machines are - we can define several classes of vulnerabilities, by extending the definition of our Turing Machine.  
Very simply, we can say that a **Vulnerability Turing Machine** (a **VTM**) is defined just like a normal Turing Machine, but with the following changes:
1. We define the **word length** of our **symbols** as the number of bits it takes to represent our **symbols**.
2. Every **cell** in our **tape** can now be marked as either `Unallocated`, `Free` or `Used`. Iniitally, all **cells** that have inputs are `Used`, and all the **cells** to the right of the input are `Unallocated`.
3. We introduce an **arithematic register** with both **overflow flag** and an **underflow flag**. Both are initially not set.
4. We extend the **transition table** by allowing each transition to emit a special **meta-instructions**:
  1. $alloc\(i\)$ - marks **cell** with index $i$ as `Used`.
  2. $free\(i\)$ - marks **cell** with index $i$ as `Free`.
  3. $read\(i\)$ - marks that **cell** with index $i$ was meant to be read by our **head**.
  4. $write\(i\)$ - marks that **cell** with index $i$ was meant to be written by our **head**.
  5. $add\(x, y\)$ - declares the intension of adding **symbol** $x$ and $y$ - if $x+y$ is greater than the **word length** then we mark an `overflow` in our **arithematic register**.
  6. $sub\(x, y\)$ - declares the intension of adding **symbol** $x$ and $y$ - if $x-y$ is less than zero then we mark an `underflow` in our **arithematic register**.

Thus, we can define the following vulnerability classes:
1. When $free\(i\)$ occurs, if the state of **cell* indexed $i$ is `Free` then we say we have a **double free vulnerability**.
2. When $read\(i\)$ occurs, if the state of **cell* indexed $i$ is `Unallocated` then we have an **out-of-bounds read vulnerability**.
3. When $write\(i\)$ occurs, if the state of **cell* indexed $i$ is `Unallocated` then we have an **out-of-bounds write vulnerability**.
4. When $read\(i\)$ occurs, if the state of **cell* indexed $i$ is `Free` then we have an **use-after-free vulnerability**.
5. When $write\(i\)$ occurs, if the state of **cell* indexed $i$ is `Free` then we have an **use-after-free vulnerability**.
6. When $add\(x, y\)$ occurs, if `Overflow` is set then we have an **integer overflow vulnerability**.
7. When $sub\(x, y\)$ occurs, if `Underflow` is set then we have an **integer underflow vulnerability**.
8. If the Turing Machine never halts, we define it as a **denial-of-service vulnerability**.

Of course, that model could be further extended with more fancy ALU operations, but the idea would be the same.

### Vulnerability discovery is undecidable
We can perform a **reduction** from any of the different vulnerability classes we've defined to the **Halting Problem**, with ease:
1. Given a **Turing Machine** $M$, we will construct a **Vulnerability Turing Machine** $Q$.
2. Machine $Q$ simulated $M$ on the computational **tape** only, emitting no special **meta-instructions**.
3. If $M$ halts then we produce exactly one violation (e.g. a **double free vulnerability**).
4. Otherwise, we exit normally (this step actually never really happens since $M$ is simulated "all the way" and gets $Q$ stuck, but note there are no any vulnerabilities in any kind during this simulation).

Now it should be easy to see that if we have an algorithm $Alg$ that gets a **Vulnerability Turing Machine** $Q$ and indicates whether it has any vulnerabilities, it'd be able to solve the **Halting Problem** as well - if we have a vulnerability then the original $M$ must have halted, otherwise we know that the original $M$ never halts.

Note this is a special case of a larger theorem called [Rice's theorem](https://en.wikipedia.org/wiki/Rice%27s_theorem), which specifies that any problem that gets Turing Machines as inputs which indicates a non-trivial property of those machines - is undecidable, where we define "trivial" as "all machines have that property" or "no machine has that property". It uses a very similar technique for proof.  
Since I defined my own new class of Turing Machines, Rice's theorem doesn't immediately work for them, but in essence it'd be the same proof altogether.

## Vulnerabilities and complexity
One thing computer scientists like to do is look at subset of problems - in our case, we will examine "efficient" Turing Machines and see how vulnerability discovery fares against them.

### Background - complexity classes
The topic of time complexity analysis is both deep and broad, but in essence, we can measure the time it takes for a Turing Machine to complete a certain task (assuming the Turing Machine halts!) by simply counting how many steps it takes to get to a final state.  
The measurement traditionally used for that is to see how much time it takes for a Turing Machine to solve the problem, as the input of the problem grows.  
For example, [binary search](https://en.wikipedia.org/wiki/Binary_search) is a well-known problem in which a program gets a sorted array and finds an element in that array (or declaring the element does not exist).  
As the size of the input array grows **linearly**, the time it takes for the program grows **logarithmically** - if the input has $n$ elements, the time it takes would be $c_1\log_{2}\(n\) + c_2$, where those $c_1$ and $c_2$ constants are independent of $n$. As $n$ grows large, those constants will become less significant, and thus we mark the time complexity as $O\(\log_{2}\(n\)\)$
