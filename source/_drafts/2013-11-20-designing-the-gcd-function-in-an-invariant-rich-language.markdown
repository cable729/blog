---
layout: post
title: "Designing the GCD function in an invariant rich language"
date: 2013-11-20 15:21:03 -0600
comments: true
categories: [invariant-lang]
---

What are invariants and what is an invariant rich language?

> In computer science, an invariant is a condition that can be relied upon to be true during execution of a program, or during some portion of it. It is a logical assertion that is held to always be true during a certain phase of execution.

Languages already have invariants. For instance, in most strongly-typed languages when you try to access a variable's value without first setting it you will get an error. But languages don't usually go much farther than that.

An ideal language has a compiler that knows the same amount of information as you do. It knows when you're multiplying by two possibly large ints and will warn you of an overflow. It will let you know when you're accessing an out-of-bound index. Basically, with invariants on every variable and function, the compiler could check every edge case, reducing the amount of errors you find in testing or possibly production.

To illustrate this fact, we design a simple language and iteratively build up a greatest common divisor out of smaller functions.

# Division and Addition
First, let's define division on integers.

	def / (
	    a : (int)
	    b : (int)
	    )
	    return low_level_int_divide(a, b) : (int)

Okay, dividing an `int` by an `int` gives an `int`. Uninteresting. Let's overload it:

	def / (
	    a : (int, positive)
	    b : (int, positive)
	    )
	    return low_level_int_divide(a, b) : (int, positive)

Finally, a useful result. This is something other programming languages don't keep track of. Let's make another quick definition:

	def + (
	    a : (int, positive)
	    b : (int, nonnegative)
	    )
	    return low_level_add(a, b) : (int, positive)

Here we see `nonnegative` for the first time. If a number is `positive` it is obviously `nonnegative`, so we are already seeing a sort of inheritence for invariants.

# Modulus
As I'm sure you remember, `mod` is the modulus function (also known as least residue), which gives the remainder when dividing two integers.


	def mod (
	    a : (int, positive, $ >= b) // this means that a >= b
	    b : (int, positive)
	    )
	    quotient_floored = a / b
	    temp_product = quotient_floored * b
	    residue = a - temp_product
	    return residue

Thus `a mod b`, for $a\ge b$, only returns integers at least `0` and at most `b - 1`. This makes sense intuitively, I.E.:

$$
\begin{alignat*}{4}
\frac{15}{3} &= 5 + \frac{0}{3}, \qquad \frac{16}{3} &= 5 + \frac{1}{3},
\qquad
\frac{17}{3} &= 5 + \frac{2}{3}, \qquad \frac{18}{3} &= 5 + \frac{0}{3}.
\end{alignat*}
$$

So what invariants does `quotient_floored` have? From our equation $a\ge b$, we have $a/b\ge 1$, so it is `positive`. So we have

	quotient_floored = a / b : (int, positive)

Then `temp_product` is the product of `(int, positive) * (int, positive)`. So the product is an `(int, positive)`. But wait, our residue has to return a `(int, nonnegative)`, so `temp_product` must be at most `a`. How do we ensure this?

	quotient_floored = a / b : (int, positive, $ <= quotient(a/b))

Since integer division is actually $\lfloor a/b \rfloor$ (rounding down the quotient), it is at most $a/b$, for positive $a,b$. So

	temp_product : (int, positive, $ <= quotient(a/b)) * (int, positive, $ == b)
		     : (int, positive, $ <= a)

So our compiler just figured out that $\lfloor a/b \rfloor b \le a$. Now it can say

	residue : (int, positive, $ == a) - (int, positive $ <= a)
		     : (int, nonnegative)

Cool, now our `mod` function returns a `(int, nonnegative)`. Let's build our `gcd` function now.

# Greatest Common Divisor

	def gcd (
	    a : (int, positive)
	    b : (int, positive, $ <= a)
	    )
	    current = a
	    divisor = b
	    while divisor > 0
	       temp = divisor
	       divisor = mod(current, divisor)
	       current = temp
	    return current

We know that $1\le\gcd(a,b)\le b$ for all integers $a,b$ with $1\le b\le a$. Thus we'd like the final `a` to be `(int,)

<!-- 

# GCD

For all positive integers $a,b$ with $a\ge b$, let $\text{gcd}(a,b)$ be a positive integer at most $b$. Using the [Euclidean algorithm](http://en.wikipedia.org/wiki/Euclidean_algorithm) we can easily implement this function.

	def gcd (
		a : (int, positive)
		b : (int, positive)
		)
	    while b != 0
	       t := b
	       b := a mod b
	       a := t
	    return a


However, we haven't defined `mod`. Let's do that now. For all $a,b\in\mathbb{N}$, let


	def mod (
		a : (int, positive)
		b : (int, positive)
		)
	    quotient_floored = a / b
	    temp_product = quotient_floored * b
	    residue = a - temp_product
	    return residue

Excellent, now how do we implement it? Notice first that we have a `mod` function that we haven't defined, so let's do that. In our pseudocode, a variable's invariants are located to the right in parentheses.


Simple enough. But how will the compiler assign invariants to the return value? Let's think about what invariants `res` should have. Remember that `a modulus b` is the remainder when dividing `a` by `b`. It makes sense that `res` should be an `int` and should satisfy the inequality `0 <= res < b`.

However, as it stands, we don't know what invariants `a / b` will assign! So let's define the `/` operator:

	def /	(
				a : (int)
				b : (int, nonzero)
			)
		res = // some low-level implementation
		return res : (int)

Cool, dividing an `int` by an `int` gives an `int`. Every statically-typed language already gives us that. But really, when we only know that `a` and `b` are `int`s, we can't say much. Consider a stronger definition:

	def /	(
				a : (int, positive)
				b : (int, positive)
			)
		res = // some low-level implementation
		return res : (int)

 -->