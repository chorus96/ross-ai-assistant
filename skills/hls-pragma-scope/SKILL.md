<!-- ko-translation: enabled -->
---
name: hls-pragma-scope
description: 'Defines HLS pragma scope rules. Must be referenced when analyzing HLS designs or generating any code for HLS designs to ensure pragmas are placed in the correct scope.'
license: MIT
argument-hint: "[<TOP_FUNCTION — top-level HLS function name e.g. 'Kernel'>]"
metadata:
  author: "Yi Gao"
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
You are focused on analyzing whether HLS pragmas are placed in the correct scope.
HLS pragma가 올바른 범위에 배치되었는지 분석하는 데 중점을 둡니다.
Strictly follow our rules 1, 2, 3, 4, 5, 6, 7, 8 and do not exert yourself.
규칙 1, 2, 3, 4, 5, 6, 7, 8을 엄격히 따르고 너무 무리하지 마십시오.

#### Concepts:
#### 개념:
- **Top function**: the top-level HLS function, get it via argument `<TOP_FUNCTION>` (e.g.
- **상위 함수**: 최상위 HLS 함수, `<TOP_FUNCTION>` 인수를 통해 가져옵니다(예:
- `Kernel`), if not provided, call `/hls-component-basic-info` skill to get top_function which is the top-level function name
- `Kernel`), 제공되지 않은 경우 `/hls-comComponent-basic-info` 스킬을 호출하여 최상위 함수 이름인 top_function을 가져옵니다.
- **Function scope**: The region inside a function body between `{` and `}`
- **함수 범위**: `{`와 `}` 사이의 함수 본문 내부 영역
- **Loop scope**: The region inside a loop body between `{` and `}`
- **루프 범위**: `{`와 `}` 사이의 루프 본문 내부 영역
- **Region scope**: Any `{ }` block, including function bodies, loop bodies, and bare (non-function, non-loop) code blocks
- **지역 범위**: 함수 본문, 루프 본문 및 기본(비함수, 비루프) 코드 블록을 포함한 모든 `{ }` 블록
- **Variable visibility**: A variable is visible in the scope where it is declared and all nested child scopes
- **변수 가시성**: 변수는 선언된 범위와 중첩된 모든 하위 범위에서 볼 수 있습니다.
- **Implicit unroll**: When `#pragma HLS PIPELINE` is applied to a loop, all sub-loops within that loop are implicitly fully unrolled
- **암시적 언롤**: '#pragma HLS PIPELINE'이 루프에 적용되면 해당 루프 내의 모든 하위 루프가 암시적으로 완전히 언롤링됩니다.

#### Scope:
#### 범위:
- Do NOT check whether the pragma options/parameters are correct (that is a separate concern)
- pragma 옵션/매개변수가 올바른지 확인하지 마세요(별도의 문제임)
- ONLY check whether the pragma is placed in a legal scope
- pragma가 법적 범위에 있는지 여부만 확인하세요.
- Check each pragma independently
- 각 pragma를 독립적으로 확인
- Check all pragmas found in code called by or within the **top function**
- **top 함수**에 의해 호출되거나 내부에서 호출된 코드에서 발견된 모든 pragma를 확인하십시오.

---

#### Pragma Scope Categories:
#### Pragma 범위 카테고리:

**Category A — Function-Body Pragmas**
**범주 A — 함수 본문 프라그마**
These pragmas MUST be placed directly inside a function body (not inside a loop or other nested block within the function).
이러한 pragma는 함수 본문 내부에 직접 배치되어야 합니다(함수 내의 루프나 다른 중첩 블록 내부가 아님).
They apply to the function that contains them.
이를 포함하는 함수에 적용됩니다.

| Pragma<br>프라그마 | Notes<br>메모 |
|--------|-------|
| `#pragma HLS TOP`<br>`#pragma HLS TOP` | Marks a function as the top-level function<br>함수를 최상위 함수로 표시합니다. |
| `#pragma HLS INLINE`<br>`#pragma HLS INLINE` | Controls inlining of the enclosing function<br>둘러싸는 함수의 인라인을 제어합니다. |
| `#pragma HLS INTERFACE`<br>`#pragma HLS 인터페이스` | Defines interface protocol; see Rule 3<br>인터페이스 프로토콜을 정의합니다. 규칙 3 참조 |
| `#pragma HLS DATAFLOW`<br>`#pragma HLS DATAFLOW` | Enables task-level pipelining; also valid in loop scope (Category B)<br>작업 수준 파이프라인을 활성화합니다. 루프 범위(범주 B)에서도 유효합니다. |
| `#pragma HLS PIPELINE`<br>`#pragma HLS 파이프라인` | Pipelines the function; also valid in loop scope (Category B)<br>기능을 파이프라인합니다. 루프 범위(범주 B)에서도 유효합니다. |
| `#pragma HLS FUNCTION_INSTANTIATE`<br>`#pragma HLS FUNCTION_INSTANTIATE` | Creates unique function instances for different constant argument values<br>다양한 상수 인수 값에 대한 고유한 함수 인스턴스를 생성합니다. |
| `#pragma HLS DEPENDENCE`<br>`#pragma HLS 의존성` | Specifies dependency information for the function; also valid in loop scope (Category B)<br>함수에 대한 종속성 정보를 지정합니다. 루프 범위(범주 B)에서도 유효합니다. |
| `#pragma HLS STABLE`<br>`#pragma HLS 안정` | Marks a function argument as stable<br>함수 인수를 안정적인 것으로 표시합니다. |

**Category B — Loop-Body Pragmas**
**범주 B — 루프 바디 프라그마**
These pragmas MUST be placed directly inside a loop body.
이러한 pragma는 루프 본문 내부에 직접 배치되어야 합니다.
They apply to the loop that contains them.
이를 포함하는 루프에 적용됩니다.

| Pragma<br>프라그마 | Notes<br>메모 |
|--------|-------|
| `#pragma HLS PIPELINE`<br>`#pragma HLS 파이프라인` | Pipelines the loop; implicitly unrolls all sub-loops<br>루프를 파이프라인합니다. 모든 하위 루프를 암시적으로 펼칩니다. |
| `#pragma HLS UNROLL`<br>`#pragma HLS UNROLL` | Unrolls the enclosing loop only<br>둘러싸는 루프만 펼칩니다. |
| `#pragma HLS LOOP_FLATTEN`<br>`#pragma HLS LOOP_FLATTEN` | Flattens from the current loop outward through enclosing loops; `off` excludes the current loop from flattening with both its sub-loops and its parent loops<br>둘러싸는 루프를 통해 전류 루프에서 바깥쪽으로 편평화됩니다. 'off'는 현재 루프를 하위 루프와 상위 루프 모두와 함께 평탄화하지 않도록 제외합니다. |
| `#pragma HLS LOOP_TRIPCOUNT`<br>`#pragma HLS LOOP_TRIPCOUNT` | Provides trip count estimate for variable-bound loops<br>가변 경계 루프에 대한 이동 횟수 추정 제공 |
| `#pragma HLS DATAFLOW`<br>`#pragma HLS DATAFLOW` | Enables task-level pipelining within the loop<br>루프 내에서 작업 수준 파이프라인을 활성화합니다. |
| `#pragma HLS DEPENDENCE`<br>`#pragma HLS 의존성` | Specifies dependency information for the loop; also valid in function scope (Category A)<br>루프에 대한 종속성 정보를 지정합니다. 함수 범위(범주 A)에서도 유효합니다. |

**Category C — Region Pragmas**
**카테고리 C — 지역 프라그마**
These pragmas are placed inside a code region.
이러한 pragma는 코드 영역 내부에 배치됩니다.
A region is any `{ }` block — this includes function bodies, loop bodies, and bare (non-function, non-loop) `{ }` blocks.
영역은 `{ }` 블록입니다. 여기에는 함수 본문, 루프 본문 및 기본(비함수, 비루프) `{ }` 블록이 포함됩니다.
Function and loop scopes are special cases of regions, so all region pragmas, except OCCURRENCE, are also valid directly inside function bodies and loop bodies.
함수 및 루프 범위는 영역의 특수한 경우이므로 OCCURRENCE를 제외한 모든 영역 pragma는 함수 본문 및 루프 본문 내에서도 직접 유효합니다.

| Pragma<br>프라그마 | Notes<br>메모 |
|--------|-------|
| `#pragma HLS ALLOCATION`<br>`#pragma HLS 할당` | Limits instance count of functions/operations within the region<br>지역 내 함수/작업의 인스턴스 수를 제한합니다. |
| `#pragma HLS EXPRESSION_BALANCE`<br>`#pragma HLS EXPRESSION_BALANCE` | Controls expression tree balancing within the region<br>지역 내에서 표현식 트리 균형 조정을 제어합니다. |
| `#pragma HLS LOOP_MERGE`<br>`#pragma HLS LOOP_MERGE` | Merges consecutive loops within the region<br>영역 내에서 연속 루프를 병합합니다. |
| `#pragma HLS LATENCY`<br>`#pragma HLS 대기 시간` | Constrains the minimum and/or maximum latency of the region<br>지역의 최소 및/또는 최대 지연 시간을 제한합니다. |
| `#pragma HLS OCCURRENCE`<br>`#pragma HLS 발생` | Specifies that the region executes at a lower rate than the enclosing pipeline; MUST be inside a conditional block<br>영역이 둘러싸는 파이프라인보다 낮은 속도로 실행되도록 지정합니다. 조건부 블록 안에 있어야 합니다. |

**Category D — Variable Pragmas**
**범주 D - 가변 프라그마**
These pragmas reference a specific variable and MUST be placed in a scope where that variable is visible.
이러한 pragma는 특정 변수를 참조하며 해당 변수가 표시되는 범위에 배치되어야 합니다.

| Pragma<br>프라그마 | Notes<br>메모 |
|--------|-------|
| `#pragma HLS ARRAY_PARTITION`<br>`#pragma HLS ARRAY_PARTITION` | Partitions an array into smaller arrays or registers<br>배열을 더 작은 배열이나 레지스터로 분할합니다. |
| `#pragma HLS ARRAY_RESHAPE`<br>`#pragma HLS ARRAY_RESHAPE` | Reshapes an array by combining partitioning with word-width increase<br>분할과 단어 너비 증가를 결합하여 배열을 재구성합니다. |
| `#pragma HLS BIND_STORAGE`<br>`#pragma HLS BIND_STORAGE` | Specifies storage implementation for a variable<br>변수에 대한 저장소 구현을 지정합니다. |
| `#pragma HLS BIND_OP`<br>`#pragma HLS BIND_OP` | Specifies implementation for an operation<br>작업 구현을 지정합니다. |
| `#pragma HLS STREAM`<br>`#pragma HLS STREAM` | Implements a variable as a FIFO stream; see Rule 5<br>변수를 FIFO 스트림으로 구현합니다. 규칙 5 참조 |
| `#pragma HLS ARRAY_STENCIL`<br>`#pragma HLS ARRAY_STENCIL` | Enables stencil pattern optimization<br>스텐실 패턴 최적화를 활성화합니다. |
| `#pragma HLS DISAGGREGATE`<br>`#pragma HLS 분리` | Disaggregates a struct into individual elements<br>구조체를 개별 요소로 분해합니다. |
| `#pragma HLS AGGREGATE`<br>`#pragma HLS 집계` | Aggregates struct fields into a single wide word<br>구조체 필드를 단일 와이드 단어로 집계합니다. |

---

#### Analysis Rules:
#### 분석 규칙:

**1.
Enclosing Scope Rule**
포함 범위 규칙**
A pragma applies to its **enclosing scope** — the nearest `{` `}` block that directly contains it.
프라그마는 **인클로징 범위**(프라그마를 직접 포함하는 가장 가까운 `{` `}` 블록)에 적용됩니다.
   - **1.1** A pragma placed before a function or loop (outside its braces) does NOT apply to that function or loop.
   - **1.1** 함수나 루프 앞에(중괄호 외부) 배치된 pragma는 해당 함수나 루프에 적용되지 않습니다.
   - It is **misplaced**.
   - **잘못 배치**되었습니다.
   - **1.2** A pragma placed after the closing brace of a function or loop does NOT apply to that function or loop.
   - **1.2** 함수나 루프의 닫는 중괄호 뒤에 배치된 pragma는 해당 함수나 루프에 적용되지 않습니다.
   - It is **misplaced**.
   - **잘못 배치**되었습니다.
   - **1.3** A pragma must be placed **inside** the `{` `}` of the scope it is intended to affect.
   - **1.3** pragma는 영향을 미치려는 범위의 `{` `}` **내부**에 배치되어야 합니다.

**2.
Function-Body Pragma Placement (Category A)**
함수 본문 Pragma 배치(범주 A)**
   - **2.1** Category A pragmas MUST appear directly inside a function body, not nested inside a loop or conditional block within the function.
   - **2.1** 카테고리 A pragma는 함수 내의 루프나 조건 블록 내에 중첩되지 않고 함수 본문 내부에 직접 나타나야 합니다.
   - **2.2** Exception: `PIPELINE`, `DATAFLOW`, and `DEPENDENCE` are dual-scope — they are legal in both function bodies and loop bodies (Category A and B).
   - **2.2** 예외: `PIPELINE`, `DATAFLOW` 및 `DEPENDENCE`는 이중 범위입니다. 함수 본문과 루프 본문(범주 A 및 B) 모두에서 유효합니다.

**3.
Interface Pragma Placement**
인터페이스 프라그마 배치**
   - **3.1** `#pragma HLS INTERFACE` is ONLY valid inside the **top function** body.
   - **3.1** `#pragma HLS INTERFACE`는 **상위 함수** 본문 내에서만 유효합니다.
   - **3.2** Placing `INTERFACE` pragmas in sub-functions is **illegal** — they will be ignored.
   - **3.2** 하위 기능에 `INTERFACE` pragma를 배치하는 것은 **불법** — 무시됩니다.
   - **3.3** The `port=` argument must reference a parameter of the top function.
   - **3.3** `port=` 인수는 최상위 함수의 매개변수를 참조해야 합니다.

**4.
Loop-Body Pragma Placement (Category B)**
Loop-Body Pragma 배치(범주 B)**
   - **4.1** Category B pragmas MUST appear inside a loop body.
   - **4.1** 카테고리 B pragma는 루프 본문 내부에 나타나야 합니다.
   - **4.2** `UNROLL` applies to the **immediately enclosing loop** only.
   - **4.2** `UNROLL`은 **즉시 포함하는 루프**에만 적용됩니다.
   - Placing it in an outer loop does NOT unroll inner loops.
   - 외부 루프에 배치해도 내부 루프가 펼쳐지지 않습니다.
   - **4.3** `LOOP_FLATTEN` (without `off`) can be placed inside **any loop** in a nest, typically the innermost.
   - **4.3** `LOOP_FLATTEN`(`off` 제외)은 둥지의 **모든 루프** 내부, 일반적으로 가장 안쪽에 배치할 수 있습니다.
   - It flattens the loop containing it outward through the enclosing loops, stopping at a loop marked `LOOP_FLATTEN off` or at an unflattenable loop boundary.
   - 이를 포함하는 루프를 둘러싸는 루프를 통해 바깥쪽으로 평면화하고 'LOOP_FLATTEN off'라고 표시된 루프 또는 평면화할 수 없는 루프 경계에서 중지합니다.
   - **4.3.1** `#pragma HLS LOOP_FLATTEN off` must also be placed inside a loop body.
   - **4.3.1** `#pragma HLS LOOP_FLATTEN off`도 루프 본문 내부에 배치해야 합니다.
   - Call the loop whose body directly contains the pragma **L** — the pragma applies to **L** only.
   - 본문에 pragma **L**이 직접 포함된 루프를 호출합니다. pragma는 **L**에만 적용됩니다.
   - It **isolates L in both directions**: **L** is NOT flattened with the loops nested **inside L**, and **L** is NOT flattened with the loops that **enclose L**.
   - 이는 **L을 양방향으로 분리**합니다. **L**은 **L** 내부에 중첩된 루프로 평면화되지 않으며 **L**은 **L**을 포함하는 루프로 평면화되지 않습니다.
   - **4.3.2** `off` acts as a boundary only at its own loop — loops nested **deeper** inside it may still be flattened among themselves.
   - **4.3.2** `off`는 자체 루프에서만 경계 역할을 합니다. 내부에 **더 깊게** 중첩된 루프는 여전히 서로 평탄화될 수 있습니다.
   - Example: in `L1 > L2 > L3`, `LOOP_FLATTEN off` inside L1 plus `LOOP_FLATTEN` inside L3 fuses L2 with L3 while L1 stays a separate loop level.
   - 예: `L1 > L2 > L3`에서 L1 내부의 `LOOP_FLATTEN off`와 L3 내부의 `LOOP_FLATTEN`은 L2를 L3과 융합하고 L1은 별도의 루프 레벨을 유지합니다.
   - **4.3.3** `LOOP_FLATTEN off` inside the **innermost** loop disables flattening of that loop entirely, including the automatic flattening the tool would apply with no pragma present.
   - **4.3.3** **가장 안쪽** 루프 내부의 `LOOP_FLATTEN off`는 pragma가 없는 상태에서 도구가 적용하는 자동 평면화를 포함하여 해당 루프의 평면화를 완전히 비활성화합니다.
   - **4.4** `LOOP_MERGE` must be placed in the scope that **contains** the consecutive loops to be merged (i.e., the parent function or outer loop body), NOT inside any of the loops being merged.
   - **4.4** `LOOP_MERGE`는 병합할 루프 내부가 아닌 병합할 연속 루프를 **포함**하는 범위(예: 상위 함수 또는 외부 루프 본문)에 배치해야 합니다.
   - **4.5** `LOOP_TRIPCOUNT` is only meaningful for loops with **variable bounds** and is used for reporting only; it does not affect synthesis.
   - **4.5** `LOOP_TRIPCOUNT`는 **변수 범위**가 있는 루프에만 의미가 있으며 보고에만 사용됩니다. 합성에는 영향을 미치지 않습니다.
   - **4.6** `DEPENDENCE` inside a loop body specifies dependencies for that loop.
   - **4.6** 루프 본문 내부의 `DEPENDENCE`는 해당 루프에 대한 종속성을 지정합니다.
   - Placing it in the function body specifies dependencies for that function.
   - 함수 본문에 배치하면 해당 함수에 대한 종속성이 지정됩니다.

**5.
Region Pragma Placement (Category C)**
지역 Pragma 배치(범주 C)**
   - **5.1** Category C pragmas are valid inside any `{ }` block — including function bodies, loop bodies, and bare code regions.
   - **5.1** 카테고리 C pragma는 함수 본문, 루프 본문 및 베어 코드 영역을 포함하여 모든 `{ }` 블록 내에서 유효합니다.
   - **5.2** `INLINE region` behaves differently from `INLINE` without the `region` keyword — `INLINE region` inlines all function calls **within** the region, while `INLINE` (Category A) inlines the enclosing function itself at its call sites.
   - **5.2** `INLINE 영역`은 `region` 키워드가 없는 `INLINE`과 다르게 동작합니다. `INLINE 영역`은 영역 내의** 모든 함수 호출을 인라인하는 반면, `INLINE`(범주 A)은 호출 사이트에서 둘러싸는 함수 자체를 인라인합니다.
   - **5.3** `OCCURRENCE` MUST be placed inside a conditional block within a pipelined scope.
   - **5.3** `OCCURRENCE`는 파이프라인 범위 내의 조건 블록 내에 배치되어야 합니다.
   - It indicates the block executes at a fraction of the pipeline rate.
   - 이는 블록이 파이프라인 속도의 일부로 실행된다는 것을 나타냅니다.
   - **5.4** `ALLOCATION` pragmas with `type=function` must be placed in the scope of the caller, not in the function being limited.
   - **5.4** `type=function`인 `ALLOCATION` pragma는 제한되는 함수가 아니라 호출자의 범위에 배치되어야 합니다.

**6.
Variable Pragma Visibility (Category D)**
가변 Pragma 가시성(범주 D)**
   - **6.1** The `variable=` argument must reference a variable that is **visible** (in scope) at the pragma's location.
   - **6.1** `variable=` 인수는 pragma 위치에서 (범위 내에서) **표시**되는 변수를 참조해야 합니다.
   - **6.2** A variable pragma placed in a scope where the target variable is not visible is **illegal**.
   - **6.2** 대상 변수가 표시되지 않는 범위에 배치된 변수 pragma는 **불법**입니다.
   - **6.3** For function parameters: variable pragmas referencing parameters must be placed inside that function's body.
   - **6.3** 함수 매개변수의 경우: 매개변수를 참조하는 변수 pragma는 해당 함수 본문 내에 배치되어야 합니다.
   - **6.4** For local variables: variable pragmas must be placed at or after the variable's declaration, within the same scope or a nested scope.
   - **6.4** 지역 변수의 경우: 변수 pragma는 동일한 범위 또는 중첩된 범위 내에서 변수 선언이나 그 뒤에 배치되어야 합니다.
   - **6.5** `#pragma HLS STREAM` cannot be applied to a **top function argument** (use `INTERFACE mode=ap_fifo` or `INTERFACE mode=axis` instead).
   - **6.5** `#pragma HLS STREAM`은 **최상위 함수 인수**에 적용할 수 없습니다(대신 `INTERFACE mode=ap_fifo` 또는 `INTERFACE mode=axis` 사용).
   - It is legal on any other visible variable, including sub-function arguments.
   - 하위 함수 인수를 포함하여 다른 모든 표시 변수에 대해 유효합니다.
   - **6.6** Variable pragmas are **propagated across function calls**.
   - **6.6** 변수 pragma는 **함수 호출 전반에 걸쳐 전파됩니다**.
   - When a variable pragma is applied to a variable in the caller, it propagates to the corresponding parameter in the callee.
   - 변수 pragma가 호출자의 변수에 적용되면 호출 수신자의 해당 매개변수로 전파됩니다.
   - This means a pragma placed on an argument in the calling function also takes effect inside the called function.
   - 이는 호출 함수의 인수에 배치된 pragma가 호출된 함수 내에서도 적용된다는 의미입니다.

**7.
PIPELINE and Implicit Unroll Interaction**
파이프라인 및 암시적 언롤 상호 작용**
   - **7.1** When `#pragma HLS PIPELINE` is applied to a loop, all loops nested inside that loop are **implicitly fully unrolled**.
   - **7.1** `#pragma HLS PIPELINE`이 루프에 적용되면 해당 루프 내에 중첩된 모든 루프가 **암시적으로 완전히 펼쳐집니다**.
   - **7.2** When `#pragma HLS PIPELINE` is applied to a function, all loops inside that function are **implicitly fully unrolled**.
   - **7.2** `#pragma HLS PIPELINE`이 함수에 적용되면 해당 함수 내부의 모든 루프가 **암시적으로 완전히 펼쳐집니다**.
   - **7.3** If an inner loop cannot be unrolled (e.g., variable trip count), the `PIPELINE` on the outer scope will fail.
   - **7.3** 내부 루프를 풀 수 없는 경우(예: 가변 트립 횟수) 외부 범위의 'PIPELINE'이 실패합니다.
   - The `PIPELINE` pragma will be reported as ignored in the synthesis log.
   - `PIPELINE` pragma는 합성 로그에서 무시된 것으로 보고됩니다.

**8.
DATAFLOW Scope Rules**
DATAFLOW 범위 규칙**
   - **8.1** `#pragma HLS DATAFLOW` in a function body enables task-level pipelining between sequential processes (function calls or loops) in that function.
   - **8.1** 함수 본문의 `#pragma HLS DATAFLOW`는 해당 함수의 순차 프로세스(함수 호출 또는 루프) 간의 작업 수준 파이프라인을 활성화합니다.
   - **8.2** `#pragma HLS DATAFLOW` in a loop body enables task-level pipelining between sequential processes within **each iteration** of that loop.
   - **8.2** 루프 본문의 `#pragma HLS DATAFLOW`를 사용하면 해당 루프의 **각 반복** 내 순차 프로세스 간 작업 수준 파이프라인이 가능합니다.
   - **8.3** `DATAFLOW` and `PIPELINE` are **mutually exclusive** in the same scope — they cannot both be applied to the same function body or the same loop body.
   - **8.3** `DATAFLOW`와 `PIPELINE`은 동일한 범위에서 **상호 배타적**입니다. 둘 다 동일한 함수 본문이나 동일한 루프 본문에 적용될 수 없습니다.
   - **8.4** Refer to the `hls-dataflow` skill for detailed canonicality rules once scope placement is validated.
   - **8.4** 범위 배치가 검증되면 자세한 정규성 규칙은 `hls-dataflow` 기술을 참조하세요.

---

#### Verdict:
#### 평결:
If **all** pragma placements follow the scope rules above, the pragmas are **correctly scoped**.
**모든** pragma 배치가 위의 범위 규칙을 따르는 경우 pragma의 범위가 올바르게 지정됩니다**.
Otherwise, list each violation with the rule number, the offending pragma, and the correction needed.
그렇지 않으면 규칙 번호, 문제가 있는 pragma 및 필요한 수정 사항과 함께 각 위반 사항을 나열합니다.
