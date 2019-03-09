# 🌊 Notes on using the C language

Small collection of notes for myself on using the C programming language.

## 1. Use struct operations for struct data classes

C permits arbitrary operations on any addressable (or non-addressable 🤦)
memory. When operating on `struct` data types, I prefer using C99 struct
operations rather than traditional pointer memory access.

### 1. Initializing structures

```c
struct foo_s {
    int a;
    char b[5];
    struct bar_s {
        int c;
    } bar;
};

// contrived pointer math approach; brittle
struct foo_s foo;
memset(foo, 0, sizeof(foo));
*((char *)&foo) = 1234;  // initialize... a?

// struct operations approach
struct foo_x foo = {
    .a = 1234,  // always use designated initializers
};
// foo is initialized by the standard to all default values (typically zero)
// except for designated fields
```

### 2. Duplicating structures

```c
const struct foo_s foo2 = {
    .a = 4321,
    .bar = {
        .c = 2,
    },
};

// we can initialize from a constant
struct foo_s foo = foo2;

// we can also initialize from a function parameter input
void foo_fn(struct foo_s *foo_ptr) {
    struct foo_s local_foo = *foo_ptr;
}
```

## 2. Use local blocks for scoping

Variable lifetimes are determined by the compiler; stack allocated variables may
not be "garbage collected" until the stack frame pops, for example.

However for readability, I like to declare variables at the innermost scope
where they are referenced.

```c
void foo_fn(int foo) {
    // traditional
    int result;

    switch(foo) {
        case 2:
            result = foo * 2;
            printf("%d\n", result);
            break;
    }

    // preferrable
    switch(foo) {
        case 2:
        {
            int result = foo * 2;
            printf("%d\n", result);
            break;
        }
    }
}
```

## 4. Compiler warnings

Compiler warnings are a cheap and simple way to increase correctness.

It's *much* easier to enable a lot of warnings at the start of a project than
later, because you may end up having to resolve a lot of violations. Be
aggressive when selecting warnings when you start, and disable later if you're
over encumbered by them.

The below warnings are gcc flags; clang has similar features (and many
additional flags that can be useful, I recommend reading through them!).

https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html

https://clang.llvm.org/docs/DiagnosticsReference.html

* `-Wall -Werror` minimum, more is better (`-Weverything` on clang... for the
brave!)
* `-Werror=conversion` - C is statically but weakly typed; the compiler will
  insert implicit type conversions based on variable types in an expression.
  This can result in subtle but disastrous data value effects. This warning can
  help avoid some of those errors.
* `-Werror=undef` - undefined indentifiers resolve to `0` when the compiler
  checks `#if` expressions. This can lead to confusing behavior, so prevent it!
* `-Werror=shadow` - prevents confusing reuse of variable names
* `-Werror=double-promotion` - only relevant for targets without efficient
  `double` hardware. Error if a double constant (decimal value lacking the `f`
  suffix) or double conversion occurs!
* `-Werror=jump-misses-init` - can detect odd control flow issues, and generally
  guides you against unintuitive design
* `-Werror=logical-op` - bit-wise operators have counter intuitive precedence.
  Error to enforce reasonable parenthesis usage.
* `-Werror=format=2` - additional safety checks on printf/scanf type format
  parameters
* `-Werror=duplicated-cond` - same conditional twice, not usually beneficial
* `-Werror=duplicated-branches` - same expression in two conditional branches,
  violates Don't Repeat Yourself
* `-Werror=null-dereference` - pretty rare that this trips usefully, but can
  catch some simple typos etc.

## 5. Static analysis

TODO `scan-build`...

## 6. Test your software

> *Chris Lattner,
cited from this
[UB white paper](http://www.yodaiken.com/wp-content/uploads/2018/05/ub-1.pdf)* :

> *[...] UB is an inseperable part of C programming, [...] this is a depressing
and faintly terrifying thing. The tooling built around the C family of languages
helps make the situation less bad, but it is still pretty bad.*

There are many kinds of errors that will persist through static analysis checks.

It's extremely important to test C programs for correctness and unintended
side-effects. The usual approach is to write unit-level (function-level) tests
that exercise inputs of functions.

There are *MANY* unit test frameworks that can be used for C software. These are
some I've used successfully, in order of my personal preference:

|Framwork|Features|Comment|
|---|---|---|
|https://github.com/cpptest/cpptest|Asserts, test runner, mocks, memory leak detection|Moderate learning curve. Very powerful and concise mocking system|
|https://github.com/google/googletest|Asserts, test runner, mocks, leak checker|Very popular and widely used. Simple + concise syntax|
|https://github.com/cgreen-devs/cgreen|Asserts, test runner, mocks, test auto-discovery|Featureful, relatively concise|
|https://github.com/ThrowTheSwitch/Unity|Asserts, test runner|Simple to set up|
|https://github.com/ThrowTheSwitch/CMock|Mock generator for Unity|Simple but verbose and boilerplately mock generater|
|https://github.com/vmg/clar|Asserts, test runner|**Very** simple; used by [libgit2](https://github.com/libgit2/libgit2). **No mocking support**|

## 6.a. Test coverage

Lots of opinions on test coverage.

**0% coverage is certainly not ideal!**

It can be *very* helpful when guiding your testing efforts, especially if you're
not using a strict TDD process.

Tracking test coverage in your build system / CI is usually pretty simple with
gcov + lcov. Lots of information and tutorials on how to do that.

## 7. Use runtime sanitizers

Todo ASAN.
