# 🌊 Notes on using the C language

Small collection of notes for myself on using the C programming language.

<!-- vim-markdown-toc GFM -->

- [1. C style / usage notes](#1-c-style--usage-notes)
  - [1.1 Use struct operations for struct data classes](#11-use-struct-operations-for-struct-data-classes)
    - [1.1.1 Initializing structures](#111-initializing-structures)
    - [1.1.2. Duplicating structures](#112-duplicating-structures)
  - [1.2. Use local blocks for scoping](#12-use-local-blocks-for-scoping)
  - [1.3. memcpy size parameter](#13-memcpy-size-parameter)
- [4. Compiler warnings](#4-compiler-warnings)
- [5. Static analysis](#5-static-analysis)
- [6. Test your software](#6-test-your-software)
- [6.a. Test coverage](#6a-test-coverage)
- [7. Use runtime sanitizers](#7-use-runtime-sanitizers)

<!-- vim-markdown-toc -->

## 1. C style / usage notes

Set of notes around useful (to me) C idioms + patterns.

### 1.1 Use struct operations for struct data classes

C permits arbitrary operations on any addressable (or non-addressable 🤦)
memory. When operating on `struct` data types, I prefer using C99 struct
operations rather than traditional pointer memory access.

#### 1.1.1 Initializing structures

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
struct foo_s foo = {
    .a = 1234,  // always use designated initializers
};
// foo is initialized by the standard to all default values (typically zero)
// except for designated fields

// assigning after declaration is permitted, but requires a cast
struct foo_s bar = { 0 };

bar = (struct foo_s) {
    .bar = {
        .c = 12,
    },
};
// cast using __typeof__ :
bar = (__typeof__(bar)) {
    .a = 34,
};
```

Note that this approach won't initialize padding
(see [here](https://www.anmolsarma.in/post/stop-struct-memset/)). This is only
a problem if you're passing your data across a trust boundary, but be aware.

#### 1.1.2. Duplicating structures

```c
struct foo_s foo2 = {
    .a = 4321,
    .bar = {
        .c = 2,
    },
};

// we can copy from another structure
struct foo_s foo = foo2;

// we can also copy from a function parameter input
void foo_fn(struct foo_s *foo_ptr) {
    struct foo_s local_foo = *foo_ptr;
}
```

### 1.2. Use local blocks for scoping

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

    // preferable
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

### 1.3. memcpy size parameter

In most cases you want to use the destination object's size:

```c
struct foo {
    int a;
};
struct foo foo1, foo2;

// very fragile
memcpy(&foo1, &foo2, 4);

// pretty fragile; relies on `foo1`s type staying the same forever
memcpy(&foo1, &foo2, sizeof(struct foo));

// best; always try to use destination size
memcpy(&foo1, &foo2, sizeof(foo1));
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

- `-Wall -Werror` minimum, more is better (`-Weverything` on clang... for the
brave!)
- `-Werror=conversion` - C is statically but weakly typed; the compiler will
  insert implicit type conversions based on variable types in an expression.
  This can result in subtle but disastrous data value effects. This warning can
  help avoid some of those errors.
- `-Werror=undef` - undefined identifiers resolve to `0` when the compiler
  checks `#if` expressions. This can lead to confusing behavior, so prevent it!
- `-Werror=shadow` - prevents confusing reuse of variable names
- `-Werror=double-promotion` - only relevant for targets without efficient
  `double` hardware. Error if a double constant (decimal value lacking the `f`
  suffix) or double conversion occurs!
- `-Werror=jump-misses-init` - can detect odd control flow issues, and generally
  guides you against unintuitive design
- `-Werror=logical-op` - bit-wise operators have counter intuitive precedence.
  Error to enforce reasonable parenthesis usage.
- `-Werror=format=2` - additional safety checks on printf/scanf type format
  parameters
- `-Werror=duplicated-cond` - same conditional twice, not usually beneficial
- `-Werror=duplicated-branches` - same expression in two conditional branches,
  violates Don't Repeat Yourself
- `-Werror=null-dereference` - pretty rare that this trips usefully, but can
  catch some simple typos etc.

## 5. Static analysis

Static analysis tools (outside of what your C compiler provides) can detect some
types of potential error conditions. They're pretty cost-free to run since they
execute separately from program compilation, so it's usually worthwhile to use
them, and as many of them as practical!

- [cppcheck](https://github.com/danmar/cppcheck) - classic C/C++ static analysis

- [clang-tidy](https://clang.llvm.org/extra/clang-tidy/) - nice set of
  linting/security related detections. Run it with the checkers not associated
  with a particular coding convention like so:
  `clang-tidy-7 -checks="performance-*,portability-*,readability-*" ./**/*.c`

- [scan-build](https://clang-analyzer.llvm.org/scan-build.html) - another clang
  analyzer utility that does fairly sophisticated program level static analysis.
  *Note: it can be somewhat difficult to get scan-build operational when
  cross-compiling C programs. Don't give up! on programs of non-trivial size
  scan-build for me has without fail detected serious errors that were missed by
  manual or unit testing.*

- [ikos](https://github.com/NASA-SW-VnV/ikos) - LLVM backed static analyzer
  developed by NASA. Pretty sophisticated. Similar usage to scan-build, see
  [ikos-scan](https://github.com/NASA-SW-VnV/ikos/blob/master/analyzer/README.md#analyze-a-whole-project-with-ikos-scan).

- [flawfinder](https://dwheeler.com/flawfinder/) - relatively naive but useful
  tool that can identify dangerous patterns used in your code. Has a patch mode!

## 6. Test your software

> *Chris Lattner,
cited from this
[UB white paper](http://www.yodaiken.com/wp-content/uploads/2018/05/ub-1.pdf)* :
>
> *[...] UB is an inseparable part of C programming, [...] this is a depressing
and faintly terrifying thing. The tooling built around the C family of languages
helps make the situation less bad, but it is still pretty bad.*

There are many kinds of errors that will persist through static analysis checks.

It's extremely important to test C programs for correctness and unintended
side-effects. The usual approach is to write unit-level (function-level) tests
that exercise inputs of functions.

There are *MANY* unit test frameworks that can be used for C software. These are
some I've used successfully, in order of my personal preference:

|Framework|Features|Comment|
|---|---|---|
|https://github.com/cpputest/cpputest|Asserts, test runner, mocks, memory leak detection|Moderate learning curve. Very powerful and concise mocking system|
|https://github.com/google/googletest|Asserts, test runner, mocks, leak checker|Very popular and widely used. Simple + concise syntax|
|https://github.com/cgreen-devs/cgreen|Asserts, test runner, mocks, test auto-discovery|Featureful, relatively concise|
|https://github.com/ThrowTheSwitch/Unity|Asserts, test runner|Simple to set up|
|https://github.com/ThrowTheSwitch/CMock|Mock generator for Unity|Simple but verbose and boilerplately mock generator|
|https://github.com/vmg/clar|Asserts, test runner|**Very** simple; used by [libgit2](https://github.com/libgit2/libgit2). **No mocking support**|

## 6.a. Test coverage

Lots of opinions on test coverage.

**0% coverage is certainly not ideal!**

It can be *very* helpful when guiding your testing efforts, especially if you're
not using a strict TDD process.

Tracking test coverage in your build system / CI is usually pretty simple with
gcov + lcov. Lots of information and tutorials on how to do that.

## 7. Use runtime sanitizers

These incur a runtime penalty (memory and execution speed, up to 30% allegedly),
so should be used judiciously in production. However, it can be enabled for unit
tests that are not used to evaluate relative or absolute computation cost for
the software under test (in those cases I think I'd use a separate test system).

The three I recommend are:
- [Address
  Sanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer)
- [Leak
  Sanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizerLeakSanitizer)
- [Undefined Behavior
  Sanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)

In gcc and clang you can enable these when building an executable by applying
these flags to your compile + link commands: `-fsanitize=address -fsanitize=leak
-fsanitize=undefined`

Here's an example of what ASAN can do for you:

```c
void foo_fn(char *arg) {
    // overflows `foo` in the outer stack frame!
    arg[4] = 1;
}

int main(int argc, char **argv) {
    char foo[3];

    foo_fn(foo);

    return 0;
}
```

*Note that cppcheck detects the above error:*

```bash
➜ cppcheck --enable=all test.c
Checking test.c ...
[test.c:8] -> [test.c:2]: (error) Array 'foo[3]' accessed at index 4, which is out of bounds.
```

Running the above program with ASAN enabled yields the following, which points
you to the line where the violation was detected:

![asan example](pics/asan_terminalizer_example.gif)
