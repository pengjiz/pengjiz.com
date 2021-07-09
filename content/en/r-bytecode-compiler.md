---
title: Bytecode Compiler of R
date: 2020-12-24T20:51:33-05:00
keywords: [dev, r]
description: It may not be superb, but does a great job.
draft: false
---

For a long time I thought that the canonical implementation of R was a tree-walk
interpreter -- one that directly evaluates the abstract syntax tree. However, a
few days ago when I accidentally evaluated a function in the REPL, I found this
line after the function body:

```
<bytecode: <address>>
```

That led me to the bytecode compiler of R, which has been there for about ten
years since R 2.13.0. Now R packages are by default byte compiled when
installed. Enter one function name in the REPL and see if there is a `bytecode`
line in the output. If so the function has been compiled. Out of curiosity I
checked the design documentation of the compiler and the virtual machine
implementation. The compiler looks rather basic and conservative, but it is
still useful.

## Compiler interface

The compiler is mostly implemented in R itself and is distributed as the
`compiler` package together with R. That package exports a few functions so we
can use the compiler in our own code. Those functions are all well documented so
I will not reiterate their usage here, but I do want to mention two functions,
`enableJIT` and `disassemble`.

The function `enableJIT` turns just-in-time (JIT) compilation on or off. There
are three levels for JIT. At level 1 large functions will be compiled before
their first use. At level 2 small functions will also be compiled before their
second use. At level 3 top level loops will be compiled before execution as
well. So what is a "large" function? Currently R thinks that if the body has
approximately more than 50 expressions the function is large, with some special
structures (for example, loops) counted differently. By default the JIT level is
3, so `f` will be compiled at the end of the following snippet:

```r
f <- function() 1
f()
f()
f
```

The function `disassemble`, as its name suggests, shows the disassembly of a
bytecode object. However, the output is not friendly. If you run
`compiler::disassemble(f)`, this is what you will get:

```
list(.Code,
     list(12L, LDCONST.OP, 0L, RETURN.OP),
     list(1,
          structure(c(1L, 6L, 1L, 17L, 6L, 17L, 1L, 1L),
                    srcfile = <environment>,
                    class = "srcref"),
          structure(c(NA, 0L, 0L, 0L),
                    class = "expressionsIndex"),
          structure(c(NA, 1L, 1L, 1L),
                    class = "srcrefsIndex")))
```

It is a list of three elements. The first one `.Code` is a marker. The second
one is the bytecode list. The third one is the constant pool. You may have
noticed that the constant pool contains some cryptic structures. They are
actually for source reference so that the interpreter can generate useful error
messages.

Now let us look at the bytecode list. The first element, `12L` is the version
number of the bytecode. Real instructions start from the second element.
`LDCONST.OP` and other symbols that end with `.OP` are operations. Numbers after
operations are operands. So the disassembly above formatted in the usual way
should be something like this:

```
0    LDCONST.OP 0
1    RETURN.OP
```

The virtual machine is stack based and the instructions above mean to load
constant 0 from the pool, which is 1 (constant pools are indexed from 0), to the
stack and return from the function. That is almost a direct translation from the
R code. Currently the R virtual machine supports 129 opcodes in total. Other
than some operations on the stack, most of them are for those primitive
functions. You may want to check the source code of the `compiler` package if
your are interested.

## Optimizations

Usually bytecode compilers [do not optimize your code much][skeeto's post] and
the R compiler is no exception. Moreover, because of the [special evaluation
mechanism][R NSE], we should expect fewer optimizations done by it. In fact, it
seems that the compiler only does constant folding and inlining of primitive
functions.

Constant folding is a common optimization. It basically means to evaluate simple
expressions involving only constants at compile time and replace the original
expression with its value. Here is an example:

```r
library(compiler)

f <- function() 1 + 2 - 3 * 4 / 5
disassemble(cmpfun(f))
# list(.Code,
#      list(12L, LDCONST.OP, 1L, RETURN.OP),
#      list(1 + 2 - 3 * 4/5,
#           0.6,
#           structure(c(1L, 6L, 1L, 33L, 6L, 33L, 1L, 1L),
#                     srcfile = <environment>, class = "srcref"),
#           structure(c(NA, 0L, 0L, 0L), class = "expressionsIndex"),
#           structure(c(NA, 2L, 2L, 2L), class = "srcrefsIndex")))
```

Here the expression `1 + 2 - 3 * 4 / 5` is evaluated by the compiler and its
value 0.6 is put into the constant pool. So in the bytecode list this value is
loaded directly from the pool without doing the calculation again. Because
function arguments are usually compiled as well, constant folding works there:

```r
library(compiler)

f <- function(x) x
g <- function() f(1 + 2 - 3 * 4 / 5)
disassemble(cmpfun(g))
# list(
#   .Code,
#   list(12L, GETFUN.OP, 1L, MAKEPROM.OP, 3L, CALL.OP, 0L, RETURN.OP),
#   list(f(1 + 2 - 3 * 4/5),
#        f,
#        structure(c(1L, 6L, 1L, 36L, 6L, 36L, 1L, 1L),
#                  srcfile = <environment>, class = "srcref"),
#        list(.Code,
#             list(12L, LDCONST.OP, 1L, RETURN.OP),
#             list(1 + 2 - 3 * 4/5, 0.6, f(1 + 2 - 3 * 4/5),
#             structure(c(1L, 6L, 1L, 36L, 6L, 36L, 1L, 1L),
#                       srcfile = <environment>, class = "srcref"),
#             structure(c(NA, 2L, 2L, 2L), class = "expressionsIndex"),
#             structure(c(NA, 3L, 3L, 3L), class = "srcrefsIndex"))),
#        structure(c(NA, 0L, 0L, 0L, 0L, 0L, 0L, 0L),
#                  class = "expressionsIndex"),
#        structure(c(NA, 2L, 2L, 2L, 2L, 2L, 2L, 2L),
#                  class = "srcrefsIndex")))
```

As we can see from the output, the forth object in the constant pool, which is
used to make a promise, is a compiled bytecode object with the constant
expression evaluated. Also, note that the object still contains the original
expression which is passed into the promise as well. So NSE works as usual.
However, because NSE does not use the value directly, constant folding here only
slows down the compiler -- yet another reason to avoid NSE. There is a way to
prevent the compilation of function arguments, but the list of such functions
are hard coded in the source code and currently it only contains the `bquote`
function. I hope that there will be a public interface to extend that list in
the future.

Another important work done by the compiler is inlining primitive functions. As
said most primitive functions have their own opcodes and the compiler will
replace calls to those functions with appropriate sequences of bytecodes. That
is not something super fancy, but is useful:

```r
library(compiler)
enableJIT(0)

f <- function() {
  for (i in 1:1000)
    i * 2
}
g <- cmpfun(f)
system.time(for (i in 1:1000) f())
#  user  system elapsed
# 0.091   0.000   0.091
system.time(for (i in 1:1000) g())
#  user  system elapsed
# 0.017   0.000   0.017
```

Here we have to disable JIT first because functions containing loops will be
compiled even at JIT level 1. As you can see, the compiled function is much
faster.

## Outro

For some reason I have a feeling that the bytecode compiler of R is overlooked.
It is a rather conservative compiler and misses many optimizations, but for R it
does a good job. The compiler is fast and correct enough so with JIT it makes
our code efficient without even being noticed.

I am not sure if we should add more optimizations to the compiler. I would
choose another language rather than optimize the R code if I want better
performance. Besides, adding more optimizations will inevitably slow down the
compilation process and will perhaps make the experience worse -- I can still
remember how the loading time of a Julia package made me crazy a few years ago.
However, I do think that the compiler deserves a better interface. It should
allow users to extend the compiler a bit.

[skeeto's post]: https://nullprogram.com/blog/2019/02/24/
[R NSE]: {{< relref "/en/r-nse" >}}
