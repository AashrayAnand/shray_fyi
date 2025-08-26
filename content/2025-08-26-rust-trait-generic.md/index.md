+++
title = "Rust Polymorphization vs Monomorphization: Traits and Generics"
date = 2025-08-26
+++

# Rust Polymorphization vs Monomorphization: Traits and Generics

Rust has the concepts of **Traits** and **Generics** that enable some of the OOP-style coding conventions and code re-use most people have seen before in Java, C++ or other OOP languages, but is not 1:1 with other languages and in some case provides safer code re-use at compile time.

## Traits and Generics Overview

Generics are something specifically that we see in other languages quite often, which allow for templatizing code for later-to-be specified data types, to reduce the amount of duplicate code in a system. An example of in C++ shows how a **max** function can be generically defined, which relies on the underlying type's comparator facility.

```cpp
template <typename T>
[[nodiscard]]
constexpr T max(T x, T y) noexcept {
    return x < y ? y : x;
}
```

Traits are most simillar to **Interfaces** in other programming languages, serving as a way to define a contract for a set of behaviors (functions) that a concrete type must implement, to be considered to "have" this trait. As an example, we could define a &**StorageEngine** trait, which specifies the following contract:

```rust

pub trait StorageEngine {
    fn set(&mut self, key: &str, value: &str);
    fn get(&self, key: &str) -> Option<String>;
    fn del(&mut self, key: &str);
}
```

Then, any concrete type which implements this contract can be considered to have the StorageEngine trait. Similar to interfaces or base classes in other languages, traits can be used as a parameter or type definition to generically specify a set of eligible concrete types e.g.

```rust

pub fn swap_keys_values(engine: Box<dyn StorageEngine>, key1: &str, key2: &str) {
    if let Some(value1) = engine.get(key1) {
        if let Some(value2) = engine.get(key2) {
            engine.set(key1, value2);
            engine.set(key2, value1);
        }
    }
}

impl StorageEngine for KeyValueStorageEngine {
    // Implement get/set/del
}

impl StorageEngine for BTreeStorageEngine

let kvs = Box::new(KeyValueStorageEngine());
let bt = Box::new(BTreeStorageEngine());
swap_keys_values(kvs, "foo", "bar"); 
swap_keys_values(bt, "foo", "bar"); 
```

Rust also has the notion of [derivable traits](https://doc.rust-lang.org/book/appendix-03-derivable-traits.html), that are common behaviors that can be automatically derived by specifying the **derive** attribute on a type. E.g. deriving the **Clone** trait automatically enables deep-copying a value.

```rust

#[derive(Clone)]
pub enum CommandType {
    GET = 1,
    SET = 2,
}

let cmd = CommandType::GET;
let cmd_clone = cmd.clone(); // We get this by deriving Clone
```

I think the main difference I see personally is that Rust is trying to "compose" behavior when implementing and deriving traits, and doesn't gravitate towards a hierarchial approach for shared behavior like inheritance. It's quite common for classes to derive or implements a set of traits, rather than incrementally building up from a base class to concrete implementations in an inheritance-like fashion.

This definitely tripped me up for a while in how I was composing code in Rust and avoiding to be too OOP-minded when defining structures and shared behaviors.

## Monomorphization: Generics Create Multiple Copies

When you use generics, Rust creates separate copies of your function for each concrete type used. This is called **monomorphization**.

```rust
fn process<T: std::fmt::Display>(value: T) {
    println!("Processing: {}", value);
}

fn main() {
    process(42i32);      // Creates process::<i32>
    process("hello");    // Creates process::<&str>
    process(3.14f64);    // Creates process::<f64>
}
```

At compile time, Rust generates three separate functions:
- `process::<i32>(value: i32)`
- `process::<&str>(value: &str)` 
- `process::<f64>(value: f64)`

This gives you **zero-cost abstractions** - no runtime overhead, but larger binary size.

## Dynamic Dispatch: Trait Objects Use Runtime Polymorphism

Trait objects like `Box<dyn Error>` use **dynamic dispatch** - one function handles multiple types at runtime through a vtable.

```rust
fn handle_error(error: Box<dyn std::error::Error>) {
    println!("Error: {}", error);  // Calls through vtable
}

fn main() {
    let io_err = std::io::Error::new(std::io::ErrorKind::Other, "IO failed");
    let parse_err = "Parse failed".to_string();
    
    handle_error(Box::new(io_err));    // Same function
    handle_error(Box::new(parse_err)); // Same function
}
```

Internally, `Box<dyn Error>` contains:
```rust
struct TraitObject {
    data: *mut (),           // Pointer to actual error
    vtable: *const VTable,   // Pointer to method implementations
}
```

## Why `Box<dyn std::error::Error>`?

This pattern solves three problems we encounter in error handling:

**1. Multiple Error Types**

```rust

pub fn send_command() -> Result<String, Box<dyn std::error::Error>> {
    let stream = TcpStream::connect("127.0.0.1:7001")?;  // io::Error
    let command = parse_command()?;                       // ParseError  
    let response = read_response(stream)?;                // io::Error
    Ok(response)
}
```

**2. Unknown Size at Compile Time**

```rust

// This won't compile - trait has unknown size
fn bad_return() -> Result<String, dyn std::error::Error> { ... }

// Box puts it on the heap with known size (pointer)
fn good_return() -> Result<String, Box<dyn std::error::Error>> { ... }
```

**3. Type Erasure**

The caller doesn't need to know the specific error type - they just know it implements `Error`:

```rust

match send_command() {
    Ok(response) => println!("{}", response),
    Err(e) => println!("Failed: {}", e),  // Works for any Error
}
```

## Rust vs C++ Comparison

**C++ Templates vs Rust Generics:** Both use monomorphization, but C++ templates are more like "smart macros" - they can generate invalid code that only fails when instantiated. Rust generics are type-checked at definition time.

```cpp
// C++ - This compiles even though T might not have foo()
template<typename T>
void bad_function(T value) {
    value.foo();  // Only checked when instantiated
}
```

```rust

// Rust - This won't compile without trait bounds
fn good_function<T>(value: T) {
    value.foo();  // ❌ Error: no method `foo` found for type `T`
}

// Must specify what T can do:
fn good_function<T: HasFoo>(value: T) {
    value.foo();  // ✅ Checked at definition time
}
```

**C++ Virtual Functions vs Rust Traits:** C++ uses inheritance and virtual functions for runtime polymorphism. Rust traits are more like C++ **concepts** (C++20) - they define behavior contracts without inheritance. C++ has no direct "trait" equivalent, but uses abstract base classes or CRTP patterns to achieve similar results.