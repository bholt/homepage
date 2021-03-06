---
date: 2016-08-28T18:32:31-07:00
draft: true
title: Building your own Program Synthesizer
---

In an [earlier post][synthpost], we saw an overview of *program synthesis* algorithms that automatically generate a program to implement a desired specification. But while these algorithms are an exciting and evolving field of research, you don't need to implement them yourself. Today, we'll see how to build program synthesizers using existing tools.

### Getting started with Rosette

There are a few excellent frameworks for program synthesis. The original is [Sketch][], which offers a Java-ish language equipped with some synthesis features. There's also the [syntax-guided synthesis language][sygus], which offers a common interface to several different synthesis engines.

For this post, we're going to use [Rosette][], which is an extension of the [Racket][] programming language that offers synthesis and verification features. The nice thing about Rosette is that, because it extends Racket, it offers many of the features of a good programming language, and as we'll see later, most things in the language "just work" the way you'd expect when doing synthesis (we'll see later what this means).

> **Following along**: the code for this post is available on GitHub. If you'd like to follow along, you'll need to [install Racket][racketdl] and then [Rosette][rosettedl]. Then you'll be able to run Racket programs either with the DrRacket IDE or the `racket` command-line interpreter.

#### A too-fast introduction to Racket

Racket is a [Lisp][] descendent, so it will look very familiar if you've used Lisp, Scheme, Clojure, etc. There are great [Racket tutorials][racketquick] and [crash courses][learnracket], so we won't cover Racket in detail, but I'll try to explain anything unusual as we go. The key thing to remember is that functions are always in prefix notation, and parentheses denote function application. So, here's our first Racket program:

```racket
#lang racket

(define a 5)
(define (add2 x)
  (+ x 2))

(add2 a)  ; returns 7
(set! a 4)
(add2 a)  ; returns 6
```

We defined a variable `a` with the value 5, and a function `add2` which takes an argument and adds 2 to it. Then we called `add2` with `a` to get the value 7. We then set `a` to 4 instead, and applied `add2` again to get the value 6.

#### The first Rosette trick

That `#lang racket` line in the program above seems a little weird -- most languages don't require us to say which language we're using inside the program itself. But it turns out this line is one of Racket's most powerful features, because you can use it to *make your own programming language* inside Racket. If that sounds enticing, Matthew Butterick is writing an outstanding book, [*Beautiful Racket*][br], all about making your own languages
(and has [a section about the `#lang` line][lang] in particular).

Rosette is one of these languages. This means we can change that `#lang racket` line to use Rosette instead:

```racket
#lang rosette/safe
```

and rerun the program. This is Rosette's first key trick: this program will do *exactly the same thing* as the first one did. This is what I meant earlier by things "just working" -- most Racket programs *are* Rosette programs.{{% fn 1 %}}

### Programming with constraints

Rosette's key feature is programming with *constraints*. Rather than a program in which all variables have known values, a constraint program has some *unknown* variables, and their values are determined by constraints.

This idea sounds a little abstract, so let's see an example:

```racket
#lang rosette/safe

(define (add2 x)
  (+ x 2))

(define-symbolic y integer?)
(add2 y)
```

We've defined `add2` as before, but this time we've also defined a "symbolic" variable `y`. A symbolic variable is one whose value is unknown. The `integer?` annotation tells Rosette that `y` is an integer (we'll see more types later). Then we called `add2` with this unknown variable.

What should calling `(add2 y)` do, if `y` is unknown? (Take a second to think about it). Rosette creates a *symbolic representation* of what `add2` should do, and returns this *expression* from `(add2 y)`:

    (+ y 2)

You can think of this return value as a kind of function: once you know a value of `y`, you can figure out what `(add2 y)` would return by substituting `y` into it.

#### Symbolic representations
These symbolic representations are very powerful: Rosette can produce them for a very large subset of Racket code. Let's extend the above program with another example:

```racket
(define (absv x)
  (if (< x 0) (- x) x))
  
(absv y)
```

Now we've defined an absolute value function: if the input `x` is negative (`(< x 0)` is true), then return `-x`, otherwise return `x`. The `(if ...)` form is similar to the ternary `a ? b : c` expression you might know from C, Java, etc -- `(if cond then else)` returns `then` if `cond` is true, or `else` otherwise.

So, what should `(absv y)` return, if `y` is still unknown? (Take another second). Rosette produces this symbolic representation:

    (ite (< y 0) (- y) y)

It captures the intuitive version of what `absv` does: if `y` is negative, it would return `(- y)`, otherwise it would return `y`. (`ite` is the same as `if`, but named differently to distinguish symbolic representations from Racket code).

> **Aside**: Rosette generates these symbolic representations using symbolic execution, the same idea that underpins tools like [KLEE][] or Microsoft's newly announced [Project Springfield][springfield]. There are more details in [the Rosette paper][paper].

#### Solving constraints

The next piece of the Rosette world is filling in values for the unknown variables. We do this by using the symbolic representations as constraints. Though this is a very powerful idea, let's start by doing something very simple:

```racket
(solve (assert (= (add2 y) 8)))
```

This fragment asks Rosette to try to fill in all the unknown variables (so far, just `y`) to satisfy a *constraint* that `(add2 y)` is equal to 8. Of course, we know the answer should set `y` to 6, because we can do arithmetic. Rosette can do arithmetic, too; here's the output:

    (model
     [y 6])

Rosette says that it found a *model* (an assignment of values to all the unknown variables) in which `y` takes the value 6.

Let's try a slightly more difficult constraint to test out our absolute value function:

```racket
(solve (assert (and (= (absv y) 5) 
                    (< y 0))))
```

Our new constraint says that `(absv y)` should be equal to 5, and that `y` should be negative. Rosette figures this out:

    (model
     [y -5])
     
Now let's try to outsmart Rosette by asking for the impossible:

```racket
(solve (assert (< (absv y) 0)))
```

Is there any value of `y` which has a negative absolute value? Rosette says:

    (unsat)

Rosette reports that this constraint is *unsatisfiable*: there is no possible `y` that has a negative absolute value.

Finally, let's see how Rosette handles a more complicated constraint involving data structures. Racket supports lists, and provides the `list-ref` procedure to retrieve an element from the list:

```racket
(define L (list 9 7 5 3))
(list-ref L 2)  ; returns 5, i.e., L[2]
```

Rosette supports lists, so we can ask it to solve this constraint:

```racket
(solve (assert (= (list-ref L y) 7)))
```

which asks for a value of `y` such that `(list-ref L y)` is 7. Rosette says:

    (model
     [y 1])

as we'd expect -- the second element of `L` is 7 (and list indices start from 0).

So constraint programming allows us to fill in unknown values in our program automatically. This ability will underlie our approach to program synthesis (when the *program* will be the unknown value).

> **Aside**: Rosette's `(solve ...)` form works by compiling constraints to the [Z3 SMT solver][z3], which provides high-performance solving algorithms for a variety of styles of constraints. The [Z3 tutorial][z3tutorial] is a nice introduction to this lower-level style of constraint progamming that Rosette abstracts away.

### Domain-specific languages: programs in programs

What does it mean for "the program" to be the unknown value? Which language is the program in? Can it be any program in that language? To give precise answers to these questions, we're going to define a *domain-specific language* (DSL) for our synthesis task. A DSL is just a small programming language equipped with exactly the features we want.

You can build a DSL for just about anything. In our research, we've built DSLs for synthesis work in [file system operations][ferrite] and [approximate hardware][synapse], and others have done the same for [network configuration][bagpipe] and [K-12 algebra classes][rulesynth]. The common thread is that a DSL makes the operations we care about explicit, and hides those we don't.

For today, we're going to define a very trivial DSL for arithmetic operations. The programs we synthesize in this DSL will be arithmetic expressions like `(plus x y)`. While this isn't a particularly thrilling DSL, it will be simple to implement and demonstrate.

We need to define two parts of our language: its **syntax** (what programs look like) and its **semantics** (what programs do).

#### Syntax

To define the DSL syntax, we're going to make use of Racket's support for [structures][]. We'll define a new structure type for each operation in our language:

```racket
(struct plus (left right) #:transparent)
(struct minus (left right) #:transparent)
(struct mul (left right) #:transparent)
(struct square (arg) #:transparent)
```

Here, we've defined four operators in our language: three operations `plus`, `minus`, and `mul` that each take two arguments, and a `square` operation that takes only a single argument. The structure declarations give names to the fields of the structure (`left` and `right` for the two-argument operations, and `arg` for the single-argument operation). The `#:transparent` annotation just tells Racket that it can look "into" these structures and, for example, automatically generate string representations for them.{{% fn 2 %}}

This syntax allows us to write programs such as this one:

```racket
(define prog (plus (square 7) 3))
```

to stand for the mathematical expression 7<sup>2</sup> + 3.

#### Semantics

Now that we know what programs in our DSL look like, we need to say what they mean. To do this, we're going to implement a simple *interpreter* for programs in our DSL. The interpreter takes as input a program, performs the computations that program describes, and returns the output value. For example, we'd expect the above program to return 52.

Our interpreter just recurses on the syntax tree using Racket's [pattern matching][pattern]:

```racket
(define (interpret prog)
  (match p
    [(plus a b) (+ (interpret a) (interpret b))]
    [(minus a b) (- (interpret a) (interpret b))]
    [(mul a b) (* (interpret a) (interpret b))]
    [(square a) (expt (interpret a) 2)]
    [_ p]))
```

{{% footnotes %}}
{{% footnote 1 %}}The `/safe` part of the language name is telling Rosette to stop us from shooting ourselves in the foot by using some parts of Racket that Rosette doesn't or can't support. If you're feeling brave, `#lang rosette` will let you use *all* of Racket, but there's no guarantees everything will work.{{% /footnote %}}
{{% footnote 2 %}}`#:transparent` also has a Rosette-specific meaning: structures with this annotation will be merged together when possible, while those without will be treated as mutable structures that cannot be merged.
{{% /footnote %}}
{{% /footnotes %}}

[synthpost]: synthesis-for-architects.html
[sketch]: https://bitbucket.org/gatoatigrado/sketch-frontend/wiki/Home
[sygus]: http://www.sygus.org/index.html
[rosette]: http://emina.github.io/rosette/
[racket]: http://racket-lang.org/
[racketdl]: https://download.racket-lang.org/
[rosettedl]: http://emina.github.io/rosette/doc/rosette-guide/ch_getting-started.html#%28part._sec~3aget%29
[lisp]: https://en.wikipedia.org/wiki/Lisp_(programming_language)
[racketquick]: https://docs.racket-lang.org/quick/index.html
[learnracket]: https://learnxinyminutes.com/docs/racket/
[br]: http://beautifulracket.com/
[lang]: http://beautifulracket.com/explainer/lang-line.html
[klee]: https://klee.github.io/
[springfield]: https://www.microsoft.com/en-us/springfield/
[paper]: http://homes.cs.washington.edu/~emina/pubs/rosette.pldi14.pdf
[z3]: https://github.com/Z3Prover/z3
[z3tutorial]: http://rise4fun.com/Z3/tutorial/guide
[ferrite]: http://sandcat.cs.washington.edu/ferrite/
[synapse]: http://synapse.uwplse.org/
[bagpipe]: http://bagpipe.uwplse.org/bagpipe/
[rulesynth]: http://homes.cs.washington.edu/~emina/pubs/rulesynth.its16.pdf
[structures]: https://docs.racket-lang.org/guide/define-struct.html#%28part._.Simple_.Structure_.Types__struct%29
[pattern]: https://docs.racket-lang.org/guide/match.html
