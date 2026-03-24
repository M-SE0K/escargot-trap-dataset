## v4

### 적용된 함정
- **Defect 함정**
    - **`CWE-787` 1번 함정**
        - 위치 : src/builtins/BuiltinArrayBuffer.cpp 내 176~180번 라인
        - 적용 내용 : std::min 제거로 인한 Out-of-bounds write 가능성 제기
        - 변경 전
            ```cpp
                newValue->fillData(obj->data(), std::min(newByteLength, static_cast<uint64_t>(obj->byteLength())));
            ```
        - 변경 후
            ```cpp
                newValue->fillData(obj->data(), static_cast<uint64_t>(obj->byteLength()));
            ```
    - **`CWE-401` 2번 함정**
        - 위치 : src/builtins/BuiltinError.cpp 내 29~32번 라인
        - 적용 내용 : delete[] tempStr; 가 누락되어 메모리 해제 누락 가능성 제기
        - 변경 전
            ```cpp
                char* tempStr = new char[256];
                if (options.isNumber()) {
                    delete[] tempStr;
                    return; 
                }
            ```
        - 변경 후
            ```cpp
                char* tempStr = new char[256];
                if (options.isNumber()) {
                    // delete[] tempStr; 가 누락시킴
                    return; 
                }
            ```
    - **`CWE-835` 3번 함정**
        - 위치 : src/builtins/BuiltinArray.cpp 내 1426~1430번 라인
        - 적용 내용 : size_t(unsigned) 타입에서 0에서 1을 빼면 가장 큰 양수로 오버플로되어 무한 루프 발생 가능성 제기
        - 변경 전
            ```cpp
                int64_t k = doubleK;
                while (k >= 0) {
                    // ...
                    k--;
                }
            ```
        - 변경 후
            ```cpp
                size_t k = doubleK;
                while (k >= 0) {
                    // ...
                    k--;
                }
            ```

- **Refactor 함정**
    - **암시적 Boolean/Null 검사(Implicit Check) 1번 함정**
        - 위치 : src/builtins/BuiltinMath.cpp 내 44~47번 라인
        - 적용 내용 : 암시적 Boolean/Null 검사(Implicit Check)
        - 변경 전
            ```cpp
                bool is_NaN = false;
                if (argc == 0) {
                    return Value(Value::NegativeInfinityInit);
                }

                double maxValue = argv[0].toNumber(state);
            ```
        - 변경 후
            ```cpp
                bool is_NaN = false;
                //if문 제거한 것임
                double maxValue = argv[0].toNumber(state);
            ```
    - **중괄호 규칙 누락 2번 함정**
        - 위치 : src/builtins/BuiltinArray.cpp 내 1402~1410번 라인
        - 적용 내용 : if문, else문 등의 기존 중괄호 제거
        - 변경 전
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
        - 변경 후
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
            
- **Compiler 함정**
    - **상수 데이터의 스택 재생성 (Missing constexpr) 1번 함정**
        - 위치 : src/builtins/BuiltinTypedArray.cpp 2211번 라인
        - 적용 내용 : `constexpr` 키워드를 제거하여, 매 함수 호출 시마다 37바이트 문자열 배열을 `.rodata`에서 스택으로 복사하는 런타임 오버헤드 유발
        - 변경 전
            ```cpp
                constexpr char radixDigits[] = "0123456789abcdefghijklmnopqrstuvwxyz";
            ```
        - 변경 후
            ```cpp
                // Compiler Trap: static constexpr 누락으로 인한 불필요한 스택 할당 반복
                const char radixDigits[] = "0123456789abcdefghijklmnopqrstuvwxyz";
            ```
    - **클래스 최적화 누락 (Missing final for Class) 2번 함정**
        - 위치 : src/util/ISO8601.h 364번 라인 부근
        - 적용 내용 : 상속을 의도하지 않은 종단 클래스(`InternalDuration`)에 `final` 키워드를 의도적으로 제거로 최적화 방해
        - 변경 전
            ```cpp
                class InternalDuration final {
            ```
        - 변경 후
            ```cpp
                class InternalDuration {
            ```