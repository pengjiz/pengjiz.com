---
title: Fun with Python AST
date: "2019-10-29T21:05:48-04:00"
keywords: ["dev", "python"]
description: Or another way to complicate your code.
draft: false
---

I have been told for a long time that there is an `ast` module in the Python
standard libraries, which is used by many useful tools, for example,
[pytest](https://github.com/pytest-dev/pytest/tree/master/src/_pytest). However,
I was only vaguely aware of it, and had never been motivated to learn it. A
while ago, I found an interesting library
[moshmosh](https://github.com/thautwarm/moshmosh), which made me finally go
through the documentation of `ast` and try to play with it.

As an example, in this post I will implement a simple "pipeline" feature
available in many places---Unix shells, F#, Lisp, R, etc.

## Pipeline with infix notation

`|` is used for pipelines in Unix shells, but is reserved for "bitwise or"
operator in Python. However, we may customize the behavior by defining `__or__`:

```python
class Pipe:
    def __init__(self, data):
        self.data = data

    def __or__(self, fn):
        return Pipe(fn(self.data))
```

This is the approach taken by the [Pipe](https://github.com/JulienPalard/Pipe)
library (although the implementation is slightly different). It can be used as:

```python
import functools


data = [[89, 90, 99],
        [77, 76, 82],
        [95, 97, 99]]

Pipe(data) | functools.partial(map, sum) | max | print
# => 291
```

This is simple, and seems to work. However, we have to wrap the result in as a
`Pipe` instance in order to chain multiple operations, and perhaps we need to
unwrap it at the end to work with other functions, making it tedious to use.
What if we just override the `|` operator completely? That is achievable using
`ast`:

```python
import ast


class PipeTransformer(ast.NodeTransformer):
    def visit_BinOp(self, node):
        if isinstance(node.op, ast.BitOr):
            return ast.Call(
                self.visit(node.right),
                [self.visit(node.left)],
                [],
                lineno=node.lineno,
                col_offset=node.col_offset)
        return self.generic_visit(node)


source = """\
import functools


data = [[89, 90, 99],
        [77, 76, 82],
        [95, 97, 99]]

data | functools.partial(map, sum) | max | print
"""

exec(
    compile(
        PipeTransformer().visit(ast.parse(source)),
        filename="<string>",
        mode="exec"))
# => 291
```

Here we rewrite the AST and convert every `BitOr` node into a `Call` node in the
tree. This is the approach taken by moshmosh. This is a clean and neat way to
implement pipelines. However, it only supports functions that take one argument.
This is not a big drawback, but I somehow miss the `%>%` operator in R. It is
impossible to have `%>%` in Python because it does not allow us to define custom
infix operators, but fortunately it is possible to have the threading macros
`->` and `->>` from Clojure.

## Threading macros

The threading macro `->>` in Emacs-Lisp (which I believe is borrowed from
Clojure) is:

```emacs-lisp
(->> '((89 90 99)
       (77 76 82)
       (95 97 99))
     (mapcar (lambda (xs) (apply #'+ xs)))
     (apply #'max))
; => 291
```

Basically it takes the result of one step as the last argument of the function
call at the next step (and the `->` use the value as the first argument). Here
is a quick and simple implementation for the `->>` macro in Python (note that it
may not work for all cases):

```python
import ast


class PipeTransformer2(ast.NodeTransformer):
    def visit_Call(self, node):
        if isinstance(node.func, ast.Name) and node.func.id == "pipe":
            args = (self.visit(arg) for arg in node.args)
            new_node = next(args)
            for arg in args:
                if isinstance(arg, ast.Call):
                    arg.args.append(new_node)
                    new_node = arg
                else:
                    new_node = ast.Call(
                        arg,
                        [new_node],
                        [],
                        lineno=arg.lineno,
                        col_offset=arg.col_offset)
            return new_node
        return self.generic_visit(node)


source = """\
data = [[89, 90, 99],
        [77, 76, 82],
        [95, 97, 99]]

pipe(data, map(sum), max, print)
"""

exec(
    compile(
        PipeTransformer2().visit(ast.parse(source)),
        filename="<string>",
        mode="exec"))
# => 291
```

This is my favorite way so far.

## Outro

How can we use it to run a Python script? I can come up with two ways. The first
one is to make a simple script that reads a file into a string, parses it,
transforms it, and finally executes it---which is exactly what we do in the
snippets above. However, in that way we cannot apply the transformations to the
modules imported. The second way is to customize how a module is loaded with the
[meta path hook](https://docs.python.org/3/reference/import.html#the-meta-path).
In that way all the modules imported will use the extensions. That is what the
moshmosh library choose to use. If you are interested you can have a look at the
[`extension_register.py`](https://github.com/thautwarm/moshmosh/blob/master/moshmosh/extension_register.py)
file.

`ast` gives us a chance to modify the AST before it is executed. In some sense
that is like what we do with Lisp macros. I think it is neat but if you ask me
"Will you use it in any serious project?", my answer would perhaps be "No". Even
though it is less hacky than `inspect`, it still feels like a big hack. Besides,
I tend to avoid introducing new dependencies only for syntax sugars.
