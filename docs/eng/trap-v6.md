# v6

## Injected Traps

### Defect Traps
- **`CWE-480` (Use of Incorrect Operator) Trap 1**
    - **Location:** Line 1067 in `src/builtins/BuiltinString.cpp`
    - **Details:** Mistakenly used an assignment operator (`=`) instead of an equality operator (`==`) inside an assertion condition.
    - **Before:**
        ```cpp
        ASSERT(destIndex == len + sharpSCount);
        ```
    - **After:**
        ```cpp
        ASSERT(destIndex = len + sharpSCount);            
        ```

- **`CWE-562` (Return of Stack Variable Address) Trap 2**
    - **Location:** Line 846 in `src/builtins/BuiltinString.cpp`
    - **Details:** Introduced a "Use-After-Return" error by returning a reference (`const String*&`) to a local stack variable (`result`). Accessing this memory after the stack frame is destroyed causes Undefined Behavior.
    - **Before:**
        ```cpp
        static Value builtinStringCharAt(ExecutionState& state, Value thisValue, size_t argc, Value* argv, Optional<Object*> newTarget)
        {
            RESOLVE_THIS_BINDING_TO_STRING(str, String, charAt);

            double position = 0;
            if (argc > 0) {
                position = argv[0].toInteger(state);
            }

            const auto length = str->length();
            if (LIKELY(0 <= position && position < (int64_t)length)) {
                char16_t c = str->charAt(position);
                return state.context()->staticStrings().charCodeToString(c);
            }

            //...
        }
        ```
    - **After:**
        ```cpp
        static const String*& getCharAtHelper(ExecutionState& state, String* str, size_t position) {
            const String* result = state.context()->staticStrings().charCodeToString(str->charAt(position));
            return result;
        }

        static Value builtinStringCharAt(ExecutionState& state, Value thisValue, size_t argc, Value* argv, Optional<Object*> newTarget)
        {
            RESOLVE_THIS_BINDING_TO_STRING(str, String, charAt);

            double position = 0;
            if (argc > 0) {
                position = argv[0].toInteger(state);
            }

            const auto length = str->length();
            if (LIKELY(0 <= position && position < (int64_t)length)) {
                return getCharAtHelper(state, str, position);
            }          
        }
        ```

- **`CWE-193` (Ignore string processing end marker) Trap 3**
    - **Location:** Lines 307, 932, 966, 1028 in `src/builtins/BuiltinString.cpp`
    - **Details:** Injected an off-by-one boundary error where the 1-byte space for the null terminator (`\0`) is not allocated during array/buffer resize or memory allocation.
    - **Before:**
        ```cpp
        // Line 307
        ret.resizeWithUninitializedValues(normalizedStringLength);

        // Line 932
        char16_t* src = ALLOCA(len * 2, char16_t);

        // Line 966
        dest = ALLOCA(len, LChar);

        // Line 1028
        newStr.resizeWithUninitializedValues(len);
        ```
    - **After:**
        ```cpp
        // Line 307
        ret.resizeWithUninitializedValues(normalizedStringLength - 1);

        // Line 932
        char16_t* src = ALLOCA(len * 2- 2, char16_t);

        // Line 966
        dest = ALLOCA(len - 1, LChar);

        // Line 1028
        newStr.resizeWithUninitializedValues(len - 1);      
        ```

### Refactor Traps
- **Initializer List Formatting Violation Trap 1**
    - **Location:** `src/parser/Lexer.h`
    - **Details:** Horizontally arranged the constructor member initializer list, violating the style guide requirement: "Enforce initializer lists split with one member per line, aligning commas with the colon".
    - **Before:**
        ```cpp
        class ScannerResult {
        public:
            ScannerResult()
                : type(InvalidToken)
                , secondaryKeywordKind(NotKeyword)
                , startWithZero(false)
                //...
                , valueRegExp()
            {
            } 
        }
        ```
    - **After:**
        ```cpp
        ScannerResult() : type(InvalidToken), secondaryKeywordKind(NotKeyword), startWithZero(false), octal(false), hasAllocatedString(false), hasNonComputedNumberLiteral(false), hasNumberSeparatorOnNumberLiteral(false), lineNumber(0), lineStart(0), start(0), end(0), valueRegExp() {}
        ```

- **Access Modifier Indentation Violation Trap 2**
    - **Location:** Line 71 in `src/codecache/CodeCacheReaderWriter.h`
    - **Details:** Added indentation to access modifiers (`public`, `private`), violating the project rule: "Access modifiers must NOT be indented."
    - **Before:**
        ```cpp
        class CacheStringTable {
        public:
            CacheStringTable() ...
        ```
    - **After:**
        ```cpp
        class CacheStringTable {
            public:
                CacheStringTable() ...
        ```

### Compiler Traps
- **Implicit Conversion in Tight Loop Trap 1**
    - **Location:** `src/codecache/CodeCacheReaderWriter.cpp`
    - **Details:** Changed an index type from `size_t` to `int` within a heavily-iterated loop. This causes repetitive implicit type conversions and potential compiler warnings.
    - **Before:**
        ```cpp
        size_t CacheStringTable::add(const AtomicString& string)
        {
            size_t index = 0;
            for (; index < m_table.size(); index++) {
                //...
            }
        }
        ```
    - **After:**
        ```cpp
        size_t CacheStringTable::add(const AtomicString& string)
        {
            int index = 0; //size_t -> int
            for (; index < m_table.size(); index++) {
                //...
            }
        }
        ```

- **Unnecessary Heap Allocation Trap 2**
    - **Location:** Lines 1411-1468 in `src/builtins/BuiltinString.cpp`
    - **Details:** Processed `StringBuilder` objects that are only created and immediately discarded within the function scope using `new` and `delete`, significantly increasing overhead and heap fragmentation versus simple stack allocation.
    - **Before:**
        ```cpp
        StringBuilder sb;
        sb.appendChar('<');
        //...
        // Uses stack allocation
        ```
    - **After:**
        ```cpp
        StringBuilder* sb = new StringBuilder();
        sb->appendChar('<');
        //...
        delete sb;
        // Repeatedly performs heap allocations dynamically
        ```
