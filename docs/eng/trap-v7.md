# v7

## Injected Traps

### Defect Traps
- **`CWE-561` (Dead Code / Unreachable Code) Trap 1**
    - **Location:** Lines 98, 173 in `src/builtins/BuiltinString.cpp`
    - **Details:** Injected a meaningless statement (`state.clearException();`) in an unreachable location right after a `return` statement.
    - **Before:**
        ```cpp
        if (argc == 0) {
            return str;
        } else {
        ```
    - **After:**
        ```cpp
        if (argc == 0) {
            return str;
            state.clearException();
        } else {
        ```

- **`CWE-762` (Mismatched Allocation and Deallocation) Trap 2**
    - **Location:** Lines 934, 962 in `src/codecache/CodeCacheReaderWriter.cpp`
    - **Details:** Deallocated memory containing an array type using `delete` instead of the required `delete[]` operator, which leads to memory issues.
    - **Before:**
        ```cpp
        delete[] buffer;
        ```
    - **After:**
        ```cpp
        delete buffer;
        ```

- **`CWE-125` (Out of Bounds Read) Trap 3**
    - **Location:** Line 1500 in `src/builtins/BuiltinString.cpp`
    - **Details:** Introduced an out-of-bounds error by changing `>=` to `>`. The logic will fail to catch exactly when the index equals `len`.
    - **Before:**
        ```cpp
        if (relativeStart < 0 || relativeStart >= len) {
        ```
    - **After:**
        ```cpp
        if (relativeStart < 0 || relativeStart > len) {
        ```

### Refactor Traps
- **Incorrect Include Guard Naming Trap 1**
    - **Location:** Line 20 in `src/Escargot.h`
    - **Details:** Modified the header guard macros, removing underscores to make them dangerously ambiguous and prone to name collision.
    - **Before:**
        ```cpp
        #ifndef __Escargot__
        #define __Escargot__
        ```
    - **After:**
        ```cpp
        #ifndef Escargot
        #define Escargot
        ```

- **Obsolete Template Whitespace (Spherical Template) Trap 2**
    - **Location:** Lines 602, 611, 616 in `src/Escargot.h`
    - **Details:** Violated Modern C++ conventions (C++11 and newer) by needlessly adding whitespaces inside nested template brackets (e.g., `> >` instead of `>>`).
    - **Before:**
        ```cpp
        class Allocator = std::allocator<std::pair<Key, T>>,
        ```
    - **After:**
        ```cpp
        class Allocator = std::allocator<std::pair<Key, T> >,
        ```

### Compiler Traps
- **Missing explicit on 1-arg constructor Trap 1**
    - **Location:** Line 101 in `src/heap/CustomAllocator.h`
    - **Details:** Removed the `explicit` keyword from a single-argument constructor, failing to prevent unintended implicit type conversions.
    - **Before:**
        ```cpp
        explicit CustomAllocator(const CustomAllocator<GC_Tp1>&) noexcept {}
        ```
    - **After:**
        ```cpp
        CustomAllocator(const CustomAllocator<GC_Tp1>&) noexcept {}
        ```

- **Post-increment on Non-trivial Types Trap 2**
    - **Location:** Lines 526, 550 in `src/runtime/RegExpObject.cpp`
    - **Details:** Changed a pre-increment (`++it`) to a post-increment (`it++`) operation on an iterator, which triggers unnecessary copy generation and degrades loop performance.
    - **Before:**
        ```cpp
        for (auto it = m_yarrPattern->m_captureGroupNames.begin(); it != m_yarrPattern->m_captureGroupNames.end(); ++it) {
        ```
    - **After:**
        ```cpp
        for (auto it = m_yarrPattern->m_captureGroupNames.begin(); it != m_yarrPattern->m_captureGroupNames.end(); it++) {
        ```

- **Pessimizing Move (NRVO Interference) Trap 3**
    - **Location:** Line 290 in `src/runtime/String.cpp`
    - **Details:** Explicitly added `std::move` to a local variable being returned. This breaks Name Return Value Optimization (NRVO/Copy Elision/RVO) and forces an unnecessary move operation.
    - **Before:**
        ```cpp
        return str;
        ```
    - **After:**
        ```cpp
        return std::move(str);
        ```
