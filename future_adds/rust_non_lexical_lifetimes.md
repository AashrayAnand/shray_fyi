+++
title = "A little bit of Rust: Non-Lexical Lifetimes and Mutability"
date = 2025-XX-XX
+++

One way where Rust has become a bit "smarter" about how ownership and borrowing are checked is by applying "non-lexical lifetime". This basically means that rather than blindly enforcing the scope that a variable is created e.g. treating the lifetime of a function-local variable as the entire lifetime of that function, Rust will look at the underlying usage of these variables and enforce a tighter lifetime e.g. scoping the lifetime of a function-local variable as the time from its declaration to its last usage.

It's important to call out that NLL only applies for references, not for owned values. 

## Non-Lexical Lifetimes (NLL)

**Before NLL**: Borrows lasted until the end of their lexical scope
**With NLL**: Borrows end at their last use

```rust

let mut vec = vec![1, 2, 3];
let r = &vec[0];        // Borrow starts
println!("{}", r);      // Last use of r
vec.push(4);           // ✅ Borrow ends here (NLL), mutation allowed

// vs. lexical scopes (old behavior):
{
    let r = &vec[0];    // Borrow starts
    println!("{}", r);  // Last use
    vec.push(4);       // ❌ Error: borrow lasts until end of scope
}                      // Borrow ends here
```

This contrasts with the obvious case below, where the NLL of the immutable reference to vec now goes beyond the mutable access and will error.

```rust

let mut vec = vec![1, 2, 3];
let r = &vec[0];        // Borrow starts
vec.push(4);           // ❌ Error: borrow NLL is still beyond this scope.
println!("{}", r);      // Last use of r
```