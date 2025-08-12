+++
title = "A little bit of Rust: Implementing a Registry-Based Singleton"
date = 2025-XX-XX
+++

The **Singleton Pattern** is a creational pattern that enforces a single instance of a class with global access. This can be taken a bit further to implement the **Registry-Based Singleton**, where a
global cache controls access to a set of singletons.

This is a pretty helpful pattern for any data that would be accessed at a process-global level, and would require things such as:

1. Lazy initialization.
2. Only-once singleton construction.
3. Thread-safe access.
4. Named access to specific global state.
5. Fixed-size global state at compile time.

In a key-value store that I am working on, I use this pattern to maintain a registry of the different namespaces accessible by clients, ensuring each namespace is only-once initialized,
and enforcing thread-safe namespace access across connections.

# Simple Singleton (Single Instance)

A singleton would look like the following to start: 

```rust

static INSTANCE: LazyLock<Arc<RwLock<KeyValueStore>>> = 
    LazyLock::new(|| Arc::new(RwLock::new(KeyValueStore::new())));

pub fn instance() -> Arc<RwLock<KeyValueStore>> {
    INSTANCE.clone()
}
```

# Once-Based Initialization

To enforce only-once initialization of a singleton, the ```Once``` primitive can be used.

```rust

use std::sync::Once;

static INIT: Once = Once::new();
static mut INSTANCE: Option<Arc<KeyValueStore>> = None;

pub fn get_instance() -> Arc<KeyValueStore> {
    INIT.call_once(|| {
        unsafe {
            INSTANCE = Some(Arc::new(KeyValueStore::new()));
        }
    });
    unsafe { INSTANCE.as_ref().unwrap().clone() }
}
```

## Creating a Singleton Registry

A combination of the above 2 steps can be implemented using a `LazyLock` with synchronized access.

### 1. LazyLock - Thread-Safe Lazy Initialization

```rust

static GLOBAL_STORES: LazyLock<T> = LazyLock::new(|| expensive_initialization());
```

In the case of ```GLOBAL_STORES```, this will delay the initialization of the registry hash to the first access, make initialization thread-safe, and results in a fixed-size compile-time global state of a ```LazyLock```.

```rust

use std::sync::{LazyLock, RwLock};
use std::collections::HashMap;

static GLOBAL_STORES: LazyLock<RwLock<HashMap<String, Arc<KeyValueStore>>>> = 
    LazyLock::new(|| RwLock::new(HashMap::new()));
```

### 2. RwLock - Reader-Writer Synchronization

```rust

let stores = GLOBAL_STORES.write().unwrap();  // Exclusive access for modifications
let stores = GLOBAL_STORES.read().unwrap();   // Shared access for reads
```

After the registry is initialized, we can enforce thread-safe registry access.

### 3. HashMap with or_insert_with - Conditional Initialization

```rust

stores.entry(name.clone())
    .or_insert_with(|| Arc::new(KeyValueStore::new(name)))
    .clone()
```

Using a synchronized ```or_insert_with``` operation on the registry, we can access an existing entry in the registry, and construct it if it doesn't already exist.

## The Complete Pattern

```rust

pub fn get_or_create(name: String) -> Arc<KeyValueStore> {
    let mut stores = GLOBAL_STORES.write().unwrap();
    stores.entry(name.clone())
        .or_insert_with(|| Arc::new(KeyValueStore::new(name)))
        .clone()
}
```

We end up being able to get the following behavior:

1. **LazyLock initialization**: The HashMap itself is created exactly once
2. **Write lock**: Only one thread can modify the HashMap at a time  
3. **or_insert_with**: Only creates new instance if key doesn't exist
4. **Arc sharing**: Multiple references to the same instance

### Execution Flow

```rust

// Thread 1: get_or_create("db1")
GLOBAL_STORES.write() 🔒
├── HashMap is empty
├── or_insert_with() creates KeyValueStore::new("db1") 
├── Stores Arc<KeyValueStore> in HashMap["db1"]
└── Returns Arc<KeyValueStore> 🔓

// Thread 2: get_or_create("db1") (concurrent)  
GLOBAL_STORES.write() 🔒 (waits for Thread 1)
├── HashMap contains "db1" 
├── or_insert_with() skips creation
└── Returns existing Arc<KeyValueStore> 🔓

// Thread 3: get_or_create("db2")
GLOBAL_STORES.write() 🔒
├── HashMap contains "db1" but not "db2"
├── or_insert_with() creates KeyValueStore::new("db2")
├── Stores Arc<KeyValueStore> in HashMap["db2"] 
└── Returns Arc<KeyValueStore> 🔓
```

### 1. **Construction Serialization**
Even if `KeyValueStore::new()` is not thread-safe, the global lock ensures only one thread can execute it at a time.

### 2. **Resource Efficiency** 
```rust
let store1 = KeyValueStore::get_or_create("db1".to_string());
let store2 = KeyValueStore::get_or_create("db1".to_string());
// store1 and store2 point to the SAME instance
```

### 3. **Named Instance Management**
Multiple named singletons can coexist:

```rust

let user_db = KeyValueStore::get_or_create("users".to_string());
let session_db = KeyValueStore::get_or_create("sessions".to_string());
// Different instances for different purposes
```