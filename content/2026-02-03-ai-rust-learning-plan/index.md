+++
title = "An AI-generated Rust Learning Plan"
date = 2026-02-03
+++

This type of plan is one of my most profilic uses of AI in both work and personally. Nobody got time in 2026 to be manually compiling resources and planning out things I want to learn about or do, I'd rather spend my time actually growing my knowledge from what it gives me.

This plan is the result of the following prompt:

```
create a markdown file in my home directory called "100 days of learning something about rust for 30-45 minutes". It should be something on a daily basis to learn about Rust, for someone who has already read "the rust programming language" and uses Rust at work on a fairly regular basis such that I have fairly decent knowledge, but not an absolute expert on:

1. rust frameworks e.g. tonic
2. rust async/tokio
3. rust interior mutability and similar concepts
4. unafe
5. zero cost abstractions e.g. iterators
6. common trait impls like Iterator/Sync/Send

I'm trying to beocme a deep domain expert and am able and willing to read from resources like "rust for rustaceans", the rustonomicon, and more that you suggest.
```

# Asking Copilot for 100 days of Rust learning

A structured deep-dive into Rust for intermediate practitioners seeking expert-level mastery.

**Prerequisites:** You've read "The Rust Programming Language," use Rust regularly at work, and are comfortable with common patterns and traits.

**Primary Resources:**
- 📘 *Rust for Rustaceans* by Jon Gjengset (RfR)
- 📖 *The Rustonomicon* (free online: https://doc.rust-lang.org/nomicon/)
- 📚 Rust Reference (https://doc.rust-lang.org/reference/)
- 🎥 Jon Gjengset's YouTube channel
- 📝 Various blog posts and RFCs

---

## Phase 1: Foundations of Mastery (Days 1-15)

### Week 1: Memory Layout & Representations

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 1 | Rust memory layout fundamentals | Read about `repr(Rust)`, `repr(C)`, `repr(transparent)` | RfR Ch. 1, Rust Reference |
| 2 | Size, alignment, and padding | Experiment with `std::mem::size_of`, `align_of`, create structs with different field orderings | Rustonomicon "Data Layout" |
| 3 | Zero-sized types (ZSTs) | Understand `PhantomData`, marker types, and why ZSTs matter | RfR Ch. 1, write examples |
| 4 | Dynamically sized types (DSTs) | Deep dive into `str`, `[T]`, trait objects, and `Sized` bound | RfR Ch. 1 |
| 5 | Fat pointers | Examine `&dyn Trait`, `&[T]` internals, vtables | Rustonomicon, use `std::mem::transmute` to inspect |
| 6 | `Option<&T>` niche optimization | Understand null pointer optimization and other niche filling | Rust Reference, compiler explorer |
| 7 | **Practice:** Build a memory inspector | Create utility functions to print layout info for any type | Hands-on coding |

### Week 2: Ownership Deep Dive

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 8 | Drop order and drop guarantees | Understand struct field drop order, `ManuallyDrop`, `mem::forget` | Rustonomicon "Drop" |
| 9 | `Pin` fundamentals | Why `Pin` exists, self-referential structs problem | RfR Ch. 8, Rustonomicon |
| 10 | `Pin` in practice | Implement a simple self-referential struct, understand `Unpin` | Write code, async book |
| 11 | Interior mutability patterns | `Cell`, `RefCell`, `UnsafeCell` internals | RfR Ch. 1, std source |
| 12 | Atomic types and memory ordering | `Ordering::SeqCst` vs `Relaxed` vs `Acquire/Release` | Rustonomicon "Atomics" |
| 13 | `Cow` and copy-on-write patterns | When to use `Cow`, implementing efficient APIs | RfR, std docs |
| 14 | **Practice:** Implement a simple `RefCell` | Build your own interior mutability primitive | Hands-on coding |

---

## Phase 2: Traits & Type System Mastery (Days 16-35)

### Week 3: Advanced Trait Patterns

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 15 | Trait objects vs generics | Monomorphization cost, binary size, vtable dispatch | RfR Ch. 2 |
| 16 | Orphan rules and coherence | Why orphan rules exist, the `#[fundamental]` attribute | Rust Reference, RFCs |
| 17 | Blanket implementations | Study `impl<T> Trait for T where T: OtherTrait` patterns | std library source |
| 18 | Associated types vs generic parameters | When to use which, `GATs` introduction | RfR Ch. 2 |
| 19 | Generic Associated Types (GATs) | Lending iterators, async traits | Rust blog, GAT RFC |
| 20 | Marker traits | `Send`, `Sync`, `Unpin`, `Sized`, and custom markers | RfR Ch. 2 |
| 21 | **Practice:** Design a trait hierarchy | Create a mini framework with well-designed traits | Hands-on coding |

### Week 4: The Borrow Checker's Mind

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 22 | Non-lexical lifetimes (NLL) | How NLL works, edge cases | Rust blog posts on NLL |
| 23 | Lifetime variance | Covariance, contravariance, invariance in Rust | Rustonomicon "Subtyping" |
| 24 | Higher-ranked trait bounds (HRTBs) | `for<'a>` syntax, when it's needed | RfR Ch. 2, Rustonomicon |
| 25 | Lifetime elision rules | All three rules, `'_` placeholder | Rust Reference |
| 26 | `'static` misconceptions | `T: 'static` doesn't mean "lives forever" | Common Rust Lifetime Misconceptions blog |
| 27 | Reborrowing | How `&mut` can be temporarily borrowed as `&` | Write examples, RfR |
| 28 | **Practice:** Fix lifetime puzzles | Solve 5-10 complex lifetime scenarios | Rust quiz, Stack Overflow |

### Week 5: Type System Advanced Features

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 29 | Type inference deep dive | How Hindley-Milner works in Rust | Academic papers, blog posts |
| 30 | Deref coercion chains | `String` → `str`, `Vec<T>` → `[T]`, custom `Deref` | RfR, std docs |
| 31 | `AsRef`, `Borrow`, `ToOwned` | When to use which, design patterns | RfR Ch. 2 |
| 32 | `From`/`Into` and `TryFrom`/`TryInto` | Error handling patterns, conversion hierarchies | std docs, API guidelines |
| 33 | Newtype pattern | Type safety without runtime cost, `Deref` considerations | RfR, Rust patterns book |
| 34 | Type-state pattern | Compile-time state machines | Blog posts, design patterns |
| 35 | **Practice:** Refactor code using type-state | Convert runtime checks to compile-time | Hands-on coding |

---

## Phase 3: Async Rust Mastery (Days 36-55)

### Week 6: Async Fundamentals

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 36 | Future trait internals | `poll()`, `Context`, `Waker` | Async book, RfR Ch. 8 |
| 37 | State machines from async/await | What the compiler generates | Compiler explorer, async book |
| 38 | Waker mechanics | How wakers work, `RawWaker`, `RawWakerVTable` | tokio source, async book |
| 39 | Executor basics | Build a minimal single-threaded executor | Async book chapter, hands-on |
| 40 | `Pin` in async contexts | Why futures must be pinned | RfR Ch. 8 |
| 41 | Cancellation safety | What happens when futures are dropped | Tokio docs, RfR |
| 42 | **Practice:** Build a simple executor | Implement wake, poll loop | Hands-on coding |

### Week 7: Tokio Deep Dive

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 43 | Tokio runtime architecture | Multi-threaded vs current-thread | Tokio docs, source code |
| 44 | Tokio task spawning | `spawn`, `spawn_blocking`, `spawn_local` | Tokio tutorial |
| 45 | Tokio channels | `mpsc`, `broadcast`, `watch`, `oneshot` | Tokio docs, compare to std |
| 46 | Tokio synchronization | `Mutex`, `RwLock`, `Semaphore`, `Notify` | Tokio docs |
| 47 | `select!` macro deep dive | Fairness, cancellation, patterns | Tokio tutorial |
| 48 | Tokio tracing integration | `tracing`, `tracing-subscriber`, spans | Tracing crate docs |
| 49 | **Practice:** Build a chat server | Use channels, tasks, graceful shutdown | Tokio tutorial project |

### Week 8: Advanced Async Patterns

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 50 | Async trait methods | `async-trait` crate, native async traits | Blog posts, crate docs |
| 51 | Streams and `AsyncIterator` | `futures::Stream`, `tokio-stream` | futures-rs docs |
| 52 | Backpressure patterns | Bounded channels, rate limiting | System design, tokio patterns |
| 53 | Structured concurrency | `JoinSet`, `TaskTracker`, nurseries concept | Tokio docs, blog posts |
| 54 | Async drop and cleanup | Patterns for cleanup in async contexts | Blog posts, discussions |
| 55 | **Practice:** Implement graceful shutdown | Handle Ctrl-C, drain connections | Tokio examples |

---

## Phase 4: Unsafe Rust & FFI (Days 56-70)

### Week 9: Unsafe Foundations

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 56 | What unsafe unlocks | The 5 unsafe superpowers | Rustonomicon "Meet Safe and Unsafe" |
| 57 | Undefined behavior in Rust | What UB exists, why it matters | Rustonomicon "What Unsafe Can Do" |
| 58 | Validity and safety invariants | Difference between validity and safety | RfR Ch. 9 |
| 59 | Raw pointers | `*const T`, `*mut T`, pointer arithmetic | Rustonomicon |
| 60 | `unsafe` traits and impls | `Send`, `Sync` manual impl, `GlobalAlloc` | Rustonomicon |
| 61 | Miri for UB detection | Run tests under miri, interpret results | Miri docs, hands-on |
| 62 | **Practice:** Find UB in examples | Debug intentionally broken unsafe code | Exercises |

### Week 10: Advanced Unsafe

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 63 | `MaybeUninit` | Safely working with uninitialized memory | std docs, Rustonomicon |
| 64 | Stacked Borrows model | Understand the aliasing model | Ralf Jung's blog posts |
| 65 | Tree Borrows (experimental) | Alternative aliasing model | Research papers, blog |
| 66 | Union types | When and why to use unions | Rust Reference |
| 67 | Inline assembly | `asm!` macro basics | Rust Reference |
| 68 | FFI basics | `extern "C"`, `#[no_mangle]`, bindgen | Rustonomicon "FFI" |
| 69 | Calling C from Rust | Linking, safety wrappers | bindgen tutorial |
| 70 | **Practice:** Wrap a C library | Create safe Rust bindings | Hands-on with bindgen |

---

## Phase 5: Frameworks & Ecosystem (Days 71-85)

### Week 11: Tonic & gRPC

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 71 | Protobuf basics | Message definitions, code generation | Protobuf docs |
| 72 | Tonic server setup | Define service, implement handlers | Tonic examples |
| 73 | Tonic client patterns | Connection pooling, retries | Tonic docs |
| 74 | Streaming with Tonic | Server, client, bidirectional streaming | Tonic examples |
| 75 | Interceptors and middleware | Authentication, logging, tracing | Tonic docs |
| 76 | Error handling in Tonic | Status codes, error details | gRPC patterns |
| 77 | **Practice:** Build a gRPC service | Complete client-server application | Hands-on project |

### Week 12: Web Frameworks & Serde

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 78 | Tower service pattern | `Service` trait, layers, middleware | Tower docs |
| 79 | Axum fundamentals | Routing, extractors, state | Axum docs |
| 80 | Serde deep dive | How derive macros work, data model | Serde docs |
| 81 | Custom Serde implementations | `Serialize`/`Deserialize` by hand | Serde docs |
| 82 | Serde attributes | `#[serde(rename)]`, `flatten`, etc. | Serde attributes reference |
| 83 | Zero-copy deserialization | `Cow`, `&'de str`, borrowing | Serde docs |
| 84 | Error handling patterns | `anyhow`, `thiserror`, when to use which | Crate docs |
| 85 | **Practice:** Build a REST API | Axum + Serde + proper error handling | Hands-on project |

---

## Phase 6: Performance & Zero-Cost Abstractions (Days 86-100)

### Week 13: Iterator Mastery

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 86 | Iterator adapter internals | How `map`, `filter`, `flat_map` work | std source code |
| 87 | Iterator fusion | How chains compile to single loops | Compiler explorer |
| 88 | Custom iterator implementations | Implement `Iterator`, `IntoIterator` | Practice |
| 89 | `ExactSizeIterator`, `DoubleEndedIterator` | When to implement, optimizations enabled | std docs |
| 90 | `FromIterator` and `Extend` | Collection building patterns | std docs |
| 91 | `collect()` type inference | Turbofish, `FromIterator` dispatch | Write examples |
| 92 | **Practice:** Implement lazy iterator chain | Custom iterator with multiple adapters | Hands-on |

### Week 14: Performance Patterns

| Day | Topic | Activity | Resource |
|-----|-------|----------|----------|
| 93 | Benchmarking with Criterion | Statistical benchmarking, profiles | Criterion docs |
| 94 | Profile-guided optimization | PGO basics, when it helps | Rust docs |
| 95 | SIMD with `std::simd` | Portable SIMD, autovectorization | Rust docs, compiler explorer |
| 96 | Cache-friendly data structures | SoA vs AoS, data-oriented design | Game programming patterns |
| 97 | String performance | `String` vs `&str` vs `Cow<str>` vs small strings | Benchmarks |
| 98 | Allocation patterns | Arena allocators, `bumpalo`, reducing allocs | Crate docs |
| 99 | `#[inline]` and LTO | When to use, link-time optimization | Rust Performance Book |
| 100 | **Capstone:** Profile and optimize | Take a real codebase, find and fix bottlenecks | Your own code |

---

## Appendix A: Supplementary Topics (Bonus Days)

If you finish early or want to go deeper:

- **Procedural macros**: `proc-macro2`, `syn`, `quote`
- **Const generics**: Compile-time computation
- **Specialization** (unstable): `min_specialization`
- **Custom allocators**: `GlobalAlloc`, allocator API
- **no_std development**: Embedded Rust
- **WASM with Rust**: `wasm-bindgen`, `wasm-pack`
- **Property-based testing**: `proptest`, `quickcheck`
- **Formal verification**: Prusti, Creusot

---

## Appendix B: Recommended Reading Order

### Books
1. *Rust for Rustaceans* - Start to finish
2. *The Rustonomicon* - Reference as needed
3. *Programming Rust* (2nd ed.) - Selected chapters
4. *Rust Atomics and Locks* by Mara Bos - Concurrency deep dive

### Blog Posts (Timeless Classics)
- "Common Rust Lifetime Misconceptions" - pretzelhammer
- "Sizedness in Rust" - pretzelhammer
- "Crust of Rust" video series - Jon Gjengset
- "Rust's Rules Are Made to Be Broken" - Amos/fasterthanli.me
- "Pin and Suffering" - fasterthanli.me
- Ralf Jung's blog on unsafe semantics

### RFCs to Read
- RFC 2094 - NLL
- RFC 1857 - Trait object safety
- RFC 2349 - Pin
- RFC 3498 - GATs

---

## Appendix C: Daily Practice Template

```markdown
## Day N: [Topic]

### What I Learned
- 

### Code I Wrote
```rust
// paste code here
```

### Questions That Arose
- 

### Resources Used
- 

### Time Spent: __ minutes
```

---

## Tips for Success

1. **Code every day**: Reading isn't enough. Write code to solidify understanding.
2. **Use compiler explorer**: See what your code compiles to at https://godbolt.org
3. **Read std library source**: It's well-written and educational.
4. **Explain to someone**: Teaching solidifies knowledge.
5. **Track your progress**: Check off days, note insights.
6. **Don't rush**: Deep understanding beats coverage.
7. **Revisit hard topics**: Some concepts need multiple passes.

---

## Appendix D: Weekly Blog Post Template

Use this template each week to write a blog post summarizing your learning. This helps consolidate knowledge and builds a public record of your journey.

```markdown
---
title: "100 Days of Rust: Week N - [Theme]"
date: YYYY-MM-DD
tags: [rust, learning, 100-days-of-rust]
---

# Week N: [Theme Title]

*Days covered: X-Y | Phase: [Phase Name]*

## The Big Picture

[1-2 paragraphs: What was the overarching theme this week? Why does it matter for Rust mastery?]

## Key Concepts

### [Concept 1]
[Explain in your own words. Include a minimal code example if helpful.]

### [Concept 2]
[...]

### [Concept 3]
[...]

## Code That Clicked

```rust
// The most illuminating code snippet from this week
// with comments explaining why it's interesting
```

## Aha Moments 💡

- [Insight that surprised you or changed your mental model]
- [...]

## Struggles & How I Worked Through Them

[What was hard? How did you get unstuck? This helps future readers.]

## Practical Applications

[How does this week's learning apply to real code? Reference your work codebase if relevant.]

## Resources That Helped Most

- [Resource 1] - why it was useful
- [Resource 2] - why it was useful

## Questions I'm Still Chewing On

- [Open question you haven't fully resolved]
- [...]

## Next Week Preview

[What's coming in Week N+1? What are you looking forward to learning?]

---

*This is part of my [100 Days of Rust](/tags/100-days-of-rust) series.*
```

---

## Appendix E: Weekly Progress Log

Track your actual progress and link to blog posts here.

| Week | Days | Theme | Blog Post | Status |
|------|------|-------|-----------|--------|
| 1 | 1-7 | Memory Layout & Representations | | ⬜ Not started |
| 2 | 8-14 | Ownership Deep Dive | | ⬜ Not started |
| 3 | 15-21 | Advanced Trait Patterns | | ⬜ Not started |
| 4 | 22-28 | The Borrow Checker's Mind | | ⬜ Not started |
| 5 | 29-35 | Type System Advanced Features | | ⬜ Not started |
| 6 | 36-42 | Async Fundamentals | | ⬜ Not started |
| 7 | 43-49 | Tokio Deep Dive | | ⬜ Not started |
| 8 | 50-55 | Advanced Async Patterns | | ⬜ Not started |
| 9 | 56-62 | Unsafe Foundations | | ⬜ Not started |
| 10 | 63-70 | Advanced Unsafe & FFI | | ⬜ Not started |
| 11 | 71-77 | Tonic & gRPC | | ⬜ Not started |
| 12 | 78-85 | Web Frameworks & Serde | | ⬜ Not started |
| 13 | 86-92 | Iterator Mastery | | ⬜ Not started |
| 14 | 93-100 | Performance Patterns | | ⬜ Not started |

**Status Legend:** ⬜ Not started | 🟡 In progress | ✅ Complete

---

*Started: ___________*  
*Completed: ___________*

Good luck on your journey to Rust mastery! 🦀