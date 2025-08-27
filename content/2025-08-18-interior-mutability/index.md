+++
title = "Interior Mutability in Rust"
date = 2025-08-18
+++

Interior mutability is a Rust pattern that allows you to mutate data even when you have an immutable reference to that data. This pattern is essential for working with the Rust borrow checker, and maintaining the safety guarantees the language already provides, in a way that less constrains your development.

## The Core Problem

In Rust, the borrow checker enforces that:
- You can have multiple immutable references (**&T**) to the same data
- You can have exactly one mutable reference (**&mut T**) to the same data
- You cannot have both immutable and mutable references simultaneously

This creates a challenge: **How can we decide (after compile time) that we want to mutate some data that has been deemed immutable, and is this possible without violating Rust's safety guarantees?**

## Rust's Solution: Interior Mutability

Rust provides several types that implement "interior mutability" - they allow mutation through immutable references in a controlled, safe way. At a high-level, this pattern enables the following:

1. Multiple threads can share immutable references while safely mutating the data in a synchronized fashion.
2. Methods can take **&self** instead of **&mut self** while still allowing mutation.
3. Memory safety guarantees.

## Why is Interior Mutability not Breaking Rust Ownership Rules?

I didn't really understand why interior mutability was allowed based on my initial understanding of Rust's ownership rules, especially the fact that it seemed to break my understanding that data must be specified, at compile time, to be referenced either mutably or immutably, while interior mutability allows for taking data, which was borrowed immutably, and then mutating inner state from this reference. Doesn't this defeat the whole purpose of ownership?

Rust enforces compile-time borrowing rules, statically determining whether there is any scope which violates the rule of: **one mutable reference or multiple immutable references at a time**.

At **runtime**, types like **RefCell** and **Mutex** continue to enforce rust ownership semantics at the "container" level. That is to say, these container types still must explicitly be stated to be mutable or not. However, these containers move the **one mutable reference or multiple immutable references at a time** of the inner data to runtime. We see this in the following example of the same borrowing violation with and without access through **RefCell**.

The point which got me over the hump of understanding interior mutability was recognizing it to be the choice to **push down some of the memory safety guarantees of Rust, from compile-time, to runtime**. No borrow checker violations are allowed by interior mutability, but rather than resulting in a compilation error, it will instead cause a runtime panic when the violation occurs, and still ensures the behavior halts prior to this violation having any adverse effect.

```rust

# compiler error
fn main() {
    let mut data = vec![1, 2, 3];
    let r1 = &data;           // Immutable borrow
    let r2 = &mut data;       // ERROR: Cannot borrow as mutable
    println!("{:?} {:?}", r1, r2);
}

# runtime error
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);
    let r1 = data.borrow();      // Immutable borrow
    let r2 = data.borrow_mut();  // PANIC: Already borrowed
    println!("{:?} {:?}", r1, r2);
}
```

Either way, Rust does not allow violating **one mutable reference or multiple immutable references at a time**, its just a matter of whether this is assessed at compile time, or runtime.