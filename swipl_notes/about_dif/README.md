# About `dif/2`

A description of predicate `dif/2` can be found [here](https://eu.swi-prolog.org/pldoc/doc_for?object=dif/2), but it is a bit short!

Let's try add some explanations.

## TL;DR

After a `dif(A,B)` call, any unification involving `A` or `B` or subterms thereof 
that would make `A` and `B` identical (`A == B`) will **fail** (and this applies to unification in clause heads, too). 
If `A` and `B` are already identical at call time, `dif(A,B)` fails at once.

### From the SWI-Prolog manual

The text in SWI-Prolog's [`library(dif)`](https://eu.swi-prolog.org/pldoc/doc/_SWI_/library/dif.pl) says:

> Among the most important coroutining predicates is dif/2, which expresses disequality of terms in a sound way.
> The actual test is delayed until the terms are sufficiently different, or have become identical.

The corresponding text for the [`dif/2`](https://eu.swi-prolog.org/pldoc/doc/_SWI_/library/dif.pl) constraint says (remarks in [] by me):

> Constraint that expresses that `Term1` and `Term2` [shall] never become identical (==/2). 
> Fails if `Term1 == Term2`. Succeeds if `Term1` can never become identical to `Term2` [at call time]. 
> In other cases the predicate succeeds after attaching constraints to the relevant parts of `Term1` and `Term2`
> that prevent the two terms to become identical.

## Some history

From: [Indexing `dif/2`](https://arxiv.org/abs/1607.01590), Ulrich Neumerkel and Stefan Kral, 2016-06-06:

> The very first Prolog, sometimes called _Prolog 0_ (Colmeraueret al. 1973) already supported `dif/2`.
> Unfortunately, the popular reimplementation _Prolog I_ (Battani and Meloni 1973) omitted `dif/2`and
> other  coroutining  features.  This  system  was  the  basis  for  Edinburgh  Prolog (Pereira et al. 1978)
> which led to ISO-Prolog (ISO/IEC 13211-1 1995). After _Prolog I_, `dif/2` was reintroduced in 
> _Prolog II_, independently reinvented in _MU-Prolog_ (Naish 1986). and soon implementation schemes to
> integrate `dif/2` and coroutining into efficient systems appeared (Carlsson 1987; Neumerkel 1990).
> The  major  achievement was that the  efficiency of general Prolog programs not using `dif/2`
> remained unaffected within a system supporting `dif/2`. In this manner `dif/2` survived in major
> high-performance implementations like _SICStus_. 
>
> However, it still has not gained general acceptance
> among programmers. We believe that the main reason for this lack of acceptance is that `dif/2` does
> not directly deliver the abstraction that is actually needed. Its direct use leads to clumsy and
> unnecessarily inefficient code. Its combination with established control constructs often leads to
> unsound results. New, pure constructs are badly needed.

From: [An implementation of _dif_ and _freeze_ in the WAM](http://www.softwarepreservation.org/projects/prolog/sics/doc/Carlsson-SICS-TR-86-12.pdf) (PDF), Mats Carlsson, 1986-12-18:

> Prolog II [Colmerauer 82a,82b], has received much attention for its theoretical model as a term
> rewriting system in the domain of infinite trees, but also for its ability to delay certain predicates
> until enough information is available: _dif(X,Y)_, read as "_X_ and _Y_ are different terms", and
> _freeze(X,P)_ , read as "delay _P_ until _X_ has been instantiated".
> 
> Similar primitives have subsequently been included into various implementations, for example
> LM-Prolog [Carlsson 83], MU-Prolog [Naish 85a], and NU-Prolog [Naish 86]. The concept of
> delay primitives is related to the idea of coroutines which were first introduced in Logic
> Programming with IC-Prolog [Clark 79], [Clark 80]. It is tightly coupled to the concept of
> constraints [Steele 80]. A study of the introduction of the constraint concept into Logic
> Programming is given in [Dincbas 86].
>
> The usefulness of delay primitives is widely recognized. By adding a data-driven component to a
> language which otherwise is goal-driven, new programming techniques become available. They
> help solve certain semantical problems, in particular by allowing a sound treatment of negation.
> MU-Prolog, for example, supply sound versions built on delays for arithmetic, negation, and
> if_then_else. They can increase the efficiency of "generate and test" programs and even prevent
> infinite loops. They can be used for simulating and-parallelism by coroutining. Certain uses of
> delay primitives cause new yet unsolved semantical problems, for example when used for
> simulating perpetual processes computing with infinite streams. These problems are being attacked
> in a scheme called Constraint Logic Programming (CLP) [Jaffar 86], and some very promising
> results have already been reported.

...

> **3.2 Dif in Prolog II**
>
> Superficially, Prolog II treats _dif_ by a completely different mechanism. Prolog II replaces
> unification by the solving of systems of equations and inequations over infinite trees, using two
> algorithms called _reduction_ and _simplification_ [Colmerauer 84]. _dif(X,Y)_ is defined as adding the
> inequation _X≠Y_ to the system, and this has nothing to do with the freeze mechanism. However,
> the simplification algorithm, which is invoked whenever the system is augmented, can select certain
> inequations for "reprocessing", an operation that is tantamount to waking a delayed goal.

## Explain!

Consider the call `dif(A,B)`.

A program that issues this call is in one of three states regarding `A` and `B` when it does so:

- **Possibly different** - In this state, `A` and `B` would unify, i.e. `A=B` would succeed. 
  However, `A` and `B` are still sufficiently unrefined/uninstantiated, that subsequent program
  operations may change that situation. `A=B`, if issued later, might fail. Example: `A=f(X),B=f(Y)`. Note that `\+ A==B`.
- **Different** - In this state, `A` and `B` won't unify and no subsequent operation will change this.
  When or once the program is in state "different", it will stay there (unless it sufficiently backtracks). 
  Example: `A=f(x),B=f(y)`. Note that `\+ A==B`.
- **Identical** - In this state, `A` and `B` will unify and term identity `A==B` will actually succeed: `A` and `B` are identical.
  No subsequent operation can change this. `A` and `B` may be nonground, and in that case, any variables in the terms appear in
  the same position and are shared. When or once the program is in state "identical", it will stay there 
  (unless it sufficiently backtracks). Example: `A=f(x),B=f(x)` or `A=f(C),B=f(C)`. Note that `A==B`. 
  
The behavious of `dif/2` in all three states is the following:

- **Possibly different**: `dif(A,B)` succeeds "optimistically" and a `dif/2` constraint is activated/posted/opened, 
  to watch for changes to `A` or `B`.
- **Different**: `dif(A,B)` succeeds deterministically.
- **Identical**: `dif(A,B)` fails immediately.

If we are still in state **possibly different** on return to the Prolog toplevel (i.e. when the program terminates), the still-active
constraints are printed out. Whether `dif(A,B)` is actually true would be an philosophical & epistemological conundrum
at this point: it's neither confirmed, nor denied. The printed constraint term does not necessarily resemble what was stated in the `dif/2` call, 
but it is equivalent.
  
The interesting case occurs if a `dif(A,B)` has been issued in state **possibly different** and a refinement 
through unification of `A` or `B` is requested, either in a goal or a head. The refinement may involve just a
subterm of `A` or `B`. 

It may be that the unification would cause a constraint violation: it would unifcation make `A` and `B` identical. 
This is resolved by **failing the unification which violated the constraint**. A way to think about this is that, after
a successful `dif(A,B)` call, Prolog makes sure that `A` stays different from `B` and fails every attempt that tries to
make them identical: `dif(A,B)` should be be read _"fail any attempts to make A and B identical after this point"_.

### What happens when `dif/2` is called

![dif call](pics/dif_call.svg)

### What happens after `dif/2` has been called

![unifications after dif call](pics/unifications_after_dif_call.svg)

## Some examples

First a predicate `check/2` which prints out information about the current situation. Note that the way to test `A=B` without retaining any
changes to either `A` or `B` is the call `\+ \+ A=B` or equivalently `\+ A\=B`.

```prolog
check(X,Y) :- 

   % "\+ \+" ensures consequence-less unification test
   ((\+ \+ (X=Y)) -> writeln("> They unify") ; true),          

   % X\=Y is already consequence-less
   ((X\=Y) -> writeln("> They don't unify") ; true),    

   % Term equivalence/identity (not unification! Terms denated by A and B are not refined by ==)
   ((X==Y) -> writeln("> They are identical") ; true),

   % Term difference/non-identity
   ((X\==Y) -> writeln("> They are different") ; true).  
```

### Behaviour of `dif/2` call

Issue a `dif(A,B)` in state _possibly different_

Note that the SWI Prolog toplevel prints the active constraint the end (suitable rearranged;
the `f` function symbol is used in `dif/2` constraint expressions and has no relationship to 
any `f` function symbol used in the program):

```text
?- A=g(x,X),B=g(Y,y),writeln("possibly different"),check(A,B),
   dif(A,B),
   writeln("END OF GOAL").

possibly different
> They unify
> They are different
END OF GOAL
A = g(x, X),
B = g(Y, y),
dif(f(Y, X), f(x, y)).
```

Issue a `dif(A,B)` in state _different_

```text
?- A=g(x),B=g(y),writeln("different"),check(A,B),
   dif(A,B),
   writeln("END OF GOAL").

different
> They don't unify
> They are different
END OF GOAL
A = g(x),
B = g(y).
```

Issue a `dif(A,B)` in state _identical_

```text
?- A=f(x),B=f(x),writeln("identical"),check(A,B),
   dif(A,B),
   writeln("END OF GOAL").

identical
> They unify
> They are identical
false.
```

### Behaviour after `dif/2` call

Issue a `dif(A,B)` in state _possibly different_, then attempt to make `A` and `B` identical by unification
operations. This triggers the active constraint and the unification fails.

```text
?- A=g(x,X),B=g(Y,y),writeln("possibly different"),check(A,B),
   dif(A,B),
   X=y,writeln("X unified with y"),
   Y=x,writeln("Y unified with x").

possibly different
> They unify
> They are different
X unified with y
false.
```

### The `dif/2` constraint stays active even if the involved terms go out of scope

A call to `p(X,Y)` constructs terms `A` and `B`, of which `X` and `Y` are subterms.
A `dif/2` constraint is then set up between `A` and `B`.

`A` and `B` go out of scope on return from `p(X.Y)` and the terms denoted by them become
unreachable. The `dif/2` constraint, however, correctly persists!

```prolog
p(X,Y) :- 
   A = f(X,y), 
   B = f(x,Y), 
   dif(A,B),
   writeln("dif/2 passed").

t1(X,Y) :- p(X,Y), 
           X=x, writeln("X is now x"), 
           Y=y, writeln("Y is now y, which should have failed").

t2(X,Y) :- p(X,Y),
           X=y, writeln("X is now y, so the constraint is closed").

t3(X,Y) :- p(X,Y), 
           writeln("the constraint remains active").
```

Then

```text
?- t1(X,Y).
dif/2 passed
X is now x
false.
```

```text
?- t2(X,Y).
dif/2 passed
X is now y, so the constraint is closed
X = y.
```

```text
?- t3(X,Y).
dif/2 passed
the constraint remains active
dif(f(X, Y), f(x, y)).
```

### Trying with if-then-else

See [`->/2`](https://www.swi-prolog.org/pldoc/doc_for?object=(-%3E)/2) 
 
```text
?- (writeln("It begins"); (writeln("Backtracked to before ->"),fail)), % print when forwarding and backtracking
   (dif(X,Y) -> writeln("YES") ; writeln("NO")),
   X=1,
   (writeln("Past X=1"); (writeln("Backtracked to before Y=1"),fail)),  % print when forwarding and backtracking
   (Y=1 ; (writeln("Y=1 failed"),fail)),                                % Y=1 fails, so print on second attempt
   writeln("END").

It begins
YES
Past X=1
Y=1 failed
Backtracked to before Y=1
Backtracked to before ->
```

### `dif/2` constraint working on head unification

```prolog
find_a_way :- dif(X,Y), findo(X,Y).

findo(x,_) :- writeln("find(x,_)").
findo(a,a) :- writeln("find(a,a)"). 
findo(_,y) :- writeln("find(_,y)"). 
```

Then:

```text
?- find_a_way.
find(x,_)
true ;
find(_,y)
true.
```

### Trying with if-then-else, again

```logtalk
doimply(WhatDo) :-
   X=k(_),
   Y=k(m),
   (dif(X,Y) 
    -> 
    optimistically_proceed(X,WhatDo)
    ;
    dif_has_failed(X)),
   format("END OF CLAUSE (X=~q, Y=~q)\n",[X,Y]). 

doimply(_) :-
   writeln("doimply/1 alternative").

optimistically_proceed(X,WhatDo) :- 
   writeln("Optimistic branch"),   
   ((WhatDo==eq) 
    -> 
    (X=k(m),writeln("dif(X,Y) constraint violation: never get here"))
    ;
    (X=k(u),writeln("dif(X,Y) late confirmation"))). 
    
dif_has_failed(X,Y) :-    
   format("We must be in state 'different': X=~q, Y=~q\n",[X,Y]).
```

The "happy path" would be to confirm `dif(A,B)`, its optimistic attitude rewarded:

```text
?- doimply(_).
Optimistic branch
dif(X,Y) late confirmation
END OF CLAUSE (X=k(u), Y=k(m))
true ;
doimply/1 alternative
true.
```

Calling `doimply/1` with the instruction to make `X` and `Y`  equal to elicit "late failure":

```text
?- doimply(eq).
Optimistic branch
doimply/1 alternative
true.
```

### An example with repeated attempts using `between/3`

```text
?- between(1,6,B),dif(3,B).
B = 1 ;
B = 2 ; 
B = 4 ;
B = 5 ;
B = 6.
```

### Constructing a more deterministic `member/2`

This approach appears in [Indexing `dif/2`](https://arxiv.org/abs/1607.01590):

```prolog
% A normal member/2 which may generate more solutions than needed

membern(X, [E|Es]) :-
   ( X = E
   ; membern(X, Es)
   ).

% A more deterministic member/2 (practically a memberchk/2) which tries to 
% not generate redundant solutions

memberd(X, [E|Es]) :-
   ( X = E
   ; dif(X, E),
     memberd(X, Es)
   ).  
```

And so:

```text
?- membern(1, [1,2,3]).
true ;
false.

?- memberd(1, [1,2,3]).
true ;
false.   % leftover choicepoint
```

Is `1` in `[1,X]`? 

```text
?- membern(1, [1,X]).
true ;   % yes it's a member
X = 1 ;  % redundant answer
false.
   
?- memberd(1, [1,X]).
true ;   % choicepoint is unnecessarily left open
false.
```

Is `1` in `[X,1]`? 

```text
?- membern(1, [X,1]).
X = 1 ;
true ;
false.   % redundant

?- memberd(1, [X,1]).
X = 1 ;
dif(X, 1) ; % dif(1,X) still open (and printed) and there is success for second element of list
false.      % no more solutions

?- bagof(X,memberd(1, [X,1]),Bag).
Bag = [1, _19152],    % solutions
dif(_19152, 1).       % dif(_,1) still open
```
  
## Beware of mixing `dif/2` and cuts.

Jan Wielemaker writes:

> The general rule of thumb is that constraints and cuts (and thus if-then-else and negation) do 
> not play together. It only works if all relevant constraints are resolved before committing.
> That is in general hard to predict. Constraint solvers have in general incomplete propagation 
> and thus may have pending constraints even in cases where all constraints can be resolved. At
> debug time, [call_residue_vars/2](https://eu.swi-prolog.org/pldoc/doc_for?object=call_residue_vars/2)
> can be used to verify there are no pending constraints. The
> implementation is fairly costly in terms of additional bookkeeping that, for example, protects
> constraints against garbage collection. This may prohibit usage at runtime.

## Addendum

According to [this discussion](https://swi-prolog.discourse.group/t/surprising-dif-2-behaviour/2317), anonymous variables
in the term passed to `dif/2` will lead to non-printing at the toplevel:

```text
% dif succeeds optimistically

?- dif((p(1) :- q),(_B:-_C)),format("Hey\n").
Hey
dif(f(_B, _C), f(p(1), q)).

% dif succeeds optimistically but doesn't print the dif/2 expression

?- dif((p(1) :- q),(_:-_)),format("Hey\n").
Hey
true.

% dif/2 fails immediately

?- dif((p(1):-q),(p(1):-q)),format("Hey\n").
false.
```

## Further explorations

### The paper "Indexing `dif/2`

**[Indexing `dif/2`](https://arxiv.org/abs/1607.01590): Ulrich Neumerkel, Stefan Kral, 2016-06-06**

> Many Prolog programs are unnecessarily impure because of inadequate means to express syntactic inequality.
> While the frequently provided built-in `dif/2` is able to correctly describe expected answers, its direct
> use in programs often leads to overly complex and inefficient definitions --- mainly due to the lack of
> adequate indexing mechanisms. We propose to overcome these problems by using a new predicate that subsumes
> both equality and inequality via reification. Code complexity is reduced with a monotonic, higher-order 
> if-then-else construct based on `call/N`. For comparable correct uses of impure definitions, our approach 
> is as determinate and similarly efficient as its impure counterparts.

### `library(reif)`

[Reification with `library(reif)`](https://www.metalevel.at/prolog/metapredicates) and the [YouTube video](https://youtu.be/-nlI33r-P70?t=675)

Code for `library(reif)`: http://www.complang.tuwien.ac.at/ulrich/Prolog-inedit/swi/reif.pl 

### Discourse thread 

This discussion: https://swi-prolog.discourse.group/t/dif-2-call-as-implication-premiss-is-the-implications-else-part-run-should-it-be/2486/26

### Reification

The idea is that the predicate always succeeds but the result of the `dif/2` is obtained as an atom (e.g. `dif`/`equal`)
unified with a third parameter.

Peter Ludemann writes in [this discussion](https://swi-prolog.discourse.group/t/dif-2-call-as-implication-premiss-is-the-implications-else-part-run-should-it-be/2486)

> Here’s a simple “reified” version of `dif/2` (it requires its arguments to be fully ground rather than sufficiently ground):

```prolog
%! test(X, Y, Result) is det.
% Result is 'dif'   when X and Y become ground and are not unifiable
% Result is 'equal' when X and Y become ground and are unifiable

test(X, Y, dif) :-
    when((ground(X),ground(Y)), X\=Y).
test(X, Y, equal) :-
    when((ground(X),ground(Y)), X==Y).

my_dif(X, Y) :- test(X, Y, dif).
```

> And running it gets:

```text
?- test(X, Y, Result), X=1, Y=1.
X = Y, Y = 1,
Result = equal.

?- test(X, Y, Result), X=1, Y=2.
X = 1,
Y = 2,
Result = dif ;
false.
```

> You could then rewrite your `if-then-else` to not have a cut but instead have both the condition and 
> its inverse – that is: `if cond then A else B becomes (cond & A) | (~cond & B)`.

```text
?- ( test(X, Y, dif), writeln(different) ; test(X, Y, equal), writeln(equal) ), X=1, Y=1.
different
equal
X = Y, Y = 1.

?- ( test(X, Y, dif), writeln(different) ; test(X, Y, equal), writeln(equal) ), X=1, Y=2.
different
X = 1,
Y = 2 
```
