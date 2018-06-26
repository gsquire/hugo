---
title: "A Rusty Go at Channels"
date: 2018-06-24T10:44:31-07:00
---

## Channels
Channels are a useful concurrency primitive that enable separate processes to safely communicate
without the need for explicit synchronization. The term processes is used here to loosely describe
independent threads of execution within a program. This can be an OS level thread or a runtime level
thread. Channels can be seen as a pipe to connect these processes and allow them to share
memory with one another. For example a program could spawn any number of processes along with a
channel to transmit results that it gathers. The main process could be configured to receive these
results and handle them accordingly. Not having to use a mutex or other form of guard can be a
useful tool when writing concurrent programs. The rest of the post will dive into channels in both
Go and Rust and how their channel support overlaps.

## Channels in Go
Go offers channels in the form of a built in type that can be used with any other type that Go
allows. Channels can be declared as send only or receive only in type declarations or function
arguments which gives the program another level of type safety. In order to create a channel you can
use the [make](https://golang.org/pkg/builtin/#make) function along with an optional capacity
argument. This capacity dictates whether or not the channel is buffered which has important
implications that I will cover later on. Below is an example of making both types of channels.

```go
// Declare a non-buffered channel of integers.
n := make(chan int)

// Declare a buffered channel of integers.
// This means that the channel can only contain one value at a time.
b := make(chan int, 1)
```

Sending and receving values with a channel is done with the `<-` operator which reads like a
sequential execution.

```go
n := make(chan int)

// Send a value on the channel in a separate goroutine.
go func() {
    n <- 1
}()

// The receive operation uses `<-` as well but is flipped to the front instead.
fmt.Printf("i received %d\n", <-n)
```

Go's channels enable many useful features when dealing with concurrent programs. Go offers
concurrency in the form of "goroutines" which are lightweight runtime level threads that are mapped
onto OS level threads. These goroutines have their own stack and are very cheap enabling the
creation of hundreds or thousands of them at any one point. They are spawned with the `go` keyword.
With that in mind, channels allow goroutines to work with one another in a safe way.

Whether or not a channel is buffered is an important distinction. Non-buffered channels will block
a goroutine on a send operation if no other goroutine is ready to receive. However a buffered channel
will only block after its buffer is filled. This is an important feature to keep in mind when
designing a program as it can cause deadlocks if used incorrectly.

Go also allows the use of the `range` keyword to iterate over a channel. The only way to stop this
iteration is by using the `close` keyword to alert the receiver that there are no more values left.
This is most likely the only time you would need to use the `close` keyword.

```go
// Make a non-buffered channel to pass integers with.
c := make(chan int)

// Spawn off a goroutine that sends values into this channel.
go func() {
    for i := 0; i < 10; i++ {
        // Remember, this will block until the main goroutine can receive its value.
        c <- i
    }
    // Alert the main goroutine that we are done sending values.
    close(c)
}()

// This will iterate until the sending goroutine calls `close`.
for i := range c {
    fmt.Println(i)
}
```

Another useful feature of Go is the `select` statement. The `select` statement allows you to perform
a blocking operation on a set of channels or a default operation if none are ready. If more than one
operation is ready then one is chosen at random. Below is an example of using `select` to manage
multiple channel operations along with a default one.

```go
select {
    // You can see if a send is ready to fire.
    case c <- i:
    // See if we should leave this by receiving a value from this channel.
    case j := <- quit:
        fmt.Printf("got %s, leaving...", j)
    // Just print waiting if either of the operations can't proceed. This is just an example
    // and may not be useful in actual code.
    default:
        fmt.Println("waiting...")
}
```

This is by no means an exhaustive list of things you can do with a channel in Go but it highlights
most of the use cases.

## Channels in Rust
Rust does not have the notion of builtin channels like Go but it does offer both flavors of channels
in the standard library. These are MPSC or "multiple producer, single consumer" enabled and can
be shared across threads. Unlike Go, Rust does not offer runtime threading and instead allows you
to spawn OS level threads through its standard library.

Below is an example of configuring a non-buffered channel in Rust.

```rust
// The standard library imports.
use std::sync::mpsc::channel;
use std::thread::spawn;

// Create a channel pair. They are distinct types unlike in Go.
let (tx, rx) = channel();

// Spawn the thread and move ownership of the sending half into the new thread. This can also be
// cloned if needed since there can be multiple producers.
spawn(move || {
    // Send a value and ignore the error by calling `unwrap`.
    tx.send(1).unwrap();
});

// Receive the value and ignore the error by calling `unwrap`.
println!("received value {:?}", rx.recv().unwrap());
```

What's different here is that the `send` call will not block even if there is no receiver to accept
it. The `recv` call will block however until a value is sent. Therefore these channels are unlike
Go's and cannot be used to synchronize two threads. The only way to achieve that is by using a
buffered channel with a size of zero.

The main inspiration for this blog post however is the newer release of a crate (Rust's form of
libraries) called [crossbeam-channel](https://github.com/crossbeam-rs/crossbeam-channel). This
library aims to bring Go's channels to Rust enabling you to take advantage of the guarantees that
they support. The feature set is not going to be one to one but the similarities outnumber the
differences.

Using the types in this crate, we can illustrate analogous examples to the Go snippets above. To
start we can show how to send values on a channel.

```rust
use crossbeam_channel as channel;

// Create a non-buffered channel.
let (tx, rx) = channel::unbounded();

// Create a buffered channel with a capcity of one.
let (s, r) = channel::bounded(1);

for i in 0..100 {
    // We can send an "infinite" amount of items into the unbounded channel without blocking.
    // This is different from Go as we don't need a receiver ready.
    tx.send(i);
}

// Try and receive one of the values in a blocking fashion.
println!("{:?}", rx.recv());

s.send(1);
// This would block until the receiving half read the value!
s.send(2);
```

Although crossbeam-channel can't use a special operator like `<-`, the API is largely the same. We can control
the behavior of the channels based on which methods are called on them. The sending half of the
channel can call `send` and will block only if it is bounded. The receiving half of the channel
can either block or not as it supports a `recv` and `try_recv` option respectively.

Iteration is also supported by implementing the [Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html)
trait for the receiving half. The only difference here is that iteration stops when the sending half
of the channel is dropped out of scope. One can emulate calling `close` like you would in Go by using the
[drop](https://doc.rust-lang.org/std/mem/fn.drop.html) function to explicitly tell the receiver
that the sender is out of scope.

```rust
use crossbeam_channel as channel;

let (tx, rx) = channel::unbounded();

tx.send(1);
tx.send(2);

// Explicitly `drop` this sender allowing the iterator to close.
drop(tx);

// This will print out:
// 1
// 2
for item in rx {
    println!("{:?}", item);
}
```

The crossbeam-channel crate also allows you select a channel operation which emulates the `select`
statement in Go. It will block until an operation is ready or choose a default case if one is
supplied. It is implemented as a macro in Rust which is expanded to source code at compile time.

```rust
// We have to declare macro usage since there is also a select macro in the standard library.
#[macro_use]
extern crate crossbeam_channel;

use crossbeam_channel as channel;

let (tx1, rx1) = channel::unbounded();
let (tx2, rx2) = channel::unbounded();

tx1.send(1);

// Using the select statement here will choose an operation at random since both will be ready
// to proceed.
select! {
    recv(rx1, val) => println!("got value {:?}", val),
    send(tx2, 2) => (),
}
```

## Putting it All Together
Below is an example of a simulated pool of executors that need to perform some expensive computation.
The thread pool crate allows me to spawn a number of threads that can execute work.

```rust
extern crate crossbeam_channel;
extern crate threadpool;

use std::thread::sleep;
use std::time::Duration;

use crossbeam_channel as channel;
use threadpool::ThreadPool;

fn work(tx: channel::Sender<usize>, task: usize) {
    // Simulate some expensive work that needs to be done.
    // This will sleep for one second.
    sleep(Duration::new(1, 0));

    tx.send(task);
}

fn main() {
    const TASKS: usize = 100;

    let (tx, rx) = channel::unbounded();
    let pool = ThreadPool::new(4);

    // Create 100 superficial units of work and let the thread pool execute them.
    for i in 0..TASKS {
        // By calling clone here, we can share our sending half with each `work` function
        // that is called.
        let tx = tx.clone();
        pool.execute(move || {
            work(tx, i);
        });
    }

    // We are done with our sending half so we can explicitly drop it here.
    drop(tx);

    for i in rx {
        if i % 5 == 0 {
            println!("done with {}% of the work", i);
        }
    }

    println!("done with all of the work");
}
```

## Conclusion
Channels are a powerful concurrency primitive that enable programs to share memory without the
overhead of a lock. They offer features such as blocking or non-blocking sends and receives while
still being straightforward to use. It's much easier to reason about message passing with a channel
than it is to try and synchronize threads in another fashion.

Thanks for reading! If you have any questions feel free to contact me at the email in my GitHub
profile.
