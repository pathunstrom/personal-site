---
title: "Building Functions in a For Loop: Understanding Late Binding"
slug: late-binding
header: web-dev
summary: Discussion has arisen regarding Python scopes in for loops, predicated on a bit of oddness due to late binding functions.
author: piper
---

## The Problem

Demonstrate for loops and functions combining to create the late binding issue. (acknowledge toy example)

## Breaking Down What is happening

1. Parts of a list comprehension
    1. format tip (break into 3/4 lines)
    2. part 2 and 3 are the `for x in collection` and function identically
    3. the `if` filter only lets values through for which the if is True
    4. The value expression is what ends up in the final comprehension.
2. lambdas
    1. "anonymous" functions: A function not bound to a name.
    2. Limitations (1 expression)
3. Whose `i` is it anyway?
    1. Scope "where variables live"
    2. global (module) scope
    3. local (function) scope
        1. comprehensions act like functions, their loop variable doesn't not exist in global scope.
4. A puzzle: When do functions evaulate their scope?
    1. Show referencing an object defined after the function inside a function body.
    2. Why does that work, but a module level calculation fails?

## So Why the Weird

1. We have our list comprehension, which creates its own function scope.
2. We build up our list of functions referencing the `i` in the local scope.
3. When we go to call each function, `i` is pointing at the last value in our range, so each function has the same result.

## A Workaround

Let's take advantage of function scope to change our comprehension to get the right results: by adding an extra scope to the pile.

```
[
    (lambda v: (lambda: v))(i) 
    for i 
    in range(10)
]
```
So the big change is our double lambda.

The inner lambda is our original `lambda: i`, so not going to spend time on it. The magic is the `(lambda v: inner)(i)` We'll start with the lambda section:
this is a function that takes a value, and creates the inner lambda referencing its parameter. This creates a new function scope. Remember, we were referencing the `i`
in the comprehensions function scope, now our final function is referencing a parameter in a second function scope. Now, what's with the `(i)`?

We're calling our outer function with the current value of `i`, which becomes the reference for the returned functions value.

So in English:

1. We will loop over every value in the range 0 to 9.
2. For each value, we'll create a function that takes a value and returns another function that returns the value passed to the first function.
3. We then call that function immediately with the current value in our loop.

Then later, we can call these functions and get different results. Unlike our original.
