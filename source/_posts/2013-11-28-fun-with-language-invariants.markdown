---
layout: post
title: "Fun with language invariants"
date: 2013-11-28 09:29:55 -0700
comments: true
categories: [invariants, language-design]
---

What are invariants and what is an invariant rich language?

> In computer science, an invariant is a condition that can be relied upon to be true during execution of a program, or during some portion of it. It is a logical assertion that is held to always be true during a certain phase of execution.

Most programming languages already have simple invariants. For instance, in most strongly-typed languages when you try to access a variable's value without first setting it you will get an error. But languages don't usually go much farther than that.

An ideal language has a compiler that knows the same (or more) amount of information as you. It knows when you're multiplying by two possibly large integers and will warn you of an overflow. It will let you know when you're accessing an out-of-bound index, or attempting to divide by zero. Basically, with invariants on every variable and function, the compiler could check every edge case, reducing the amount of errors you find in testing or possibly production.

## Setup

Let's set up integer division in our new language. First, let's take a look at a fundamental mathematics result, the Division Theorem. It states that for all integers $a$ and $b$ with $b\ne0$, there exist integers $q$ and $r$ such that

$$
a = qb+r, \qquad 0\le r< b.
$$

So the rational $a/b$ is equal to $q+r/b$. This means $q$ is the floor of $a/b$, and $r/b$ is the remainder. Using the floor value, we can define integer division, so we solve for $q$:

$$
\begin{align*}
0\le &a-qb < b\\
-a\le &-qb < b-a\\
a-b < &qb \le a \\
\frac{a-b}{b} < &q \le \frac{a}{b}.
\end{align*}
$$

We write type invariants for division as follows, with `/` in the invariants representing a fraction that is not truncated like integer division:

    def / (a : int, nonnegative, b : int, positive)
        returns : (int, a/b - 1 < $ <= a/b)

We use the syntax `:` to denote type and use the symbol `$` to denote the value of the type we're defining invariants on. In this case, we say that the returned value (`$`) is at most $a/b$ and strictly greater than $a/b-1$.

## Applying the division definition

You may be thinking, what's the point? Don't worry, doubtful reader! Let's define the `mod` function, and I guarantee you'll see he light. In our definition we write out the types, but in an ideal IDE the compiler would derive the types and display them for the programmer.

    def mod (a : int, nonnegative, b : int, positive)
        let q = a / b : (int, a/b - 1 < $ <= a/b)
        let t = q * b : (int, a - b < $ <= a)
        return a - t : (int, 0 <= $ < b)

Nice! How cool is that? Now we know our `mod` function always returns a nonnegative integer that is strictly less than what we're modding by! This is obvious when programming, but compilers just can't seem to figure it out.

Also note that `0 < $` and `0 <= $` are synonyms for `positive` and `nonnegative`, respectively. We could also say that `positive` is a subtype of `nonnegative`, or in more mathematical terms, `positive` implies `nonnegative`.

## Greatest common divisor

When we find the gcd of two positive integers, we should be assured that when dividing either of the inputs by the gcd results in an integer.

    def gcd (a : (int, positive), b : (int, 0 < $ <= a))
        let m = a mod b : (int, 0 <= $ < b)
        case m:
            of 0: return b : (int, $ == b, 0 < $ <= a)
            else: return gcd(b, m) : ???

Up until this point we haven't encountered a recursive function, so we don't know how our compiler should handle this. Our base case has bounds and types on its return value, but our recursive case doesn't.

Well, if our function doesn't infinitely recurse, it will return the second argument. The arguments in `gcd` look like this: `(a, b)`, `(b, a mod b)`, `(a mod b, b mod (a mod b))`, etc... The second arguments all obey the inequality `b > a mod b > b mod (a mod b) > ... > 0`. So if `gcd` returns a value at all, it will be an `(int, 0 < $ <= b)`.

## Closing

Hopefully you're getting a sense for the possibilities of a language designed like this. We figured out all of this by hand, but we only used information available from code and the definitions. Therefore it should be possible to design a compiler that can figure all of this out as well. We haven't considered any functions that modify state, so it's still unclear if this kind of design would work in a non-functional language.