% Compiler Optimizations, hw3 CS 6241
% Arash Rouhani (902951864)
%

## Constant propagation

I have made up a scheme that is pretty much like the one in the book, but also
handles phi functions. First consider the simplest of of the $\phi$-function.

![Illustration of phi function in a simple merge](./constant-just-phi.pdf)

Here, we can think of a variable, like `a1`, having three different kinds of values:

  * `NAC` - Not A Constant
  * A constant - say we only deal with integers, so then it would be a
    particular integer instance, like 3, 7, etc.
  * `Undefined` - We don't know what it is, it is of course one of the two above
    but we don't know which.

Same goes for `a2` of course. Given this mindset, `a3` is totally trivial:

  * If any of them (`a1`/`a2`) is `undefined`, then `a3` has the value of the other value
  * If any of them (`a1`/`a2`) is `NAC`, then `a3` is `NAC` too
  * If both of them is the same constant, then `a3` is that constant too, otherwise it's NAC

This should be very intuitive. Handling of operators like `+` should be
like how the book handles the non-SSA case.

### Efficiency and example walk through

Given the constrained (and therfor simplified) IR that SSA provides, one
wonders if the complexity can be improved. I claim it's still $O(N^2)$.
I'll walk us through how the algorithm handles figure 2, which should
make motivate my claim.

![An example that is walked through in the text](./constant-many-phi.pdf)

Here I'll describe my algorithm, by telling how it would go about
cracking the figure. Here goes:

Initially, we'll say all of the `a`s are undefined, then we traverse the
basic blocks in a dependency graph fashion, hey `a1` is the constant 17.
Let's propagate that to `a2` and `a3`, note how those are uninfluenced
by `a4` and `a5` since they are `undefined` to begin with. Now we've
visited all of the nodes, but we're not done yet. Instead, first `a4`
ignites a reestimation of `a3`, now `a3` didn't change so we don't need
to propagate down any changes here. But as for when `a5` ignites the
reestimation of `a2`, it will realize that `a2` actually is `NAC`, then
`a2` must go on and update all of it's dependencies, `a3` that is, which
also must update it's dependencies, however it has none.

Indeed, we realized that `a2` and `a3` are `NAC` whilst the others are
actual constants (hence correctness). The hunch of the complexity is $O(n!)$ which is way
worse than quadratic. But it's calming to realize that the variables,
say `a2` in our case. Will at most only be changed at most twice! Once a
value is `NAC` it will remain `NAC` and not make any recomputations.  In
the worst case it will be a constant and then `NAC`. so at worst each
phi function can traverse all other code twice. If we're really timing
it bad it will end up only being $O(n^2)$ though it'll probably be
near-linear for all practical purposes.

## Available Expression

Available Expression analysis is a joy with SSA. There is no kill set!
That's fantastic, we can expect a higher availability now. Remember, if
we say we believe that `a3+b4` is available, nothing like `a3 = ...` nor
`b4 = ...` will spoil the party since we basically have SSA.

Also, unlike for constant propagation, we don't see $\phi$-assignments
like $a5 = \phi(a2, a4)$ like any special kind of assignment. We simply
just *generate* when we see something like `a1 + b1`. Now, just treat
SSA version of the Available Expression flow problem as the original
one, that is let it be a *forward&intersection* flow problem.

### Efficiency

Unlike for constant propagation, we'll never need to recalculate
anything, as what we'll consider available won't suddently be
overwritten (unless it's just marked as available for the first time).
With this in mind, there is no ignitions, so one looping through the
blocks in a dependecy graph traversal fashion should be enough and we
conclude linear time, $O(n)$.

## Very Busy Expression

Again we have no such thing as an kill set as expressions are never
reassigned.

I even claim that there is no need for flow analysis here given that
variables are never used uninitialized. While that assumption isn't
perfectly reasonable, I'm going to go ahead and present a interesting
non-flow approach to solving this.

  1. For any expression that is used multiple times (and thus always can
     be hoisted, because of SSA). Let's say `a + b`
  2. Now find the definition of `a` and `b`, as we assumed the blocks
     which where they are defined must dominate all blocks using `a +
     b`, that way we know `a` and `b` are defined. Put the statement `t
     = a + b` after the definition of whichever of `a` and `b` that are
     defined latest. Here `t` is any temporary. Now replace all the
     other occurences of `a + b` with `t`.

This will work, but as I said earlier only under the assumption
variables are always defined once they are used. Let's see this in this
example series:

![Before appling VBE optimization](./busy-before.pdf)

![After appling VBE optimization](./busy-after.pdf)

I'll now walk us through the VBE figures (figure 3 and figure 4). We'll
try to hoist the expression `a + b` and we'll ultimately succeed. To
begin, we obsereve that our assumption is fulfilled, namely, that `b`
isn't defined in `B2` nor `B3` as they don't dominate the usage blocks
`B5` and `B6`. So now we compare the definition points of `a` and `b`,
`b` clearly is defined after `b`, so put the new `t = ...` statement
after the definition of `b` and use `t` where it's now possible.
