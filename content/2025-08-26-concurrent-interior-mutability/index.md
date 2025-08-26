+++
title = "Concurrent Interior Mutability in Rust"
date = 2025-08-26
+++

I wrote a bit about how Rust implements the **interior mutability** pattern, as a way to augment its compile-time enforcement of ownership and borrowing rules, specifically, the **one mutable reference or multiple immutable references at a time**, to enforce compile-time checking at the "container" type level, but allow safe mutable access to the inner data, which instead is checked for **one mutable reference or multiple immutable references at a time** violations at runtime.

## Using Interior Mutability for Thread-Safety

Some interior mutability types, like **RefCell**, govern interior mutable access, but are not sufficient for guaranteeing thread-safe data access. This is achieved by using the **Mutex** and **RwLock** interior mutability types, which both:

1. Enforce **one mutable reference or multiple immutable references at a time** at runtime.
2. Enforce thread-safe interior data access.

A simple example showing this access pattern is below, where we see a thread-safe integer mutation:

```rust

use std::sync::Mutex;

let x = Mutex::new(5);
*x.lock().unwrap() = 10;  // Lock, mutate, unlock
```

Compared to other interior mutability types, which directly borrow the inner data as mutable, these types additionally will block similarly to mutexes in other programming languages, and only on acquisition, will allow for interior mutability to take place. These two conditions together means that Rust can reliably enforce **one mutable reference or multiple immutable references at a time**, as well as enforcing thread-safe access to the inner data at the same time.

## **Rust vs. C++ Thread-Safety**

### **Type-Level Safety vs Runtime Safety**

```rust

// The Mutex is part of the type - impossible to forget to lock
Arc<Mutex<HashMap<String, String>>>
//     ↑
//     Built into the type system
```

```cpp
// Separate mutex - easy to forget
std::mutex mutex_;           // Separate variable
std::unordered_map<...> data_; // Separate variable

void set(...) {
    // Easy to forget this line:
    std::lock_guard<std::mutex> lock(mutex_);
    data_[key] = value;  // Race condition if lock is forgotten!
}
```

### **Compile-Time Guarantees**

```rust

// Compile-time enforcement of thread safety
let kvs = Arc::new(KeyValueStore::new());
let ref1 = &kvs;  // Immutable reference
let ref2 = &kvs;  // Another immutable reference - OK!
// let ref3 = &mut kvs;  // This would NOT compile
```

```cpp

// No compile-time guarantees
auto kvs = std::make_shared<KeyValueStore>();
// Can accidentally create multiple mutable references at runtime
```

### **Automatic Resource Management**

```rust

pub fn set(&self, key: String, value: String) {
    if let Ok(mut map) = self.data.lock() {
        map.insert(key, value);
        // Lock automatically released when map goes out of scope
        // Even if insert() panics, lock is still released
    }
}
```

```cpp

void set(const std::string& key, const std::string& value) {
    std::lock_guard<std::mutex> lock(mutex_);
    data_[key] = value;
    // Lock automatically released when lock_guard goes out of scope
    // But if an exception is thrown, behavior depends on exception safety
}
```

## **When to Use Each Interior Mutability Type**

1. **Mutex\<T>** -> Thread-safe, Simple, Automatic deadlock prevention.
2. **RwLock\<T>** -> Multiple concurrent readers, Single writer
3. **RefCell\<T>** -> Single-threaded interior mutability, Runtime borrow checking
4. **Cell\<T>** -> Single-threaded interior mutability, Copy types only, Zero runtime overhead