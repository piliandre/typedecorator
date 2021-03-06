# Typedecorator

A decorator-based implementation of type checks for Python.

[![Build Status](https://travis-ci.org/dobarkod/typedecorator.svg?branch=master)](https://travis-ci.org/dobarkod/typedecorator?branch=master)

Provides `@params`, `@returns` and `@void` decorators for describing
the type of the function arguments and return values. If the types mismatch,
an exception can be thrown, the mismatch can be logged, or it can be ignored.

Example:

    @returns({str: int})
    @params(data={str: int}, key=str, val=int)
    def add_to_data(data, key, val):
        data[key] = val
        return data

Works on Python 2.5+ and Python 3.2+.

## Quickstart

Install the package using pip:

    pip install typedecorator

You can now start using it in your code

    # import the decorators and the typecheck setup function
    from typedecorator import params, returns, setup_typecheck

    # decorate your functions
    @returns(int)
    @params(a=int, b=int)
    def add(a, b):
        return a + b

    # set up the type checking
    setup_typecheck()

    add(1, 2)  # works fine
    add('one', 2)  # will raise a TypeError

You ony need to call `setup_typecheck` once to enable it and optionally
configure exceptions thrown and logging level. You can use it multiple
time to change, disable, or re-enable typechecks at runtime. See the
Setup section for more information.

## Type Signatures

Both `@params` and `@returns` take type signatures that can describe both
simple and complex types. The `@void` decorator is a shorthand for
`@returns(type(None))`, describing a function returning nothing.

A type signature can be:

1. A type, such as `int`, `str`, `bool`, `object`, `dict`, `list`, or a
custom class, requiring that the value be of specified type or a subclass of
the specified type. Since every type is a subclass of `object`, `object`
matches any type.

2. A list containing a single element, requiring that the value be a list of
values, all matching the type signature of that element. For example, a type
signature specifying a list of integers would be `[int]`.

3. A tuple containing one or more elements, requiring that the value be a tuple
whose elements match the type signatures of each element of the tuple. For
eample, type signature `(int, str, bool)` matches tuples whose first element
is an integer, second a string and third a boolean value.

4. A dictionary containing a single element, requiring that the value be a
dictionary with keys of type matching the dict key, and values of type
matching the dict value. For example, `{str:object}` describes a dictionary
with string keys, and anything as values.

5. A set containing a single element, requiring that the value be a set
with elements of type matching the set element. For example, `{str}` matches
any set consisting solely of strings.

6. `xrange` (or `range` in Python 3), matching any iterable (including
generators and lists).

7. An instance of `typedecorator.Union`, requiring that the value be of any of
the types listed when creating the `Union` instance. For example,
`Union(int, str, type(None))` matches integers, strings and `None`.

8. An instance of `typedecorator.Nullable`, requiring that the value is either
of the type specified when creating the `Nullable` instance, or None. For
example, `Nullable(str)` matches strings and `None`.

9. A string containing the type name. This is useful in places where the
type is not yet available when the decorator is wrapping the function, for
example in method definitions (where class is not yet available)

These rules are recursive, so it is possible to construct arbitrarily
complex type signatures. Here are a few examples:

* `{str: (int, [MyClass])}` - dictionary with string keys, where values are
    tuples with first element being an integer, and a second being a list
    of instances of MyClass

* `{str: types.FunctionType}` - dictionary mapping strings to functions

* `{xrange}` - set of iterators

* `{str: Union(int, float)}` - dictionary mapping strings to values
    that are either integers or floating point numbers

* `MyObject` - any type named *MyObject* (more precisely,
    any type whose `__name__` attribute is equal to `MyObject`)

Note that `[object]` is the same as `list`, `{object:object}` is the same
as `dict` and `{object}` is the same as  `set`.


## Setup

The function `setup_typecheck` takes care of enabling, disabling, and
configuring type checks at "compile" (parse) time and at runtime.

The function takes three optional arguments:

* `enabled` - whether to enable checks of any kind (default: `True`)
* `exception` - which exception to raise if type check fails (default:
  `TypeError`), or `None` to disable raising the exception
* `loglevel` - the log level at which to log the type error (see the
  standard `logging` module for possible levels), or `None` to disable type
  error logging.

By default, the type checking system is inactive unless activated through
this function. However, the type-checking wrappers are in place, so the
type checking can be enabled or disabled at runtime (multiple times).
These wrappers do incur a very small but real performance cost. If you
want to disable the checks at "compile" time, call this function with
`enabled=False` *before* defining any functions or methods using the
typecheck decorator.

For example, if you have a `config.py` file with `USE_TYPECHECK` constant
specifying whether you want the type checks enabled:

    #!/usr/bin/env python

    from typecheck import setup_typecheck, params
    import config

    setup_typecheck(enabled=config.USE_TYPECHECK)

    @params(a=int, b=int):
    def add(a, b):
        return a + b

Note that in this case, the checks cannot be enabled at runtime.

## Type checking methods

When using `@params` with instance methods, you should specify `object` as
the type of the `self` argument. This is required because when the decorator
is executing, the class itself still isn't created so the type that `self`
will have doesn't exist yet.

There is no problem in using a catch-all `object` type though, since Python
will enforce correct type for the instance upon method invocation anyways.

When using `@params` on a class method, there is no problem since the class
itself is instance of the `type` type.

But there's still a problem if a method takes additional parameters
that must be instances of the same class, or if it returns an instance of
the class. In this case, and in any other case where the type might not
be available at the time of decorator running, a string specifying the type
name can be used instead. Using strings should be avoided in other cases,
because they're more prone to type errors (the existence of the type is never
checked, the check only does a name match).

When combining type checking decorators with `@classmethod`, `@staticmethod`
or `@property` decorators, the type checking decorators must run first (ie.
be "closer" to the body of the function being defined).

Here's an example:

    class Accumulator(object):
        sum = 0

        @void
        @params(self=object, a=int)
        def add(self, a):
            self.sum = self.sum + a

        @classmethod
        @returns(int)
        @params(cls=type, a=int, b=int)
        def add2(cls, a, b):
            return a + b

        @property
        @returns(int)
        def total(self):
            return self.sum

        @returns('Accumulator')
        @params(self=object, other='Accumulator')
        def __add__(self, other):
            result = Accumulator()
            result.sum = self.sum + other.sum
            return result

## Mocking

Since the type signatures compare the actual value types, the parameters
can't be mocked. To minimize the problem, `Mock` type from `mock` library
(the most used mocking library in Python, and part of Python 3 standard
library) is special cased - an instance of `Mock` (or any of its subclasses,
such as `MagicMock`) will pass any check.

## Python 3 annotations

If used with Python 3, the parameters and return type signatures can also
be specified using the Python 3 function annotation syntax. To enable these,
use the `@typed` decorator on an annotated function. For example:

    @typed
    def add_to_data(data: {str: int}, key: str, val: int) -> {str: int}:
        data[key] = val
        return data

The behaviour is identical as if `@params` and `@returns` were used, the only
difference is in nicer syntax.


## License

Copyright (C) 2014. Senko Rasic <senko.rasic@goodcode.io>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
