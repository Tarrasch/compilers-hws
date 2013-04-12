% Compiler Optimizations, hw2 CS 6241
% Arash Rouhani
%

## Question I

### An EBB finding algorithm (a)

Here is a polynomial time algorithm. Familiarity with the disjoint set data structure
is assumed.

  1. Partition the basic blocks into disjoint sets such that each basic block
     is singleton. We're next going to merge the singletons to form extended
     basic blocks.
  2. For every block
     i. Look at its successors. Would merging the current basic block's set
     with the successors set make a valid EBB? If yes then merge them.
  3. The disjoint sets now form the members of each EBB.

#### Correctness

The set of returned BBs will be all of the BBs, clearly. Also the sets that
form the EEB's definitely are valid by definition. It remains to
show that these EEB's are maximum. But if they aren't, then there must be two
extended blocks $E_1$ and $E_2$ that can be merged. But then step two would
have included it when looking at the edge crossing the two EBBs, so it is
indeed maximum too.

### Exploiting the structure of basic blocks (b)

Basic blocks are nice because they only have one block who has more than one
join edge. That means that is always the entry point and that it will be the
most visited block (during execution) among the other blocks. For the examples
below I'll use a `for` loop since essentially it'll have the structure of a
EBB as long as there is no iteration in it (selection is ok though).

#### Constant propagation

Consider

      for(i = 1..1000){
        int b;
        int a = 5;

        b = a + 5;
        if(b > 13){
          b = 20;
        } else {
          b = 0;
        }

        print(b);
      }

This can through simple constant propagation and `if(true)`-removals be
reduced to

      for(i = 1..1000){
        a = 5;
        b = 10;
        b = 0;
      
        print(0);
      }

Note that I didn't do the liveliness analysis to remove unused variables. On
the other hand if we just complicate our structure slightly. Replacing the
`if` with a `for`. Making backedges possible within the EEB (so it's no longer
a EEB anymore). We don't dare to do constant propagation, because we don't
know which execution path leads to the executed body. See the example:

      for(i = 1..1000){
        int b;
        int a = 5;

        b = a + 5;
        for(;b > 13;){
          b = 20;
        }

        print(b);
      }

This can only be reduced to

      for(i = 1..1000){
        int b;
        int a = 5;

        b = 10;
        for(;b > 13;){
          b = 20;
        }

        print(b);
      }

Since the block containing the `b > 13` part both can come from either a `b =
10` path or a `b = 20` path. As smart humans we can of course see that the
whole loop is not necessary, but it's not as easy for the compiler.

## Question II

I'm using a C style syntax because the motivation for this question is to do
analysis of code written by a programmer, which probably uses a high level
language as C.

### Defining and restating the problem (a)

> An uninitialized variable is a variable that isn't set but takes the value
> of random garbage memory. An unsound definition is a definition in the
> code that at least partially is based on an uninitialized variable.

So a *variable* can be uninitialized and a *definition* can be unsound.

Let's look at a few examples of code and then say how we determine these two
properties.

      ...
      int a;
      if(something)
        a = 5;
      else
        b = 5;
      end
      (*)
      ...

At the (\*) marker in the code above, it's clear that a is potentially
uninitialized and we'll say that it's just uninitialized even though it could
be if `something` evaluated to true. We do this to always be on the safe side. As
for soundness of the definitions for `a = 5` and `b = 5`. They are of course
sound as there even are no variables on the right hand side. Lets look at
another example

      ...
      int x;
      b = x;
      ...

Here `b = x` is unsound because `x` is uninitialized. `x` is uninitialized
because it never occurs on a left hand side of a sound definition.

      ...
      int x;
      x = 5;  (A sound definition with x on its LHS ==> mark x initialized)
      b = x;  (Another sound definition. Sound because x is initialized)
      ...

A clear pattern starts to emerge, let's formalize this!

#### Flow equations

Initially set all the "initialized bits" to `false`. $initialized(a) ==
true$ means that `a` is initialized and $sound(d) == true$ means that
definition $d$ is sound.
 
When we have a statement *S* of the form

      d: X = Y + Z

Perform this calculation:

$$
  initialized[X] = initialized[Y] \wedge initialized[Z]
$$

Or when thinking in terms of data "in and out" of a statement, it's possible
to give this definition

$$
  out[S] = (in[S] - \{X\}) \text{  when  }(in[S]_Y \wedge in[S]_Z)
$$

$$
  out[S] = (in[S] \cup \{X\}) \text{  otherwise  }
$$


Lets explore bit vectors further ...

### The bit-vector algorithms (b)

When $B_1 \dots B_n$ are the predecessors of $B$ and we want the
bit-vector/set for $B$:

$$
  in[B]= \cap_i^n in[B_i]
$$

This should make sense because we only claim a variable to be initialized is
all paths leading to it are considered to have initialized that variable. In
the case of $n=0$ we set $in[start] = \{\}$

#### Sound definitions

We can calculate all the sound definitions in the end when we are sure about
where which variable is sound. Any definition is sound if the variable being
defined is initialized right afterwards.

### Example run in CFG (c)

Using the equations from above applied to this figure:

![Figure](cfg.pdf)

\newpage

          in_abcd  out_abcd
      B1     0000      0000     
      B2     0000      0000
      B3     0000      0000
      B4     0000      0000


          in_abcd  out_abcd
      B1     0000      0001     
      B2     0000      0000
      B3     0000      0100
      B4     0000      0000


          in_abcd  out_abcd
      B1     0000      0001     
      B2     0001      1001
      B3     0001      0101
      B4     0100      0000


          in_abcd  out_abcd
      B1     0000      0001     
      B2     0001      1001
      B3     0001      0101
      B4     0001      0011


          in_abcd  out_abcd
      B1     0000      0001     
      B2     0001      1001
      B3     0001      0101
      B4     0001      0001 (We've reached the fixpoint)

We can see that this algorithm will terminate because it always gets more and
more ones. This example is fairly short, but demonstrates that for example the
join of $B2$ and $B3$ happens using intersection. Also, the bit of $c$ never
get set because $a$ is not ever certainly initialized in $B4$. But for the
assignment in $B2$ we show that $a$ is initialized because of $d$ is.

From this we can inspect the individual definitions to determine if they are
sound. Well, The definitions in $B1 \dots B3$ are sound because if we look at
the last out table, we see that the variable they set in it takes on value
$1$. So the first three definitions are sound but not the last ($B4).

### Concerns regarding aliasing (4)

Consider the program

      int a;
      int *p = &a;
      *p = 5;

Our current scheme will judge `a` to be uninitialized but it clearly isn't.
One dream we definitely can forget is too have it work perfectly with pointers
(unless we do initialization checks at run-time, of course). Just imagine how
complicated this would be for arrays! Also consider this program

      /* (declerations of a and b) */
      int *p;
      if(something){
        p = &a;
      }  else {
        p = &b;
      }
      *p = 5;

Here it's really hard to say which of `a` or `b` that is uninitialized.
Remember that there are multiple basic blocks involved here. Even worse,
imagine the program if we also allowed aliases like in C++. The program could
be that `int c, &a=c, &b=c;`. Then both would be initialized! So for the
program to not give false positives or negatives is very difficult.  But,
perhaps there is at least some static guarantees we can give?

One idea is to hard code some cases. Like for cases similar to the first code
example. Alternatively one can use heuristics. One could say `a` becomes
initialized just by the user using `&a`. Like in the example

      int d, m;
      divMod(7,3, &d, &m); (now d==2 and m==1)

However, I conclude that I at least can't see any trivial way to without
heuristics find uninitialized values when aliasing can be. Just like most
cases, aliasing ruins the program analysis!

## Question III

I used this as my reference for x86:
<http://www.cs.virginia.edu/~evans/cs216/guides/x86.html>.

### Uncertainty of branch targets (a)

We know for sure where to jump in cases like this:

      label: my_label
        ... // some code
        jmp my_label
     
Here we always jump to the same place that is hard coded. Occasionally we want
to jump to either of two locations, like in an if-else statement.

        cmp eax, ebx
        je on_true
        jmp on_false
      label: on_true
        ...
      label: on_false
        ...

Finally, we can jump to an arbitrary position.

        jmp eax

This could be considered an "efficient" way of handling case statements, but
it's probably a bad idea as it really makes it impossible to do CFG analysis
**anywhere** in the whole program!

### Constructing the CFG

  1. In the rare case of there being statements like `jmp eax` present, or
     `jmp` on any nonfixed value. Return one huge basic block for all the
     pogram code.
  2. By now, all the jumps jump to predetermined locations, we call these
     *labels*. Put a label at the programs entry point, Now, at each label,
     start a basic block and let it run until the next label or the program
     ends. The `call` instruction will also start a basic block.
  3. Now, for each `jmp` or `call`, draw a directed edge from the basic block
     that the `jmp` is in to the basic block that the destination label
     represents.

It's quite clear that this is safe. Let's demonstrate how it works with an
example.

      label start:
        ...
        je lab1
        jmp lab2

      label lab1:
        jmp lab2

      label lab2:
        jmp start
        jmp lab3

      label lab3:
        jmp lab2

Here we create 4 basic blocks with 6 edges.

### Safe edge pruning in the CFG (c)

One good idea is to identify all unreachable code and then never include the
edges that was added from jumps in the dead code. For example the `jmp lab3`
in the example above is unreachable, but it creates a edge `lab2 --> lab3`
without that edge being possible.

We formalize this idea and also extend it to apply the dead code checking
until fix point.

  1. Construct the safe CFG as the algorithm above
  2. Consider all code between a `jmp` and another label as unreachable.
     Remove the edges caused by unreachable code.
  3. Now, it might happen that we removed some actual labels and more
     unreachable code has been created, go back to (2.). If not, we're done.

With the exception of this method, I see no other simple and general way to do
more CFG edge pruning.

## Question IV

In this exercise I use `+` but I mean any binary operator, Let `typeof(a)
== typeof(b) == int` and `typeof(p) == int*`. Like for `+`, `int` could really
be any arbitrary type.

While the exercise doesn't explicitly mention the scheme. I assume that the
"bit" for `a+b` being set means that an **up-to-date** `a+b` is stored in a
register available for use. The whole set of actual up-to-date expressions is
denoted $CSE_A$ and the ones our scheme finds is $CSE_F$.

### A sound safety condition (a)

Naturally we have:

$$
  CSE_F \subseteq CSE_A
$$

Why? Because we must be on the safe side. If we would substitute an expression
with an incorrect value, that would be absolutely disastrous.

### Pointers (b)

In an expression like

      _ = a + b

We've originally set the "bit" to true. We know from this point and on what
`a + b` is and should it occur later we can apply CSE.

Furthermore, traditionally we unset the bit in either of 

      a = _

or

      b = _

But with pointers (or any kind of aliasing) involved we must be more careful.
For example in the expression

      *p = _

We don't really know what `p` is, it could be so that `p == &a`. So at *any*
expression of the form `*p = _` we must unset *all* bits. Notice the word
unset which in set terminology means to remove elements. And since we aim for
the invariant $CSE_F \subseteq CSE_A$, having less elements will never violate
it. To clarify this further, consider this example

      p = &a
      a = 5
      b = 3
      c = a + b     (we now set the bit for a+b)
      *p = 7
      d = a + b     (Unless we would have unset the bit for the last line, we
                     would substitute this with c, which is incorrect since
                     now a+b equals 10 and not 8)

### Demand Driven CSE analysis (c)

Consider this code:

      ...
      c = a + b         (That's great, we set the bit)
      a = 5             (No, we must now un set the bit)
      for(10000 times)
      {
         v = a + b      (We set the bit)
      }                 (Also, we never un set it!)
      ...

This situation is sad. We'll run the body of the loop 10000 times, the first
entrance is from the block above, but the rest 9999 are from the block itself.
The expression analysis in it's simplest form will not be able to do CSE here.
But it would have if the code was:

      ...
      c = a + b
      a = 5
      dummy = a + b     (Add a dummy assignment, just to set the bit)
      for(10000 times)
      {
         v = a + b
      }
      ...

Wow! That's how simple it can be, now you can do CSE in the loop basic block.
This simple source level transformation can always be done. Let's formalize it
a tiny bit.

> For every basic block that is a loop. Look for the CSE bits it does set. In
> the case that it's CFG-predecessors does not set the bit, add a dummy
> assignment in the end of those basic blocks just so set the bit.
