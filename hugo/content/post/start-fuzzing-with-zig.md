+++
date = "2021-06-06T23:23:29+00:00"
title = "Start Fuzzing With Zig"

+++

Fuzzing is a testing technique in which a program generates random data as input for another program
to consume. The goal is to find data which could cause corruption or other fatal issues in the
software that is being tested. Machines will be able to generate test cases much quicker than humans
thus making it a great way to test your software.

Fuzzing has proven effective enough to be included as a first class citizen in Go. At the
time of writing this post you can read about its inclusion in the beta version [here](https://blog.golang.org/fuzz-beta).
This may set a precedent for other languages to follow depending on how successful it is.

Throughout the rest of this short post I hope to cover an interesting way that one may be able to
start fuzzing with Zig.

## Setup
[AFL++](https://aflplus.plus/) is the successor the AFL library that incorporates many more features.
In order to use this library, I used the [`cargo afl`](https://github.com/rust-fuzz/afl.rs) tool which
handles the linking and AFL++ library usage for us quite easily. So you will need both a Rust and Zig tool chain
to follow this example. To see how to install these tool chains you can see the instructions on their
[respective](https://www.rust-lang.org) [websites](https://ziglang.org).

## Demonstration

For this post, we will exercise Zig's `json.validate` function from its standard library. This function
accepts a byte slice and returns a boolean signifying if the JSON is valid or not. In order for the
`cargo-afl` tool to be able to pass input to this function, we must expose it as a C API. Luckily Zig
makes this trivial via the `export` keyword:

```zig
const json = @import("std").json;

export fn validate_json(j: [*]const u8, j_len: usize) usize {
    const source = j[0..j_len];
    if (json.validate(source)) {
        return 0;
    }
    return 1;
}
```

At this point we want to make a library that `cargo-afl` can use so we go ahead and build a library
after writing this in a file called `json.zig`:

```sh
zig build-lib json.zig
```

This should produce an archive called `libjson.a` that you can use to link against later on.

Now we will get the actual fuzzing tool configured. With Rust installed you can run this command:

```sh
cargo install afl
```

This may take a while but once you see it install successfully, we'll make a new binary as such:

```sh
cargo new fuzzig
````

You should be able to jump into this newly created directory and get your manifest edited to use
`cargo-afl`. The file you will want to edit is called `Cargo.toml`. In here you will list the
following dependencies:

```toml
[dependencies]
afl = "0.10"
libc = "0.2"
```

Another file you will want to create is a custom build script. Name this file `build.rs` and add the
following contents in:

```rust
fn main() {
    println!("cargo:rustc-link-search=.");
}
```

This will tell `cargo` to search in the current directory for any static libraries we may be interested
in. Go ahead and copy `libjson.a` into the current directory now as well:

```sh
cp ../libjson.a .
```

Next we will edit the binary file under `src/main.rs`:

```rust
use afl::fuzz;
use libc::size_t;

#[link(name = "json", kind = "static")]
extern "C" {
    fn validate_json(j: *const u8, j_len: size_t) -> size_t;
}

fn main() {
    fuzz!(|data: &[u8]| {
        unsafe {
            validate_json(data.as_ptr(), data.len());
        }
    });
}
```

We set up the `validate_json` API with a string input as well as a corresponding length. Inside the
main function we use the `fuzz` macro that accepts a single parameter as a slice of bytes as input.
Inside here we call the function we want to fuzz which in this case is our `validate_json` function
provided by the Zig API. We have to use `unsafe` here since we are going across FFI boundaries.

The last step we want to do before running the fuzz tool is create a couple directories:

```sh
mkdir in out
```

The `cargo-afl` tool will put any failures inside the out directory and use the in directory as input
seeds. In this case we will create a few files with valid JSON that the library will then build new
inputs with.

The final commands are:

```sh
cargo afl build

# The fuzzing tool will run indefinitely until canceled with CTRL-C.
cargo afl fuzz -i in -o out target/debug/fuzzig
````

## Conclusion
This *is not* the most straightforward way to get started with fuzzing in Zig. But it should provide
little friction in getting started with any fuzz testing you want to do. If you find any bugs, feel
free to drop an issue [here](https://github.com/gsquire/fuzzig/issues).
