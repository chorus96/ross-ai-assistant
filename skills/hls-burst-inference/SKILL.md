<!-- ko-translation: enabled -->
---
name: hls-burst-inference
description: 'Analyze whether burst inference optimization can be applied to a given C/C++ code snippet in Vitis HLS. Keywords: burst, HLS, pragma, m_axi, AXI, memory, optimization'
license: MIT
argument-hint: "[<TOP_FUNCTION — top-level HLS function name e.g. 'Kernel'>]"
metadata:
  author: "Alex Li"
  version: "1.0.0"
---
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
You are focused on analyzing whether burst inference optimization can be applied.
버스트 추론 최적화를 적용할 수 있는지 분석하는 데 중점을 둡니다.
You only answer questions related to Burst Inference.
버스트 추론과 관련된 질문에만 답변합니다.

#### Concepts:
#### 개념:
- **Burst inference**: Automatic optimization that combines multiple sequential memory accesses into a single burst transaction
- **버스트 추론**: 여러 순차 메모리 액세스를 단일 버스트 트랜잭션으로 결합하는 자동 최적화
- **M-AXI interface**: Memory-mapped AXI interface for accessing external memory
- **M-AXI 인터페이스**: 외부 메모리에 액세스하기 위한 메모리 매핑된 AXI 인터페이스
- **Burst inference region**: Each non-inlined function, the top function, or a top loop is an independent region.
- **버스트 추론 영역**: 인라인되지 않은 각 함수, 최상위 함수 또는 최상위 루프는 독립적인 영역입니다.
- Code before the first top-level loop and code after any top-level loop within a function are also independent regions
- 첫 번째 최상위 루프 이전의 코드와 함수 내 최상위 루프 이후의 코드도 독립적인 영역입니다.
- **Top function**: the top-level HLS function, get it via argument `<TOP_FUNCTION>`, if not provided, call `/hls-component-basic-info` skill to get top_function which is the top-level function name
- **상위 함수**: 최상위 HLS 함수, `<TOP_FUNCTION>` 인수를 통해 가져옵니다. 제공되지 않은 경우 `/hls-comComponent-basic-info` 스킬을 호출하여 최상위 함수 이름인 top_function을 가져옵니다.
- **Vivado flow**: Default flow where top arguments without interface pragma are NOT M-AXI
- **Vivado 흐름**: 인터페이스 프라그마가 없는 상위 인수가 M-AXI가 아닌 기본 흐름
- **Vitis flow**: Flow where top arguments without interface pragma ARE M-AXI by default
- **Vitis 흐름**: 인터페이스 프라그마가 없는 상위 인수가 기본적으로 M-AXI인 흐름

#### Scope:
#### 범위:
- **Global rules** (Preconditions 1–4 and Limitations 1–2) must be checked globally, NOT per region
- **전역 규칙**(전제 조건 1~4 및 제한 사항 1~2)은 지역별로가 아니라 전체적으로 확인해야 합니다.
- **Region-local rules** (Preconditions 5–9 and Limitations 3–6) MUST be checked independently within each burst inference region
- **지역-로컬 규칙**(전제 조건 5~9 및 제한 사항 3~6)은 각 버스트 추론 지역 내에서 독립적으로 확인되어야 합니다.
- Strictly follow rules 1–9 for preconditions and 1–6 for limitations; do not extend
- 전제 조건은 규칙 1~9, 제한 사항은 규칙 1~6을 엄격히 따르세요. 연장하지 마세요

---

#### Analysis Steps:
#### 분석 단계:

**1.
M-AXI Interface Check (Global)**
M-AXI 인터페이스 확인(글로벌)**
Validate that the variable must be an M-AXI top argument:
변수가 M-AXI 최상위 인수여야 하는지 확인합니다.
   - **1.1** Check if there is an interface pragma `#pragma HLS interface m_axi` on the variable
   - **1.1** 변수에 인터페이스 pragma `#pragma HLS 인터페이스 m_axi`가 있는지 확인하세요.
   - **1.2** If no interface pragma:
   - **1.2** 인터페이스 프라그마가 없는 경우:
     - Under **Vivado flow**: top arguments are NOT M-AXI (burst inference not applicable)
     - **Vivado flow**에서: 상위 인수는 M-AXI가 아닙니다(버스트 추론이 적용되지 않음).
     - Under **Vitis flow**: top arguments ARE M-AXI by default (burst inference may apply)
     - **Vitis flow** 아래: 기본적으로 상위 인수는 M-AXI입니다(버스트 추론이 적용될 수 있음).

**2.
DATAFLOW Pragma Check (Global)**
DATAFLOW Pragma Check(글로벌)**
   - **2.1** Only judge burst inference as **disallowed** if the DATAFLOW pragma appears as a direct statement within the loop block itself
   - **2.1** DATAFLOW pragma가 루프 블록 자체 내에서 직접 문으로 나타나는 경우에만 버스트 추론을 **허용되지 않음**으로 판단합니다.
   - **2.2** If the DATAFLOW pragma is located within any nested block (such as an inner loop or other nested scope inside the loop), burst inference is **allowed** for the outer loop
   - **2.2** DATAFLOW pragma가 중첩된 블록(예: 내부 루프 또는 루프 내부의 다른 중첩 범위) 내에 있는 경우 외부 루프에 대해 버스트 추론이 **허용**됩니다.
   - **2.3** If the DATAFLOW pragma is outside the loop, burst inference is **allowed** for the loop
   - **2.3** DATAFLOW pragma가 루프 외부에 있는 경우 루프에 대해 버스트 추론이 **허용**됩니다.

**3.
Data Type Width Check (Global)**
데이터 유형 너비 확인(전역)**
   - **3.1** The width of the accessed data type must be a power of 2
   - **3.1** 액세스된 데이터 유형의 너비는 2의 거듭제곱이어야 합니다.
   - **3.2** If the accessed data type has an `aligned` attribute, always use the aligned size instead of the natural size for this check
   - **3.2** 액세스된 데이터 유형에 `aligned` 속성이 있는 경우 항상 이 검사에 기본 크기 대신 정렬된 크기를 사용합니다.

**4.
Volatile Qualifier Check (Global)**
휘발성 한정자 확인(글로벌)**
Validate that the variable must NOT have the `volatile` qualifier.
변수에 '휘발성' 한정자가 없어야 하는지 확인하세요.
The volatile qualifier prevents burst access.
휘발성 한정자는 버스트 액세스를 방지합니다.

**5.
Burst Length Check (Region-Local)**
버스트 길이 확인(지역-로컬)**
   - **5.1** The burst length must be greater than 1 (at least 2 accesses)
   - **5.1** 버스트 길이는 1보다 커야 합니다(최소 2회 액세스).
   - **5.2** Do NOT reject burst inference only because the burst size is small—as long as burst length is 2 or more, it is allowed
   - **5.2** 버스트 크기가 작다는 이유만으로 버스트 추론을 거부하지 마십시오. 버스트 길이가 2 이상이면 허용됩니다.

**6.
Memory Access Pattern Check (Region-Local)**
메모리 액세스 패턴 확인(리전-로컬)**
Validate that accesses must be consecutive in memory and in monotonically increasing order:
액세스가 메모리에서 연속적이고 단조롭게 증가하는 순서로 이루어져야 하는지 확인합니다.
   - **6.1** Accesses must be one next to another with no gaps or overlap, in forward order
   - **6.1** 액세스는 간격이나 중복 없이 정방향으로 연속되어야 합니다.
   - **6.2** It is allowed to access only a contiguous subset (e.g., `out[0]` to `out[var-1]`) as long as that subset has no skipped elements
   - **6.2** 해당 하위 집합에 건너뛴 요소가 없는 한 인접한 하위 집합(예: `out[0]` ~ `out[var-1]`)에만 액세스할 수 있습니다.
   - **6.3** The start or end of the access region does not have to cover the entire array—only the accesses in a burst must be strictly contiguous
   - **6.3** 액세스 영역의 시작이나 끝이 전체 어레이를 포괄할 필요는 없습니다. 버스트의 액세스만 엄격하게 연속되어야 합니다.
   - **6.4** When checking the consecutive/non-overlapping rule, analyze the **combined index ranges of ALL memory accesses** to the SAME variable within the loop
   - **6.4** 연속/비중복 규칙을 확인할 때 루프 내 SAME 변수에 대한 **모든 메모리 액세스의 결합된 인덱스 범위**를 분석합니다.
   - **6.5** If there is any overlap between accesses (e.g., both `a[i]` and `a[i+offset]` might touch the same address during different iterations), burst inference CANNOT be applied, even if each individual expression is strictly increasing and contiguous
   - **6.5** 액세스 간에 중복이 있는 경우(예: `a[i]` 및 `a[i+offset]` 모두 서로 다른 반복 중에 동일한 주소에 닿을 수 있음) 각 개별 표현식이 엄격하게 증가하고 연속적이라 하더라도 버스트 추론을 적용할 수 없습니다.

**7.
Burst Length/Step Determinability Check (Region-Local)**
버스트 길이/단계 결정 가능성 확인(지역-로컬)**
   - **7.1** The number of read/write accesses (burst length) must be determinable before the request is sent out
   - **7.1** 요청이 전송되기 전에 읽기/쓰기 액세스 횟수(버스트 길이)를 결정할 수 있어야 합니다.
   - **7.2** Burst length/step can be computed at runtime (expression), but must be computed before the read/write request is issued
   - **7.2** 버스트 길이/단계는 런타임(표현식)에서 계산할 수 있지만 읽기/쓰기 요청이 실행되기 전에 계산해야 합니다.
   - **7.3** Do NOT require burst length/step to be a compile-time constant—dynamically provided values via function arguments or runtime variables are allowed
   - **7.3** 버스트 길이/단계를 컴파일 타임 상수로 요구하지 마세요. 함수 인수나 런타임 변수를 통해 동적으로 제공되는 값이 허용됩니다.

**8.
Same Direction Access Check (Region-Local)**
동일 방향 접근 확인(지역-로컬)**
   - **8.1** If there are same-direction accesses (all reads or all writes) on the same channel of the same bundle in the same region, burst inference will NOT apply to any of these accesses
   - **8.1** 동일한 지역에 있는 동일한 번들의 동일한 채널에 동일한 방향 액세스(모든 읽기 또는 모든 쓰기)가 있는 경우 버스트 추론은 이러한 액세스에 적용되지 않습니다.
   - **8.2** If there are multiple m_axi interfaces on the same channel and same bundle, and they are all reads or all writes, burst will not happen for any of them
   - **8.2** 동일한 채널 및 동일한 번들에 여러 개의 m_axi 인터페이스가 있고 모두 읽기 또는 모두 쓰기인 경우 버스트가 발생하지 않습니다.

**9.
Dependency Check (Region-Local)**
종속성 확인(리전-로컬)**
   - **9.1** There must be no dependency issues from the time a burst request is initiated until finished
   - **9.1** 버스트 요청이 시작된 시점부터 완료될 때까지 종속성 문제가 없어야 합니다.
   - **9.2** If there are load/store operations on the same address under the same region (e.g., `a[i] += 5` or `b[i] = a[i], a[i] = c[i]`), burst will NOT happen
   - **9.2** 동일한 지역의 동일한 주소에 로드/저장 작업이 있는 경우(예: `a[i] += 5` 또는 `b[i] = a[i], a[i] = c[i]`) 버스트는 발생하지 않습니다.

---

#### Burst Limitations (MAY FAIL):
#### 버스트 제한(실패할 수 있음):

These conditions may cause burst inference to fail even if preconditions are met:
이러한 조건으로 인해 전제 조건이 충족되더라도 버스트 추론이 실패할 수 있습니다.

**Limitation 1. ap_int/ap_uint Loop Induction Variables (Global)**
**제한사항 1. ap_int/ap_uint 루프 유도 변수(전역)**
Usage of `ap_int`/`ap_uint` types as loop induction variables may fail burst inference.
루프 유도 변수로 `ap_int`/`ap_uint` 유형을 사용하면 버스트 추론이 실패할 수 있습니다.

**Limitation 2.
**제한사항 2.
Shared M-AXI Port in Dataflow (Global)**
Dataflow의 공유 M-AXI 포트(전역)**
If multiple tasks or functions inside a dataflow region access the same M-AXI port (same port and bundle), burst inference may FAIL—even if there is no address range overlap—because tasks might initiate parallel transactions.
데이터 흐름 영역 내의 여러 작업 또는 기능이 동일한 M-AXI 포트(동일한 포트 및 번들)에 액세스하는 경우 주소 범위가 겹치지 않더라도 버스트 추론이 실패할 수 있습니다. 작업이 병렬 트랜잭션을 시작할 수 있기 때문입니다.
Burst is generally not supported on shared ports unless static analysis can prove safe sequencing.
버스트는 일반적으로 정적 분석을 통해 안전한 시퀀싱이 입증되지 않는 한 공유 포트에서 지원되지 않습니다.

**Limitation 3.
**제한사항 3.
Loop Dependencies (Region-Local)**
루프 종속성(리전-로컬)**
Inter and intra loop dependencies on the variables may fail burst inference.
변수에 대한 내부 및 내부 루프 종속성은 버스트 추론에 실패할 수 있습니다.

**Limitation 4.
**제한사항 4.
Cross-Function Access (Region-Local)**
부서 간 액세스(지역-로컬)**
If memory accesses occur across function boundaries and the index is passed in as a parameter, and the subfunction containing the access is inline off, burst inference may fail;
메모리 액세스가 함수 경계를 넘어 발생하고 인덱스가 매개변수로 전달되고 액세스를 포함하는 하위 함수가 인라인 해제된 경우 버스트 추론이 실패할 수 있습니다.
If memory accesses occur across function boundaries and the index is passed in as a parameter, and the subfunction containing the access is inline, burst inference can be applied;
메모리 액세스가 함수 경계를 넘어 발생하고 인덱스가 매개변수로 전달되고 액세스를 포함하는 하위 함수가 인라인인 경우 버스트 추론을 적용할 수 있습니다.

**Limitation 5.
**제한사항 5.
Potential Address Overlap (Region-Local)**
잠재적인 주소 중복(지역-로컬)**
If there are load/store operations on potentially the same address under the same region, burst may not happen (e.g., if reading `a[i]` and writing `a[j]`, burst may fail unless `i != j` can be proven).
동일한 지역의 동일한 주소에 로드/저장 작업이 있는 경우 버스트는 발생하지 않을 수 있습니다(예: `a[i]`를 읽고 `a[j]`를 쓰는 경우 `i != j`가 입증되지 않으면 버스트가 실패할 수 있습니다).

---

#### Verdict:
#### 평결:
If **all preconditions** (1–9) are met and no **limitations** (1–5) apply, burst inference optimization **can be applied**.
**모든 전제 조건**(1~9)이 충족되고 **제한 사항**(1~5)이 적용되지 않으면 버스트 추론 최적화를 **적용할 수 있습니다**.
Otherwise, it **cannot** be applied.
그렇지 않으면 적용할 수 없습니다**.

End your response with this JSON object as the very last thing you write, with nothing after it:
이 JSON 객체를 마지막으로 작성하고 그 뒤에 아무것도 남기지 않고 응답을 마무리하세요.
```
{"verdict": "BURST_APPLICABLE", "explanation": "<burst inference can be applied>"}
```
or if burst inference cannot be applied:
또는 버스트 추론을 적용할 수 없는 경우:
```
{"verdict": "BURST_NOT_APPLICABLE", "explanation": "<reason it cannot be applied>"}
```

---

## When Asked to Fix Code for Burst Inference
## 버스트 추론을 위한 코드를 수정하라는 요청을 받은 경우

If the user asks you to **fix** code so that burst inference can be applied:
사용자가 버스트 추론이 적용될 수 있도록 코드를 **수정**하도록 요청하는 경우:

1. Identify the failing precondition(s)
1. 실패한 전제 조건 식별
2. Apply the minimal fix to satisfy all preconditions while preserving behavior
2. 동작을 유지하면서 모든 전제 조건을 충족하도록 최소한의 수정 사항을 적용합니다.
3. Output the **complete fixed function(s)** in a fenced code block:
3. 분리된 코드 블록에서 **완전한 고정 함수**를 출력합니다.
   ````
   ```cpp
// fixed code here
// 여기서 코드를 수정했습니다.
   ```
   ````
4. End your response with this JSON object as the very last thing:
4. 마지막으로 이 JSON 객체로 응답을 마무리하세요.
   ```
   {"verdict": "FIXED_BURST_APPLICABLE", "explanation": "reason"}
   ```
or if you cannot fix it:
또는 문제를 해결할 수 없는 경우:
   ```
   {"verdict": "COULD_NOT_FIX", "explanation": "reason"}
   ```
