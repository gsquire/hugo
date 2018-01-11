+++
title = "Rust In 2018"
date = "2018-01-10T19:05:55-08:00"
draft = true

+++

# Rust in 2018
This short post is a reflection on Rust in 2017 and some ideas I would love to see in 2018 written
in the spirit of [this](https://blog.rust-lang.org/2018/01/03/new-years-rust-a-call-for-community-blogposts.html)
blog post. I will keep things brief and try not to repeat what others have already shared.
There have been many wonderful ideas that makes me even more enthusiastic about Rust so I hope that
I can add to the excitement with a few of my own.

## Asynchronous Programming
Throughout the course of 2017 Rust saw tremendous efforts to make its asynchronous story better.
Futures has laid the groundwork for providing abstractions over things like I/O and CPU bound work.
Rust also saw the addition of [generators](https://doc.rust-lang.org/1.22.0/unstable-book/language-features/generators.html)
in nightly allowing a pleasant [async/await](https://github.com/alexcrichton/futures-await) syntax
that is familiar to developers coming from languages that support this style. This is a feature I
can not wait to see stabilize as it seems like one of the biggest blockers from having networking
programs that compete with Go or C++. A friend of mine at work has even mentioned this as being one
of the last features that Rust needs in order for him to adopt it for any new project.

## Lifetimes
Lifetimes are already a core part of Rust but the reason I am mentioning it is to see if there are
any more resources out there I can learn from. I know the book has a brief part on it as well as the
nomicon but I still think my knowledge only scratches the surface. It would be great to see the
community write up some more in-depth documentation on how I can leverage this feature to make
my programs faster or more elegant. Perhaps this could be seen as the need for better internal documentation
as others have mentioned but it's one of the last corners of Rust I feel I haven't been comfortable with.

## Participation
We are already fortunate to have a brilliant group of individuals who make Rust what it is today. This
includes the core team members, influential contributors, and all those who use Rust. I am constantly
impressed by the amount of effort people have made to ensure that Rust is seen as more than just a
programming language. I want to ask those reading to try and become a more active participant of the
Rust community in 2018. This is an intentionally broad statement as there are a seemingly infinite amount of
ways you can do this. You can contribute patches to Rust projects, write documentation, host a meeting,
or just respond to forum posts to voice your thoughts. I am sure we continually read about how open
and polite our community is and I want to echo that sentiment. Everyone wants Rust to succeed and will
appreciate any effort you can give. I am making it a goal myself to give back to the community more than
I did last year and it would be great even if only one other person shared this idea.

## Conclusion
Thank you to the entire community for your continued efforts that make Rust one of the most
exciting things I have been a part of for a long time.
