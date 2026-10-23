<!-- ko-translation: enabled -->
---
name: hls-dataflow
description: 'Analyze whether a given C/C++ code snippet contains a canonical dataflow region for Vitis HLS. Keywords: dataflow, HLS, pragma, canonical, stream, PIPO, hls::task'
license: MIT
argument-hint: "[<TOP_FUNCTION — top-level HLS function name e.g. 'Kernel'>]"
metadata:
  author: "Alexandre Isoard, Alain Darte"
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
You are focused on analyzing whether a dataflow region follows the canonical dataflow coding style.
데이터 흐름 영역이 표준 데이터 흐름 코딩 스타일을 따르는지 여부를 분석하는 데 중점을 둡니다.
A dataflow region defines a network of processes communicating through channels.
데이터 흐름 영역은 채널을 통해 통신하는 프로세스의 네트워크를 정의합니다.
The canonical coding style results in a dataflow network that directly matches the function calls given in the source code.
표준 코딩 스타일은 소스 코드에 제공된 함수 호출과 직접적으로 일치하는 데이터 흐름 네트워크를 생성합니다.
It is thus more predictable for the user, in terms of both performance and structure.
따라서 성능과 구조 측면에서 사용자가 더 예측하기 쉽습니다.
Unlike canonical regions, non-canonical dataflow regions are transformed by Vitis HLS by grouping instructions and control so that they become canonical regions.
정규 영역과 달리 비정규 데이터 흐름 영역은 지침과 제어를 그룹화하여 Vitis HLS에 의해 변환되어 정규 영역이 됩니다.
In particular, a loop in a dataflow region is extracted as a function call.
특히 데이터 흐름 영역의 루프는 함수 호출로 추출됩니다.

#### Concepts:
#### 개념:
- **Dataflow pragma**: `#pragma HLS dataflow` or `#pragma HLS dataflow disable_start_propagation`
- **데이터 흐름 pragma**: `#pragma HLS 데이터 흐름` 또는 `#pragma HLS 데이터 흐름 비활성화_시작_전파`
- **Dataflow region**: A region that contains directly a dataflow pragma
- **데이터 흐름 지역**: 데이터 흐름 pragma를 직접 포함하는 지역
- **Function dataflow region**: A function whose body is a dataflow region
- **함수 데이터 흐름 영역**: 본문이 데이터 흐름 영역인 함수
- **Loop dataflow region**: A loop whose body is a dataflow region
- **루프 데이터 흐름 영역**: 본문이 데이터 흐름 영역인 루프
- **Dataflow function**: A function that is either a function dataflow region or that contains only a loop dataflow region
- **데이터 흐름 함수**: 함수 데이터 흐름 영역이거나 루프 데이터 흐름 영역만 포함하는 함수
- **Dataflow function arguments**: Arguments of a dataflow function
- **데이터 흐름 함수 인수**: 데이터 흐름 함수의 인수
- **Global variable**: A variable defined outside of all functions
- **전역변수**: 모든 함수 외부에서 정의된 변수
- **Static variable**: A variable declared as `static` within a function
- **정적 변수**: 함수 내에서 '정적'으로 선언된 변수
- **m_axi type**: Marked with `#pragma HLS interface m_axi`
- **m_axi 유형**: `#pragma HLS 인터페이스 m_axi`로 표시됨
- **Top function**: The top-level HLS function, get it via argument `<TOP_FUNCTION>`, if not provided, call `/hls-component-basic-info` skill to get top_function which is the top-level function name
- **상위 함수**: 최상위 HLS 함수, `<TOP_FUNCTION>` 인수를 통해 가져옵니다. 제공되지 않은 경우 `/hls-comComponent-basic-info` 스킬을 호출하여 최상위 함수 이름인 top_function을 가져옵니다.
- **Top dataflow function**: The top function if it is a dataflow function
- **상위 데이터 흐름 함수**: 데이터 흐름 함수인 경우 최상위 함수
- **Overlapping dataflow region**: A dataflow region that is a top dataflow function, or the body of a loop dataflow region, or a function dataflow region called by an overlapping dataflow region
- **겹치는 데이터흐름 영역**: 최상위 데이터흐름 함수인 데이터흐름 영역, 루프 데이터흐름 영역의 본체, 또는 겹치는 데이터흐름 영역에 의해 호출되는 함수 데이터흐름 영역
- **Read process**: A process (function/task/block) that reads a variable's value
- **읽기 프로세스**: 변수의 값을 읽는 프로세스(함수/태스크/블록)
- **Write process**: A process (function/task/block) that writes a variable's value
- **쓰기 프로세스**: 변수의 값을 쓰는 프로세스(함수/태스크/블록)
- **Read/write process**: A process that both reads and writes the same variable
- **읽기/쓰기 프로세스**: 동일한 변수를 읽고 쓰는 프로세스
- **Class instance rule**: If the dataflow region is defined within a non-static class method, treat `this` (and its member variables) as dataflow function arguments
- **클래스 인스턴스 규칙**: 데이터 흐름 영역이 비정적 클래스 메서드 내에 정의된 경우 `this`(및 해당 멤버 변수)를 데이터 흐름 함수 인수로 처리합니다.

#### Scope:
#### 범위:
- Strictly follow rules 1–10; each rule should only be checked within that specific rule—do not extend
- 규칙 1~10을 엄격히 따르십시오. 각 규칙은 해당 특정 규칙 내에서만 확인해야 합니다. 확장하지 마세요.
- Follow the literal text of each rule strictly; do not extend rules based on general design principles
- 각 규칙의 문자 그대로의 텍스트를 엄격하게 따르십시오. 일반적인 설계 원칙에 따라 규칙을 확장하지 마십시오.
- For loop dataflow regions, analyze only a single iteration
- 루프 데이터 흐름 영역의 경우 단일 반복만 분석

---

#### Analysis Steps:
#### 분석 단계:

**1.
Pragma Presence**
프라그마 존재**
Validate that the code must contain `#pragma HLS dataflow` or `#pragma HLS dataflow disable_start_propagation`.
코드에 '#pragma HLS dataflow' 또는 '#pragma HLS dataflow disable_start_propagation'이 포함되어 있는지 확인하세요.

**2.
Pragma Location**
프라그마 위치**
Validate that the dataflow pragma is in a loop body or function body.
데이터 흐름 pragma가 루프 본문 또는 함수 본문에 있는지 확인합니다.

**3.
Loop Dataflow Region Rules (skip if pragma is not inside a loop body)**
루프 데이터 흐름 영역 규칙(pragma가 루프 본문 내부에 없으면 건너뛰기)**
   - **3.1** The loop must be a `for`-loop.
   - **3.1** 루프는 'for' 루프여야 합니다.
   - **3.2** The parent function of the for-loop contains only this for-loop and no other code except local variable declarations and pragma statements.
   - **3.2** for 루프의 상위 함수에는 이 for 루프만 포함되어 있으며 지역 변수 선언 및 pragma 문을 제외한 다른 코드는 포함되어 있지 않습니다.
   - **3.3** The loop counter is declared in the loop header and is of `int` type.
   - **3.3** 루프 카운터는 루프 헤더에 선언되며 `int` 유형입니다.
   - **3.4** The iterator must be initialized to a non-negative integer constant in the loop header.
   - **3.4** 반복자는 루프 헤더에서 음수가 아닌 정수 상수로 초기화되어야 합니다.
   - **3.5** The loop bounds must be a non-negative integer constant or a scalar argument of the function.
   - **3.5** 루프 범위는 음수가 아닌 정수 상수이거나 함수의 스칼라 인수여야 합니다.
   - **3.6** The step of the iterator must be a positive integer constant.
   - **3.6** 반복자의 단계는 양의 정수 상수여야 합니다.

**4.
Dataflow Region Content Rules (check line-by-line)**
데이터 흐름 영역 콘텐츠 규칙(라인별로 확인)**
   - **4.1** The dataflow region must not contain any statements other than:
   - **4.1** 데이터 흐름 영역에는 다음 이외의 문이 포함되어서는 안 됩니다.
     - Pragma statements
     - 프라그마 문
     - Local variable declarations (of any kind, including class objects instantiations), which become dataflow channels
     - 데이터 흐름 채널이 되는 지역 변수 선언(클래스 객체 인스턴스화를 포함한 모든 종류)
     - Function call statements (including `hls::task` instantiations and class method calls), which become dataflow processes
     - 데이터 흐름 프로세스가 되는 함수 호출 문(`hls::task` 인스턴스화 및 클래스 메서드 호출 포함)
   - **4.2** If `hls::task` is present, it must be declared as `hls_thread_local`:
   - **4.2** `hls::task`가 있는 경우 `hls_thread_local`로 선언해야 합니다.
     ```cpp
     hls_thread_local hls::task t1(proc, arg1, arg2, arg3);
     ```
   - **4.3** If `hls::task` is present, `hls::stream` and `hls::stream_of_blocks` within it must be declared as `hls_thread_local`.
   - **4.3** `hls::task`가 있는 경우 `hls::stream` 및 `hls::stream_of_blocks`는 `hls_thread_local`로 선언되어야 합니다.
   - **4.4** All local variables within the dataflow region must be declared as non-static.
   - **4.4** 데이터 흐름 영역 내의 모든 지역 변수는 비정적으로 선언되어야 합니다.
   - **4.5** Local variables with initializing default constructors (e.g., `std::complex`) lead to implicit write processes.
   - **4.5** 기본 생성자를 초기화하는 지역 변수(예: `std::complex`)는 암시적 쓰기 프로세스로 이어집니다.
   - To avoid this, use the `no_ctor` attribute:
   - 이를 방지하려면 'no_ctor' 속성을 사용하세요.
     ```cpp
     std::complex<float> arr[SIZE] __attribute__((no_ctor));
     ```
   - **4.6** Function calls must not pass array element values (e.g., `arg_mem[offset_var]`, `mem[i]`, `array[index]`) as arguments.
   - **4.6** 함수 호출은 배열 요소 값(예: `arg_mem[offset_var]`, `mem[i]`, `array[index]`)을 인수로 전달해서는 안 됩니다.
   - Only pass the entire array/pointer variable itself.
   - 전체 배열/포인터 변수 자체만 전달하세요.
   - **4.7** Function calls must not perform arithmetic or logical operations on arguments at the call site (e.g., use `i`, not `i + 1`; use `ptr`, not `ptr + offset`).
   - **4.7** 함수 호출은 호출 위치에서 인수에 대해 산술 또는 논리 연산을 수행해서는 안 됩니다(예: `i + 1`이 아닌 `i` 사용, `ptr + offset`이 아닌 `ptr` 사용).
   - **4.8** List all function parameters and explicitly verify type compatibility between function parameters and arguments.
   - **4.8** 모든 함수 매개변수를 나열하고 함수 매개변수와 인수 간의 유형 호환성을 명시적으로 확인합니다.
   - No type conversion is allowed during function calls, except value-to-reference conversions.
   - 값-참조 변환을 제외하고 함수 호출 중에는 유형 변환이 허용되지 않습니다.

**5.
Multiple Invocation Rules**
다중 호출 규칙**
If a function is invoked multiple times within a dataflow region, ensure static variables follow these rules:
데이터 흐름 영역 내에서 함수가 여러 번 호출되는 경우 정적 변수가 다음 규칙을 따르는지 확인하세요.
   - **5.1** If the function is invoked by a loop within the dataflow region:
   - **5.1** 데이터 흐름 영역 내의 루프에 의해 함수가 호출되는 경우:
     - **5.1.a** If the same function is called in each iteration (repeatedly across all iterations), skip rules 5, 5.1, 5.2.
     - **5.1.a** 각 반복에서 동일한 함수가 호출되는 경우(모든 반복에서 반복적으로) 규칙 5, 5.1, 5.2를 건너뜁니다.
     - **5.1.b** If the function is called multiple times within a single iteration, ensure no static variables within that function are accessed.
     - **5.1.b** 단일 반복 내에서 함수가 여러 번 호출되는 경우 해당 함수 내의 정적 변수에 액세스하지 않는지 확인하십시오.
   - **5.2** If the function is invoked multiple times by a non-loop within the dataflow region, ensure no static variables within that function are accessed.
   - **5.2** 데이터 흐름 영역 내의 비루프에 의해 함수가 여러 번 호출되는 경우 해당 함수 내의 정적 변수에 액세스하지 않는지 확인하십시오.

**6.
Dataflow Function Argument Rules**
Dataflow 함수 인수 규칙**
List dataflow function arguments before checking.
확인하기 전에 데이터 흐름 함수 인수를 나열합니다.
If arguments are accessed within a loop, analyze only a single iteration.
루프 내에서 인수에 액세스하는 경우 단일 반복만 분석하십시오.
   - **6.1** If the argument is of type `hls::stream`, ensure it is accessed (read or written) by only one dataflow process.
   - **6.1** 인수가 'hls::stream' 유형인 경우 하나의 데이터 흐름 프로세스에서만 액세스(읽기 또는 쓰기)되는지 확인하세요.
   - **6.2** If the argument is an array of m_axi type, it must be accessed by only a single process, OR if accessed by both a read process and a write process, the write process must appear textually after the read process.
   - **6.2** 인수가 m_axi 유형의 배열인 경우 단일 프로세스에서만 액세스해야 합니다. 또는 읽기 프로세스와 쓰기 프로세스 모두에서 액세스하는 경우 쓰기 프로세스는 읽기 프로세스 다음에 텍스트로 나타나야 합니다.
   - **6.3** If the argument is an array and not of m_axi type, ensure:
   - **6.3** 인수가 배열이고 m_axi 유형이 아닌 경우 다음을 확인하세요.
     - If written, it is accessed (read or written) by only one process
     - 쓰여지면 하나의 프로세스에서만 액세스(읽기 또는 쓰기)됩니다.
     - If read-only, it is read by at most two processes
     - 읽기 전용인 경우 최대 두 개의 프로세스에서 읽습니다.
   - **6.4** If a scalar argument (passed by address or reference) of an overlapping dataflow region has a process that reads the scalar strictly before a *different* process that writes the scalar, ensure that this scalar is always written in the region before it is read.
   - **6.4** 겹치는 데이터 흐름 영역의 스칼라 인수(주소 또는 참조로 전달됨)에 스칼라를 쓰는 *다른* 프로세스보다 엄격하게 스칼라를 읽는 프로세스가 있는 경우 이 스칼라가 읽기 전에 항상 영역에 기록되는지 확인하세요.
   - When applying this rule, consider that a conditional write corresponds to a read followed by a write (in the same process).
   - 이 규칙을 적용할 때 조건부 쓰기는 (동일한 프로세스에서) 읽기와 쓰기에 해당한다는 점을 고려하세요.
   - If the same single process performs both the read and the write of the scalar, that access does not trigger this rule.
   - 동일한 단일 프로세스가 스칼라 읽기 및 쓰기를 모두 수행하는 경우 해당 액세스는 이 규칙을 트리거하지 않습니다.
   - If the dataflow region is not an overlapping dataflow region, skip this entire rule.
   - 데이터 흐름 영역이 겹치는 데이터 흐름 영역이 아닌 경우 이 전체 규칙을 건너뜁니다.
   - The reason for this rule is that a flow "loop-carried" dependence within dataflow is supported only within a single process.
   - 이 규칙의 이유는 데이터 흐름 내의 흐름 "루프 전달" 종속성이 단일 프로세스 내에서만 지원되기 때문입니다.

**7.
Dataflow Function Global/Static Scalar Rule**
데이터 흐름 함수 전역/정적 스칼라 규칙**
Ensure all scalar-type global/static variables accessed within an overlapping dataflow region follow the rule 6.4 (substituting scalar argument by global/static scalar variable).
겹치는 데이터 흐름 영역 내에서 액세스되는 모든 스칼라 유형 전역/정적 변수가 규칙 6.4(스칼라 인수를 전역/정적 스칼라 변수로 대체)를 따르는지 확인하세요.

**8.
Top Dataflow Function Scalar Argument Rules**
상위 데이터 흐름 함수 스칼라 인수 규칙**
If `<TOP_FUNCTION>` function is a top dataflow function, and a scalar argument of the top function is used in a dataflow region, recursively included in dataflow regions up to the top, ensure that if written, it is accessed (read or written) by only one process.
`<TOP_FUNCTION>` 함수가 최상위 데이터 흐름 함수이고 최상위 함수의 스칼라 인수가 데이터 흐름 영역에서 사용되어 최상위까지 데이터 흐름 영역에 반복적으로 포함되는 경우, 작성된 경우 하나의 프로세스에서만 액세스(읽기 또는 쓰기)되도록 해야 합니다.

**9.
Global/Static Array Rule**
전역/정적 배열 규칙**
If an array-type global/static variable is accessed within the dataflow region, validate that no more than one dataflow process reads or writes the global/static array in the dataflow region.
데이터 흐름 영역 내에서 배열 유형 전역/정적 변수에 액세스하는 경우 데이터 흐름 영역에서 전역/정적 배열을 읽거나 쓰는 데이터 흐름 프로세스가 두 개 이상인지 확인합니다.

**10.
Local Variable Rules**
지역 변수 규칙**
Validate local variables within the dataflow region.
데이터 흐름 영역 내의 지역 변수를 검증합니다.
For loop dataflow regions, analyze a single iteration only.
루프 데이터 흐름 영역의 경우 단일 반복만 분석합니다.
   - **10.1** Any local variable within a dataflow region must be a scalar, an array, `hls::stream`, `hls::stream_of_blocks`, or a class containing any of these types.
   - **10.1** 데이터 흐름 영역 내의 모든 지역 변수는 스칼라, 배열, `hls::stream`, `hls::stream_of_blocks` 또는 이러한 유형을 포함하는 클래스여야 합니다.
   - **10.2** If a local array is marked with `#pragma HLS bind_storage variable=... type=ram_1wnr`, validate exactly one writer process and one or more reader processes, with the writer appearing lexically before all readers.
   - **10.2** 로컬 배열이 '#pragma HLS 바인딩_storage 변수=... type=ram_1wnr'로 표시된 경우 정확히 하나의 작성기 프로세스와 하나 이상의 판독기 프로세스를 검증합니다. 이때 기록기는 모든 판독기 앞에 어휘적으로 표시됩니다.
   - **10.3** If a local array is not marked with `#pragma HLS bind_storage variable=... type=ram_1wnr`, validate exactly one writer process and one reader process, with the writer appearing lexically before the reader.
   - **10.3** 로컬 배열이 '#pragma HLS 바인딩_storage 변수=... type=ram_1wnr'로 표시되지 않은 경우, 판독기 앞에 어휘적으로 표시되는 기록기를 사용하여 정확히 하나의 기록기 프로세스와 하나의 판독기 프로세스를 검증합니다.
   - **10.4** If a local variable is of `hls::stream` type, validate exactly one writer process and one reader process.
   - **10.4** 지역 변수가 `hls::stream` 유형인 경우 정확히 하나의 기록기 프로세스와 하나의 판독기 프로세스를 검증합니다.
   - **10.5** If a local variable is of `hls::stream_of_blocks` type, validate:
   - **10.5** 지역 변수가 `hls::stream_of_blocks` 유형인 경우 다음을 확인하세요.
     - **a.** It is declared as:
     - **a.** 다음과 같이 선언됩니다.
       ```cpp
       #include "hls_streamofblocks.h"
       ...
       hls::stream_of_blocks<block_type, depth> block;
       // or
       hls::stream_of_blocks<block_type> block;
       ```
     - **b.** Exactly one write process, and it is using `hls::write_lock lock(block)` in the region that writes
     - **b.** 정확히 하나의 쓰기 프로세스이며, 쓰기 영역에서 `hls::write_lock lock(block)`을 사용하고 있습니다.
     - **c.** Exactly one read process, and it is using `hls::read_lock lock(block)` in the region that reads
     - **c.** 정확히 하나의 읽기 프로세스이며, 읽는 영역에서 `hls::read_lock lock(block)`을 사용하고 있습니다.

---

#### Verdict:
#### 평결:
If **all** criteria are met, the code forms a **canonical dataflow region**.
**모든** 기준이 충족되면 코드는 **표준 데이터 흐름 영역**을 형성합니다.
Otherwise, it is **not canonical**.
그렇지 않으면 **표준이 아닙니다**.
