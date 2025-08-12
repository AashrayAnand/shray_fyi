+++
title = "A little bit of Rust: Interior Mutability"
date = 2025-XX-XX
+++

Interior mutability is a Rust pattern that allows you to mutate data even when you have an immutable reference to that data. This pattern is essential for safe concurrent programming and is one of Rust's most powerful features for achieving thread safety.

## The Core Problem

In Rust, the borrow checker enforces that:
- You can have multiple immutable references (`&T`) to the same data
- You can have exactly one mutable reference (`&mut T`) to the same data
- You cannot have both immutable and mutable references simultaneously

This creates a challenge: **How do we safely mutate data when multiple threads need to share it?**

## Rust's Solution: Interior Mutability

Rust provides several types that implement "interior mutability" - they allow mutation through immutable references in a controlled, safe way.

### Key Interior Mutability Types

1. **`Mutex<T>`** - Mutual exclusion for thread-safe mutation
2. **`RwLock<T>`** - Read-write lock for multiple readers, single writer
3. **`RefCell<T>`** - Single-threaded interior mutability
4. **`Cell<T>`** - Single-threaded interior mutability (copy types only)
5. **`Atomic<T>`** - Lock-free atomic operations

## Example: Thread-Safe Key-Value Store

Let's examine how Rust's interior mutability differs from C++ approaches.

### Rust Implementation

```cpp
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Clone)]
pub struct KeyValueStore {
    data: Arc<Mutex<HashMap<String, String>>>
}

impl KeyValueStore {
    pub fn new() -> Self {
        Self { 
            data: Arc::new(Mutex::new(HashMap::new())) 
        }
    }

    // Note: &self, not &mut self!
    pub fn set(&self, key: String, value: String) {
        if let Ok(mut map) = self.data.lock() {
            map.insert(key, value);  // Mutation through immutable reference
        }
    }

    pub fn get(&self, key: &str) -> Option<String> {
        if let Ok(map) = self.data.lock() {
            map.get(key).cloned()
        } else {
            None
        }
    }
}

// Usage
fn main() {
    let kvs = Arc::new(KeyValueStore::new());
    
    // Multiple threads can have immutable references
    let kvs1 = Arc::clone(&kvs);
    let kvs2 = Arc::clone(&kvs);
    
    // Both threads can mutate the data safely
    std::thread::spawn(move || {
        kvs1.set("key1".to_string(), "value1".to_string());
    });
    
    std::thread::spawn(move || {
        kvs2.set("key2".to_string(), "value2".to_string());
    });
}
```

### C++ Equivalent

```cpp
#include <unordered_map>
#include <mutex>
#include <memory>
#include <thread>

class KeyValueStore {
private:
    std::mutex mutex_;
    std::unordered_map<std::string, std::string> data_;
    
public:
    void set(const std::string& key, const std::string& value) {
        std::lock_guard<std::mutex> lock(mutex_);  // Manual locking
        data_[key] = value;
    }
    
    std::string get(const std::string& key) {
        std::lock_guard<std::mutex> lock(mutex_);  // Manual locking
        auto it = data_.find(key);
        return it != data_.end() ? it->second : "";
    }
};

// Usage
int main() {
    auto kvs = std::make_shared<KeyValueStore>();
    
    // Manual thread management
    std::thread t1([kvs]() {
        kvs->set("key1", "value1");
    });
    
    std::thread t2([kvs]() {
        kvs->set("key2", "value2");
    });
    
    t1.join();
    t2.join();
}
```

## Key Differences

### 1. **Type-Level Safety vs Runtime Safety**

**Rust:**
```cpp
// The Mutex is part of the type - impossible to forget to lock
Arc<Mutex<HashMap<String, String>>>
//     ↑
//     Built into the type system
```

**C++:**
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

### 2. **Compile-Time Guarantees**

**Rust:**
```cpp
// Compile-time enforcement of thread safety
let kvs = Arc::new(KeyValueStore::new());
let ref1 = &kvs;  // Immutable reference
let ref2 = &kvs;  // Another immutable reference - OK!
// let ref3 = &mut kvs;  // This would NOT compile
```

**C++:**
```cpp
// No compile-time guarantees
auto kvs = std::make_shared<KeyValueStore>();
// Can accidentally create multiple mutable references at runtime
```

### 3. **Automatic Resource Management**

**Rust:**
```cpp
pub fn set(&self, key: String, value: String) {
    if let Ok(mut map) = self.data.lock() {
        map.insert(key, value);
        // Lock automatically released when map goes out of scope
        // Even if insert() panics, lock is still released
    }
}
```

**C++:**
```cpp
void set(const std::string& key, const std::string& value) {
    std::lock_guard<std::mutex> lock(mutex_);
    data_[key] = value;
    // Lock automatically released when lock_guard goes out of scope
    // But if an exception is thrown, behavior depends on exception safety
}
```

## How Interior Mutability Works

### The Magic of `DerefMut`

```cpp
pub fn set(&self, key: String, value: String) {
    if let Ok(mut map) = self.data.lock() {  // ← Returns MutexGuard<HashMap>
        map.insert(key, value);               // ← Automatically dereferences to &mut HashMap
    }
}
```

**What happens:**
1. `self.data.lock()` returns `MutexGuard<HashMap<String, String>>`
2. `MutexGuard` implements `DerefMut`
3. When you use `map` in a context expecting `&mut HashMap`, it automatically dereferences
4. The lock is automatically released when `MutexGuard` is dropped

### Why This is Safe

```cpp
// Multiple threads can have immutable references
let kvs = Arc::new(KeyValueStore::new());
let kvs1 = Arc::clone(&kvs);  // &KeyValueStore
let kvs2 = Arc::clone(&kvs);  // &KeyValueStore

// But only one can access the HashMap at a time
kvs1.set("key1".to_string(), "value1".to_string());  // Locks, mutates, unlocks
kvs2.set("key2".to_string(), "value2".to_string());  // Waits, then locks, mutates, unlocks
```

## Other Interior Mutability Patterns

### `RefCell<T>` - Single-Threaded Interior Mutability

```cpp
use std::cell::RefCell;

struct Counter {
    count: RefCell<i32>
}

impl Counter {
    fn new() -> Self {
        Self { count: RefCell::new(0) }
    }
    
    fn increment(&self) {  // &self, not &mut self!
        let mut count = self.count.borrow_mut();
        *count += 1;
    }
    
    fn get(&self) -> i32 {
        *self.count.borrow()
    }
}
```

### `RwLock<T>` - Multiple Readers, Single Writer

```cpp
use std::sync::RwLock;

struct SharedData {
    data: Arc<RwLock<HashMap<String, String>>>
}

impl SharedData {
    fn read(&self, key: &str) -> Option<String> {
        if let Ok(map) = self.data.read() {  // Multiple readers can access simultaneously
            map.get(key).cloned()
        } else {
            None
        }
    }
    
    fn write(&self, key: String, value: String) {
        if let Ok(mut map) = self.data.write() {  // Exclusive access for writing
            map.insert(key, value);
        }
    }
}
```

## Benefits of Rust's Approach

### 1. **Zero-Cost Abstractions**
- The mutex is part of the type, enabling better compiler optimizations
- No runtime overhead for checking if synchronization is needed

### 2. **Compile-Time Safety**
- Impossible to forget to acquire locks
- Type system prevents data races at compile time

### 3. **Automatic Resource Management**
- Locks are automatically released when guards go out of scope
- Exception-safe (panic-safe in Rust)

### 4. **Clear Ownership Semantics**
- The type system makes ownership and borrowing explicit
- No hidden aliasing or undefined behavior

## When to Use Each Pattern

### `Mutex<T>` - When you need:
- Thread-safe mutation
- Simple exclusive access
- Automatic deadlock prevention (single mutex)

### `RwLock<T>` - When you need:
- Multiple concurrent readers
- Single writer
- Better performance for read-heavy workloads

### `RefCell<T>` - When you need:
- Single-threaded interior mutability
- Runtime borrowing checks
- Breaking the borrowing rules safely

### `Cell<T>` - When you need:
- Single-threaded interior mutability
- Copy types only
- Zero runtime overhead

## Conclusion

Rust's interior mutability pattern provides a powerful way to safely mutate data through immutable references. This is essential for:

1. **Thread-safe programming** - Multiple threads can share immutable references while safely mutating data
2. **API design** - Methods can take `&self` instead of `&mut self` while still allowing mutation
3. **Memory safety** - The type system ensures safe concurrent access

The key insight is that interior mutability types act as "gates" that provide controlled access to mutable data, while the Rust type system ensures that this access is always safe and properly synchronized.

This pattern is one of Rust's most powerful features for building safe, concurrent systems. 