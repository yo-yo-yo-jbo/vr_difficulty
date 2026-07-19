# Why vulnerability discovery is mathematically difficult
With the recent news of folks finding vulnerabilities [left](https://en.wikipedia.org/wiki/Copy_Fail) and [right](https://chromereleases.googleblog.com/2026/05/stable-channel-update-for-desktop.html) using LLMs, some folks hope that we'd be able to find every single vulnerability.  
Today, I hope to shatter that idea. This post will be slightly more mathematical than usual, but it doesn't require much background.

## Vulnerabilities and computability
In this section we will discuss the computability aspect of vulnerability discovery.

### Background - on programs and Turing Machines
In the real world, we use programming languages to express ideas. Those, in turn, eventually run code that runs on your CPU (via compilation, interpretation etc.) - each programming language might express similar ideas but run different instructions.  
Moreover, there are different CPUs and instruction sets, so comparing all different languages and architectures is quite difficult - especially when we want to discuss types of vulnerabilities and runtimes.  
The framework used in universities is called a **Turing Machine**, and it's a computation model that represents any "reasonable" programmable computer.

Without getting into the rigorous definition, that machine is defined as:
1. A strip of **tape**, divided into **cells**, that stretches from index 1 (leftmost) to infinity. That represents the working memory of the machine. We further define a set of **symbols** - think of those as the supported character set of the tape.
2. A **head** that can read or write symbols on the tape cells, and move the tape left or right (one cell at a time).
3. A **state register**, which saves the current "state" of the machine. There are only a finite number of states for a Turing Machine. Initially, that state register has a special value - the "start state", which is also a part of the Turing Machine definition.
4. A finite **transition table** - which for every state and symbol currently pointed by the **head**, tells the machine what symbol to write to the current cell, what new state the **state register** should be assigned to, and where to move the head (left, right or stay where it is).
5. A set of states that are considered to be **final states**. When the machine transitions to such a state (using the **transition table**) we say that the machine **halts**.

Initially, the **tape** is given some **input** (from our set of allowed **symbols**). When the machine halts, everything written on the tape to the left of the **head** is considered the machine's **output**.

This simplistic model is equivalent to any programming language, and is a convenient model for proofs about runtime complexity (since it's programming-language structure independent and architecture independent).  
One of the most important properties of Turing Machines is that they can run other Turing Machines that they get as inputs - that makes them **Turing Complete**.  
One other important aspect is that all programs can be represented by some natural number - just like your day-to-day programs can be represented as text files on disk, and those are just bytes that could be represented by (commonly huge) numbers. We will refer to this property as the **encoding** of the Turing Machine.

Note: throughout this blogpost I will be using the terms "Turing Machine", "Program" and "Algorithm" interchangeably, with the understanding that all of those models are equivalent.

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
2. Every **cell** in our **tape** can now be marked as either `Unallocated`, `Free` or `Used`. Initially, all **cells** that have inputs are `Used`, and all the **cells** to the right of the input are `Unallocated`.
3. We introduce an **arithmetic register** with both **overflow flag** and an **underflow flag**. Both are initially not set.
4. We extend the **transition table** by allowing each transition to emit a special **meta-instructions**:
  1. $alloc\(i\)$ - marks **cell** with index $i$ as `Used`.
  2. $free\(i\)$ - marks **cell** with index $i$ as `Free`.
  3. $read\(i\)$ - marks that **cell** with index $i$ was meant to be read by our **head**.
  4. $write\(i\)$ - marks that **cell** with index $i$ was meant to be written by our **head**.
  5. $add\(x, y\)$ - declares the intention of adding **symbol** $x$ and $y$ - if $x+y$ is greater than $2^{w}-1$ (where $w$ is the **word length**) then we mark an `overflow` in our **arithmetic register**.
  6. $sub\(x, y\)$ - declares the intention of subtracting **symbol** $x$ and $y$ - if $x-y$ is less than zero then we mark an `underflow` in our **arithmetic register**.

Thus, we can define the following vulnerability classes:
1. When $free\(i\)$ occurs, if the state of **cell** indexed $i$ is `Free` then we say we have a **double free vulnerability**.
2. When $read\(i\)$ occurs, if the state of **cell** indexed $i$ is `Unallocated` then we have an **out-of-bounds read vulnerability**.
3. When $write\(i\)$ occurs, if the state of **cell** indexed $i$ is `Unallocated` then we have an **out-of-bounds write vulnerability**.
4. When $read\(i\)$ occurs, if the state of **cell** indexed $i$ is `Free` then we have a **use-after-free read vulnerability**.
5. When $write\(i\)$ occurs, if the state of **cell** indexed $i$ is `Free` then we have a **use-after-free write vulnerability**.
6. When $add\(x, y\)$ occurs, if `Overflow` is set then we have an **integer overflow vulnerability**.
7. When $sub\(x, y\)$ occurs, if `Underflow` is set then we have an **integer underflow vulnerability**.
8. If the Turing Machine never halts, we define it as a **denial-of-service vulnerability**.

Of course, that model could be further extended with more fancy ALU operations, but the idea would be the same.

### Vulnerability discovery is undecidable
We can perform a **reduction** from any of the different vulnerability classes we've defined to the **Halting Problem**, with ease:
1. Given a **Turing Machine** $M$, we will construct a **Vulnerability Turing Machine** $Q$.
2. Machine $Q$ simulated $M$ on the computational **tape** only, emitting no special **meta-instructions**.
3. If $M$ halts then we produce exactly one violation (e.g. a **double free vulnerability**).
4. Otherwise, we exit normally (this step actually never really happens since $M$ is simulated "all the way" and gets $Q$ stuck, but note there are no vulnerabilities of any kind during this simulation).

Now it should be easy to see that if we have an algorithm $Alg$ that gets a **Vulnerability Turing Machine** $Q$ and indicates whether it has any vulnerabilities, it'd be able to solve the **Halting Problem** as well - if we have a vulnerability then the original $M$ must have halted, otherwise we know that the original $M$ never halts.

Note this is a special case of a larger theorem called [Rice's theorem](https://en.wikipedia.org/wiki/Rice%27s_theorem), which specifies that any problem that gets Turing Machines as inputs which indicates a non-trivial property of those machines - is undecidable, where we define "trivial" as "all machines have that property" or "no machine has that property". It uses a very similar technique for proof.  
Since I defined my own new class of Turing Machines, Rice's theorem doesn't immediately work for them, but in essence it'd be the same proof altogether.

## Vulnerabilities and complexity
One thing computer scientists like to do is look at subset of problems - in our case, we will examine "efficient" Turing Machines and see how vulnerability discovery fares against them.

### Background - time complexity
The topic of time complexity analysis is both deep and broad, but in essence, we can measure the time it takes for a Turing Machine to complete a certain task (assuming the Turing Machine halts!) by simply counting how many steps it takes to get to a final state.  
The measurement traditionally used for that is to see how much time it takes for a Turing Machine to solve the problem, as the input of the problem grows.  
For example, [binary search](https://en.wikipedia.org/wiki/Binary_search) is a well-known problem in which a program gets a sorted array and finds an element in that array (or declaring the element does not exist).  
As the size of the input array grows **linearly**, the time it takes for the program grows **logarithmically** - if the input has $n$ elements, the time it takes would be $c_1\log_{2}\(n\) + c_2$, where those $c_1$ and $c_2$ constants are independent of $n$. As $n$ grows large, those constants will become less significant, and thus we mark the time complexity as $O\(\log_{2}\(n\)\)$.

### Background - P and NP
An important thing computer scientists like to do is examine "efficient" Turing Machines, or more precisely - examine **problems** for which an "efficient" Turing machine exists.  
The term "efficient" is quite vague, and one common definition is that the time complexity is a **polynomial** (or better).  
For example, a time complexity of $3n^4 + 5n + 7$ is really $O\(n^4\)$, which is a polynomial (and thus considered "efficient"), while $O\(2^n\)$ is not a polynomial (in fact, it's called **exponential**) is thus not considered efficient.  
When we talk about problems and efficiency, we define two important classes of problems:
- $P$ - these are the set of problems with a known Turing Machine that can **find** an efficient solution (in polynomial time).
- $NP$ - these are the set of problems with known Turing Machines that can easily **verify** if a solution is valid or not, to a certain problem, efficiently (in polynomial time).  
An example of that could be factoring large numbers - if we are given a number and a potential divisor, it's quite easy to check if that's a valid solution (performing that division is linear in time), but it's currently unknown whether we could find a divisor for a large number efficiently (note that would break [RSA](https://en.wikipedia.org/wiki/RSA_cryptosystem), for example).
There is a well-known problem called [P versus NP](https://en.wikipedia.org/wiki/P_versus_NP_problem) which asks whether those two classes are the same - currently it's an open mathematical problem (and I expect it to stay that way for a very long time). Most computer scientists assume that $P \neq NP$.

### Background - NP-completeness
An important subset of problems under $NP$ is called [NP-complete](https://en.wikipedia.org/wiki/NP-completeness) or $NPC$ for short.  
Those problems are problems in $NP$ but also proven to be "the hardest NP problems", in a sense that solving one of them can be used to solve any NP problem (without losing the efficiency aspect!), by simulating the desired NP problem.  
One such problem is the [Boolean_satisfiability_problem](https://en.wikipedia.org/wiki/Boolean_satisfiability_problem) (commonly referred to as $SAT$), which asks whether a certain formula of Boolean variables can be satisfied by assigning `false` or `true` to those variables.  

### Vulnerability discovery is NP-complete
Interestingly, we can reduce $SAT$ to any of our vulnerability discovery problems.  
Given a Boolean formula $\varphi$ with variables $x_1, x_2, ... x_n$, we can build a **Vulnerability Turing Machine** that gets input $x$ (of length $n$ symbols) and:
1. Scans $\varphi$ clause by clause and computes $\varphi\(x\)$, emitting no **meta-instructions**.
2. If $\varphi\(x\)$ is `true`, cause a vulnerability (e.g. `out-of-bounds`).
3. Otherwise, halt without a violation.

Note that this machine runs in $O\(\varphi\)$ time complexity on every input, so it's polynomial time.  
Also note if $\varphi\$ is in $SAT$ and $a$ is the assignment that satisfies it, then the machine reaches step 2 when given the input $a$ and causes a violation. Conversely, if the machine has a violation, then there exists some input that causes the violation - that input, when interpreted as an assignment to $SAT$, satisfies it.

## Implications for practical systems

### Static analyzers
[Static analyzers](https://en.wikipedia.org/wiki/Static_program_analysis) reason about all possible executions without running the program. Since detecting any non-trivial vulnerability property is undecidable, no static analyzer can be simultaneously sound (no false negatives), complete (no false positives), and terminating on all
inputs.

### Fuzzers
[Fuzzers](https://en.wikipedia.org/wiki/Fuzzing) generate concrete inputs and observe program behavior. Each execution confirms or refutes the presence of a violation on one input, but says nothing about other inputs.  
The undecidability of vulnerability discovery implies that a fuzzer can confirm a bug (by finding it) but can never rule one out.  
Coverage-guided fuzzers improve efficiency by prioritizing novel paths but face path explosion: the number of distinct execution paths grows without bound as a function of program size. Fuzzing therefore operates as a probabilistic sampler of the execution space, not as a decision procedure.

### Symbolic execution
[Symbolic execution](https://en.wikipedia.org/wiki/Symbolic_execution) encodes program paths as logical formulas and uses constraint solvers to find bug-triggering inputs.  
This is more systematic than fuzzing - it can in principle explore all paths - but suffers from path explosion for exactly the same reason.  
Bounding exploration (by path depth, loop unrolling count, or solver timeout) makes the problem decidable but NP-complete, and thus hypothesized to be difficult from a time complexity perspective.

### Runtime instrumentation
Tools such as [AddressSanitizer](https://github.com/google/sanitizers/wiki/addresssanitizer) instrument programs to detect violations dynamically on executed paths.  
This corresponds to running the **Vulnerability Turing Machine** simulation on a specific input: given $x$, it is always possible to detect whether the machine exhibits a violation on $x$ (by simulating).  
Run-time tools correctly detect violations when they occur on the tested input, but cannot certify absence across all inputs.

## Some thoughts about LLMs
[LLMs](https://en.wikipedia.org/wiki/Large_language_model) are not a new computational model - they orchestrate the techniques we described above (generating fuzzing harnesses, driving symbolic execution tools, triaging static-analyzer output, reading sanitizer traces) and combine them with a learned prior over what vulnerable code *looks like*.

That prior is genuinely useful. The vast majority of real-world vulnerabilities are not novel - they are yet another off-by-one, yet another use-after-free on a familiar lifecycle pattern, yet another [TOCTOU](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use). LLMs are very good at recognizing the *shape* of a known vulnerability class in unfamiliar code - a skill that previously required an experienced auditor - and this is a real capability that should not be dismissed.

However, the results we've shown still apply:
1. An LLM is itself a Turing-equivalent computation, so it cannot decide a problem we've already proven undecidable. There will always be programs for which it cannot tell us "vulnerable or not" with certainty.
2. On bounded programs, the NP-hardness result means an LLM searching for triggering inputs is approximating a search over an exponential space, not solving it. High confidence is not the same as coverage.
3. Pattern-matching has a recall ceiling: an LLM finds bugs that resemble its training distribution. Novel logic bugs, specification-mismatch bugs, and bugs in unusual protocols or codebases remain systematic blind spots.

What actually changes with LLMs is throughput, cost, and accessibility. The time-to-find for known-shape bugs is collapsing for both attackers and defenders, but the residual population of undiscovered bugs - the deep, semantic, novel ones - has roughly the same shape as before.

A related caveat: benchmarks based on curated CVEs or CTF challenges measure how well an LLM recognizes known shapes, not how close it is to solving vulnerability discovery in general. A high score there is evidence of pattern-matching capacity, not of completeness.

## Summary
In this blogpost, I tried to share some analysis on why vulnerability research is truly a difficult problem, even in a toy-model such as a Turing Machine.  
The recent explosion in vulnerability discovery and exploitation via LLMs is mostly an engineering breakthrough (which should not be underestimated!) but does not yield new technical abilities - in my view, it mostly commoditizes the framework that has been used until this point by experts and ad-hoc (e.g. fuzzing harnesses).

Stay tuned!

Jonathan Bar Or (https://jonathanbaror.com)

