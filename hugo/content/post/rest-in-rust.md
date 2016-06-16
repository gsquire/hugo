+++
date = "2016-06-14T14:56:18-07:00"
title = "REST in Rust"

+++

# Writing a simple REST app in Rust
If you have ever worked with HTTP in Rust, you have probably referred to the
[hyper](hyper.rs) crate. Hyper provides a safe abstraction over HTTP and offers both
a client and server type. It is also the foundation for frameworks such as [Iron](https://github.com/iron/iron)
and [Nickel](https://github.com/nickel-org/nickel.rs).

One of the coolest things about hyper is that its server can be started with anything that
implements the [`Handler`](http://hyper.rs/hyper/v0.9.8/hyper/server/trait.Handler.html) trait.
There is one required method called `handle` and it denotes how the type which implements it
should respond to an incoming connection.

`handle` has the following function signature:

```
fn handle<'a, 'k>(&'a self, Request<'a, 'k>, Response<'a, Fresh>)
```

As you can see it gives us access to a `Request` and `Response` type which are aptly named. If you take a look
at the documentation for these, you will find that they provide the basis to check
for HTTP headers or set the body of an HTTP response.

## `RegexSet`
The regex crate recently added a type called a [`RegexSet`](https://doc.rust-lang.org/regex/regex/struct.RegexSet.html).
This is a powerful construct in that it can match multiple patterns that you provide in a single
scan. Using this idea, I thought it would be fun to write a router with a `RegexSet` matching
requests under the hood.

The result of this is [reroute](https://github.com/gsquire/reroute). This crate provides a `Router`
type that can match patterns you provide and call corresponding functions accordingly. The functions have access
to the `Request` and `Response` along with a type that captures matches in the URI.

## A simple example
```rust
extern crate hyper;
extern crate reroute;

use hyper::Server;
use hyper::server::{Request, Response};
use reroute::{Captures, Router};

fn digit_handler(_: Request, res: Response, c: Captures) {
    println!("captures: {:?}", c);
    res.send(b"It works for digits!").unwrap();
}

fn main() {
    let mut router = Router::new();

    // Use raw strings so you don't need to escape patterns.
    router.get(r"/(\d+)", digit_handler);

    router.finalize().unwrap();

    // You can pass the router to hyper's Server's handle function as it
    // implements the Handle trait.
    Server::http("127.0.0.1:3000").unwrap().handle(router).unwrap();
}
```

Since `Router` implements the `Handler` trait, hyper's `Server` can use it to respond
to connections. With this in mind, you can write a REST app in a few lines and extend this
functionality to suit your own needs. The `Captures` type gives you matches in your URI that the regex
engine finds. So in the example above, if I do a GET against the route "/123", `c` will be
`Some(["/123", "123"])`. This is a `Vector` of the matches returned from the `RegexSet`. You
can use any pattern with grouping that you like and the router will collect them for you. Of course
you don't need to provide groups and can just add routes like "/v1/some/endpoint" as well.

Reroute surely isn't as complete as Iron but its only goal is to route requests in an app.

## Questions or thoughts?
Feel free to contact me through the email on my [GitHub](https://github.com/gsquire) profile.
