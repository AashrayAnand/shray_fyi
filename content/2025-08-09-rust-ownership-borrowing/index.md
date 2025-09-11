+++
title = "Ownership and Borrowing in Rust"
date = 2025-08-09
+++

Rust's type system provides two complementary safety mechanisms, **Ownership**, and **Borrowing**, which together provide memory and logical safety guarantees that are impossible to achieve at compile time in other languages.

### Ownership

- Prevents double-free, use-after-free, memory leaks
- Ensures each piece of memory has exactly one owner
- Guarantees proper cleanup when values go out of scope

### Borrowing

- Prevents data races, undefined behavior, non-deterministic access
- Ensures controlled access to shared data
- Prevents iterator invalidation and other logical bugs

### Rust's Ownership Rules

```rust

// Each value has exactly one owner
let s = String::from("hello");  // s owns the string
let t = s;                      // ownership moves to t
// println!("{}", s);           // Error: s no longer owns the string
```

### C++ Equivalent (Manual Memory Management)

```cpp
#include <string>
#include <memory>

int main() {
    std::string* s = new std::string("hello");  // Manual allocation
    std::string* t = s;                         // Both pointers point to same memory
    delete s;                                   // Memory freed
    delete t;                                   // Double-free! Undefined behavior
    
    // Or with smart pointers (better, but still possible to misuse)
    auto s = std::make_unique<std::string>("hello");
    auto t = std::move(s);  // s is now nullptr
    // std::cout << *s;     // Runtime error: null pointer dereference
}
```

### Benefits of Rust's Ownership

#### 1. **No Double-Free**

```rust
// Rust: Impossible to double-free
let s = String::from("hello");
// Only one owner can drop the string
// Compiler prevents multiple owners
```

```cpp
// C++: Easy to double-free
std::string* s = new std::string("hello");
std::string* t = s;  // Two pointers to same memory
delete s;            // Memory freed
delete t;            // Double-free! Undefined behavior
```

#### 2. **No Use-After-Free**

```rust

// Rust: Impossible to use after free
fn create_string() -> String {
    String::from("hello")
}

let s = create_string();  // s owns the string
// String is automatically dropped when s goes out of scope
// No possibility of use-after-free
```

```cpp
// C++: Easy to use after free
std::string* create_string() {
    return new std::string("hello");
}

std::string* s = create_string();
delete s;           // Memory freed
std::cout << *s;    // Use-after-free! Undefined behavior
```

#### 3. **Automatic Memory Management**

Although Rust is not a garbage-collected language like Java, it is able to achieve automatic memory management without additional overhead, by relying on the lifetime of owned data to dictate when to release this data.

```rust

// Rust: Automatic cleanup
fn process_data() {
    let data = vec![1, 2, 3, 4, 5];  // Allocated on stack
    // Process data...
    // Automatically dropped when function ends and data is out of scope.
}
```

```cpp
// C++: Manual cleanup required
void process_data() {
    std::vector<int>* data = new std::vector<int>{1, 2, 3, 4, 5};
    // Process data...
    delete data;  // Must remember to free
    // If we forget, memory leak!
}
```

### Rust's Borrowing Rules

```rust

// You can have either:
// - One mutable reference (&mut T)
// - Any number of immutable references (&T)
// - But never both simultaneously

let mut data = vec![1, 2, 3];
let ref1 = &mut data;  // Mutable reference
// let ref2 = &mut data;  // Error: cannot borrow as mutable more than once
```

### C++ Equivalent (No Borrow Checker)

```cpp
// C++: No compile-time borrowing rules
std::vector<int> data = {1, 2, 3};
int* ref1 = &data[0];
int* ref2 = &data[0];  // Multiple mutable references - allowed but dangerous
*ref1 = 10;
*ref2 = 20;  // Data race! Undefined behavior
```

### Benefits of Rust's Borrowing

#### 1. **No Data Races**

```rust

// Rust: Impossible to have data races in single thread
let mut counter = 0;
let ref1 = &mut counter;
// let ref2 = &mut counter;  // Compile error
*ref1 += 1;  // Only one way to modify counter
```

```cpp
// C++: Easy to create data races
int counter = 0;
int* ref1 = &counter;
int* ref2 = &counter;  // Multiple mutable references
*ref1 += 1;  // counter = 1
*ref2 += 1;  // counter = 2
// But what if order changes? Undefined behavior!
```

#### 2. **Deterministic Behavior**

```rust

// Rust: Predictable results
let mut data = vec![1, 2, 3];
let ref_data = &mut data;
ref_data.push(4);  // data = [1, 2, 3, 4]
ref_data.push(5);  // data = [1, 2, 3, 4, 5]
// Result is always deterministic
```

```cpp
// C++: Unpredictable results possible
std::vector<int> data = {1, 2, 3};
int* ref1 = &data[0];
int* ref2 = &data[0];
// Multiple ways to modify same data
// Result depends on order of operations
```

#### 3. **Iterator Safety**

```rust

// Rust: Iterator invalidation prevented at compile time
let mut vec = vec![1, 2, 3];
let iter = vec.iter();  // Immutable borrow
// vec.push(4);         // Error: cannot borrow as mutable while borrowed as immutable
```

```cpp
// C++: Iterator invalidation possible
std::vector<int> vec = {1, 2, 3};
auto iter = vec.begin();
vec.push_back(4);  // Iterator invalidated!
// *iter;           // Undefined behavior
```

## Concurrency Safety

Rust handles concurrency gracefully, with a pattern called **interior mutability**. I will dedicate a full post to how this is achieved in the future.
