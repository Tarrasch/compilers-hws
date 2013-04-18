% Compiler Optimizations, hw3 CS 6241
% Arash Rouhani (902951864)
%

## Q1 -- An indepedent loop

Consider this loop:

    for i = 0..1 (inclusive)
      for j = 0..1 (inclusive)
        A[101*i] = A[j*99 + 100]
      end
    end

Call the LHS indexing $\alpha = 101 i_1$ and RHS $\beta = 99 j_2 + 100$. We can
in fact table both these guys for all 4 iterations.


  Iteration      $\alpha$         $\beta$
  ---------     -----------      ---------
  (0, 0)        0                100
  (0, 1)        0                199
  (1, 0)        101              100
  (1, 1)        101              199

Clearly there is on dependence as all $\alpha$s are different from the
$\beta$s. Ok, but now the question is if GCD or Banerjee will say *may* for this
loop. Lets check that in that order:

### GCD test

`gcd(101, 99) ==> 1` and 1 divides all so the GCD test is useless here because
1 divides everything. GCD therfor reports *may*.

### Banerjee test

Ok lets calculate low and high.

  * `low  = (101*i_1 - 99*j_2 - 100) = -199`
  * `high = (101*i_1 - 99*j_2 - 100) = 1`

And 0 is within the range `[-199..1]` so Banerjee will also report *may*.

## Q3 -- A paper on register allocation

#### (1) The problem

One new idea of compilers is to have compilation during runtime and in
particular the JVMs come to mind. Traditionally it has not been a priority to
lower compile times as it is done offline, however it's important nowadays due
to the popular usage of Just In Time compilers.

#### (2) Why important and the improvements

Since compilation now happens during runtime, all the time runtime is compiling
will be experienced by the one running the program. Hence, reducing running
times is important. As we know compilation consists of many phases and one of
them is Register Allocation (RA). A good RA is crucial for performence and this
paper shows a nice way of doing a good RA fast.

#### (3) Assumptions and scope of solution

The setting is to take register based IR code which have unlimited registers
and synthesizing that code and allocating physical registers to these.
Registers are a very scarce resource on a regular CPU.

A stack based machine (like JVM) will eventually also eventually need to do
register allocation. The paper does not address for instance if the algorithm
would work for stack based machines.

#### (4) The key techniques

The fact is that `total runtime = executing time + compilation time` for a JIT
compiler. Now, Theare are two famlies of RA methods have existed before, the
*linear scan* methods compile fast meanwhile the *graph coloring* methods
produce code that executes fast. The paper presented a graph coloring method
but modified to also compile decently fast.

The new method recallibrate's it's graph structure faster because the previous
graph coloring algorithms always rebuilt the graph once it became out of date.
The new method kinda only adjusts the parts of the graphs that are necessary.
While it cannot do this exactly it's always estimating on the safe side.

#### (5) Results

From the results I see that their method work as they claim. I'm happy to see
they had a graph with the `total time` as I equated above. I was also impressed
to see that even though the signifcant %3 increase in spills, the actual
programs got slowed down with a smaller proportion (when ignoring compilation
times of course). According to the authors, the reason was that the more used
code got allocation more similar to the old graph coloring algorithms.

#### (6) What the paper leaves open

Meanwhile i symphatize with their choice of going with the LLVM infrasturcture
becuase it's well documented. I would like to see an implementation and see if the
results are as promising on a JVM implementation.

#### (7) Future theoretical work

I choose to give a bullet list of ideas I have:

  * Make a meta JIT, there is no reason to run the very same RA algorithm for
    all functions. Two sub ideas are

      * Make a super quick static analysis of the bytecode and choose RA
        algorithm based on that.
      * Or, even better I believe, start using the fastest compiling method
        first, then when the function gets called more frequently, reoptimize
        with a RA algorithm that produces faster code. This is very demand
        driven thinking.

  * While building the graph still is the bottleneck, the faster build step
    makes the other phases int the graph coloring to be more significant. Maybe
    one can optimize the spilling step in the next paper?
