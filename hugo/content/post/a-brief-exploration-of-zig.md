+++
date = "2020-05-17T10:57:01-07:00"
title = "A Brief Exploration of Zig"

+++

## Why Zig?
[Zig](https://ziglang.org/) is an intriguing new language that aims to fill a niche in the
low-level development world. It offers a syntax that looks like a derivative of C along with
some exciting features that make it an appealing modern alternative. The main website highlights
a lot of the ideas it tries to encompass so I won't list them here. What drew me in most though is
Zig's builtin support for cross-compilation and the great support for embedding C sources. Considering
that these are large endeavors within themselves, I wanted to explore the language a little more by
writing a trivial command line program with it. At the time of writing this I am using Zig version 0.6.0 on
MacOS. Since Zig is rapidly evolving, some of the code snippets may not compile in the future. The code
may not adhere to best practices as well. If you find mistakes or non-idiomatic parts, the source is
hosted on GitHub. I will always review patches or suggestions. I also draw parallels to other languages
merely as a reference point for those familiar with them. This post is not intended to compare languages
or posit any ideas that one may find superior.

## Getting Up to Speed
Zig can be easily installed on your host system with a package manager or pre-built binary from
[GitHub](https://github.com/ziglang/zig/releases). The `zig` tool helps you build and test your source
among many other things. Zig source files have a `.zig` extension so I created `cat.zig` for my first
project. Since `cat` is ubiquitous on Unix platforms, I wanted to write a small clone of it. The Zig
version does not support any command line switches or arguments but it can print any number of files to
standard output. I also wrote a Makefile to help with the compile and edit cycle. Once I had some
familiar tooling in place, I started reading through the Zig [documentation](https://ziglang.org/documentation/0.6.0/).
This was the best way for me to understand some of the philosophy of Zig while getting a better
grasp on how to layout the source. I highly recommend looking at the basics of this page before writing
any Zig.

## Diving In
For those familiar with C like languages, program execution starts at the main function. Zig is no different
except it requires `main` to be public. This is denoted with the `pub` keyword. An example of how this may look
is here:

```c
pub fn main() void {

}
```

In this case, `void` signifies that there is no return value but `main` can return anything you wish. Since we
are writing a command line application we will likely want to access the program's arguments. I found that the
standard library allows you to access them through an iterator type. Backing up a bit, we will import the
standard library via Zig's builtin `@import` function. Here is what we need:

```c
// Bring in the standard library.
const std = @import("std");

// Now that we are using the `std` namespace, we want to bring in the `fs` module too.
const fs = std.fs;
```

Now that we have imported these, we can go ahead and access our program's arguments. Like I said previously, the
arguments were part of an iterator type returned by the `args` function in `std.process`. The first thing I noticed
however is that calling `next` to advance the iterator required an `allocator` as seen
[here](https://ziglang.org/documentation/0.6.0/std/#std;process.ArgIterator.next). I decided to read through the
source instead and stumbled upon `nextPosix` which although is limited to Unix platforms, did what I wanted.
This function returns a slice of bytes or `null` as denoted by the return type of `?[]const u8`. This syntax is
what Zig refers to as an optional type which is akin to Rust's [Option](https://doc.rust-lang.org/std/option/enum.Option.html) type.
Instead of having it be generic over some type, it uses `null` in place of `None`. So after consuming the first argument returned by the iterator, we can
check our next value returned to see if we got a file name as an argument. If it is in fact `null`, we can print a message
to standard error and exit with a non-zero return status. Something I noticed with the
[warn](https://ziglang.org/documentation/0.6.0/std/#std;debug.warn) function is that it requires a second argument regardless if you use a format
specifier or not. Perhaps that is a design decision or maybe a limitation of the current implementation. Anyway, if we got a valid file name
we will go ahead and try and read it. Since the argument iterator returns an optional type, we extract the value from it via the `?` operator as such:

```c
readFile(fname.?)
```

Calling our `readFile` function leads us into the core of the program. The definition of this function reads as such:

```c
fn readFile(fname: []const u8) !void
```

It takes a single argument as a slice of bytes and either returns nothing or an error as denoted by `!`. In this case we use
Zig's `anyerror` [error set](https://ziglang.org/documentation/0.6.0/#The-Global-Error-Set). You must capture error values
and the compiler will enforce this. Luckily Zig has a lot of ways to handle them as described in the documentation. I will
illustrate one of them in this example. First off, we get a handle to standard output through the standard library. This was
relatively straightforward via the documentation. However, opening a file for reading wasn't as obvious to me. I fumbled around
with various functions that dealt with absolute paths at first but that would impose an strict requirement for the program. Again I
tried reading through the Zig source and found a function called `cwd` that allowed you to get a handle to a `Dir` structure. This
type had functions that let you open files much more easily so I ended up with:

```c
// Note the `try` keyword here.
var f = try fs.cwd().openFile(fname, fs.File.OpenFlags{ .read = true });

// Note that Zig has `defer` like Go does. This way we know our file handle will be closed regardless of function flow.
defer f.close();
```

The most significant part of this snippet is the [`try`](https://ziglang.org/documentation/0.6.0/#try) keyword. This offers a
shortcut for catching errors and returning them. If we cannot open the file then the function will short-circuit and return the
error associated with the call to `openFile`. If the call succeeds though, invoking `defer f.close()` ensures that our open handle
gets closed. This is another nice-to-have so we don't have to litter the function flow with calls to `close`. It also enforces
good developer habits since it is easy to forget to close resources if there is a lot of complex function logic.

Now that we have an opened file, we can read its contents and print them to standard output. We initialize an array of fixed size to
fill with bytes and continually call `read` until it returns 0 meaning that we reached the end. This is the same behavior as the C
system call. Here is how that looks:

```c
// `undefined` here means that the memory is not initialized.
var buf: [std.mem.page_size]u8 = undefined;

// `read` can fail so we have to `try`.
// The `[0..]` turns the `buf` array into a slice for `read` to use.
var bytes_read = try f.read(buf[0..]);

// Fill up our buffer and print it to standard output.
while (bytes_read > 0) {
    try stdout.print("{}", .{buf[0..bytes_read]});
    bytes_read = try f.read(buf[0..]);
}
```

## Building Executables
As I mentioned earlier, the `zig` tool lets you build executables very easily. With a couple one line make targets, I was able to build
one for my native machine and another for my Linux server. This is fascinating to me since I didn't need to install any other dependencies.
Zig ships with various libc libraries so all I had to do was pass the `-target` flag to the `zig` tool:

```sh
zig build-exe cat.zig -target x86_64-linux-gnu --release-fast
```

After that I copied it up to my Linux server and was able to use it there as if I had built it on that machine. That is incredible.

## Conclusion
Zig was a lot of fun to work with for me. I know the language is still young and changing daily so I expected a few paper cuts here and there.
I hope to take what I learned and create some patches myself. If nothing more I hope I encouraged you to do the same or to help out in another way.
If you have any comments or criticisms, my email is available in the page's header. The source is also available [here](https://github.com/gsquire/cat-zig).
Thanks for taking the time to read this!
