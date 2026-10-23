<!-- ko-translation: enabled -->
# Oasys Synthesis Message Handlers
# Oasys 합성 메시지 핸들러

Per-message-ID fix instructions for **Oasys synthesis messages** in Vivado.
Vivado의 **Oasys 합성 메시지**에 대한 메시지 ID별 수정 지침.
Messages appear as `[Synth 8-XXX]` in Vivado logs.
메시지는 Vivado 로그에 '[Synth 8-XXX]'로 표시됩니다.

> **Mandatory**: For every message, read the RTL source at the reported file:line using
> **필수**: 모든 메시지에 대해 다음을 사용하여 보고된 파일:줄에서 RTL 소스를 읽습니다.
> `read_file` before generating a fix.
> 수정 사항을 생성하기 전 `read_file`.
> Never fabricate code.
> 코드를 조작하지 마십시오.

---

## How Messages Appear in Vivado Logs
## Vivado 로그에 메시지가 나타나는 방식

```
ERROR: [Synth 8-36] 'my_signal' is not declared [/path/to/file.v:42]
WARNING: [Synth 8-327] inferring latch for variable 'state_reg' [/path/to/file.v:100]
CRITICAL WARNING: [Synth 8-5796] RAM (ram_inst) has conflicting writes [/path/to/file.v:55]
```

The parser extracts: severity, ID (36/327/5796), message text, file path, line number.
파서는 심각도, ID(36/327/5796), 메시지 텍스트, 파일 경로, 줄 번호를 추출합니다.

---

## ID Coverage Summary
## ID 적용 범위 요약

| Severity<br>심각도 | Count<br>개수 | ID Range<br>ID 범위 |
|----------|-------|----------|
| **Critical Warnings**<br>**중요한 경고** | 8 | 4442, 5796, 5967, 6030, 6781, 6790, 6858, 13329 |
| **Errors**<br>**오류** | 457 | 27-13331 |
| **Warnings**<br>**경고** | 86 | 85-13374 |
| **Total**<br>**합계** | **551** | |

---

## Dispatch Table - Critical Warnings (8 IDs)
## 디스패치 테이블 - 심각한 경고(ID 8개)

| ID | Message<br>메시지 | Fix Strategy<br>전략 수정 |
|----|---------|--------------|
| 4442 | BlackBox module !%1! has unconnected pin !%2!<br>블랙박스 모듈!%1! 연결되지 않은 핀이 있습니다!%2! | Connect the pin or mark as intentionally unconnected<br>핀을 연결하거나 의도적으로 연결되지 않은 것으로 표시 |
| 5796 | RAM (!%1!) has conflicting writes using multiple ports with same address<br>RAM(!%1!)에 동일한 주소를 가진 여러 포트를 사용하여 쓰기 충돌이 있습니다. | Ensure ports write to different addresses or add arbitration logic<br>포트가 다른 주소에 쓰는지 확인하거나 중재 논리를 추가하세요. |
| 5967 | Detected mixed tristate reset assignment for register<br>레지스터에 대한 혼합 삼상태 재설정 할당이 감지되었습니다. | Split vector; keep tristate assignments separate<br>분할 벡터; 삼국 할당을 별도로 유지 |
| 6030 | Inout pin read/written without tristate logic<br>삼상태 논리 없이 입력 핀 읽기/쓰기 | Use explicit direction or tristate buffer<br>명시적 방향 또는 삼상태 버퍼 사용 |
| 6781 | Nested interface items may not be available<br>중첩된 인터페이스 항목을 사용하지 못할 수 있습니다. | Pass interface items explicitly; flatten interface usage<br>인터페이스 항목을 명시적으로 전달합니다. 평면화 인터페이스 사용 |
| 6790 | Constant variable cannot be reassigned<br>상수 변수는 재할당할 수 없습니다. | Remove assignment to constant or change to variable<br>상수 할당 제거 또는 변수 변경 |
| 6858 | Multi-driven net connected to constant driver<br>상수 드라이버에 연결된 다중 구동 넷 | Remove duplicate drivers; preserve intended driver only<br>중복 드라이버를 제거하십시오. 의도한 드라이버만 보존 |
| 13329 | Null range not supported<br>Null 범위는 지원되지 않습니다. | Fix range bounds; ensure start <= end (or proper direction)<br>범위 경계를 수정합니다. 시작 <= 끝(또는 올바른 방향)을 확인하세요. |

---

## Dispatch Table - Errors (457 IDs)
## 디스패치 테이블 - 오류(ID 457개)

### Range 27-99 (17 IDs)
### 범위 27-99(ID 17개)

| ID | Message<br>메시지 | Fix Strategy<br>전략 수정 |
|----|---------|--------------|
| 27 | !%1! not supported<br>!%1! 지원되지 않음 | Use synthesizable alternative construct<br>합성 가능한 대체 구성 사용 |
| 34 | '!%1!' expects !%2! arguments<br>'!%1!' !%2!을(를) 기대합니다! 인수 | Add missing arguments to function/task call<br>함수/작업 호출에 누락된 인수 추가 |
| 35 | '!%1!' is not a constant<br>'!%1!' 상수가 아니다 | Replace with parameter/localparam<br>매개변수/localparam으로 교체 |
| 36 | '!%1!' is not declared<br>'!%1!' 선언되지 않았습니다 | Add missing declaration for signal/variable<br>신호/변수에 대한 누락된 선언 추가 |
| 78 | a value must be associated with generic !%1!<br>값은 일반 !%1!과 연결되어야 합니다. | Provide value for generic parameter<br>일반 매개변수에 대한 값 제공 |
| 83 | all members of packed union must have same size<br>Packed Union의 모든 구성원은 동일한 크기를 가져야 합니다. | Pad union members to equal width<br>유니온 멤버를 너비와 동일하게 패드 |
| 91 | ambiguous clock in event control<br>이벤트 제어의 모호한 시계 | Use single clock edge; restructure sensitivity<br>단일 클록 에지를 사용하십시오. 민감도 재구성 |
| 99 | array type required<br>배열 유형이 필요합니다. | Change to array type or fix indexing<br>배열 유형으로 변경하거나 인덱싱 수정 |

### Range 120-285 (21 IDs)
### 범위 120-285(ID 21개)

| ID | Message<br>메시지 | Fix Strategy<br>전략 수정 |
|----|---------|--------------|
| 120 | call depth limit exceeded<br>통화 심도 제한을 초과했습니다. | Reduce recursion depth; flatten hierarchy<br>재귀 깊이를 줄입니다. 계층구조 평탄화 |
| 196 | conditional expression could not be resolved to a constant<br>조건식을 상수로 확인할 수 없습니다. | Replace with constant condition<br>상수 조건으로 대체 |
| 206 | constant expression required<br>상수 표현이 필요함 | Replace variable with constant/parameter<br>변수를 상수/매개변수로 바꾸기 |
| 211 | could not evaluate expression<br>표현을 평가할 수 없습니다 | Simplify or make constant<br>단순화하거나 일정하게 만드세요. |
| 248 | direction of slice does not match direction of prefix<br>슬라이스 방향이 접두사 방향과 일치하지 않습니다. | Fix part-select direction [MSB:LSB]<br>부품 선택 방향 수정 [MSB:LSB] |
| 250 | disable statement must apply to surrounding named block<br>비활성화 문은 주변의 명명된 블록에 적용되어야 합니다. | Add block label or fix disable target<br>블록 레이블 추가 또는 비활성화 대상 수정 |
| 258 | duplicate association<br>중복 연관 | Remove duplicate port/parameter association<br>중복된 포트/매개변수 연결 제거 |
| 274 | error: !%1!<br>오류: !%1! | Fix syntax error at reported location<br>보고된 위치의 구문 오류 수정 |
| 277 | exponentiation is not supported<br>지수화는 지원되지 않습니다. | Use shift operations or lookup table<br>교대 연산 또는 조회 테이블 사용 |
| 279 | expression in generate should resolve to constant<br>생성의 표현식은 상수로 해석되어야 합니다. | Use parameter for generate index<br>인덱스 생성을 위해 매개변수 사용 |
| 280 | expression must be constant<br>표현식은 일정해야 합니다. | Replace variable with constant<br>변수를 상수로 바꾸기 |
| 281 | expression must be of a packed type<br>표현식은 압축된 유형이어야 합니다. | Add `packed` to type definition<br>유형 정의에 `packed` 추가 |
| 285 | failed synthesizing module<br>실패한 합성 모듈 | Fix all errors in the module<br>모듈의 모든 오류 수정 |

### Range 316-439 (25 IDs)
### 범위 316-439(ID 25개)

| ID | Message<br>메시지 | Fix Strategy<br>전략 수정 |
|----|---------|--------------|
| 316 | illegal module recursion detected<br>불법 모듈 재귀가 감지되었습니다. | Remove circular instantiation<br>순환 인스턴스화 제거 |
| 317 | illegal recursive instantiation<br>불법 재귀 인스턴스화 | Flatten hierarchy; remove recursion<br>계층 구조를 단순화합니다. 재귀 제거 |
| 318 | illegal unconstrained array<br>제한되지 않은 잘못된 배열 | Add range/size constraint to array<br>배열에 범위/크기 제약 조건 추가 |
| 349 | instance passed wrong connection count<br>인스턴스가 잘못된 연결 수를 전달했습니다. | Check module port list; fix connection count<br>모듈 포트 목록을 확인하세요. 연결 수 수정 |
| 354 | integer overflow computing array size<br>정수 오버플로 컴퓨팅 배열 크기 | Reduce array size; use smaller indices<br>배열 크기를 줄입니다. 더 작은 인덱스 사용 |
| 366 | invalid definition of precompiled function<br>미리 컴파일된 함수의 잘못된 정의 | Fix function signature<br>함수 서명 수정 |
| 403 | loop limit exceeded<br>루프 제한을 초과했습니다. | Add iteration limit or restructure loop<br>반복 제한 추가 또는 루프 재구성 |
| 421 | mismatched array sizes in assignment<br>할당 시 배열 크기가 일치하지 않습니다. | Match array dimensions on both sides<br>양쪽의 배열 차원 일치 |
| 422 | missing association<br>연관성 누락 | Add missing port/parameter association<br>누락된 포트/매개변수 연결 추가 |
| 426 | missing choices in case statement<br>Case 문에 선택 항목이 누락되었습니다. | Add missing case items or default<br>누락된 케이스 항목 또는 기본값 추가 |
| 428 | missing elements in aggregate<br>총체적으로 누락된 요소 | Add missing array/struct elements<br>누락된 배열/구조체 요소 추가 |
| 434 | mixed level/edge triggered events<br>혼합 레벨/에지 트리거 이벤트 | Separate into combo and sequential blocks<br>콤보 및 순차 블록으로 분리 |
| 436 | modport mismatch for interface port<br>인터페이스 포트에 대한 modport 불일치 | Fix modport specification<br>모드포트 사양 수정 |
| 439 | module '!%1!' not found<br>모듈 '!%1!' 찾을 수 없음 | Add missing file to project; fix typo<br>프로젝트에 누락된 파일을 추가합니다. 오타 수정 |

### Range 462-561 (16 IDs)
### 범위 462-561(ID 16개)

| ID | Message<br>메시지 | Fix Strategy<br>전략 수정 |
|----|---------|--------------|
| 462 | no clock signal in event control<br>이벤트 제어에 클럭 신호 없음 | Add clock after `posedge`/`negedge`<br>`posedge`/`negedge` 뒤에 시계 추가 |
| 465 | no constraint on function return<br>함수 반환에 대한 제약 없음 | Add return type constraint<br>반환 유형 제약 조건 추가 |
| 470 | no interface found<br>인터페이스를 찾을 수 없습니다 | Add interface definition or fix reference<br>인터페이스 정의 추가 또는 참조 수정 |
| 476 | no matching instance for defparam<br>defparam과 일치하는 인스턴스가 없습니다. | Fix defparam path; use parameter override<br>defparam 경로를 수정합니다. 매개변수 재정의 사용 |
| 478 | no matching signal for .* connection<br>.* 연결에 대해 일치하는 신호가 없습니다. | Add matching signal declaration<br>일치하는 신호 선언 추가 |
| 485 | no port on instance<br>인스턴스에 포트 없음 | Fix port name or add to module<br>포트 이름 수정 또는 모듈에 추가 |
| 493 | no such design unit<br>그런 디자인 단위가 없습니다 | Add missing file or fix reference<br>누락된 파일 추가 또는 참조 수정 |
| 509 | operands have different lengths<br>피연산자의 길이가 다릅니다 | Resize operands to match<br>일치하도록 피연산자 크기 조정 |
| 517 | overlapping choice in case<br>경우에 따라 중복 선택 | Fix case values to be unique<br>케이스 값을 고유하게 수정하세요. |
| 521 | parameter could not resolve to constant<br>매개변수를 상수로 확인할 수 없습니다. | Use compile-time constant value<br>컴파일 타임 상수 값 사용 |
| 522 | parsing error<br>구문 분석 오류 | Fix syntax error at reported location<br>보고된 위치의 구문 오류 수정 |
| 523 | part-select does not match declaration<br>부분 선택이 선언과 일치하지 않습니다 | Fix part-select to match signal direction<br>신호 방향과 일치하도록 부품 선택 수정 |
| 524 | part-select out of range<br>범위를 벗어난 부분 선택 | Adjust part-select bounds<br>부분 선택 범위 조정 |
| 549 | port width mismatch<br>포트 너비 불일치 | Resize signal or add explicit slice<br>신호 크기 조정 또는 명시적 슬라이스 추가 |
| 550 | port width mismatch in instance array<br>인스턴스 배열의 포트 너비 불일치 | Match signal width to (port_width x instance_count)<br>신호 너비를 (port_width x instance_count)에 일치시킵니다. |
| 561 | range expression not constant<br>범위 표현식이 상수가 아님 | Use parameter for range bounds<br>범위 경계에 매개변수 사용 |

### Range 647-734 (12 IDs)
### 범위 647-734(ID 12개)

| ID | Message<br>메시지 | Fix Strategy<br>전략 수정 |
|----|---------|--------------|
| 647 | too many elements in subexpression<br>하위 표현식에 요소가 너무 많습니다. | Split complex expression<br>분할 복합 표현식 |
| 650 | too many ports for instance<br>예를 들어 포트가 너무 많습니다. | Remove excess port connections<br>과도한 포트 연결 제거 |
| 658 | type mismatch for port<br>포트 유형 불일치 | Fix port/signal type compatibility<br>포트/신호 유형 호환성 수정 |
| 659 | type mismatch<br>유형 불일치 | Add cast or change declaration<br>캐스트 추가 또는 선언 변경 |
| 660 | unable to resolve '!%1!'<br>'!%1!'을(를) 확인할 수 없습니다. | Search for correct identifier; fix typo<br>올바른 식별자를 검색하세요. 오타 수정 |
| 685 | variable should not be in output port<br>변수는 출력 포트에 있어서는 안 됩니다. | Change to net type or use procedural block<br>Net 유형으로 변경하거나 절차적 블록을 사용하십시오. |
| 690 | width mismatch in assignment<br>할당의 너비 불일치 | Resize source/target to match<br>일치하도록 소스/타겟 크기 조정 |
| 734 | Invalid parameter<br>잘못된 매개변수 | Fix parameter name or value<br>매개변수 이름 또는 값 수정 |

### Range 837-983 (8 IDs)
### 범위 837-983(ID 8개)

| ID | Message<br>메시지 | Fix Strategy<br>전략 수정 |
|----|---------|--------------|
| 837 | assignment operator only in blocking context<br>차단 컨텍스트에서만 할당 연산자 | Use `=` in combinational, `<=` in sequential<br>조합에는 `=`를 사용하고 순차적으로 `<=`를 사용합니다. |
| 859 | too many parameter overrides<br>매개변수 재정의가 너무 많습니다. | Remove excess parameter overrides<br>초과 매개변수 재정의 제거 |
| 944 | multiple operator definitions match<br>여러 연산자 정의가 일치합니다. | Disambiguate operator usage<br>연산자 사용법 명확화 |
| 981 | statement illegal in always_comb/latch<br>Always_comb/latch의 문이 잘못되었습니다. | Move statement outside combinational block<br>조합 블록 외부로 명령문 이동 |
| 982 | statement illegal in always_comb/latch/ff<br>Always_comb/latch/ff의 문이 잘못되었습니다. | Use appropriate always block type<br>적절한 항상 블록 유형을 사용하십시오. |
| 983 | statement illegal in compilation scope<br>컴파일 범위에서 잘못된 명령문 | Move to appropriate scope<br>적절한 범위로 이동 |

### Range 1067-1958 (21 IDs)
### 범위 1067-1958(ID 21개)

| ID | Message<br>메시지 | Fix Strategy<br>전략 수정 |
|----|---------|--------------|
| 1067 | statement is illegal here<br>여기서 진술은 불법입니다 | Move statement to appropriate location<br>명령문을 적절한 위치로 이동 |
| 1101 | .* token can only appear once<br>.* 토큰은 한 번만 나타날 수 있습니다. | Remove duplicate .* in port list<br>포트 목록에서 중복된 .* 제거 |
| 1560 | actual of formal must be correct type<br>실제와 형식이 올바른 유형이어야 합니다. | Fix type of actual parameter<br>실제 매개변수의 유형 수정 |
| 1561 | actual and formal for ref must be equivalent<br>참조에 대한 실제 및 공식은 동일해야 합니다. | Match types exactly<br>유형을 정확히 일치시키세요. |
| 1576 | aliasing only on net types<br>네트 유형에서만 앨리어싱 | Change to wire/net type<br>와이어/네트형으로 변경 |
| 1621 | assigning type to string requires cast<br>문자열에 유형을 할당하려면 캐스트가 필요합니다. | Add explicit cast<br>명시적 캐스트 추가 |
| 1697 | base value must be 2-16<br>기본 값은 2-16이어야 합니다. | Fix number base<br>숫자 베이스 수정 |
| 1718 | cannot assign type to type<br>유형에 유형을 할당할 수 없습니다. | Fix type compatibility<br>유형 호환성 수정 |
| 1819 | choice must be discrete range<br>선택 항목은 개별 범위여야 합니다. | Use discrete range in case<br>경우에 따라 개별 범위를 사용하십시오. |
| 1859 | constant not allowed here<br>여기서는 상수가 허용되지 않습니다 | Use variable instead<br>대신 변수를 사용하세요 |
| 1875 | default case should appear only once<br>기본 사례는 한 번만 나타나야 합니다. | Remove duplicate default<br>중복된 기본값 제거 |
| 1892 | digits must follow decimal point<br>숫자는 소수점 뒤에 와야 합니다 | Fix real number format<br>실수 형식 수정 |
| 1925 | elements must both be signed or unsigned<br>요소는 서명되거나 서명되지 않아야 합니다. | Match signedness<br>부호 일치 |
| 1956 | error in use clause<br>사용 절에 오류가 있습니다 | Fix VHDL use clause syntax<br>VHDL 사용 절 구문 수정 |
| 1958 | event expressions must be singular type<br>이벤트 표현식은 단일 유형이어야 합니다. | Simplify event expression<br>이벤트 표현 단순화 |

### Range 2043-2948 (54 IDs)
### 범위 2043-2948(ID 54개)

2043, 2056, 2068, 2088, 2105, 2112, 2114, 2119, 2121, 2132, 2148, 2158, 2179, 2195, 2203, 2210, 2212, 2254, 2291, 2340, 2371, 2409, 2415, 2458, 2481, 2496, 2506, 2512, 2543, 2548, 2549, 2551, 2558, 2572, 2577, 2590, 2599, 2666, 2680, 2691, 2716, 2757, 2784, 2798, 2856, 2901, 2902, 2908, 2910, 2913, 2914, 2916, 2918, 2948

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

### Range 3302-4760 (18 IDs)
### 범위 3302-4760(ID 18개)

3302, 3380, 3391, 3492, 3493, 3522, 3892, 3918, 3942, 4169, 4374, 4556, 4564, 4675, 4678, 4687, 4752, 4760

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

### Range 5535-6902 (26 IDs)
### 범위 5535-6902(ID 26개)

5535, 5549, 5551, 5731, 5732, 5743, 5765, 5767, 5826, 5832, 5838, 6036, 6038, 6043, 6046, 6058, 6062, 6073, 6156, 6716, 6735, 6777, 6799, 6828, 6901, 6902

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

### Range 7028-8986 (32 IDs)
### 범위 7028-8986(ID 32개)

7028, 7051, 7054, 7055, 7068, 7136, 7213, 8742, 8863, 8879, 8882, 8891, 8892, 8895, 8896, 8899, 8902, 8910, 8922, 8923, 8925, 8926, 8927, 8936, 8937, 8943, 8945, 8958, 8960, 8983, 8985, 8986

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

### Range 9101-9963 (94 IDs)
### 범위 9101-9963(ID 94개)

9101, 9109, 9111, 9112, 9114, 9116, 9117, 9121, 9123, 9143, 9173, 9174, 9179, 9182, 9185, 9195, 9210, 9212, 9222, 9223, 9224, 9248, 9249, 9250, 9263, 9280, 9287, 9288, 9312, 9315, 9339, 9372, 9380, 9382, 9386, 9397, 9398, 9400, 9404, 9414, 9429, 9448, 9469, 9480, 9481, 9486, 9492, 9493, 9502, 9510, 9563, 9565, 9570, 9571, 9587, 9597, 9606, 9610, 9652, 9653, 9654, 9664, 9696, 9699, 9714, 9721, 9729, 9735, 9736, 9747, 9764, 9767, 9771, 9775, 9842, 9843, 9844, 9871, 9872, 9873, 9875, 9891, 9901, 9910, 9913, 9915, 9921, 9923, 9926, 9932, 9934, 9960, 9963

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

### Range 10021-11899 (156 IDs)
### 범위 10021-11899(ID 156개)

10021, 10033, 10039, 10042, 10043, 10047, 10059, 10093, 10095, 10097, 10140, 10149, 10157, 10180, 10219, 10221, 10275, 10287, 10298, 10300, 10302, 10304, 10307, 10314, 10320, 10327, 10334, 10344, 10366, 10387, 10388, 10438, 10440, 10458, 10478, 10487, 10491, 10495, 10507, 10511, 10515, 10530, 10531, 10551, 10561, 10571, 10592, 10604, 10606, 10615, 10616, 10617, 10631, 10632, 10636, 10656, 10661, 10672, 10681, 10690, 10694, 10703, 10712, 10713, 10714, 10729, 10738, 10752, 10768, 10790, 10791, 10797, 10813, 10826, 10829, 10848, 10872, 10882, 10886, 10890, 10911, 10929, 10940, 10945, 10949, 10976, 10992, 11001, 11055, 11057, 11062, 11065, 11066, 11067, 11077, 11095, 11098, 11112, 11117, 11119, 11121, 11137, 11145, 11152, 11155, 11160, 11161, 11166, 11182, 11187, 11212, 11213, 11214, 11218, 11229, 11234, 11242, 11259, 11261, 11262, 11287, 11290, 11303, 11323, 11324, 11325, 11356, 11365, 11495, 11501, 11534, 11536, 11537, 11558, 11560, 11574, 11584, 11587, 11615, 11635, 11772, 11803, 11815, 11817, 11899

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

### Range 12188-13331 (10 IDs)
### 범위 12188-13331(ID 10개)

12188, 12189, 12523, 12526, 13076, 13099, 13162, 13322, 13324, 13331

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

---

## Dispatch Table - Warnings (86 IDs)
## 디스패치 테이블 - 경고(ID 86개)

### Range 85-693 (23 IDs)
### 범위 85-693(ID 23개)

| ID | Message<br>메시지 | Fix Strategy<br>전략 수정 |
|----|---------|--------------|
| 85 | always block has no event control<br>항상 차단에는 이벤트 제어가 없습니다. | Add `@(*)` or explicit sensitivity list<br>`@(*)` 또는 명시적 민감도 목록 추가 |
| 90 | always_latch did not result in a latch<br>Always_latch로 인해 래치가 발생하지 않았습니다. | Check latch inference conditions<br>래치 추론 조건 확인 |
| 98 | array of blackboxes: assuming ports one bit<br>블랙박스 배열: 포트를 1비트로 가정 | Provide module definition or port widths<br>모듈 정의 또는 포트 너비 제공 |
| 151 | case item is unreachable<br>케이스 항목에 연결할 수 없습니다 | Remove or reorder case item<br>케이스 항목 삭제 또는 재주문 |
| 153 | case item will never be executed<br>케이스 항목은 절대 실행되지 않습니다. | Remove dead case item<br>데드 케이스 항목 제거 |
| 264 | enable of latch is always disabled<br>래치 활성화는 항상 비활성화됩니다. | Check enable condition; may be dead code<br>활성화 상태를 확인하십시오. 죽은 코드일 수도 있습니다 |
| 290 | fork/join as sequential block<br>순차 블록으로 포크/조인 | Review parallel intent<br>병렬 의도 검토 |
| 302 | global signal treated as local<br>전역 신호는 로컬로 처리됩니다. | Review signal scope; make explicit<br>신호 범위를 검토합니다. 명백하게 하다 |
| 308 | ignoring empty port<br>빈 포트 무시 | Remove empty port or add connection<br>빈 포트 제거 또는 연결 추가 |
| 311 | ignoring non-constant in initial<br>초기에 상수가 아닌 것을 무시합니다. | Use constant for initial values<br>초기값에 상수 사용 |
| 312 | ignoring unsynthesizable construct<br>합성 불가능한 구조 무시 | Replace with synthesizable alternative<br>합성 가능한 대안으로 교체 |
| 324 | index out of range<br>범위를 벗어난 인덱스 | Fix array/vector index bounds<br>배열/벡터 인덱스 경계 수정 |
| 327 | inferring latch for variable<br>변수에 대한 래치 추론 | Add `else`/`default` for combinational logic<br>조합 논리에 `else`/`default` 추가 |
| 330 | inout connections inferred for interface<br>인터페이스에 대해 추론된 inout 연결 | Specify modport explicitly<br>modport를 명시적으로 지정 |
| 445 | multiply driven output in instance array<br>인스턴스 배열의 곱셈 구동 출력 | Fix driver conflicts<br>드라이버 충돌 수정 |
| 506 | null port ignored<br>널 포트가 무시됨 | Remove null port or add connection<br>null 포트 제거 또는 연결 추가 |
| 547 | port direction mismatch<br>포트 방향 불일치 | Fix input/output/inout direction<br>입력/출력/입력 방향 고정 |
| 567 | signal should be on sensitivity list<br>신호는 민감도 목록에 있어야 합니다. | Add signal to `@(...)` or use `@(*)`<br>`@(...)`에 신호를 추가하거나 `@(*)`를 사용하세요. |
| 589 | replacing case/wildcard equality<br>대소문자/와일드카드 동등성 대체 | Use `==`/`!=` instead of `===`/`!==`<br>`===`/`!==` 대신 `==`/`!=`를 사용하세요. |
| 613 | shared variable as local<br>공유 변수를 로컬로 | Review variable scope<br>변수 범위 검토 |
| 614 | signal not in sensitivity list<br>민감도 목록에 없는 신호 | Add to sensitivity list<br>민감도 목록에 추가 |
| 689 | port connection width mismatch<br>포트 연결 폭 불일치 | Resize signal or add slice<br>신호 크기 조정 또는 슬라이스 추가 |
| 693 | zero replication count<br>복제 횟수 0 | Fix replication expression<br>복제 표현식 수정 |

### Range 2887-3936 (15 IDs)
### 범위 2887-3936(ID 15개)

2887, 2897, 2898, 3301, 3330, 3332, 3514, 3819, 3848, 3917, 3919, 3936

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

### Range 4179-5974 (16 IDs)
### 범위 4179-5974(ID 16개)

4179, 4446, 4557, 4767, 5411, 5639, 5640, 5730, 5733, 5753, 5837, 5863, 5928, 5974

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

### Range 6014-7257 (25 IDs)
### 범위 6014-7257(ID 25개)

6014, 6026, 6040, 6060, 6090, 6104, 6426, 6430, 6774, 6778, 6849, 6850, 6880, 6896, 7023, 7032, 7043, 7071, 7129, 7134, 7137, 7143, 7186, 7193, 7257

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

### Range 11357-13374 (12 IDs)
### 범위 11357-13374(ID 12개)

11357, 11376, 11581, 11585, 12181, 13159, 13161, 13234, 13327, 13328, 13373, 13374

> Look up message in `oasys_severity_lists.csv`; read RTL at reported location
> `oasys_severity_lists.csv`에서 메시지를 찾습니다. 보고된 위치에서 RTL을 읽습니다.

---

## Notes
## 메모

- Message patterns use `!%1!`, `!%2!` etc. as placeholders for dynamic values
- 메시지 패턴은 `!%1!`, `!%2!` 등을 동적 값의 자리 표시자로 사용합니다.
- For IDs not in tables above, look up the full message in `oasys_severity_lists.csv`
- 위 표에 없는 ID의 경우 `oasys_severity_lists.csv`에서 전체 메시지를 찾아보세요.
- Testcases for each ID are in `synth_code_8_xxx_testcase/{error,warnings,critical_warning}/8_XXX/`
- 각 ID에 대한 테스트 케이스는 `synth_code_8_xxx_testcase/{error,warnings,tical_warning}/8_XXX/`에 있습니다.
- Always verify the fix by re-running synthesis
- 항상 합성을 다시 실행하여 수정 사항을 확인하세요.

---

## Reference Files
## 참조 파일

- **CSV**: `oasys_severity_lists.csv` - Full message catalog
- **CSV**: `oasys_severity_lists.csv` - 전체 메시지 카탈로그
- **Testcases**: `synth_code_8_xxx_testcase/` - RTL samples for each ID
- **테스트 케이스**: `synth_code_8_xxx_testcase/` - 각 ID에 대한 RTL 샘플
- **Resolution Guides**: `resolution/` - Detailed fix templates for common patterns
- **해결 가이드**: `해결 방법/` - 일반적인 패턴에 대한 자세한 수정 템플릿
