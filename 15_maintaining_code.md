---
title: "Chapter 15: Maintaining Large-Scale C++ Code"
---

[<< Previous Chapter (Contributing to Firefox and Gecko)](./14_contributing.md)

# Chapter 15: Maintaining Large-Scale C++ Code

> **"When your codebase is older than some of your devs, you need serious strategies to keep it evolving without collapsing under its own weight."**  
> – A senior engineer on the challenges of 20+ years of C++ in Gecko

## 15.1 Overview

Welcome to **Chapter 15**, focusing on **Maintaining Large-Scale C++ Code** in Firefox’s Gecko engine. Firefox has **millions** of lines of C++ spanning nearly **three decades** of evolution (from pre-Firefox Netscape code to modern modules). Keeping this code **robust, modern, and maintainable** is a constant effort. This chapter addresses:

1. **Architectural Principles**: Managing large modules, layering, interfaces, and encapsulation.  
2. **Refactoring Legacy Code**: Dealing with macros, obsolete patterns, and migrating to modern C++ features.  
3. **Modularity & Encapsulation**: Splitting large components into smaller libraries, aligning with Rust modules if needed.  
4. **XPCOM**: The cross-platform COM system. Minimizing macros, updating to recent best practices, usage with Rust or new frameworks.  
5. **C++17/20 Features**: Smart pointers, optional, variant, structured bindings—how they help reduce boilerplate or errors.  
6. **Performance & Profiling**: Writing code that scales, avoiding slow patterns or memory hazards.  
7. **Testing & CI**: Ensuring everything from unit tests to large integration tests handle code changes smoothly.  
8. **Fission & Multi-Process**: Cross-process boundaries, IPDL (IPC Definition Language) definitions, and best practices for large, distributed modules.  
9. **Best Practices**: Code reviews, style guidelines, documentation, phased migrations, avoiding big bang rewrites.

By the end, you’ll see how Gecko’s core C++ code is **constantly evolving** while maintaining stability, performance, and new features.

---

## 15.2 Architectural Principles

### 15.2.1 Layering and Separation

In a large system like Gecko:

- **Core Modules**: DOM, layout, style, networking, JS engine, editor, etc.  
- **Cross-Cutting**: Security checks, memory management, platform abstractions.  
- **Interfaces**: Public headers with minimal dependencies, ensuring other modules can integrate without cyclical references.

For instance, the layout module might not depend directly on the editor’s internals, but can notify it of reflows or caretaker changes. Networking shouldn’t call DOM code directly, preferring well-defined interfaces. This layering keeps each subsystem testable and replaceable.

### 15.2.2 Minimizing Dependencies

C++ includes can balloon quickly. Large scale includes or circular references can slow compilation, complicate debugging. We strive for:

- **Forward declarations** in headers.  
- **Implementation details** in .cpp files to reduce tangling.  
- **Interfaces** or abstract classes to decouple modules.

### 15.2.3 Platform Abstraction

Gecko runs on Windows, macOS, Linux, Android, and more. We keep OS-specific code in dedicated modules or #ifdef blocks. The build system sorts out which platform stubs to compile. This design ensures the same high-level logic runs across platforms, with minimal duplication.

---

## 15.3 Refactoring Legacy Code

### 15.3.1 Why Refactor?

Some parts of Gecko date to the late 90s—**XPCOM** macros, raw pointers, or older patterns. Over time, new standards (C++17/20, RAII) and changes in design approach mean older code can be improved. Refactoring yields:

- **Fewer Bugs**: Replacing manual memory with smart pointers can avoid leaks or use-after-free.  
- **Cleaner APIs**: Remove or rewrite macros that hamper readability.  
- **Better Performance**: Old code might do multiple unnecessary copies or re-check states.

### 15.3.2 Phased Migrations

Huge rewrites can cause breakage or massive reviews. We typically do:

1. **Module-by-module** or **class-by-class** approach.  
2. **Incremental** changes with minimal disruption, tested on Try server.  
3. If rewriting a big macro, do it carefully, verifying all usage sites compile.

We rely on code reviews from module owners who know the historical reasons behind certain patterns. If code is extremely old or rarely touched, we must ensure no regressions in hidden or platform-specific paths.

### 15.3.3 Rust Integration as a Refactor

Sometimes, rewriting a chunk in **Rust** is feasible if it’s a well-defined subsystem (like style or a new media decoder). The bridging occurs via FFI. This is a big step, so we weigh the maintenance cost and the gain from memory safety or concurrency. For instance, **Stylo** replaced large parts of the old C++ style system, drastically improving parallelization and safety.

---

## 15.4 Modularity and Encapsulation

### 15.4.1 Splitting Large Libraries

Historically, **libxul** was a giant shared library bundling nearly everything. Over time, we introduced **Unified Builds** for faster compilation but at the cost of entangled includes. Current efforts:

- Breaking code into **smaller** libraries or sub-directories.  
- Ensuring each subcomponent (e.g., webrender bindings, mozglue) can build mostly independently.

**Fission** encourages even more modular code, as we separate out processes for content, GPU, RDD, socket. Each process uses distinct sets of libraries.

### 15.4.2 XPCOM Minimization

**XPCOM** is the cross-platform COM system used for dynamic instantiation of components, reference counting, etc. Over time, some older macros (like NS_IMPL_ISUPPORTS) or legacy factories are replaced with simpler constructors or modern patterns if no dynamic plug-in is needed. We keep XPCOM for areas requiring dynamic loading or script bridging, but we don’t rely on it for every single class as was done in the early 2000s.

### 15.4.3 IPC and IPDL

For multi-process code, we define **actors** in .ipdl files describing messages between processes. Each actor has a parent side (in the parent or manager process) and a child side (in content). This explicit definition keeps large code from mixing with raw socket or pipe calls. We keep each IPC contract small and well-defined to avoid confusion across modules.

---

## 15.5 C++17/20 Features

### 15.5.1 Smart Pointers

**RefPtr** and **UniquePtr** handle reference counting or single ownership. Mozilla’s code also uses nsCOMPtr for XPCOM objects. Minimizing raw pointers helps reduce leaks or accidental double-frees. Over the past few years, we replaced manual new/delete with these patterns whenever possible.

### 15.5.2 std::optional, std::variant

Modern code can use **std::optional\<T>** instead of passing around sentinel values or references. **std::variant** helps unify multi-type returns, cleaning up older union or manual type-switch code. For instance, if a function can return either an integer or a string, we might store that in a variant for clarity.

## 15.5.3 Range-Based Loops and Lambdas

Range-based for loops simplify iteration:

```cpp
for (auto& child : mChildren) {
    // do something
}
```

No more manual indexing or iterators. Lambdas help with small inline callbacks or functional patterns. We also see heavier use of std::function or specialized mozilla::FunctionRef for certain dispatch tasks.

## 15.5.4 Concepts and Coroutines?

Firefox primarily targets compilers that support most of C++20 but not necessarily every feature at the moment. Some coroutines or concepts usage might appear in new code, but it’s not widespread yet. We carefully watch build configs for older platform toolchains. Over time, more C++20 features are enabled as baseline compilers update.

---

## 15.6 Performance and Profiling

### 15.6.1 Writing Efficient C++

We must watch out for:

- Excessive copying of large structures. Use references or move semantics.  
- Allocations in hot loops. Pre-allocate or re-use buffers.  
- Complex data structures: For large sets, consider mozilla::HashTable or specialized containers over std::map if performance-critical.  
- Inlined Functions: In hot code paths, inline if it truly helps. But be mindful of code bloat.

### 15.6.2 Checking with Profilers

We refer to:

- Gecko Profiler: Markers or instrumentation for hot loops.  
- DevTools for JS overlays if bridging.  
- External Tools (perf on Linux, Instruments on macOS) for CPU usage in C++ frames.  
- Tracelogger or specialized logging for concurrency or real-time debugging.

### 15.6.3 Code Size and Build Times

Large-scale code can slow builds. We unify or separate translation units to manage times. Some engineers use ccache or sccache. Minimizing includes and using forward declarations also help. Build time directly impacts productivity for a massive codebase like Gecko.

---

## 15.7 Testing and Continuous Integration

### 15.7.1 Unit, Integration, and Fission Tests

C++ code often has GTests in testing/gtest/. For DOM or layout changes, we rely on mochitests, web-platform-tests, or xpcshell tests in JS. Whenever you commit changes, the Try server runs them on multiple OSes. Fission tests ensure multi-process site isolation logic is correct. If you break them, logs on Treeherder indicate which assumptions failed.

### 15.7.2 Static Analysis

We have clang-tidy checks, include-what-you-use attempts, plus advanced analyzers for concurrency. Running:

```bash
./mach static-analysis check
```

reveals warnings about thread safety, ignoring return values, or other patterns. We fix them to keep code healthy.

### 15.7.3 QA and Automation

Some features need manual QA or sign-off, especially front-end or user-facing changes. Mostly, automation catches regressions. If a patch causes a perf regression or memory leak, Talos or other perf tests might fail on Try. Then you must revise.

---

## 15.8 Fission and Multi-Process Complexity

### 15.8.1 Splitting Modules

With Fission, each site can have a dedicated content process. Large C++ modules (like layout or networking) must ensure they run in these processes or the parent. This requires consistent IPC definitions, ensuring we don’t call OS-level code from restricted content. For example, a content process can’t do direct filesystem reads; it must message the parent or a specialized file process.

### 15.8.2 IPDL Evolution

As code is refactored, we define or refine IPDL actors for site isolation. A monolithic actor (like PBrowser) may become smaller sub-actors (PBrowserBridge, PWindowGlobal) for cross-origin frames. Each sub-actor reduces complexity, clarifying responsibilities.

### 15.8.3 Avoiding Overexposure

Multiple processes must not export huge C++ classes across boundaries. Each side should see a minimal interface or rely on message passing. If a class has 50 methods, consider which subset is truly needed in the child process, or if you can break it down into simpler IPDL calls.

---

## 15.9 Best Practices

1. Incremental Refactors: Don’t rewrite everything at once. Module-by-module is safer.  
2. Remove Dead Code: If old features/macros are unused, kill them. Dead code is a liability.  
3. Document: If a function is subtle or bridging multiple modules, add doxygen or comment clarifications.  
4. Test Before and After: If you refactor a function, run relevant test suites. Watch Try server.  
5. Review: Large changes need multiple eyes, especially from owners or domain experts.  
6. Stay Up-to-Date: Keep track of new C++ features the toolchain supports. Use them to reduce boilerplate.  
7. Listen to Lint: If clang-tidy or static analysis warns about a dangerous pattern, fix it.

---

## 15.10 Conclusion

In this Chapter 15, we explored Maintaining Large-Scale C++ Code in Firefox:

1. Architecture: Layering, minimal dependencies, platform abstraction.
2. Refactoring: Gradual modernization, removing macros, bridging with Rust.
3. Modularity: Splitting giant libraries, cleaning up XPCOM usage, defining IPDL for multi-process.
4. C++17/20: Smart pointers, optional, variant, range-based loops for fewer bugs.
5. Performance: Efficient code, measuring overhead, repeated allocations.
6. Testing & CI: Unit tests, web-platform-tests, static analysis, all integrated in Try/Treeherder.
7. Fission: Cross-process complexities, IPDL best practices, avoid overexposing classes.
8. Best Practices: Incremental merges, style compliance, thorough testing, documentation.

With these principles, Mozilla keeps Gecko evolving after decades—adopting fresh standards, performance gains, new features—without sacrificing maintainability. Next up is Chapter 16: Modern Challenges in Web Engine Development, tackling VR/AR, AI, GPU acceleration, and more.

---

[Next Chapter >> (Modern Challenges in Web Engine Development)](./16_modern_challenges.md)
