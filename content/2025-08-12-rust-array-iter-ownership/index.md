+++
title = "A little bit of Rust: Comparing Array Iteration vs. C++ and Borrow Checker Safety"
date = 2025-08-12
+++

The way that even simple algorithms are implemented in Rust vs. C++ or other languages is often a big departure, both in syntax and in what safe guards are enforced for getting it done.

I wanted to use a small but interesting problem to show how Rust, at compile time, provides more memory safety and avoids certain classes of issues that may be seen otherwise.

On one hand, the safe guards enforced by the borrow checker mean that compiled Rust code can make greater guarantees than other code, but especially for people using it for the first time, a lot of code that may seem totally innocous, or compile-able in another language, becomes a stream of compiler errors and headache with Rust. I definitely spent a lot of time initially just monkey-debugging around this, but over time have realized that the errors are actually almost always insightful and helpful when I actually take 30 seconds to read them.

## The Problem: First Missing Positive

Find the smallest positive integer that is not present in the array.

**Example:**
- Input: `[1, 2, 0]` → Output: `3`
- Input: `[3, 4, -1, 1]` → Output: `2`
- Input: `[7, 8, 9, 11, 12]` → Output: `1`

## C++ Solution: Unrestricted Access

```cpp
class Solution {
public:
    int firstMissingPositive(vector<int>& nums) {
        int n = nums.size();
        bool contains1 = false;

        // Replace negative numbers, zeros,
        // and numbers larger than n with 1s.
        // After this nums contains only positive numbers.
        for (int i = 0; i < n; i++) {
            // Check whether 1 is in the original array
            if (nums[i] == 1) {
                contains1 = true;
            }
            if (nums[i] <= 0 || nums[i] > n) {
                nums[i] = 1;
            }
        }

        if (!contains1) return 1;

        // Mark whether integers 1 to n are in nums
        // Use index as a hash key and negative sign as a presence detector.
        for (int i = 0; i < n; i++) {
            int value = abs(nums[i]);
            if (value == n) {
                nums[0] = -abs(nums[0]);
            } else {
                nums[value] = -abs(nums[value]);
            }
        }

        // First positive in nums is smallest missing positive integer
        for (int i = 1; i < n; i++) {
            if (nums[i] > 0) return i;
        }

        // nums[0] stores whether n is in nums
        if (nums[0] > 0) {
            return n;
        }

        // If nums contained all elements 1 to n
        // the smallest missing positive number is n + 1
        return n + 1;
    }
};
```

## Rust Solution: Borrow Checker Violations (Original)

```cpp
impl Solution {
    pub fn first_missing_positive(mut nums: Vec<i32>) -> i32 {
        // get rid of numbers we don't care for.
        let mut has_one = false;
        for num in &mut nums {
            if *num == 1 {
                has_one = true;
            }
            if *num <= 0 || *num > nums.len() as i32 {  // ❌ BORROW CHECKER ERROR
                *num = 1;
            }
        }

        if !has_one { return 1; }
        for num in &nums {
            let abs_num = num.abs();
            if let Some(num_entry) = nums.get_mut((abs_num - 1) as usize) {  // ❌ BORROW CHECKER ERROR
                *num_entry = -1 * num_entry.abs();
            }
        }
        for i in 0..nums.len() {
            if nums[i] > 0 {
                return (i + 1) as i32;
            }
        }
        return (nums.len() + 1) as i32;
    }
}
```

## Borrow Checker Violations in Rust

### **Violation 1: Mutable and Immutable Borrow Simultaneously**

```cpp
for num in &mut nums {  // Mutable borrow of nums
    if *num <= 0 || *num > nums.len() as i32 {  // Immutable borrow of nums
        *num = 1;
    }
}
```

**Error:**

```
error: cannot borrow `nums` as immutable because it is also borrowed as mutable
```

**Why This Happens:**
- `&mut nums` creates a mutable borrow for the entire loop
- `nums.len()` tries to create an immutable borrow while mutable borrow is active

### **Violation 2: Immutable and Mutable Borrow Simultaneously**

```cpp
for num in &nums {  // Immutable borrow of nums
    let abs_num = num.abs();
    if let Some(num_entry) = nums.get_mut((abs_num - 1) as usize) {  // Mutable borrow
        *num_entry = -1 * num_entry.abs();
    }
}
```

**Error:**
```
error: cannot borrow `nums` as mutable because it is also borrowed as immutable
```

**Why This Happens:**
- `&nums` creates an immutable borrow for the entire loop
- `nums.get_mut()` tries to create a mutable borrow while immutable borrow is active

## C++ vs Rust: What C++ Allows vs What Rust Prevents

### **1. Simultaneous Access Patterns**

#### **C++: Unrestricted Access**

```cpp
for (int i = 0; i < n; i++) {
    if (nums[i] <= 0 || nums[i] > n) {  // Read access
        nums[i] = 1;                      // Write access
    }
}
```

**C++ allows this because:**
- No compile-time safety checks
- Manual memory management
- Runtime discipline required
- Potential for undefined behavior

#### **Rust: Controlled Access**

```cpp
// This would be unsafe in Rust:
for num in &mut nums {
    if *num <= 0 || *num > nums.len() as i32 {  // Compile error
        *num = 1;
    }
}
```

**Rust prevents this because:**
- Compile-time borrow checking
- Prevents data races
- Ensures memory safety
- Prevents undefined behavior

### **2. Iterator Invalidation**

#### **C++: Runtime Errors Possible**

```cpp
vector<int> nums = {1, 2, 3, 4, 5};
for (int i = 0; i < nums.size(); i++) {
    if (nums[i] == 3) {
        nums.push_back(6);  // Modifies vector while iterating
        // Potential undefined behavior
    }
}
```

#### **Rust: Compile-Time Prevention**

```cpp
let mut nums = vec![1, 2, 3, 4, 5];
for num in &mut nums {
    if *num == 3 {
        // nums.push(6);  // Compile error: cannot borrow as mutable
    }
}
```

### **3. Memory Safety**

#### **C++: Manual Management**

```cpp
int* arr = new int[10];
// ... use array ...
delete[] arr;  // Must remember to free
// arr[0] = 5;  // Use after free - undefined behavior
```

#### **Rust: Automatic Management**

```cpp
let nums = vec![1, 2, 3, 4, 5];
// nums is automatically dropped when it goes out of scope
// No possibility of use-after-free
```

## Fixed Rust Solution

```cpp
impl Solution {
    pub fn first_missing_positive(mut nums: Vec<i32>) -> i32 {
        // Get length before any borrows to avoid conflicts
        let len = nums.len() as i32;
        
        // First pass: normalize numbers
        let mut has_one = false;
        for num in &mut nums {
            if *num == 1 {
                has_one = true;
            }
            if *num <= 0 || *num > len {  // Use stored length
                *num = 1;
            }
        }

        if !has_one { return 1; }
        
        // Second pass: mark numbers as negative using index-based iteration
        for i in 0..nums.len() {
            let abs_num = nums[i].abs();
            if let Some(num_entry) = nums.get_mut((abs_num - 1) as usize) {
                *num_entry = -1 * num_entry.abs();
            }
        }
        
        // Third pass: find first positive
        for i in 0..nums.len() {
            if nums[i] > 0 {
                return (i + 1) as i32;
            }
        }
        return (nums.len() + 1) as i32;
    }
}
```

## Key Differences in the Fixed Solution

### **1. Pre-compute Values**

```cpp
// Before: nums.len() called during mutable borrow
for num in &mut nums {
    if *num > nums.len() as i32 {  // Error
    }
}

// After: Pre-compute length
let len = nums.len() as i32;
for num in &mut nums {
    if *num > len {  // Safe
    }
}
```

### **2. Index-Based Iteration**

```cpp
// Before: Immutable borrow conflicts with mutable access
for num in &nums {
    nums.get_mut(...);  // Error
}

// After: Index-based iteration, avoid an immutable borrow
// in a scope where we need mutable access.
for i in 0..nums.len() {
    let abs_num = nums[i].abs();
    nums.get_mut(...);  // Safe
}
```

## Performance Comparison

**Rust provides the same performance as C++ with additional safety guarantees.**

### **C++ Performance**

```c
// Direct array access - fastest
for (int i = 0; i < n; i++) {
    nums[i] = 1;  // Direct memory access
}
```

### **Rust Performance**

```rust

// Zero-cost abstractions - same performance
for num in &mut nums {
    *num = 1;  // Same assembly as C++
}
```

## Common Rust Patterns for This Problem

### **1. Pre-compute Values**

```rust

let len = nums.len() as i32;  // Avoid borrow conflicts
```

### **2. Index-Based Iteration**
```rust

for i in 0..nums.len() {  // When you need to modify while iterating
    // Safe to modify nums[i]
}
```

### **3. Use `iter_mut()` for Simple Cases**

```rust

for num in &mut nums {  // When you only need to modify elements
    *num = 1;
}
```

### **4. Use `enumerate()` for Index + Value**

```rust

for (i, num) in nums.iter_mut().enumerate() {
    if *num == 1 {
        // i is the index, num is the value
    }
}
```