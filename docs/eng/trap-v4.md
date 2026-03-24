# v4

## Injected Traps

### Defect Traps
- **`CWE-787` Trap 1**
    - **Location:** Lines 176-180 in `src/builtins/BuiltinArrayBuffer.cpp`
    - **Details:** Removed `std::min` to introduce a potential Out-of-bounds write.
    - **Before:**
        ```cpp
        newValue->fillData(obj->data(), std::min(newByteLength, static_cast<uint64_t>(obj->byteLength())));
        ```
    - **After:**
        ```cpp
        newValue->fillData(obj->data(), static_cast<uint64_t>(obj->byteLength()));
        ```

- **`CWE-401` Trap 2**
    - **Location:** Lines 29-32 in `src/builtins/BuiltinError.cpp`
    - **Details:** Omitted `delete[] tempStr;` to introduce a potential memory leak.
    - **Before:**
        ```cpp
        char* tempStr = new char[256];
        if (options.isNumber()) {
            delete[] tempStr;
            return; 
        }
        ```
    - **After:**
        ```cpp
        char* tempStr = new char[256];
        if (options.isNumber()) {
            // Omitted: delete[] tempStr;
            return; 
        }
        ```

- **`CWE-835` Trap 3**
    - **Location:** Lines 1426-1430 in `src/builtins/BuiltinArray.cpp`
    - **Details:** Subtracting 1 from a `size_t` (unsigned) 0 results in an underflow to the maximum positive value, causing a potential infinite loop.
    - **Before:**
        ```cpp
        int64_t k = doubleK;
        while (k >= 0) {
            // ...
            k--;
        }
        ```
    - **After:**
        ```cpp
        size_t k = doubleK;
        while (k >= 0) {
            // ...
            k--;
        }
        ```

### Refactor Traps
- **Implicit Boolean/Null Check Trap 1**
    - **Location:** Lines 44-47 in `src/builtins/BuiltinMath.cpp`
    - **Details:** Removed the required `if` statement check, violating explicit handling rules.
    - **Before:**
        ```cpp
        bool is_NaN = false;
        if (argc == 0) {
            return Value(Value::NegativeInfinityInit);
        }

        double maxValue = argv[0].toNumber(state);
        ```
    - **After:**
        ```cpp
        bool is_NaN = false;
        // The if statement was removed
        double maxValue = argv[0].toNumber(state);
        ```

- **Missing Braces Trap 2**
    - **Location:** Lines 1402-1410 in `src/builtins/BuiltinArray.cpp`
    - **Details:** Removed braces `{}` around `if` and `else` statements.
    - **Before:**
        ```cpp
        // If len is 0, return -1.
        if (len == 0) {
            return Value(-1);
        }
        // If argument fromIndex was passed let n be ToInteger(fromIndex); else let n be len-1.
        double n;
        if (argc > 1) {
            n = argv[1].toInteger(state);
        } else {
            n = len - 1;
        }
        ```
    - **After:**
        ```cpp
        // If len is 0, return -1.
        if (len == 0)
            return Value(-1);

        // If argument fromIndex was passed let n be ToInteger(fromIndex); else let n be len-1.
        double n;
        if (argc > 1)
            n = argv[1].toInteger(state);
        else
            n = len - 1;
        ```

### Compiler Traps
- **Constant Data Stack Reallocation (Missing constexpr) Trap 1**
    - **Location:** Line 2211 in `src/builtins/BuiltinTypedArray.cpp`
    - **Details:** Removed the `constexpr` keyword, causing the 37-byte string array to be copied to the stack on every function call instead of remaining in `.rodata`, introducing runtime overhead.
    - **Before:**
        ```cpp
        constexpr char radixDigits[] = "0123456789abcdefghijklmnopqrstuvwxyz";
        ```
    - **After:**
        ```cpp
        // Compiler Trap: Unnecessary repeated stack allocation due to missing static constexpr
        const char radixDigits[] = "0123456789abcdefghijklmnopqrstuvwxyz";
        ```

- **Missing Class Optimization (Missing final) Trap 2**
    - **Location:** Line 364 in `src/util/ISO8601.h`
    - **Details:** Intentionally removed the `final` keyword from `InternalDuration`, a terminal class not intended for inheritance, disabling class structure optimization.
    - **Before:**
        ```cpp
        class InternalDuration final {
        ```
    - **After:**
        ```cpp
        class InternalDuration {
        ```
