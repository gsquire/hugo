---
title: "Rust Redis Modules"
date: 2018-01-21T19:23:57-08:00
draft: true
---

# Redis Modules with Rust
With newer versions of Redis, the modules API lets users extend Redis' already powerful functionality.
To quote the Redis website:

> Redis modules make possible to extend Redis functionality using external modules, implementing
> new Redis commands at a speed and with features similar to what can be done inside the core itself.

The module API is defined in a single header file with the main goal of writing them in C but also
allowing for any language with C bindings. With that in mind, I decided to try and write a simple
module using Rust since it offers a stable FFI story. Once I learned that you can introduce new data
types I ended up creating a simple `MultiMap` type considering Redis' hash type only supports one
value. I was able to leverage many builtin types in Rust with the hardest part being getting values
from the API into a Rust type. It made more sense as I continued to develop but it wasn't obvious at
first. The rest of this post will go from start to finish while writing a module using Rust.

## Generating Bindings
This was an intimidating part at first but it got immensely simpler once I found the correct tools.
I was quite impressed with how easy it was to integrate binding generation into a cargo pipeline.
Cargo offers a [build script](https://doc.rust-lang.org/cargo/reference/build-scripts.html) feature
which let me create the API bindings to use in my FFI module. The first thing I tried was the
[bindgen](https://docs.rs/bindgen) crate since it would handle a lot of boilerplate for me. 
I was able to copy the example verbatim in to a `build.rs` script and then copy the the module
example from Redis' documentation only to run into an issue that I wasn't familiar with. I hadn't used
C in a while and forgot that `static` functions will not be exported in a header file. Since the
`RedisModule_OnLoad` function is static, there were no bindings generated for it. In order to work
around this I had to write a few lines of C to export my own function which was then called from
the API. In order to use this I ended up downloading the [cc](https://docs.rs/cc) crate. After
I had my simple example working I didn't need to worry about the bindings again since they were all
generated behind the scenes with these helpful tools.

## Rust MultiMap
Rust offers a great set of common data structures which I was able to use to implement a `MultiMap`.
By embedding the type inside my own structure, I could write functions that the FFI calls could use
to store new data or read values from a given key. Here is the definition of the `MultiMap`:

```rust
/// `MultiMap` represents a map of `String` keys to a `Vec` of `String` values.
#[derive(Clone, Debug, Default)]
pub struct MultiMap {
    inner: HashMap<String, Vec<String>>,
}
```

Since this is really just a [`HashMap`](https://doc.rust-lang.org/std/collections/struct.HashMap.html)
my implementation just used its available functions for everything I needed. Given the power of
Rust's trait-based generics I was able to write a simple insert function for example.

```rust
/// Insert will add all values passed as an argument to the specified key.
pub fn insert<K, I>(&mut self, key: K, values: I)
where
    K: Into<String>,
    I: IntoIterator<Item = String>,
{
    let entry = self.inner.entry(key.into()).or_insert_with(|| vec![]);
    values.into_iter().for_each(|item| entry.push(item));
}
```

The `MultiMap` module turned out to be less than eighty lines allowing me to focus on using the
Redis API to implement it natively.

## FFI
In order to implement a new data type, the API asks for a custom structure along with a function
call to `RedisModule_CreateDataType`. This structure allows the user to define things like how it
is loaded into memory or serialized to disk along with a unique name. After that is initialized, you
are able to create commands for this type using the `RedisModule_CreateCommand` function.

### Data Type Initialization
In order to create a new data type, you must define how it can be serialized to disk, read from the
snapshot file, and how to handle freeing its memory. In the [`MultiMap` example](https://github.com/gsquire/redis-multi-map/blob/65df97effbf43a16d5ceaba8c4131c3a0afcf17f/src/lib.rs#L352) we have to use `Option` types to represent
function pointers that the API expects. These functions help with the persistence that was mentioned
earlier. These functions were relatively straightforward to write once I found a good way to represent
the structure on disk.

### Command Functions
In order to create a command that Redis can understand you can use the `RedisModule_CreateCommand`
function and pass it values such as a command name, command function, and various flags. As an example,
I will walk through my length function which returns the length of the values stored at a given key.

The first few lines enable automatic memory management as defined by the API as well as a check
for the number of arguments for the function:

```rust
ffi::RedisModule_AutoMemory.unwrap()(ctx);

if argc != 3 {
    return ffi::RedisModule_WrongArity.unwrap()(ctx);
}
```

Then I unpack the arguments from the C array into a Rust slice and then grab the key name denoted
by the second argument in the slice:

```rust
let args = slice::from_raw_parts(argv, argc as usize);
let key = ffi::RedisModule_OpenKey.unwrap()(
    ctx,
    args[1],
    (ffi::REDISMODULE_READ as i32) | (ffi::REDISMODULE_WRITE as i32),
) as *mut ffi::RedisModuleKey;
```

After this, we do a validity check on the key to make sure it is actually a `MultiMap` type:

```rust
if invalid_key_type(key) {
    return reply_with_error(ctx, ffi::ERRORMSG_WRONGTYPE);
}
```

Then we can work with the key and access the Rust API that I created:

```rust
let map = ffi::RedisModule_ModuleTypeGetValue.unwrap()(key) as *mut MultiMap;
if map.is_null() {
    ffi::RedisModule_ReplyWithLongLong.unwrap()(ctx, 0);
} else {
    let m = &mut *map;
    let map_key = string_from_module_string(args[2]);
    ffi::RedisModule_ReplyWithLongLong.unwrap()(ctx, m.key_len(map_key) as i64);
}
```

This is the function used above to convert a `RedisModuleString` into something that Rust could use
while interacting with the `MultiMap`.

```rust
/// Perform a lossy conversion of a module string into a `Cow<str>`.
pub unsafe extern "C" fn string_from_module_string(
    s: *const ffi::RedisModuleString,
) -> Cow<'static, str> {
    let mut len = 0;
    let c_str = ffi::RedisModule_StringPtrLen.unwrap()(s, &mut len);
    CStr::from_ptr(c_str).to_string_lossy()
}
```

We end our function by returning an integer to denote the success or failure following the Unix
tradition of zero and one respectively.

## Conclusion
I enjoyed working with Rust's FFI capabilities and didn't run into too many issues. Once I grasped how
Rust interacts with C enough it was easy to make progress using the Redis API. My current
implementation could use some polish and more tests. I will list the repository below for anyone who
is interested. Feel free to open issues, make pull requests, or just ask questions as well.

Thanks for reading. The repository can be found [here](https://github.com/gsquire/redis-multi-map).
More information on Redis modules can be found [here](https://redis.io/topics/modules-intro).
