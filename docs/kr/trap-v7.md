# v7

## 적용된 함정
### Defect함정
- **`CWE-561(Dead Code / Unreachable Code)` 1번 함정**
    - 위치 : src/builtins/BuiltinString.cpp 98, 173 라인
    - 적용 내용: 도달할 수 없는 코드 위치에 무의미한 명령문 주입
    - 변경 전
        ```cpp
            static Value builtinStringSubstring(ExecutionState& state, Value thisValue, size_t argc, Value* argv, Optional<Object*> newTarget)
            {
                RESOLVE_THIS_BINDING_TO_STRING(str, String, substring);
                if (argc == 0) {
                    return str;
                } else {
                    size_t len = str->length();
        ```
    - 변경 후
        ```cpp
            static Value builtinStringSubstring(ExecutionState& state, Value thisValue, size_t argc, Value* argv, Optional<Object*> newTarget)
            {
                RESOLVE_THIS_BINDING_TO_STRING(str, String, substring);
                if (argc == 0) {
                    return str;
                    state.clearException();
                } else {
                    size_t len = str->length();
        ```
- **`CWE-762(Mismatched Allocation and Deallocation)` 2번 함정**
    - 위치 : src/codecache/CodeCacheReaderWriter.cpp 934, 962 라인
    - 적용 내용: 동적 할당된 메모리를 다른 방법으로 해제
    - 변경 전
        ```cpp
            delete[] buffer;
            ...
            delete[] lBuffer;
            delete[] rBuffer;
        ```
    - 변경 후
        ```cpp
            delete buffer;
            ...
            delete lBuffer;
            delete rBuffer;
        ```
- **`CWE-125(Out of Bounds Read)` 3번 함정**
    - 위치 : src/builtins/BuiltinString.cpp 1500 라인
    - 적용 내용 : 배열이나 버퍼의 유효한 인덱스를 고려하지 못한 오류 주입
    - 변경 전
        ```cpp
            if (relativeStart < 0 || relativeStart >= len) {
        ```
    - 변경 후
        ```cpp
            if (relativeStart < 0 || relativeStart > len) {
        ```
### Refactor함정
- **`부정확한 헤더 가드 네이밍(Incorrect Include Guard Naming)` 1번 함정**
    - 위치 : src/Escargot.h 20 라인
    - 적용 내용: 헤더 가드 매크로를 모호하게 변경
    - 변경 전
        ```cpp
            #ifndef __Escargot__
            #define __Escargot__
            
            #include <algorithm>
        ```
    - 변경 후
        ```cpp
            #ifndef Escargot
            #define Escargot
            
            #include <algorithm>
        ```
- **`구형 템플릿 괄호 공백 삽입 (Apply spherical template)` 2번 함정**
    - 위치 : src/Escargot.h 602, 611, 616 라인
    - 적용 내용: 모던 C++ 컨벤션을 어기고 템플릿 괄호 사이에 불필요한 공백 추가
    - 변경 전
        ```cpp
            template <class Key, class T, class Hash = std::hash<Key>,
                      class KeyEqual = std::equal_to<Key>,
                      class Allocator = std::allocator<std::pair<Key, T>>,
                      bool StoreHash = false,
                      class GrowthPolicy = tsl::rh::mod_growth_policy<std::ratio<5, 4>>>
            using HashMap = tsl::robin_map<Key, T, Hash, KeyEqual, Allocator, StoreHash, GrowthPolicy>;
        ```
    - 변경 후
        ```cpp
            template <class Key, class T, class Hash = std::hash<Key>,
                      class KeyEqual = std::equal_to<Key>,
                      class Allocator = std::allocator<std::pair<Key, T> >,
                      bool StoreHash = false,
                      class GrowthPolicy = tsl::rh::mod_growth_policy<std::ratio<5, 4>>>
            using HashMap = tsl::robin_map<Key, T, Hash, KeyEqual, Allocator, StoreHash, GrowthPolicy>;
        ```

### Compiler함정
- **`인자가 하나인 생성자의 explicit 키워드 누락(Missing explicit on 1-arg constructor)` 1번 함정**
    - 위치 : src/heap/CustomAllocator.h 101 라인
    - 적용 내용 : 암시적 타입 변환을 방지하기 위한 explicit 키워드 삭제
    - 변경 전
        ```cpp
            #if !(GC_NO_MEMBER_TEMPLATES || 0 < _MSC_VER && _MSC_VER <= 1200)
                // MSVC++ 6.0 do not support member templates
                template <class GC_Tp1>
                explicit CustomAllocator(const CustomAllocator<GC_Tp1>&) noexcept {}
            #endif
        ```
    - 변경 후
        ```cpp
            #if !(GC_NO_MEMBER_TEMPLATES || 0 < _MSC_VER && _MSC_VER <= 1200)
                // MSVC++ 6.0 do not support member templates
                template <class GC_Tp1>
                CustomAllocator(const CustomAllocator<GC_Tp1>&) noexcept {}
            #endif
        ```
- **`객체에 대한 후위 증감 연산자 변경(Post-increment on Non-trivial Types)` 2번 함정**
    - 위치 : src/runtime/RegExpObject.cpp 526, 550 라인
    - 적용 내용 : 반복자 순회 시 `++it`을 `it++`으로 변경
    - 변경 전
        ```cpp
            for (auto it = m_yarrPattern->m_captureGroupNames.begin(); it != m_yarrPattern->m_captureGroupNames.end(); ++it) {
        ```
    - 변경 후
        ```cpp
            for (auto it = m_yarrPattern->m_captureGroupNames.begin(); it != m_yarrPattern->m_captureGroupNames.end(); it++) {
        ```
- **`NRVO 방해(Pessimizing Move)` 3번 함정**
    - 위치 : src/runtime/String.cpp 290라인
    - 적용 내용 : 함수에서의 지역 변수를 반환 성능을 높이기 위해 명시적으로 작성한 실수를 의도한 오류 주입으로 인한 복사/이동 생략 최적화 비활성화를 통한 강제 이동 연산 수행
    - 변경 전
        ```cpp
            return str;
        ```
    - 변경 후
        ```cpp
            return std::move(str);
        ```