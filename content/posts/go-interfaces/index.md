---
title: Go Interfaces
date: 2016-07-03
---

Though plenty of people built up hype around interfaces in Go, I saw them as little more than a keyword. `interface` had obvious expressive powers but nothing more than `for` or `struct`. Definitely nothing that warranted the adjectives – “powerful”, “expressive” – that so many Go developers happily attached to them.

<!--more-->

Go’s `interface` is similar to Java’s `interface` that you might be familiar with, but with some key differences. Like in Java, interfaces in Go are all about _behavior_. You can’t wrap instance variables or other data, only the the methods on top of that data. For example, a simple interface would be:

    type NamedReader interface {
        Name() string
        io.Reader
    }


Any type that satisfies this interface knows how to generate an ID. The implementation, however, is entirely hidden. We could have a counter that is incremented on each call to `NextID`, we could read from `/dev/urandom` or we could generate a GUID.
