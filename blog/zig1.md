# Iterative Replacement of C with Zig
<div class="published"><time datetime="2017-07-19">19 Jul 2017</time></div>

[Zig](http://ziglang.org/) is a programming language in a similar realm as C.
Being more modern, it has a number of useful constructs such as sum types,
compile-time introspection, improved error handling and no preprocessor!

This post will not describe the language itself (check the project page for
that), but will show how it can be
used to convert an existing C code-base into zig. We will look at a simplistic
example, but the general strategy remains the same.

The `zig` compiler that will be used in this post can be found [here](https://github.com/zig-lang/zig).

*The finished conversion result is at [this](https://github.com/tiehuis/zig-replace-c)
repository. For each heading, there is a corresponding commit which describes
all changes made at each point in the conversion process.*

## The Project
<small>[commit](https://github.com/tiehuis/zig-replace-c/commit/572d12a60187799cacb649c44b0e2a8eb7be33c7)</small>

The project we will be replacing has the following structure:

```text
$ tree .
.
├── compute.c
├── compute.h
├── compute_helper.c
├── compute_helper.h
├── display.c
├── display.h
├── main.c
└── Makefile
```

The contents of the files are:

#### main.c

```c
#include "display.h"
#include "compute.h"

int main(void)
{
    display_char(compute('A'));
}
```

#### display.c

```c
#include <stdio.h>

void display_char(char c)
{
    printf("%c\n", c);
}
```

#### compute.c

```c
#include "compute_helper.h"

char compute(char a)
{
    return compute_helper(a) + 5;
}
```

#### compute_helper.c

```c
char compute_helper(char a)
{
    return a + 1;
}
```

#### Makefile

```make
SRCS := compute.c compute_helper.c display.c main.c
OBJS := $(SRCS:%.c=build/%.o)

main: $(OBJS)
	gcc -o main $(OBJS)

$(OBJS): build/%.o: %.c | mkdirs
	gcc -std=c99 -c $< -o $@

mkdirs:
	@mkdir -p build

clean:
	rm -rf build main
```

The `.h` file contents simply expose their corresponding `.c` implementations.

```text
$ make
gcc -std=c99 -c compute.c -o build/compute.o
gcc -std=c99 -c compute_helper.c -o build/compute_helper.o
gcc -std=c99 -c display.c -o build/display.o
gcc -std=c99 -c main.c -o build/main.o
gcc -o main build/compute.o build/compute_helper.o build/display.o build/main.o

$ ./main
G
```

## The Zig Build System
<small>[commit](https://github.com/tiehuis/zig-replace-c/commit/0704b4455bc23f1dae820fe2791dcc56368e7aaa)</small>

The first thing we will change is replacing the `Makefile` with zigs own custom
build system. The build system of zig is written in zig itself, which reduces
the requirement of knowing the oft-arcane Makefile idiosyncrasies.

#### build.zig

```text
const Builder = @import("std").build.Builder;

pub fn build(b: &Builder) {
    const exe = b.addCExecutable("main");
    exe.addCompileFlags([][]const u8 {
        "-std=c99"
    });

    const source_files = [][]const u8 {
        "compute.c",
        "compute_helper.c",
        "display.c",
        "main.c"
    };

    for (source_files) |source| {
        exe.addSourceFile(source);
    }

    exe.setOutputPath("./main");
    b.default_step.dependOn(&exe.step);
}
```

First, we begin by specifying the main executable which we will be building.
This constructs an object which represents a build-step. For each source
file in our project, we simply add the file to main executable step.

This approach is far more imperative than the declarative approach of a
Makefile. In my view, this is a good choice. Makefiles whilst concise can
become exceedingly opaque and hard to parse, especially as a project grows and
extra conditions need to be handled.

```text
$ zig build --verbose
cc -c compute.c -o zig-cache/compute.c.o -std=c99
cc -c compute_helper.c -o zig-cache/compute_helper.c.o -std=c99
cc -c display.c -o zig-cache/display.c.o -std=c99
cc -c main.c -o zig-cache/main.c.o -std=c99
cc zig-cache/compute.c.o zig-cache/compute_helper.c.o zig-cache/display.c.o \
    zig-cache/main.c.o -o main -Wl,-rpath,zig-cache -rdynamic

$ ./main
G
```

## First C Replacement
<small>[commit](https://github.com/tiehuis/zig-replace-c/commit/c74cba439b36fa7f6de41a155f156c06083bccfb)</small>

The first actual source code we will replace is `compute.c`.

#### compute.zig

```text
use @cImport(@cInclude("compute_helper.h"));

export fn compute(a: u8) -> u8 {
    compute_helper(a) + 5
}
```

This snippet demonstrates a few features of zig. First, zig is able to parse
C header files directly. No binding interface needing! In this case, the `use`
statement will bring all definitions from `compute_helper.h` into the global
namespace, allowing us to call the `compute_helper` function.

The other important thing to note here is the `export` specifier on our function.
This is important as it tells zig that it should compile this against the C ABI.
This means we can call this function from within other C files.

Since our header files are simple, we can continue using them unmodified. Zig does
automatically generate C headers as well however. We can compare these against
the expected definitions to make sure that we implemented the function
correctly.

#### zig-cache/compute.zig.h

```c
#ifndef COMPUTE_2E_ZIG_H
#define COMPUTE_2E_ZIG_H

#include <stdint.h>

#ifdef __cplusplus
#define COMPUTE_2E_ZIG_EXTERN_C extern "C"
#else
#define COMPUTE_2E_ZIG_EXTERN_C
#endif

#if defined(_WIN32)
#define COMPUTE_2E_ZIG_EXPORT COMPUTE_2E_ZIG_EXTERN_C __declspec(dllimport)
#else
#define COMPUTE_2E_ZIG_EXPORT COMPUTE_2E_ZIG_EXTERN_C __attribute__((visibility ("default")))
#endif

COMPUTE_2E_ZIG_EXPORT uint8_t compute(uint8_t a);
COMPUTE_2E_ZIG_EXPORT __attribute__((__noreturn__)) void __zig_panic(const uint8_t * message_ptr, uintptr_t message_len);

#endif
```

### Build System Modification

The second step we need to perform is modifying `build.zig` to compile both
C and zig files and link them together.

#### build.zig

```text
const Builder = @import("std").build.Builder;

pub fn build(b: &Builder) {
    const exe = b.addCExecutable("main");
    b.addCIncludePath(".");
    exe.addCompileFlags([][]const u8 {
        "-std=c99"
    });

    const source_files = [][]const u8 {
        "compute_helper.c",
        "display.c",
        "main.c"
    };

    for (source_files) |source| {
        exe.addSourceFile(source);
    }

    const zig_source_files = [][]const u8 {
        "compute.zig",
    };

    for (zig_source_files) |source| {
        const object = b.addObject(source, source);
        exe.addObject(object);
    }

    exe.setOutputPath("./main");
    b.default_step.dependOn(&exe.step);
}
```

This is mostly same, except we now have a list of zig source files as well. This
should be fairly self-explanatory; for each zig source, we create an object
build step. This is then added to the exe build step.

Note that we also add the current directory to the C include path. This is
important since the `@cInclude` function used by zig does not read headers from
the local directory.

```text
$ zig build --verbose
zig build-obj compute.zig --cache-dir zig-cache --output zig-cache/compute.zig.o \
    --output-h zig-cache/compute.zig.h --name compute.zig -isystem .
cc -c compute_helper.c -o zig-cache/compute_helper.c.o -std=c99 -I zig-cache
cc -c display.c -o zig-cache/display.c.o -std=c99 -I zig-cache
cc -c main.c -o zig-cache/main.c.o -std=c99 -I zig-cache
cc zig-cache/compute.zig.o zig-cache/compute_helper.c.o zig-cache/display.c.o \
    zig-cache/main.c.o -o main -Wl,-rpath,zig-cache -rdynamic

$ ./main
G
```

## Using the Zig Standard Library
<small>[commit](https://github.com/tiehuis/zig-replace-c/commit/8a8f571d7802dd416b81269830ccd7313b810b51)</small>

The next file we will replace is `display.c`.

#### display.zig
```text
const std = @import("std");
const printf = std.io.stdout.printf;

export fn display_char(c: u8)
{
    %%printf("{c}\n", c);
}
```

Since we want to end up using only zig, we can replace the C printf statement
with zig's own stdlib implementation. Zig does not depend on libc at all.
Because this is the only use of libc in our project, we can use the `nostdlib`
to enforce this during our C compilation.

#### build.zig
```text
exe.addCompileFlags([][]const u8 {
    "-std=c99",
    "-nostdlib",
});
```

The only other changes are removing `display.c` from the C sources, and adding
`display.zig` to the zig sources.

```text
$ zig build --verbose
zig build-obj compute.zig --cache-dir zig-cache --output zig-cache/compute.zig.o \
    --output-h zig-cache/compute.zig.h --name compute.zig -isystem .
zig build-obj display.zig --cache-dir zig-cache --output zig-cache/display.zig.o \
    --output-h zig-cache/display.zig.h --name display.zig -isystem .
cc -c compute_helper.c -o zig-cache/compute_helper.c.o -std=c99 -nostdlib -I zig-cache -I zig-cache
cc -c main.c -o zig-cache/main.c.o -std=c99 -nostdlib -I zig-cache -I zig-cache
cc zig-cache/compute.zig.o zig-cache/display.zig.o zig-cache/compute_helper.c.o \
    zig-cache/main.c.o -o main -Wl,-rpath,zig-cache -rdynamic

$ ./main
G
```

## Removing Header Files
<small>[commit](https://github.com/tiehuis/zig-replace-c/commit/e24d3141956521f7c950ce8ff8a48e6210fe2e4e)</small>

As we get further along in our replacement, we will eventually reach the point
where we have zig files which are not used by any other C files. This is great
as it means we can remove the header files.

Consider now as we change `compute_helper.c`.

#### compute_helper.zig

```text
pub fn compute_helper(a: u8) -> u8
{
    a + 1
}
```

The only dependency on this is `compute.zig`. We don't need to export this using
the C ABI and can just mark it `pub` for visibility. `compute.c` can then be
changed to import a zig file instead.

#### compute.zig

```text
pub use @import("compute_helper.zig");

export fn compute(a: u8) -> u8 {
    compute_helper(a) + 5
}
```

```text
$ zig build --verbose
zig build-obj compute.zig --cache-dir zig-cache --output zig-cache/compute.zig.o \
    --output-h zig-cache/compute.zig.h --name compute.zig -isystem .
zig build-obj compute_helper.zig --cache-dir zig-cache --output zig-cache/compute_helper.zig.o \
    --output-h zig-cache/compute_helper.zig.h --name compute_helper.zig -isystem .
zig build-obj display.zig --cache-dir zig-cache --output zig-cache/display.zig.o \
    --output-h zig-cache/display.zig.h --name display.zig -isystem .
cc -c main.c -o zig-cache/main.c.o -std=c99 -nostdlib -I zig-cache -I zig-cache -I zig-cache
cc zig-cache/compute.zig.o zig-cache/compute_helper.zig.o zig-cache/display.zig.o \
    zig-cache/main.c.o -o main -Wl,-rpath,zig-cache -rdynamic

$ ./main
G
```

## The Final File
<small>[commit](https://github.com/tiehuis/zig-replace-c/commit/1bbee329aebbbb8578623d4b8e1967904fa6dcb1)</small>

Our project now has only 1 remnant left of C. Let's remove it all!

#### main.zig

```text
use @import("display.zig");
use @import("compute.zig");

pub fn main() -> %void {
    display_char(compute('A'));
}
```

The main things to note here are the `use` statements for import. Since we were
converting a C project, we didn't initially have any namespacing. Since zig has
a proper module system we usually strongly prefer assigning our imports to a
constant. e.g. `const display = @import("display.zig")`.

Now, we need to edit our `build.zig` file.

#### build.zig

```
const Builder = @import("std").build.Builder;

pub fn build(b: &Builder) {
    const exe = b.addExecutable("main", "main.zig");

    exe.setOutputPath("./main");
    b.default_step.dependOn(&exe.step);
}
```

Much simpler! Zig can make use of the implicit dependency graph formed between
imports. Individual object files do not need to be built for each file
explicitly.

```text
$ zig build --verbose
zig build-exe main.zig --cache-dir zig-cache --output main --name main

./main
G
```

## Closing

Zig makes this type of iterative conversion comparatively easier than most other
languages. For larger projects there will be unknown difficulties however. These
will be continually improved as the language becomes more stable and refined.

Being able to easily replace C with a newer modern alternative is a real bonus
in terms of safety and ergonomics. See this
[post](http://andrewkelley.me/post/intro-to-zig.html) by the creator of the
language some short examples of improvements.

If you want to know more about zig as a language, check out the [project
page](http://ziglang.org/).
