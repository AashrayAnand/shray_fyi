+++
title = "A little bit of Rust: Dereferencing and Deref Coercion"
date = 2025-XX-XX
+++

Rust's `Deref` trait is pretty cool and adds some useful syntactic sugar in the language, for helping define how to convert between reference types, and making smart pointers and owned types work seamlessly with their underlying data.

Not only that but there are different types of trait-based deref that Rust uses to write some pretty interesting code.

## Basic Dereferencing

**Manual dereferencing** with `*`:

```rust

let x = 5;
let r = &x;
assert_eq!(5, *r);  // Explicit dereference
```

**Automatic dereferencing** in method calls:

```rust

let s = String::from("hello");
let len = s.len();  // Calls str::len() automatically
```

## Deref Coercion

Automatic conversion from `&T` to `&U` when `T: Deref<Target = U>`:

```rust

// String implements Deref<Target = str>
fn takes_str(s: &str) { }

let owned = String::from("hello");
takes_str(&owned);     // &String → &str via deref coercion
```

This one is pretty cool and explained a question I had early on, that I assume a lot of people wondered, which is why would a refernce to a ```String``` become a ```&str```.

In reality, ```&String``` becomes ```&String.as_str()``` which internally returns a reference to the underlying byte vector -> ```&string_owned_data[..]```.

### Multiple Levels

```rust

let s = String::from("hello");
let r1 = &s;      // &String  
let r2 = &r1;     // &&String
takes_str(r2);    // &&String → &String → &str
```

## Deref vs DerefMut

### `Deref` - Immutable Access

```rust

use std::ops::Deref;

impl Deref for Box<T> {
    type Target = T;
    fn deref(&self) -> &T { &**self }
}

let boxed = Box::new(String::from("hello"));
let len = boxed.len();  // &Box<String> → &String → &str
```

### `DerefMut` - Mutable Access

```rust

use std::ops::DerefMut;

impl DerefMut for Box<T> {
    fn deref_mut(&mut self) -> &mut T { &mut **self }
}

let mut boxed = Box::new(String::from("hello"));
boxed.push('!');  // &mut Box<String> → &mut String
```

## Key Differences

| Trait | Access | Requirement | Use Case |
|-------|--------|-------------|----------|
| `Deref` | `&T → &U` | Shared reference | Reading data |
| `DerefMut` | `&mut T → &mut U` | Exclusive reference | Modifying data |

This is another pretty cool idea, and was the first thing that lead to me learning about the ```interior mutability``` pattern, which I will definitely write about in the future.

## Custom Implementation

```rust

struct MyString {
    data: String,
}

impl Deref for MyString {
    type Target = str;
    fn deref(&self) -> &str { &self.data }
}

impl DerefMut for MyString {
    fn deref_mut(&mut self) -> &mut str { &mut self.data }
}

let mut my_s = MyString { data: "hello".to_string() };
let len = my_s.len();        // Deref: MyString → str
my_s.make_ascii_uppercase(); // DerefMut: &mut MyString → &mut str
```

## Important Rules

1. **Only works with references** - No coercion for owned values
2. **Happens at call sites** - Function boundaries and method calls
3. **Transitive** - Can chain multiple deref steps
4. **One-way** - `&str` cannot coerce back to `&String`
5. **Zero-cost** - Resolved at compile time

## Common Examples

- `String` → `str`: Text processing
- `Vec<T>` → `[T]`: Array operations  
- `Box<T>` → `T`: Smart pointer transparency
- `Rc<T>` → `T`: Shared ownership
- `Arc<T>` → `T`: Thread-safe sharing

This all ends ups enabling writing functions that accept `&str` but work with `String`, `Box<str>`, `Rc<String>`, etc.