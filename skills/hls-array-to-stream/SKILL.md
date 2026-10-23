<!-- ko-translation: enabled -->
---
name: hls-array-to-stream
description: 'Analyze whether a given C/C++ code snippet follows the recommended coding style for array-to-stream conversion in Vitis HLS. Keywords: array, stream, HLS, pragma, ap_fifo, axis, interface'
license: MIT
argument-hint: "[<TOP_FUNCTION — top-level HLS function name e.g. 'Kernel'>]"
metadata:
  author: "Huina Li, Alex Li"
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
You are focused on analyzing whether code follows the recommended coding style for array-to-stream conversion.
코드가 배열에서 스트림으로의 변환에 권장되는 코딩 스타일을 따르는지 분석하는 데 중점을 둡니다.

#### Concepts:
#### 개념:
- **Array-to-stream conversion**: Converting arrays to streaming interfaces for efficient data transfer
- **어레이-스트림 변환**: 효율적인 데이터 전송을 위해 어레이를 스트리밍 인터페이스로 변환
- **Stream pragma**: `#pragma HLS stream variable=A type=T depth=D`
- **스트림 프라그마**: `#pragma HLS 스트림 변수=A 유형=T 깊이=D`
- **Interface pragma**: `#pragma HLS interface [mode=ap_fifo|axis] port=A` (`mode` is optional; for example, `mode=ap_fifo` or just `ap_fifo`)
- **인터페이스 pragma**: `#pragma HLS 인터페이스 [mode=ap_fifo|axis] port=A`(`mode`는 선택 사항입니다. 예를 들어 `mode=ap_fifo` 또는 그냥 `ap_fifo`)
- **Top function**: the top-level HLS function, get it via argument `<TOP_FUNCTION>`, if not provided, call `/hls-component-basic-info` skill to get top_function which is the top-level function name
- **상위 함수**: 최상위 HLS 함수, `<TOP_FUNCTION>` 인수를 통해 가져옵니다. 제공되지 않은 경우 `/hls-comComponent-basic-info` 스킬을 호출하여 최상위 함수 이름인 top_function을 가져옵니다.
- **dataflow region**: dataflow region is a function or for-loop that contains a #pragma HLS dataflow.
- **데이터 흐름 영역**: 데이터 흐름 영역은 #pragma HLS 데이터 흐름을 포함하는 함수 또는 for 루프입니다.
- **Top function arguments**: Parameters of the top-level function
- **상위 함수 인수**: 최상위 함수의 매개변수

#### Scope:
#### 범위:
- Strictly follow rules 1–6; do not extend
- 규칙 1~6을 엄격히 따르십시오. 연장하지 마세요
- Must get `<TOP_FUNCTION>` value which is the top-level function name (e.g.
- 최상위 함수 이름인 `<TOP_FUNCTION>` 값을 가져와야 합니다(예:
- `Kernel`)
- `커널`)

---

#### Analysis Steps:
#### 분석 단계:

**1.
Pragma Presence**
프라그마 존재**
Validate that the code contains at least one of the following two array-to-stream pragmas:
코드에 다음 두 배열-스트림 pragma 중 하나 이상이 포함되어 있는지 확인하십시오.
   - Stream pragma: `#pragma HLS stream variable=A type=T depth=D`
   - 스트림 pragma: `#pragma HLS 스트림 변수=A 유형=T 깊이=D`
   - Interface pragma: `#pragma HLS interface [mode=ap_fifo|axis] port=A` (`mode` is optional; for example, `mode=ap_fifo` or just `ap_fifo`)
   - 인터페이스 pragma: `#pragma HLS 인터페이스 [mode=ap_fifo|axis] port=A`(`mode`는 선택 사항입니다. 예를 들어 `mode=ap_fifo` 또는 그냥 `ap_fifo`)

**2.
Stream Pragma Rules** (check if `#pragma HLS stream variable=<VAR> [type=<T>] [depth=<D>]` is used)
스트림 프라그마 규칙**(`#pragma HLS 스트림 변수=<VAR> [유형=<T>] [깊이=<D>]`가 사용되는지 확인)
   - **2.1** It must be a legal HLS stream pragma with the form `#pragma HLS stream variable=<VAR> [type=<T>] [depth=<D>]`
   - **2.1** `#pragma HLS 스트림 변수=<VAR> [유형=<T>] [깊이=<D>]` 형식의 유효한 HLS 스트림 pragma여야 합니다.
   - **2.2** `<VAR>` can be:
   - **2.2** `<VAR>`은 다음과 같습니다.
     - An array (single or multi-dimensional)
     - 배열(단일 또는 다차원)
     - A scalar variable
     - 스칼라 변수
   - **2.3** `<VAR>` must be a local variable or a global variable
   - **2.3** `<VAR>`은 지역 변수이거나 전역 변수여야 합니다.
   - **2.4** `<VAR>` must be passed within the dataflow region
   - **2.4** `<VAR>`은 데이터 흐름 영역 내에서 전달되어야 합니다.
   - **2.5** `<VAR>` cannot be a top-level function argument
   - **2.5** `<VAR>`은 최상위 함수 인수가 될 수 없습니다.
   - **2.6** `type=<T>` is optional:
   - **2.6** `type=<T>`는 선택 사항입니다.
     - If omitted, defaults to "fifo"
     - 생략하면 기본값은 "fifo"입니다.
     - If provided, `<T>` must be "fifo"
     - 제공된 경우 `<T>`는 "fifo"여야 합니다.
   - **2.7** `depth=<D>` is optional:
   - **2.7** `깊이=<D>`는 선택 사항입니다.
     - If omitted, uses default depth
     - 생략하면 기본 깊이를 사용합니다.
     - If provided, `<D>` must be a positive integer literal or const expression that evaluates to a positive integer
     - 제공된 경우 `<D>`는 양의 정수 리터럴이거나 양의 정수로 평가되는 const 표현식이어야 합니다.

**3.
Interface Pragma Rules** (check if `#pragma HLS interface [mode=ap_fifo|axis] port=A` is used; `mode` is optional; for example, `mode=ap_fifo` or just `ap_fifo`)
인터페이스 Pragma 규칙**(`#pragma HLS 인터페이스 [mode=ap_fifo|axis] port=A`가 사용되는지 확인하세요. `mode`는 선택 사항입니다. 예를 들어 `mode=ap_fifo` 또는 그냥 `ap_fifo`)
   - **3.1** `A` must be an array (possibly multi-dimensional) or a scalar
   - **3.1** `A`는 배열(다차원 가능) 또는 스칼라여야 합니다.
   - **3.2** `A` must be passed within the dataflow region
   - **3.2** 'A'는 데이터 흐름 영역 내에서 전달되어야 합니다.
   - **3.3** `A` must be a top function argument
   - **3.3** `A`는 최상위 함수 인수여야 합니다.
   - **3.4** `mode` must be either `ap_fifo` or `axis`
   - **3.4** `mode`는 `ap_fifo` 또는 `axis`여야 합니다.

**4.
Struct Element Type Rules** (skip if array element type is not a struct type)
구조체 요소 유형 규칙**(배열 요소 유형이 구조체 유형이 아닌 경우 건너뛰기)
If the array element type is a struct type, ensure that the whole struct is prepared in a separate struct variable and then stored at once in the array.
배열 요소 유형이 구조체 유형인 경우 전체 구조체가 별도의 구조체 변수로 준비된 다음 배열에 한 번에 저장되는지 확인하세요.

**5.
Struct Copy Assignment Rules** (skip if array element type is not a struct type)
구조체 복사 할당 규칙**(배열 요소 유형이 구조체 유형이 아닌 경우 건너뛰기)
If the array element type is a struct type, ensure that no custom copy assignment operator is defined (such as `A& operator=(const A& a)`) except mark it as default (such as `A& operator=(const A& a) = default`) for that struct type.
배열 요소 유형이 구조체 유형인 경우 해당 구조체 유형에 대해 기본값(예: `A& 연산자=(const A& a) = default`)으로 표시하는 것을 제외하고는 사용자 지정 복사 할당 연산자가 정의되지 않았는지 확인하세요(예: `A& 연산자=(const A& a)`).

**6.
Stream Pragma Access Pattern Rules** (check only if stream pragma `#pragma HLS stream variable=A type=T depth=D` is used)
스트림 프라그마 액세스 패턴 규칙**(스트림 프라그마 `#pragma HLS 스트림 변수=A 유형=T 깊이=D`가 사용되는 경우에만 확인)
Ensure that each accessed array element is read exactly once, written exactly once, and that the read/write order is consistent:
액세스된 각 배열 요소가 정확히 한 번 읽고, 정확히 한 번 기록되고, 읽기/쓰기 순서가 일관되는지 확인하세요.
   - **6.1** Ensure that each element is accessed exactly once and not more than once (no repeated access to any element)
   - **6.1** 각 요소는 정확히 한 번만 액세스되고 한 번 이상은 액세스되지 않도록 해야 합니다(어떤 요소에도 반복적으로 액세스하지 않음).
   - **6.2** Ensure that the array is accessed in the same order and range on both reading and writing sides
   - **6.2** 읽기 및 쓰기 측면 모두에서 동일한 순서와 범위로 어레이에 액세스하는지 확인하세요.
   - **6.3** Ensure the iteration order must be the same (e.g., both ascending or both descending)
   - **6.3** 반복 순서가 동일해야 합니다(예: 둘 다 오름차순 또는 둘 다 내림차순).
   - **6.4** Each side must access the same number of elements (e.g., if write accesses N-1 elements, read must also access N-1 elements, not N)
   - **6.4** 각 측은 동일한 수의 요소에 액세스해야 합니다. 예를 들어 쓰기가 N-1개의 요소에 액세스하는 경우 읽기는 N이 아닌 N-1개의 요소에도 액세스해야 합니다.

---

#### Verdict:
#### 평결:
If **all** applicable criteria are met, the code follows the **recommended coding style for array-to-stream conversion**.
**모든** 해당 기준이 충족되면 코드는 **배열에서 스트림으로의 변환에 권장되는 코딩 스타일**을 따릅니다.
Otherwise, it does **not** follow the recommended coding style.
그렇지 않으면 권장 코딩 스타일을 따르지 **않습니다**.



---

## When Asked to Fix Code to Follow Array-to-Stream Style
## Array-to-Stream 스타일을 따르도록 코드를 수정하라는 요청을 받은 경우

If the user asks you to **fix** code so it follows the recommended array-to-stream style:
사용자가 권장되는 배열-스트림 스타일을 따르도록 코드를 **수정**하도록 요청하는 경우:

1. Identify the failing rule(s)
1. 실패한 규칙 식별
2. Apply the minimal fix while preserving behavior
2. 동작을 유지하면서 최소한의 수정 사항을 적용합니다.
3. Output the **complete fixed code** in a fenced code block:
3. 분리된 코드 블록에서 **완전한 고정 코드**를 출력합니다.
   ````
   ```cpp
// fixed code here
// 여기서 코드를 수정했습니다.
   ```
   ````
4. Then end your response with this JSON object as the very last thing:
4. 그런 다음 마지막으로 이 JSON 개체를 사용하여 응답을 종료합니다.
   ```
   {"verdict": "FIXED_FOLLOWS_STYLE", "explanation": "<brief reason>"}
   ```
or if you cannot fix it:
또는 문제를 해결할 수 없는 경우:
   ```
   {"verdict": "COULD_NOT_FIX", "explanation": "<reason>"}
   ```
