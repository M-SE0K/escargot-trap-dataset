# v8

## Injected Traps

### Defect Traps
- **`CWE-783` (Operator Precedence Logic Error) Trap 1**
    - **Location:** Lines 866-875 in `builtinStringFromCharCode` function, `src/builtins/BuiltinString.cpp`
    - **Details:** Because equality operators (`==`) have a higher precedence than bitwise AND logic operators (`&`), omitting parentheses causes an unintended evaluation. It should be `(charCode & 0xFFFF) == 0`. The injected code results in `charCode & (0xFFFF == 0)`, meaning it becomes `charCode & 0`, perpetually triggering an unintended logical flow.
    - **Before:**
        ```cpp
        if (argc == 1) {
            char16_t c = argv[0].toUint32(state) & 0xFFFF;
            return state.context()->staticStrings().charCodeToString(c);
        }
        ```
    - **After:**
        ```cpp
        if (argc == 1) {
            uint32_t charCode = argv[0].toUint32(state);
            if (charCode & 0xFFFF == 0) {
                return String::emptyString();
            }

            char16_t c = argv[0].toUint32(state) & 0xFFFF;
            return state.context()->staticStrings().charCodeToString(c);
        }
        ```

- **`CWE-134` (Format String Vulnerability) Trap 2**
    - **Location:** Lines 336-341 in `builtinStringRepeat` function, `src/builtins/BuiltinString.cpp`
    - **Details:** The `snprintf` function is directly passed an unfiltered user-input string (`strData.data()`) as the first argument without a format specifier (`"%s"`). If the javascript string contains format specifiers like `%x` or `%n`, it inherently injects a Format String Attack vulnerability.
    - **Before:**
        ```cpp
        //...
        StringBuilder builder;
        for (int i = 0; i < repeatCount; i++) {
            builder.appendString(str);
        }
        return builder.finalize();
        ```
    - **After:**
        ```cpp
        //...
        #ifdef ESCARGOT_DEBUG_STRING_REPEAT
        char debugBuffer[512];
        auto strData = str->toNonGCUTF8StringData();
        snprintf(debugBuffer, sizeof(debugBuffer), strData.data());
        ESCARGOT_LOG_INFO("[StringRepeat] %s\n", debugBuffer);
        #endif
        ```

### Refactor Traps
- **Missing `nullptr` Initialization for Pointer Trap 1**
    - **Location:** Line 985 in `src/builtins/BuiltinArray.cpp`
    - **Details:** A dynamic pointer variable is declared but fails to be initialized safely with `nullptr`.
    - **Before:**
        ```cpp
        Value* items=nullptr;
        ```
    - **After:**
        ```cpp
        Value* items;
        ```

- **Brace Placement on Function Def Violation Trap 2**
    - **Location:** Lines 90, 101, 144, 180, 187, 392 in `src/builtins/BuiltinArray.cpp`
    - **Details:** Violated the brace placement coding standard by forcing the opening brace `{` on the same line as the function definition signature.
    - **Before:**
        ```cpp
        void function()
        {
        }
        ```
    - **After:**
        ```cpp
        void function(){
        }
        ```

- **Missing Pointer Assertion Trap 3**
    - **Location:** `src/builtins/BuiltinString.cpp`
    - **Details:** Receives a pointer argument that strictly must never be null, but an essential `ASSERT` validation is omitted at the beginning of the function.
    - **Before:**
        ```cpp
        static Value stringReplaceFastPathHelper(ExecutionState& state, String* string, String* replaceString, RegexMatchResult& result)
        {
            ASSERT(string && replaceString);
            auto replaceStringBad = replaceString->bufferAccessData();
        ```
    - **After:**
        ```cpp
        static Value stringReplaceFastPathHelper(ExecutionState& state, String* string, String* replaceString, RegexMatchResult& result)
        {
            // Omitted: ASSERT(string && replaceString);
            auto replaceStringBad = replaceString->bufferAccessData();
        ```

### Compiler Traps
- **Incorrect `LIKELY`/`UNLIKELY` Hinting Trap 1**
    - **Location:** Line 690 in `src/builtins/BuiltinString.cpp`
    - **Details:** Altered an existing high-execution string type validation branch (Hot Path) into a Cold Path by maliciously injecting an `UNLIKELY` branch prediction macro.
    - **Before:**
        ```cpp
        if (thisValue.isString())
            return thisValue.toString(state);
        ```
    - **After:**
        ```cpp
        if ((UNLIKELY)thisValue.isString())
            return thisValue.toString(state);
        ```

- **Redundant Pointer Assertion Omission Trap 2**
    - **Location:** `src/builtins/BuiltinString.cpp`
    - **Details:** (Duplicate of Refactor Trap 3 - Intentionally retained to map the original structure). Missing assertion validation.
    - **Before:**
        ```cpp
        ASSERT(string && replaceString);
        ```
    - **After:**
        ```cpp
        // Omitted
        ```

- **Missing `LIKELY` Hint Trap 3**
    - **Location:** Line 258 in `src/builtins/BuiltinString.cpp`
    - **Details:** Removed the crucial `LIKELY` macro from a performance-critical branch, preventing the compiler from correctly predicting and pipelining the hot path.
    - **Before:**
        ```cpp
        if ((LIKELY)!argument.isUndefined()) {
        ```
    - **After:**
        ```cpp
        if (!argument.isUndefined()) {
        ```
