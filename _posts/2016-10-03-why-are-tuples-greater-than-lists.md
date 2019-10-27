---
layout: post
status: publish
published: true
title: Why are tuples greater than lists?
author: Travis Mick
date: 2016-10-03
excerpt: I pose this question in quite a literal sense. Why does Python 2.7 have this behavior? No matter what the tuple, and no matter what the list, the tuple will always be considered greater.
---

I pose this question in quite a literal sense. Why does Python 2.7 have this behavior?

```python
>>> (1,) > [2]
True
```

No matter what the tuple, and no matter what the list, the tuple will always be considered greater. On the other hand, Python 3 gives us an error, which actually makes a bit more sense:

```python
>>> (1,) > [2]
Traceback (most recent call last):
 File "<stdin>", line 1, in <module>
TypeError: unorderable types: tuple() > list()
```

The following post is a journey into some CPython internals, with a goal of finding out why 2.7 gives us such a weird comparison result.

Those of you who have implemented nontrivial classes in Python are probably aware of the two different comparison interfaces in the data model: rich comparison, and simple comparison. Rich comparison is implemented by defining the functions `__lt__`, `__le__`, `__eq__`, `__ne__`, `__gt__`, and `__ge__`. That is, there is one function for each possible comparison operator. Simple comparison uses only one function, `__cmp__`, which has a similar interface to C's `strcmp`.

Any comparison operation you write in Python compiles down to the `COMPARE_OP` bytecode, which itself is handled by a function called `cmp_outcome`. For the types of comparisons we're concerned with today (i.e., inequalities rather than exact comparisons), this function will end up calling `PyObject_RichCompare`, the user-facing comparison function in the C API.

At this point, the runtime will attempt to use the rich comparison interface, if possible. Assuming that neither operand's class is a subclass of the other, the first class's comparison functions will be checked first; the second class would be checked if the first class does not yield a useful result. In the case of `tuple` and `list`, both calls return `NotImplemented`.

Having failed to use the rich comparison interface, we now try to call `__cmp__`. The actual semantics here are quite complicated, but in the case at hand, all attempts fail. One penultimate effort before hitting the last-ditch "default" compare function is to convert both operands to numeric types (which fails here, of course).

CPython's `default_3way_compare` is somewhat of a collection of terrible ideas. If the two objects are of the same type, it will try to compare them by address and return that result. Otherwise, we then check if either value is `None`, which would be considered smaller than anything else. The second-to-last option, which we will actually end up using in the case of `tuple` vs. `list`, is to compare the names of the two classes (essentially returning `strcmp(v->ob_type->tp_name, w->ob_type->tp_name)`). Note, however, that any numeric type would have its type name switched to the empty string here, so a number ends up being considered smaller than anything non-numeric. If we end up in a case where both type names are the same (either they actually have the same name, or they are incomparable numeric types), then we get a final result by comparing pointers to the type definitions.

To validate our findings, consider the following:

```python
>>> tuple() > list()
True
>>> class abc (tuple):
... pass
... 
>>> abc() > list()
False
>>> class xyz (tuple):
... pass
... 
>>> xyz() > list()
True
```

The only difference between classes `abc` and `xyz` (and `tuple`, even) are their names, however we can see that the instances are compared differently. Now, we have certainly found quite the footgun here, so it's fortunate that Python 3 has a more sane comparison operation.

