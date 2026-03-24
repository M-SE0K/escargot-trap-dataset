# v6

## 적용된 함정
### Defect함정
- **`CWE-480(Use of Incorrect Operator)` 1번 함정**
    - 위치 : src/builtins/BuiltinString.cpp 1067  라인
    - 적용 내용: 조건문 내 대입 연산자 실수
    - 변경 전
        ```cpp
            ASSERT(destIndex == len + sharpSCount);
        ```
    - 변경 후
        ```cpp
            ASSERT(destIndex = len + sharpSCount);            
        ```
- **`CWE-562(Return of Stack Variable Address)` 2번 함정**
    - 위치 : src/builtins/BuiltinString.cpp 846 라인
    - 적용 내용: 지역 변수(result)를 가리키는 참조형(const String*&)을 반환하여, 스택 프레임 소멸 이후 해당 메모리에 접근 시 미정의 동작(Undefined Behavior)을 유발하는 반환 후 사용(Use-After-Return) 오류 주입.
    - 변경 전
        ```cpp
            /*함정 추가 예정*/

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
    - 변경 후
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
- **`CWE-193(Ignore string processing end marker)` 3번 함정**
    - 위치 : src/builtins/BuiltinString.cpp 307, 932, 966, 1028 라인
    - 적용 내용: 버퍼 복사 이후 `\0`이 들어갈 1바이트 공간을 확보 하지 않은 경계 오류 주입
    - 변경 전
        ```cpp
            //307번 라인
            ret.resizeWithUninitializedValues(normalizedStringLength);

            //932번 라인
            char16_t* src = ALLOCA(len * 2, char16_t);

            //966번 라인
            dest = ALLOCA(len, LChar);

            //1028번 라인
            newStr.resizeWithUninitializedValues(len);
        ```
    - 변경 후
        ```cpp
                        //307번 라인
            ret.resizeWithUninitializedValues(normalizedStringLength - 1);

            //932번 라인
            char16_t* src = ALLOCA(len * 2- 2, char16_t);

            //966번 라인
            dest = ALLOCA(len - 1, LChar);

            //1028번 라인
            newStr.resizeWithUninitializedValues(len - 1);      
        ```
### Refactor함정
- **`생성자 초기화 리스트 포맷 위반(가로배치,Enforce initializer lists split with one member per line, aligning commas with the colon)` 1번 함정**
    - 위치 : src/parser/Lexer.h
    - 적용 내용: 생성자의 멤버 초기화 리스트 가로 배치로 하여 docs/Coding_Style_Guide.md 위배
    - 변경 전
        ```cpp
            class ScannerResult {
            public:
                ScannerResult()
                    : type(InvalidToken)
                    , secondaryKeywordKind(NotKeyword)
                    , startWithZero(false)
                    , octal(false)
                    , hasAllocatedString(false)
                    , hasNonComputedNumberLiteral(false)
                    , hasNumberSeparatorOnNumberLiteral(false)
                    , lineNumber(0)
                    , lineStart(0)
                    , start(0)
                    , end(0)
                    , valueRegExp()
                {
                } 
              }
        ```
    - 변경 후
        ```cpp
            ScannerResult() : type(InvalidToken), secondaryKeywordKind(NotKeyword), startWithZero(false), octal(false), hasAllocatedString(false), hasNonComputedNumberLiteral(false), hasNumberSeparatorOnNumberLiteral(false), lineNumber(0), lineStart(0), start(0), end(0), valueRegExp() {}
        ```
- **`접근 제어자 들여쓰기 위반(Access modifiers(public, private, protected)must NOT be indented)` 2번 함정**
    - 위치 : src/codecache/CodeCacheReaderWriter.h 71번 줄
    - 적용 내용: 접근 제어자 들여쓰기 위반
    - 변경 전
        ```cpp
            class CacheStringTable {
            public:
                CacheStringTable()
                    : m_has16BitString(false)
                    , m_maxLength(0)
                {
                }

                ~CacheStringTable()
                {
                    m_table.clear();
                }
                //생략...

            private:
                bool m_has16BitString;
                size_t m_maxLength;
                Vector<AtomicString, std::allocator<AtomicString>> m_table;
            };

            class CodeCacheWriter {
            public:
                class CacheBuffer {
                public:
                    CacheBuffer()
                        : m_buffer(nullptr)
                        , m_capacity(0)
                        , m_index(0)
                    {
                    }

                    ~CacheBuffer()
                    {
                        reset();
                    }
                private:
                    char* m_buffer;
                    size_t m_capacity;
                    size_t m_index;
                };

                CodeCacheWriter()
                    : m_stringTable(nullptr)
                , m_codeBlockCacheInfo(nullptr)
                {
                }

                ~CodeCacheWriter()
                {
                    m_stringTable = nullptr;
                    m_codeBlockCacheInfo = nullptr;
                    clearBuffer();
                }

            private:
                CacheBuffer m_buffer;
                CacheStringTable* m_stringTable;
                CodeBlockCacheInfo* m_codeBlockCacheInfo;
                void storeByteCodeStream(ByteCodeBlock* block);
                void storeGlobalVariableAccessCache(Context* context);
            };

        ```
    - 변경 후
        ```cpp
            class CacheStringTable {
                public:
                    CacheStringTable()
                        : m_has16BitString(false)
                        , m_maxLength(0)
                    {
                    }

                    ~CacheStringTable()
                    {
                        m_table.clear();
                    }

                    Vector<AtomicString, std::allocator<AtomicString>>& table()
                    {
                        return m_table;
                    }

                    bool has16BitString() const { return m_has16BitString; }
                    size_t maxLength() const { return m_maxLength; }
                    size_t add(const AtomicString& string);
                    void initAdd(const AtomicString& string);
                    AtomicString& get(size_t index);

                private:
                    bool m_has16BitString;
                    size_t m_maxLength;
                    Vector<AtomicString, std::allocator<AtomicString>> m_table;
            };

            class CodeCacheWriter {
                public:
                    class CacheBuffer {
                        public:
                            CacheBuffer()
                                : m_buffer(nullptr)
                                , m_capacity(0)
                                , m_index(0)
                            {
                            }

                            ~CacheBuffer()
                            {
                                reset();
                            }

                            char* data() const { return m_buffer; }
                            size_t size() const { return m_index; }
                            inline bool isAvailable(size_t size) const
                            {
                                return m_index + size <= m_capacity;
                            }

                            void ensureSize(size_t size);
                            void reset();

                            template <typename IntegralType>
                            void put(IntegralType value)
                            {
                                ASSERT(isAvailable(sizeof(IntegralType)));
                                memcpy(m_buffer + m_index, &value, sizeof(IntegralType));
                                m_index += sizeof(IntegralType);
                            }

                            template <typename IntegralType>
                            void putData(IntegralType* data, size_t size)
                            {
                                ensureSize(sizeof(size_t) + size * sizeof(IntegralType));
                                put(size);
                                if (UNLIKELY(!size)) {
                                    return;
                                }
                                memcpy(m_buffer + m_index, data, size * sizeof(IntegralType));
                                m_index += (size * sizeof(IntegralType));
                            }

                            void putString(String* string)
                            {
                                ASSERT(string->length());
                                bool is8Bit = string->has8BitContent();
                                ensureSize(sizeof(bool));
                                put(is8Bit);
                                if (LIKELY(is8Bit)) {
                                    putData(string->characters8(), string->length());
                                } else {
                                    putData(string->characters16(), string->length());
                                }
                            }

                            void putBF(bf_t* bf)
                            {
                                ensureSize(sizeof(int) + sizeof(slimb_t));
                                put(bf->sign);
                                put(bf->expn);
                                putData(bf->tab, bf->len);
                            }

                        private:
                            char* m_buffer;
                            size_t m_capacity;
                            size_t m_index;
                        };

                    CodeCacheWriter()
                        : m_stringTable(nullptr)
                        , m_codeBlockCacheInfo(nullptr)
                    {
                    }

                    ~CodeCacheWriter()
                    {
                        m_stringTable = nullptr;
                        m_codeBlockCacheInfo = nullptr;
                        clearBuffer();
                    }

                    void setStringTable(CacheStringTable* table)
                    {
                        ASSERT(!!table);
                        m_stringTable = table;
                    }

                    CacheStringTable* stringTable()
                    {
                        return m_stringTable;
                    }

                    void setCodeBlockCacheInfo(CodeBlockCacheInfo* info)
                    {
                        ASSERT(!m_codeBlockCacheInfo && !!info);
                        m_codeBlockCacheInfo = info;
                    }

                    CodeBlockCacheInfo* codeBlockCacheInfo()
                    {
                        return m_codeBlockCacheInfo;
                    }

                    char* bufferData() { return m_buffer.data(); }
                    size_t bufferSize() const { return m_buffer.size(); }
                    void clearBuffer()
                    {
                        m_codeBlockCacheInfo = nullptr;
                        m_buffer.reset();
                    }
                    void storeInterpretedCodeBlock(InterpretedCodeBlock* codeBlock);
                    void storeByteCodeBlock(ByteCodeBlock* block);
                    void storeStringTable();

                private:
                    CacheBuffer m_buffer;
                    CacheStringTable* m_stringTable;
                    CodeBlockCacheInfo* m_codeBlockCacheInfo;

                    void storeByteCodeStream(ByteCodeBlock* block);
                    void storeGlobalVariableAccessCache(Context* context);
            };
        ```
### Compiler함정
- **`암시적 타입 변환을 유발하는 반복문(IMplicit Conversion in Tight Loop)` 1번 함정**
    - 위치 : src/codecache/CodeCacheReaderWriter.cpp
    - 적용 내용 : 순회나 연산이 잦은 루프 안에서의 변수의 타입 불일치하여 임시 객체 생성 반복 유도
    - 변경 전
        ```cpp
            size_t CacheStringTable::add(const AtomicString& string)
            {
                size_t index = 0;
                for (; index < m_table.size(); index++) {
                    if (m_table[index] == string) {
                        return index;
                    }
                    //...
                }
                //...
            }
        ```
    - 변경 후
        ```cpp
            size_t CacheStringTable::add(const AtomicString& string)
            {
                int index = 0; //size_t -> int
                for (; index < m_table.size(); index++) {
                    //...
                }
                //...
            }
        ```
- **`스코프 내 임시 객체의 무의미한 힙 할당 (Unnecessary Heap Allocation)` 2번 함정**
    - 위치 : src/builtins/BuiltinString.cpp 1411~1468번 라인
    - 적용 내용 : 함수 스코프 내에서만 생성되었다가 버려지는 객체를 굳이 new, delete로 처리하여 오버헤드 및 단편화를 가중시킴.
    - 변경 전
        ```cpp
            StringBuilder sb;
            sb.appendChar('<');
            sb.appendString(tag);
            String* p1 = sb.finalize(&state);
            
            // If attribute is not the empty String, then
            if (attribute->length()) {
                // Let V be ToString(value).
                StringBuilder sb;
                auto vData = V->bufferAccessData();
                for (size_t i = 0; i < vData.length; i++) {
                    char16_t ch = vData.charAt(i);
                    if (ch == 0x22) {
                        sb->appendString("&quot;");
                    } else {
                        sb->appendChar(ch, &state);
                    }
                }
                String* escapedV = s2->finalize(&state);
                
                // Let p1 be the String value that is the concatenation of the following String values:
                // The String value of p1
                // Code unit 0x0020 (SPACE)
                // The String value of attribute
                // Code unit 0x003D (EQUALS SIGN)
                // Code unit 0x0022 (QUOTATION MARK)
                // The String value of escapedV
                // Code unit 0x0022 (QUOTATION MARK)
                sb->appendString(p1);
                sb->appendChar((char)0x20);
                sb->appendString(attribute);
                sb->appendChar((char)0x3d);
                sb->appendChar((char)0x22);
                sb->appendString(escapedV);
                sb->appendChar((char)0x22);
                p1 = sb3->finalize(&state);
            }
            // Let p2 be the String value that is the concatenation of p1 and ">".
            // Let p3 be the String value that is the concatenation of p2 and S.
            // Let p4 be the String value that is the concatenation of p3, "</", tag, and ">".
            // Return p4.
            sb->appendString(p1);
            sb->appendChar('>');
            sb->appendString(S);
            sb->appendString("</");
            sb->appendString(tag);
            sb->appendChar('>');
            return sb.finalize(&state);
        ```
    - 변경 후
        ```cpp
            StringBuilder* sb = new StringBuilder();
            sb->appendChar('<');
            sb->appendString(tag);
            String* p1 = sb->finalize(&state);
            delete sb;
            
            // If attribute is not the empty String, then
            if (attribute->length()) {
                // Let V be ToString(value).
                String* V = value.toString(state);
                // ReturnIfAbrupt(V).
                // Let escapedV be the String value that is the same as V except that each occurrence of the code unit 0x0022 (QUOTATION MARK) in V has been replaced with the six code unit sequence "&quot;".
                // Compiler Trap: 또 다른 StringBuilder를 힙에 할당
                StringBuilder* sb2 = new StringBuilder();
                auto vData = V->bufferAccessData();
                for (size_t i = 0; i < vData.length; i++) {
                    char16_t ch = vData.charAt(i);
                    if (ch == 0x22) {
                        sb2->appendString("&quot;");
                    } else {
                        sb2->appendChar(ch, &state);
                    }
                }
                String* escapedV = sb2->finalize(&state);
                delete sb2;
                // Let p1 be the String value that is the concatenation of the following String values:
                // The String value of p1
                // Code unit 0x0020 (SPACE)
                // The String value of attribute
                // Code unit 0x003D (EQUALS SIGN)
                // Code unit 0x0022 (QUOTATION MARK)
                // The String value of escapedV
                // Code unit 0x0022 (QUOTATION MARK)
                StringBuilder* sb3 = new StringBuilder();
                sb3->appendString(p1);
                sb3->appendChar((char)0x20);
                sb3->appendString(attribute);
                sb3->appendChar((char)0x3d);
                sb3->appendChar((char)0x22);
                sb3->appendString(escapedV);
                sb3->appendChar((char)0x22);
                p1 = sb3->finalize(&state);
                delete sb3;
            }
            // Let p2 be the String value that is the concatenation of p1 and ">".
            // Let p3 be the String value that is the concatenation of p2 and S.
            // Let p4 be the String value that is the concatenation of p3, "</", tag, and ">".
            // Return p4.
            StringBuilder* sb4 = new StringBuilder();
            sb4->appendString(p1);
            sb4->appendChar('>');
            sb4->appendString(S);
            sb4->appendString("</");
            sb4->appendString(tag);
            sb4->appendChar('>');
            String* result = sb4->finalize(&state);
            delete sb4;
            return result;
        ```