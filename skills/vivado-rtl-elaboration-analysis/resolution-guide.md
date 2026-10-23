<!-- ko-translation: enabled -->
# Resolution Guide - Oasys Synthesis Messages
# 해결 가이드 - Oasys 합성 메시지

This file maps **all 551 Oasys synthesis message IDs** from testcases to resolution patterns.
이 파일은 테스트 케이스의 **모든 551개 Oasys 합성 메시지 ID**를 해결 패턴에 매핑합니다.

---

## Overview
## 개요

Resolution guides are pre-written, validated fix templates for known message types.
해결 가이드는 알려진 메시지 유형에 대해 미리 작성되고 검증된 수정 템플릿입니다.
They provide:
그들은 다음을 제공합니다:

- **Consistency** - Same message type always gets the same fix pattern
- **일관성** - 동일한 메시지 유형은 항상 동일한 수정 패턴을 갖습니다.
- **Accuracy** - Fixes validated against IEEE standards and Xilinx documentation
- **정확성** - IEEE 표준 및 Xilinx 문서에 대해 검증된 수정 사항
- **Completeness** - Includes rationale, multiple fix options, and verification
- **완전성** - 근거, 여러 수정 옵션 및 확인이 포함됩니다.
- **Speed** - Pre-written templates reduce analysis time
- **속도** - 미리 작성된 템플릿으로 분석 시간 단축

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

## Resolution Pattern Categories
## 해상도 패턴 카테고리

### Category 1: Declaration Issues
### 카테고리 1: 선언 문제
**Pattern**: Signal/variable/module not found or already declared
**패턴**: 신호/변수/모듈을 찾을 수 없거나 이미 선언되었습니다.

| Oasys ID<br>오아시스 아이디 | Issue<br>이슈 | Fix Pattern<br>패턴 수정 |
|----------|-------|-------------|
| 36 | not declared<br>선언되지 않음 | Add declaration; check spelling and scope<br>선언을 추가합니다. 철자와 범위를 확인하세요 |
| 258 | duplicate association<br>중복 연관 | Remove duplicate<br>중복 제거 |
| 439 | module not found<br>모듈을 찾을 수 없습니다 | Add file to project; fix typo<br>프로젝트에 파일을 추가합니다. 오타 수정 |
| 470 | no interface found<br>인터페이스를 찾을 수 없습니다 | Add interface definition<br>인터페이스 정의 추가 |
| 478 | no matching signal for .*<br>.*에 대해 일치하는 신호가 없습니다. | Add matching signal<br>일치하는 신호 추가 |
| 485 | no port on instance<br>인스턴스에 포트 없음 | Fix port name<br>포트 이름 수정 |
| 493 | no such design unit<br>그런 디자인 단위가 없습니다 | Add missing file<br>누락된 파일 추가 |
| 660 | unable to resolve<br>해결할 수 없다 | Fix typo; check scope<br>오타를 수정하세요. 범위를 확인하세요 |

### Category 2: Constant Expression Issues
### 카테고리 2: 상수 표현식 문제
**Pattern**: Expression must be constant for synthesis
**패턴**: 합성을 위해서는 발현이 일정해야 합니다.

| Oasys ID<br>오아시스 아이디 | Issue<br>이슈 | Fix Pattern<br>패턴 수정 |
|----------|-------|-------------|
| 35 | not a constant<br>상수가 아닌 | Use parameter/localparam<br>매개변수/localparam 사용 |
| 196 | conditional not constant<br>조건부 상수가 아님 | Use constant condition<br>상수 조건 사용 |
| 206 | constant expression required<br>상수 표현이 필요함 | Replace variable with constant<br>변수를 상수로 바꾸기 |
| 211 | could not evaluate expression<br>표현을 평가할 수 없습니다 | Simplify; use constants<br>단순화하다; 상수를 사용하다 |
| 279 | generate expression not constant<br>상수가 아닌 표현식 생성 | Use parameter<br>매개변수 사용 |
| 280 | expression must be constant<br>표현식은 일정해야 합니다. | Use constant value<br>상수 값 사용 |
| 521 | parameter not constant<br>매개변수가 일정하지 않음 | Use compile-time constant<br>컴파일 타임 상수 사용 |
| 561 | range expression not constant<br>범위 표현식이 상수가 아님 | Use parameter for bounds<br>경계에 매개변수 사용 |

### Category 3: Width/Type Mismatch Issues
### 카테고리 3: 너비/유형 불일치 문제
**Pattern**: Signal widths or types don't match
**패턴**: 신호 폭 또는 유형이 일치하지 않습니다.

| Oasys ID<br>오아시스 아이디 | Issue<br>이슈 | Fix Pattern<br>패턴 수정 |
|----------|-------|-------------|
| 83 | packed union size mismatch<br>포장된 조합 크기 불일치 | Pad to equal width<br>너비를 동일하게 패드 |
| 281 | must be packed type<br>포장형이어야 합니다. | Add `packed` keyword<br>`packed` 키워드 추가 |
| 421 | array size mismatch<br>배열 크기 불일치 | Match dimensions<br>차원 일치 |
| 509 | operand length mismatch<br>피연산자 길이 불일치 | Resize operands<br>피연산자 크기 조정 |
| 549 | port width mismatch<br>포트 너비 불일치 | Resize signal or add slice<br>신호 크기 조정 또는 슬라이스 추가 |
| 550 | instance array width mismatch<br>인스턴스 배열 너비 불일치 | Match width to port x count<br>너비를 포트 x 개수와 일치 |
| 658 | type mismatch for port<br>포트 유형 불일치 | Fix type compatibility<br>유형 호환성 수정 |
| 659 | type mismatch<br>유형 불일치 | Add cast or change type<br>캐스트 추가 또는 유형 변경 |
| 690 | width mismatch in assignment<br>할당의 너비 불일치 | Resize source/target<br>소스/타겟 크기 조정 |
| 689 | port connection width mismatch<br>포트 연결 폭 불일치 | Resize or slice<br>크기 조정 또는 슬라이스 |

### Category 4: Part-Select/Index Issues
### 카테고리 4: 부품 선택/인덱스 문제
**Pattern**: Array/vector indexing errors
**패턴**: 배열/벡터 인덱싱 오류

| Oasys ID<br>오아시스 아이디 | Issue<br>이슈 | Fix Pattern<br>패턴 수정 |
|----------|-------|-------------|
| 248 | slice direction mismatch<br>슬라이스 방향 불일치 | Fix [MSB:LSB] direction<br>[MSB:LSB] 방향 수정 |
| 324 | index out of range<br>범위를 벗어난 인덱스 | Fix bounds<br>경계 수정 |
| 523 | part-select mismatch<br>부분 선택 불일치 | Match declaration direction<br>일치 선언 방향 |
| 524 | part-select out of range<br>범위를 벗어난 부분 선택 | Adjust bounds<br>경계 조정 |
| 354 | integer overflow<br>정수 오버플로 | Reduce array size<br>배열 크기 줄이기 |

### Category 5: Case Statement Issues
### 카테고리 5: 사례 진술서 문제
**Pattern**: Case/switch statement problems
**패턴**: Case/switch 문 문제

| Oasys ID<br>오아시스 아이디 | Issue<br>이슈 | Fix Pattern<br>패턴 수정 |
|----------|-------|-------------|
| 151 | case item unreachable<br>케이스 항목에 연결할 수 없음 | Remove or reorder<br>제거 또는 재정렬 |
| 153 | case item never executed<br>케이스 항목이 실행되지 않음 | Remove dead item<br>죽은 항목 제거 |
| 426 | missing choices<br>누락된 선택 | Add missing cases or default<br>누락된 케이스 또는 기본값 추가 |
| 517 | overlapping choice<br>중복 선택 | Make values unique<br>가치를 고유하게 만드세요 |
| 1875 | duplicate default<br>중복 기본값 | Remove extra default<br>추가 기본값 제거 |

### Category 6: Latch Inference Issues
### 범주 6: 래치 추론 문제
**Pattern**: Unintended latch inferred
**패턴**: 의도하지 않은 래치가 추론됨

| Oasys ID<br>오아시스 아이디 | Issue<br>이슈 | Fix Pattern<br>패턴 수정 |
|----------|-------|-------------|
| 90 | always_latch no latch<br>Always_latch 래치 없음 | Check inference conditions<br>추론 조건 확인 |
| 264 | latch enable always disabled<br>래치 활성화는 항상 비활성화됩니다. | Check enable; may be dead code<br>활성화를 확인하십시오. 죽은 코드일 수도 있습니다 |
| 327 | inferring latch<br>래치 추론 | Add `else`/`default` clause<br>`else`/`default` 절 추가 |

### Category 7: Clock/Timing Issues
### 카테고리 7: 클록/타이밍 문제
**Pattern**: Clock or event control problems
**패턴**: 시계 또는 이벤트 제어 문제

| Oasys ID<br>오아시스 아이디 | Issue<br>이슈 | Fix Pattern<br>패턴 수정 |
|----------|-------|-------------|
| 85 | no event control<br>이벤트 제어 없음 | Add `@(*)` or sensitivity list<br>`@(*)` 또는 민감도 목록 추가 |
| 91 | ambiguous clock<br>모호한 시계 | Use single edge<br>단일 모서리 사용 |
| 434 | mixed level/edge triggers<br>혼합 레벨/에지 트리거 | Separate blocks<br>별도의 블록 |
| 462 | no clock in event control<br>이벤트 제어에 시계가 없습니다. | Add clock signal<br>클럭 신호 추가 |
| 567 | signal not in sensitivity<br>신호가 민감하지 않음 | Add to sensitivity list<br>민감도 목록에 추가 |
| 614 | signal not in sensitivity<br>신호가 민감하지 않음 | Add to sensitivity list<br>민감도 목록에 추가 |

### Category 8: Loop/Recursion Issues
### 범주 8: 루프/재귀 문제
**Pattern**: Infinite or unsupported iteration
**패턴**: 무한하거나 지원되지 않는 반복

| Oasys ID<br>오아시스 아이디 | Issue<br>이슈 | Fix Pattern<br>패턴 수정 |
|----------|-------|-------------|
| 120 | call depth exceeded<br>통화 심도를 초과했습니다. | Reduce recursion; flatten<br>재귀를 줄입니다. 평평하게 하다 |
| 316 | module recursion<br>모듈 재귀 | Remove circular instantiation<br>순환 인스턴스화 제거 |
| 317 | recursive instantiation<br>재귀 인스턴스화 | Flatten hierarchy<br>계층구조 평탄화 |
| 403 | loop limit exceeded<br>루프 제한을 초과했습니다. | Add iteration limit<br>반복 제한 추가 |

### Category 9: Port Connection Issues
### 카테고리 9: 포트 연결 문제
**Pattern**: Instance port connection problems
**패턴**: 인스턴스 포트 연결 문제

| Oasys ID<br>오아시스 아이디 | Issue<br>이슈 | Fix Pattern<br>패턴 수정 |
|----------|-------|-------------|
| 34 | wrong argument count<br>잘못된 인수 개수 | Add missing arguments<br>누락된 인수 추가 |
| 349 | wrong connection count<br>잘못된 연결 개수 | Fix port count<br>포트 수 수정 |
| 422 | missing association<br>연관성 누락 | Add missing connection<br>누락된 연결 추가 |
| 476 | no matching instance<br>일치하는 인스턴스가 없습니다 | Fix defparam path<br>defparam 경로 수정 |
| 650 | too many ports<br>포트가 너무 많음 | Remove excess<br>초과분 제거 |
| 685 | variable in output port<br>출력 포트의 변수 | Change to net or use procedural<br>net으로 변경하거나 절차적 사용 |

### Category 10: Unsupported Constructs
### 카테고리 10: 지원되지 않는 구성
**Pattern**: Feature not supported for synthesis
**패턴**: 합성이 지원되지 않는 기능

| Oasys ID<br>오아시스 아이디 | Issue<br>이슈 | Fix Pattern<br>패턴 수정 |
|----------|-------|-------------|
| 27 | not supported<br>지원되지 않음 | Use synthesizable alternative<br>합성 가능한 대안 사용 |
| 277 | exponentiation not supported<br>지수화는 지원되지 않습니다. | Use shifts or LUT<br>교대근무 또는 LUT 사용 |
| 312 | unsynthesizable construct<br>합성할 수 없는 구조물 | Replace with supported construct<br>지원되는 구성으로 바꾸기 |
| 589 | case/wildcard equality<br>대소문자/와일드카드 동등성 | Use == or !=<br>== 또는 !=를 사용하세요. |

---

## Critical Warning Resolution (8 IDs)
## 심각한 경고 해결(ID 8개)

| ID | Issue<br>이슈 | Resolution<br>해상도 |
|----|-------|------------|
| 4442 | BlackBox unconnected pin<br>블랙박스 연결되지 않은 핀 | Connect pin or document as intentional<br>의도적으로 핀이나 문서를 연결하세요. |
| 5796 | RAM conflicting writes<br>RAM 충돌 쓰기 | Use different addresses or add arbitration<br>다른 주소를 사용하거나 중재를 추가하세요. |
| 5967 | Mixed tristate reset<br>혼합 삼상태 재설정 | Split vector; separate tristate assignments<br>분할 벡터; 별도의 삼국 할당 |
| 6030 | Inout without tristate<br>삼상태가 없는 Inout | Use explicit direction or tristate buffer<br>명시적 방향 또는 삼상태 버퍼 사용 |
| 6781 | Nested interface items<br>중첩된 인터페이스 항목 | Pass items explicitly; flatten usage<br>항목을 명시적으로 전달합니다. 사용량을 평탄화하다 |
| 6790 | Constant reassigned<br>상수 재할당 | Remove assignment or change to variable<br>할당 제거 또는 변수 변경 |
| 6858 | Multi-driven with constant<br>상수로 다중 구동 | Remove duplicate drivers<br>중복 드라이버 제거 |
| 13329 | Null range<br>널 범위 | Fix range bounds and direction<br>범위 경계 및 방향 수정 |

---

## Complete ID List by Severity
## 심각도별 전체 ID 목록

### Critical Warnings (8 IDs)
### 심각한 경고(ID 8개)
4442, 5796, 5967, 6030, 6781, 6790, 6858, 13329

### Errors (457 IDs)
### 오류(ID 457개)

**27-99**: 27, 34, 35, 36, 78, 83, 91, 99

**120-285**: 120, 196, 206, 211, 248, 250, 258, 274, 277, 279, 280, 281, 285

**316-439**: 316, 317, 318, 349, 354, 366, 403, 421, 422, 426, 428, 434, 436, 439

**462-561**: 462, 465, 470, 476, 478, 485, 493, 509, 517, 521, 522, 523, 524, 549, 550, 561

**647-734**: 647, 650, 658, 659, 660, 685, 690, 734

**837-983**: 837, 859, 944, 981, 982, 983

**1067-1958**: 1067, 1101, 1560, 1561, 1576, 1621, 1697, 1718, 1819, 1859, 1875, 1892, 1925, 1956, 1958

**2043-2948**: 2043, 2056, 2068, 2088, 2105, 2112, 2114, 2119, 2121, 2132, 2148, 2158, 2179, 2195, 2203, 2210, 2212, 2254, 2291, 2340, 2371, 2409, 2415, 2458, 2481, 2496, 2506, 2512, 2543, 2548, 2549, 2551, 2558, 2572, 2577, 2590, 2599, 2666, 2680, 2691, 2716, 2757, 2784, 2798, 2856, 2901, 2902, 2908, 2910, 2913, 2914, 2916, 2918, 2948

**3302-4760**: 3302, 3380, 3391, 3492, 3493, 3522, 3892, 3918, 3942, 4169, 4374, 4556, 4564, 4675, 4678, 4687, 4752, 4760

**5535-6902**: 5535, 5549, 5551, 5731, 5732, 5743, 5765, 5767, 5826, 5832, 5838, 6036, 6038, 6043, 6046, 6058, 6062, 6073, 6156, 6716, 6735, 6777, 6799, 6828, 6901, 6902

**7028-8986**: 7028, 7051, 7054, 7055, 7068, 7136, 7213, 8742, 8863, 8879, 8882, 8891, 8892, 8895, 8896, 8899, 8902, 8910, 8922, 8923, 8925, 8926, 8927, 8936, 8937, 8943, 8945, 8958, 8960, 8983, 8985, 8986

**9101-9963**: 9101, 9109, 9111, 9112, 9114, 9116, 9117, 9121, 9123, 9143, 9173, 9174, 9179, 9182, 9185, 9195, 9210, 9212, 9222, 9223, 9224, 9248, 9249, 9250, 9280, 9287, 9288, 9312, 9315, 9339, 9372, 9380, 9382, 9386, 9397, 9398, 9400, 9404, 9414, 9429, 9448, 9469, 9480, 9481, 9486, 9492, 9493, 9502, 9510, 9563, 9565, 9570, 9571, 9587, 9597, 9606, 9610, 9652, 9653, 9654, 9664, 9696, 9699, 9714, 9721, 9729, 9735, 9736, 9747, 9764, 9767, 9771, 9775, 9842, 9843, 9844, 9871, 9872, 9873, 9875, 9891, 9901, 9910, 9913, 9915, 9921, 9923, 9926, 9932, 9934, 9960, 9963

**10021-11899**: 10021, 10033, 10039, 10042, 10043, 10047, 10059, 10093, 10095, 10097, 10140, 10149, 10157, 10180, 10219, 10221, 10275, 10287, 10298, 10300, 10302, 10304, 10307, 10314, 10320, 10327, 10334, 10344, 10366, 10387, 10388, 10438, 10440, 10458, 10478, 10487, 10491, 10495, 10507, 10511, 10515, 10530, 10531, 10551, 10561, 10571, 10592, 10604, 10606, 10615, 10616, 10617, 10631, 10632, 10636, 10656, 10661, 10672, 10681, 10690, 10694, 10703, 10712, 10713, 10714, 10729, 10738, 10752, 10768, 10790, 10791, 10797, 10813, 10826, 10829, 10848, 10872, 10882, 10886, 10890, 10911, 10929, 10940, 10945, 10949, 10976, 10992, 11001, 11055, 11057, 11062, 11065, 11066, 11067, 11077, 11095, 11098, 11112, 11117, 11119, 11121, 11137, 11145, 11152, 11155, 11160, 11161, 11166, 11182, 11187, 11212, 11213, 11214, 11218, 11229, 11234, 11242, 11259, 11261, 11262, 11287, 11290, 11303, 11323, 11324, 11325, 11356, 11365, 11495, 11501, 11534, 11536, 11537, 11558, 11560, 11574, 11584, 11587, 11615, 11635, 11772, 11803, 11815, 11817, 11899

**12188-13331**: 12188, 12189, 12523, 12526, 13076, 13099, 13162, 13322, 13324, 13331

### Warnings (86 IDs)
### 경고(ID 86개)

**85-693**: 85, 90, 98, 151, 153, 264, 290, 302, 308, 311, 312, 324, 327, 330, 445, 506, 547, 567, 589, 613, 614, 689, 693

**2887-3936**: 2887, 2897, 2898, 3301, 3330, 3332, 3514, 3819, 3848, 3917, 3919, 3936

**4179-5974**: 4179, 4446, 4557, 4767, 5411, 5639, 5640, 5730, 5733, 5753, 5837, 5863, 5928, 5974

**6014-7257**: 6014, 6026, 6040, 6060, 6090, 6104, 6426, 6430, 6774, 6778, 6849, 6850, 6880, 6896, 7023, 7032, 7043, 7071, 7129, 7134, 7137, 7143, 7186, 7193, 7257

**11357-13374**: 11357, 11376, 11581, 11585, 12181, 13159, 13161, 13234, 13327, 13328, 13373, 13374

---

## Mandatory Workflow (Per Message)
## 필수 워크플로(메시지별)

```
1. Parse log - extract [Synth 8-XXX] messages (Oasys synthesis)
2. For each Oasys message ID:
   a. Check Resolution Pattern Categories above
   b. If pattern matches, apply the fix pattern
   c. If no pattern match, look up message in oasys_severity_lists.csv
   d. Read testcase in synth_code_8_xxx_testcase/*/8_XXX/ for example RTL
3. Always read the RTL source at reported file:line before generating a fix
4. Never fabricate code - only propose fixes based on actual source context
```

---

## Reference Files
## 참조 파일

- **CSV**: `oasys_severity_lists.csv` - Full message catalog (5000+ entries)
- **CSV**: `oasys_severity_lists.csv` - 전체 메시지 카탈로그(5000개 이상의 항목)
- **Testcases**: `synth_code_8_xxx_testcase/` - 551 testcase folders with RTL samples
- **테스트 케이스**: `synth_code_8_xxx_testcase/` - RTL 샘플이 포함된 551개의 테스트 케이스 폴더
- **Message Handlers**: `message-handlers.md` - Per-ID fix strategies
- **메시지 핸들러**: `message-handlers.md` - ID별 수정 전략
