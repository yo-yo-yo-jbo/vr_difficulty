# Why vulnerability discovery is difficult mathematically
With the recent news of folks finding vulnerabilities [left](https://en.wikipedia.org/wiki/Copy_Fail) and [right](https://chromereleases.googleblog.com/2026/05/stable-channel-update-for-desktop.html) using LLMs, some folks hope that we'd be able to find every single one vulnerability.  
Today, I hope to shatter that idea. This post will be slightly more mathematical than usual, but it doesn't require much background.

## Background - on programs and Turing Machines
In the real world, we use programming languages to express ideas. Those, in turn, eventually run code that runs on your CPU (via compilation, interpretation etc.) - each programming language might express similar ideas but run different instructions.  
Moreover, there are different CPUs and instruction sets, so comparing all different languages and architectures is quite difficult - especially when we want to discuss types of vulnerabilities and runtimes.  
The frmaework used in universities is called a **Turing Machine**, and it's a computation model that represents any "reasonable" programmable computer.

Without getting into the rigorous definition, that machine is defined as:
1. A strip of **tape**, divided into **cells**, that streches from index 1 (leftmost) to infinity. That represents the working memory of the machine. We further define a set of **symbols** - think of those as the supported character set of the tape.
2. A **head** that can read or write symbols on the tape cells, and move the tape left or right (one cell at a time).
3. A **state register**, which saves the current "state" of the machine. There are only a finite number of states for a Turing Machine. Initially, that state register has a special value - the "start stage", which is also a part of the Turing Machine definition.
4. A finite **transition table** - which for every state and symbol currently pointed by the **head**, tells the machine what symbol to write to the current cell, what new state the **state register** should be assigned to, and where to move the head (left, right or stay where it is).

This simplistic model is equivalent to any programming language, and is a convenient model for proofs about runtime complexity (since it's programming-language structure independent and architecture independent).
