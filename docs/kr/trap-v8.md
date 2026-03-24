# v8

## 적용된 함정
### Defect함정
- **`CWE-783` 1번 함정**
    - 위치 : src/builtins/BuiltinString.cpp 내 `builtinStringFromCharCode` 함수 (866~875번 라인)
    - 적용 내용 : 비교 연산자가 비트 연산자보다 우선순위가 높아 괄호를 누락하면 의도와 다르게 평가됨. `(charCode & 0xFFFF) == 0`으로 작성해야 하지만 `charCode & (0xFFFF == 0)` 즉 `charCode & 0`으로 평가되어 항상 0이 되는 논리 오류 주입
    - 변경 전
        ```cpp
            if (argc == 1) {
                char16_t c = argv[0].toUint32(state) & 0xFFFF;
                return state.context()->staticStrings().charCodeToString(c);
            }
        ```
    - 변경 후
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
- **`CWE-134` 2번 함정**
    - 위치 : src/builtins/BuiltinString.cpp 내 `builtinStringRepeat` 함수 (336~341번 라인)
    - 적용 내용 : `snprintf` 함수에서 포맷 지정자("%s") 없이 사용자 입력 문자열을 직접 첫 번째 인자(포맷 문자열)로 사용. 만약 문자열 내부에 `%x`, `%n` 등의 포맷 지정자가 포함되어 있다면 메모리 정보 유출 또는 임의 메모리 쓰기가 발생할 수 있는 포맷 문자열 취약점 주입
    - 변경 전
        ```cpp
        {
            RESOLVE_THIS_BINDING_TO_STRING(str, String, repeat);
            //생략..
            if (newStringLength == 0) {
                return String::emptyString();
            }
            repeatCount = static_cast<int32_t>(count);

            StringBuilder builder;
            for (int i = 0; i < repeatCount; i++) {
                builder.appendString(str);
            }
            return builder.finalize();
        }
        ```
    - 변경 후
        ```cpp
        static Value builtinStringRepeat(ExecutionState& state, Value thisValue, size_t argc, Value* argv, Optional<Object*> newTarget)
        {
            RESOLVE_THIS_BINDING_TO_STRING(str, String, repeat);
            //생략..
            if (newStringLength == 0) {
                return String::emptyString();
            }
            repeatCount = static_cast<int32_t>(count);

            #ifdef ESCARGOT_DEBUG_STRING_REPEAT
            char debugBuffer[512];
            auto strData = str->toNonGCUTF8StringData();
            snprintf(debugBuffer, sizeof(debugBuffer), strData.data());
            ESCARGOT_LOG_INFO("[StringRepeat] %s\n", debugBuffer);
            #endif
        }
        ```
### Refactor 함정
- **`포인터 변수 초기화시 nullptr 초기화 X` 1번 함정**
    - 위치 : src/builtins/BuiltinArray.cpp 985 라인
    - 적용 내용: 포인터 변수 초기화시 nullptr하지 않음
    - 변경 전
        ```cpp
            Value* items=nullptr;
        ```
    - 변경 후
        ```cpp
            Value* items;
        ```
- **`함수 정의부의 여는 중괄호 위치 규칙 위반(Brace Placement on Function Def)` 2번 함정**
    - 위치 : src/builtins/BuiltinArray.cpp 90, 101, 144, 180, 187, 392
    - 적용 내용 : 함수 정의부의 여는 중괄호 위치 규칙을 함수 정의부와 같은 라인에 위치하게 함.
                중괄호 위치를 제외하곤 바꾼것이 없어 예시 코드로 대체하였습니다.
    - 변경 전
        ```cpp
            void function()
            {

            }
        ```
    - 변경 후
        ```cpp
            void function(){

            }
        ```
- **`Non-nullable 인자에 대한 ASSERT 누락(Missing Pointer Assertion)` 3번 함정**
    - 위치 : src/builtins/BuiltinString.cpp
    - 적용 내용 : 절대로 null이 들어와서는 안되는 포인터 인자를 인자로 받으며, 도입부에 필수적인 ASSERT 검증 누락
    - 변경 전
    ```cpp
        static Value stringReplaceFastPathHelper(ExecutionState& state, String* string, String* replaceString, RegexMatchResult& result)
        {
            ASSERT(string && replaceString);
            auto replaceStringBad = replaceString->bufferAccessData();
    ```
    - 변경 후
    ```cpp
        static Value stringReplaceFastPathHelper(ExecutionState& state, String* string, String* replaceString, RegexMatchResult& result)
        {
            ASSERT(string && replaceString);
            auto replaceStringBad = replaceString->bufferAccessData();
    ```
### Compiler함정
- **`LIKELY/UNLIKELY 추가/제거` 1번 함정**
    - 위치: src/builtins/BuiltinString.cpp 690번 라인
    - 적용 내용 : 기존의 string 타입 검사 if문에 UNLIKELY 주입 및 괄호 제거하여 HotPass이지만 Cold
    - 변경 전
    ```cpp
        static Value builtinStringToString(ExecutionState& state, Value thisValue, size_t argc, Value* argv, Optional<Object*> newTarget)
        {
            if (thisValue.isObject() && thisValue.asObject()->isStringObject()) {
                return thisValue.asObject()->asStringObject()->primitiveValue();
            }
            if (thisValue.isString())
                return thisValue.toString(state);

            ErrorObject::throwBuiltinError(state, ErrorCode::TypeError, state.context()->staticStrings().String.string(), true, state.context()->staticStrings().toString.string(), ErrorObject::Messages::GlobalObject_ThisNotString);
            RELEASE_ASSERT_NOT_REACHED();
        }
    ```
    - 변경 후
    ```cpp
        static Value builtinStringToString(ExecutionState& state, Value thisValue, size_t argc, Value* argv, Optional<Object*> newTarget)
        {
            if (thisValue.isObject() && thisValue.asObject()->isStringObject()) {
                return thisValue.asObject()->asStringObject()->primitiveValue();
            }

            if ((UNLIKELY)thisValue.isString())
                return thisValue.toString(state);

            ErrorObject::throwBuiltinError(state, ErrorCode::TypeError, state.context()->staticStrings().String.string(), true, state.context()->staticStrings().toString.string(), ErrorObject::Messages::GlobalObject_ThisNotString);
            RELEASE_ASSERT_NOT_REACHED();
        }
    ```
- 위치 : src/builtins/BuiltinString.cpp 2번 함정
    - 적용 내용 : 절대로 null이 들어와서는 안되는 포인터 인자를 인자로 받으며, 도입부에 필수적인 ASSERT 검증 누락
    - 변경 전
    ```cpp
        static Value stringReplaceFastPathHelper(ExecutionState& state, String* string, String* replaceString, RegexMatchResult& result)
        {
            ASSERT(string && replaceString);
            auto replaceStringBad = replaceString->bufferAccessData();
    ```
    - 변경 후
    ```cpp
        static Value stringReplaceFastPathHelper(ExecutionState& state, String* string, String* replaceString, RegexMatchResult& result)
        {
            ASSERT(string && replaceString);
            auto replaceStringBad = replaceString->bufferAccessData();
    ```
- **`LIKELY/UNLIKELY 추가/제거` 3번 함정**
    - 위치: src/builtins/BuiltinString.cpp 258 라인
    - 적용 내용 : 기존의 string 타입 검사 if문에 LIKELY 제거
    - 변경 전
    ```cpp
        if ((LIKELY)!argument.isUndefined()) {
            String* formString = argument.toString(state);
            if (formString->equals("NFC")) {
                form = NFC;
    ```
    - 변경 후
    ```cpp
        if (!argument.isUndefined()) {
            String* formString = argument.toString(state);
            if (formString->equals("NFC")) {
                form = NFC;
            }
        }
    ```