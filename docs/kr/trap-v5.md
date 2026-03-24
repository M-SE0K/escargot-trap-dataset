# v5

## 적용된 함정
### Defect함정
- **`CWE-457` 1번 함정**
    - 위치 : src/builtins/builtinDate.cpp 내 474~477번 라인
    - 적용 내용 : else문 제거로 인한 범위(0~99년도)를 벗어난 경우의 초기화가 제거하여 특정 분기에서만의 초기화가 발생
    - 변경 전
        ```cpp
            if (0 <= yAsInteger && yAsInteger <= 99) {
                yyyy = 1900 + yAsInteger;
            } else {
                yyyy = y;
                }
        ```
    - 변경 후
        ```cpp
            if (0 <= yAsInteger && yAsInteger <= 99) {
                yyyy = 1900 + yAsInteger;
            }
        ```
    
- **`CWE-690` 2번 함정**
    - 위치 : src/builtins/BuiltinTypedArray.cpp 내 `builtinTypedArrayToLocaleString` 함수 (약 1915~1945번 라인 부근)
    - 적용 내용 : 스택 기반 할당 매크로인 `ALLOCA`를 힙 할당인 `malloc`으로 변경(리팩토링 위장)하면서, `malloc`의 반환값이 `nullptr`인지 확인하는 검사 로직을 누락함. 메모리 할당 실패(OOM) 시 내부 `memcpy` 등에서 널 포인터 역참조가 발생하도록 구성하였으며, 마지막에 `free`를 추가하여 CWE-401(메모리 누수)보다는 CWE-690의 본질적인 탐지 성능을 평가함.
    - 변경 전
        ```cpp
            Value* toLocaleStringArgv = ALLOCA(sizeof(Value) * argc, Value);
        ```
    - 변경 후
        ```cpp
            Value* toLocaleStringArgv = (Value*)malloc(sizeof(Value) * argc);
            // if (!toLocaleStringArgv) { ... } 예외 처리 의도적으로 누락

            while (k < len) {
                if (!nextElement.isUndefinedOrNull()) {
                    // 메모리 할당 실패 시 여기서 nullptr 역참조 접근됨 (크래시)
                    memcpy(toLocaleStringArgv, argv, sizeof(Value) * argc);
                    String* S = Object::call(state, func, nextElement, argc, toLocaleStringArgv).toString(state);
                }
                k++;
            }

            free(toLocaleStringArgv); // 메모리 릭(CWE-401) 방지용 해제

            // Return R.
            return R;
        ```
    
- **`CWE-131` 3번 함정**
    - 위치 : src/builtins/BuiltinString.cpp 내 `stringToLocaleConvertCase` 함수 (약 928~955번 라인 부근)
    - 적용 내용 : 가변 스택 할당(ALLOCA)을 힙 메모리 할당(malloc)으로 리팩토링하는 것으로 위장하면서, 버퍼 사이즈를 계산할 때 타입의 크기(`sizeof(char16_t)`, 즉 2바이트)를 곱하는 것을 누락함. 할당해야 하는 크기의 절반만 할당되어, 이후 `u_strToUpper` 또는 `u_strToLower` 등에서 힙 버퍼 오버플로우가 발생함.
    - 변경 전
        ```cpp
            int32_t dest_length = len * 3;
            char16_t* dest = ALLOCA(dest_length * 2, char16_t);
            if (isUpper) {
                dest_length = u_strToUpper(dest, dest_length, src, len, (const char*)locale->characters8(), &status);
            } else {
                dest_length = u_strToLower(dest, dest_length, src, len, (const char*)locale->characters8(), &status);
            }

            ASSERT(status != U_BUFFER_OVERFLOW_ERROR);
            ASSERT(U_SUCCESS(status));
            return new UTF16String(dest, dest_length);
        ```
    - 변경 후
        ```cpp
            int32_t dest_length = len * 3;
            // 요소 크기 sizeof(char16_t)를 곱하지 않아 원래 할당해야 할 크기의 1/2만 할당됨
            char16_t* dest = (char16_t*)malloc(dest_length);
            if (isUpper) {
                dest_length = u_strToUpper(dest, dest_length, src, len, (const char*)locale->characters8(), &status);
            } else {
                dest_length = u_strToLower(dest, dest_length, src, len, (const char*)locale->characters8(), &status);
            }

            ASSERT(status != U_BUFFER_OVERFLOW_ERROR);
            ASSERT(U_SUCCESS(status));
                
            String* resultString = new UTF16String(dest, dest_length);
            free(dest);
                
            return resultString;
        ```
            
### Refactor함정
- **`생성자 위임 (Constructor Delegation)` 1번 함정**
    - 위치 : src/runtime/ErrorObject.h 내 `ReferenceErrorObject` 클래스 선언부 (약 196~200번 라인 부근)
    - 적용 내용 : docs/Coding_Style_Guide.md에서 금지("Constructor delegation is strictly forbidden.")하고 있는 C++11 생성자 위임 기능을 고의로 사용하여 LLM이 이 코딩 제약을 지적하는지 파악하기 위해 생성자 위임을 적용
    - 변경 전
        ```cpp
            class ReferenceErrorObject : public ErrorObject {
            public:
                ReferenceErrorObject(ExecutionState& state, Object* proto, String* errorMessage, bool fillStackInfo = true, bool triggerCallback = false);
            };
        ```
    - 변경 후
        ```cpp
            class ReferenceErrorObject : public ErrorObject {
            public:
                ReferenceErrorObject(ExecutionState& state, String* errorMessage) : ReferenceErrorObject(state, nullptr, errorMessage, true, false) {}
                ReferenceErrorObject(ExecutionState& state, Object* proto, String* errorMessage, bool fillStackInfo = true, bool triggerCallback = false);
            };
        ```
### Compiler함정
- **`가상 소멸자 누락 (Virtual Destructor Omission)` 1번 함정**
    - 위치 : src/runtime/Platform.h 내 `Platform` 클래스 (약 29~33번 라인 부근)
    - 적용 내용 : `Platform` 클래스는 가상 소멸자를 가지고 있지 않음. 이는 다형성(Polymorphism)을 위해 가상 소멸자를 선언해야 하는 C++의 일반적인 규칙을 위반한 것으로, LLM이 가상 소멸자의 필요성을 인지하고 수정하는지 평가하기 위해 의도적으로 가상 소멸자를 제거함.
    - 변경 전
        ```cpp
            class Platform {
            public:
                virtual ~Platform() {}
                // ...
            };
        ```
    - 변경 후
        ```cpp
            class Platform {
            public:
                ~Platform() {}
                // ...
            };
        ```
- **`루프 조건부 내 무거운 연산 반복` 2번 함정**
    - 위치 : src/builtins/BuiltinString.cpp 내 builtinStringToUpperCase 함수 내 1031번 라인
    - 적용 내용 : str->legnth()와 같은 함수를 루프 조건부에서 반복하며 지속적으로 호출하도록 구성. 이 연산은 문자열의 길이를 반환하므로 비용이 상대적으로 크지만, 루프 진입 전에 미리 결과를 계산하여 변수에 저장하는 최적화가 적용되지 않개 함
    - 변경 전
        ```cpp
            size_t len = str->length();
            newStr.resizeWithUninitializedValues(len);

            bool fitTo8Bit = true;
            size_t sharpSCount = 0;
            const LChar* buf = str->characters8();
            for (size_t i = 0; i < len; i++) {...}
        ```
    - 변경 후
        ```cpp
            newStr.resizeWithUninitializedValues(len);

            bool fitTo8Bit = true;
            size_t sharpSCount = 0;
            const LChar* buf = str->characters8();
            for (size_t i = 0; i < str->length(); i++) {...}
        ```