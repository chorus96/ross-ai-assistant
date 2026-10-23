<!-- ko-translation: enabled -->
---
name: hls-perf-pragma
description: Vitis HLS Performance Pragma — calculate target_ti from a throughput target, cascade through architecture and loops, and present the pragma placement table for user confirmation. Run this before placing any #pragma HLS performance.
license: MIT
argument-hint: <throughput-target e.g. "140 FPS" | "500 Msps" | "1 GFLOPS" | "minimize II">
metadata:
  author: "Sai Akhil Ayyagari"
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
# Skill: perf-pragma
# 스킬: perf-pragma

Calculate `target_ti` from the user's throughput target and produce a cascade table for review.
사용자의 처리량 목표에서 'target_ti'를 계산하고 검토용 캐스케이드 테이블을 생성합니다.
This skill is a pure calculator — it does not run synthesis or interpret reports.
이 기술은 순수한 계산기입니다. 즉, 합성을 실행하거나 보고서를 해석하지 않습니다.

**Throughput target:** $ARGUMENTS
**처리량 목표:** $ARGUMENTS

Print at the start:
시작 시 인쇄:
```
─────────────────────────────────────────────────────
[perf-pragma]  flow overview
  Steps 0 → 5
  Step 0  — Gather inputs (clock, architecture, loop structure)
  Step 1  — Calculate top-level target_ti
  Step 2  — Cascade through architecture
  Step 3  — Cascade through loops → build table
  Step 4  — Present table for review
  Step 5  — Apply pragmas to source code
─────────────────────────────────────────────────────
```

---

---

## Step 0: Gather inputs
## 0단계: 입력 수집

The throughput requirement comes from $ARGUMENTS.
처리량 요구 사항은 $ARGUMENTS에서 비롯됩니다.

**Read the configuration and source code to determine:**
**구성 및 소스 코드를 읽고 다음 사항을 확인하세요.**

1. **Clock frequency** — Read from the `.cfg` file:
1. **클럭 주파수** — `.cfg` 파일에서 읽습니다.
   ```bash
   CFG_FILE=$(ls *.cfg)
   CLOCK_NS=$(grep "^clock=" $CFG_FILE | cut -d'=' -f2)
   clock_hz = 1e9 / CLOCK_NS
   ```

2. **Architecture type** — Inspect the top-level function:
2. **아키텍처 유형** — 최상위 기능을 검사합니다.
   - Look for `#pragma HLS dataflow` or `hls::task` → dataflow pipeline
   - `#pragma HLS dataflow` 또는 `hls::task`를 찾습니다 → 데이터 흐름 파이프라인
   - Multiple function calls in sequence → sequential functions
   - 순차적으로 여러 함수 호출 → 순차 함수
   - Single pipelined function otherwise
   - 그렇지 않으면 단일 파이프라인 함수

3. **Loop structure** — Identify loop nesting levels and trip counts:
3. **루프 구조** — 루프 중첩 수준 및 이동 횟수를 식별합니다.
   - Extract all labeled loops (e.g., `Row_Loop:`, `Col_Loop:`)
   - 레이블이 지정된 모든 루프 추출(예: `Row_Loop:`, `Col_Loop:`)
   - Determine nesting depth
   - 중첩 깊이 결정
   - **Trip counts:** Read loop bounds from source code
   - **여행 횟수:** 소스 코드에서 루프 범위 읽기
   - **If trip counts are runtime parameters (not compile-time constants):**: **warn the user** : "Performance pragma requires loop trip counts.
   - **여행 횟수가 런타임 매개변수(컴파일 시간 상수가 아님)인 경우:**: **사용자에게 경고**: "성능 pragma에는 루프 여행 횟수가 필요합니다.
   - Please specify the trip count values for: [list loops with variable bounds]"
   - [변수 범위가 있는 루프 목록]"에 대한 이동 횟수 값을 지정하십시오.

4. **Throughput conversion** — If throughput is given as data rate (MB/s, GB/s):
4. **처리량 변환** — 처리량이 데이터 속도(MB/s, GB/s)로 제공되는 경우:

**Read source code to determine input data size:**
**입력 데이터 크기를 결정하려면 소스 코드를 읽어보세요.**
   
   ```bash
   # Find top function
   CFG_FILE=$(ls *.cfg)
   TOP_FUNCTION=$(grep "^top=" $CFG_FILE | cut -d'=' -f2)
   
   # Find source file containing top function
   SRC_FILE=$(grep -l "void $TOP_FUNCTION" src/*.cpp)
   ```
   
Read the function signature.
함수 서명을 읽어보세요.
The function will have multiple inputs/outputs. **Identify the input arrays** and calculate:
이 함수에는 여러 개의 입력/출력이 있습니다. **입력 배열을 식별**하고 다음을 계산합니다.
   
   ```
   bytes_per_invocation = total_input_array_size_in_bytes
   ```
   
**Convert data rate to invocations per second:**
**데이터 속도를 초당 호출로 변환:**
   
   ```
   data_rate_bytes_per_sec = throughput_value × (1,000,000 for MB/s, 1,000,000,000 for GB/s)
   invocations_per_sec = data_rate_bytes_per_sec / bytes_per_invocation
   ```
   
Use this `invocations_per_sec` value in Step 1.
1단계에서 이 'invocations_per_sec' 값을 사용하세요.

5. **Computational throughput** — If throughput is given as GOPS or GFLOPS:
5. **계산 처리량** — 처리량이 GOPS 또는 GFLOPS로 제공되는 경우:

   ```bash
   # Parse value (e.g., "100 GOPS" → 100e9 ops/sec)
   THROUGHPUT_VALUE=$(echo "$ARGUMENTS" | grep -oE '[0-9.]+')
   OPS_PER_SEC=$(echo "$THROUGHPUT_VALUE * 1e9" | bc)
   
   # Read source code to determine operations per invocation
   # Look for dimensions in defines or comments
   
   # Convert: invocations/sec = ops/sec / ops_per_invocation
   # Note: 1 OP = 1 MAC = 2 FLOPs
   invocations_per_sec=$(echo "$OPS_PER_SEC / $OPS_PER_INVOCATION" | bc)
   ```
   
If operation count cannot be determined, warn the user to specify it or use FPS/MB/s instead.
작업 횟수를 확인할 수 없는 경우 사용자에게 이를 지정하도록 경고하거나 대신 FPS/MB/s를 사용하세요.
   
Use the calculated `invocations_per_sec` in Step 1.
1단계에서 계산된 `invocations_per_sec`을 사용합니다.

Print after Step 0:
0단계 이후 인쇄:
```
─────────────────────────────────────────────────────
[perf-pragma]  Step 0 — done
  ✓ Step 0  clock=<CLOCK_NS> ns  arch=<single/sequential/dataflow>  loops=<N levels>
  ← NEXT    Step 1 — Calculate top-level target_ti
─────────────────────────────────────────────────────
```

---

## Step 1: Calculate top-level target_ti
## 1단계: 최상위 target_ti 계산

**Formula:**
**공식:**
```
target_ti = clock_hz / throughput_rate
```

Where `throughput_rate` is the target invocations per second (e.g., FPS, invocations/s, samples/s).
여기서 '처리량_속도'는 초당 대상 호출(예: FPS, 호출/초, 샘플/초)입니다.

Print after Step 1:
1단계 이후 인쇄:
```
─────────────────────────────────────────────────────
[perf-pragma]  Step 1 — done
  ✓ Step 0  Inputs gathered
  ✓ Step 1  top-level target_ti = <N> cycles  (<throughput> @ <CLOCK_NS> ns)
  ← NEXT    Step 2 — Cascade through architecture
─────────────────────────────────────────────────────
```

---

## Step 2: Cascade through architecture
## 2단계: 아키텍처를 통한 캐스케이드

### Case A — Single function
### 사례 A - 단일 기능

The full `target_ti` applies to the one function.
전체 `target_ti`는 하나의 함수에 적용됩니다.
Go to Step 3.
3단계로 이동합니다.

### Case B — Sequential functions
### 사례 B — 순차 함수

Functions execute one after the other — total latency is the **sum**:
함수는 차례로 실행됩니다. 총 지연 시간은 **합계**입니다.
```
target_ti_total = target_ti_fn1 + target_ti_fn2 + ...
```
Allocate per-function budget by trip-count ratio.
기능별 예산을 운행 횟수 비율에 따라 할당합니다.
Apply the top-level pragma first; add function- or loop-scope pragmas only after synthesis confirms the bottleneck.
최상위 수준 pragma를 먼저 적용합니다. 합성이 병목 현상을 확인한 후에만 함수 또는 루프 범위 pragma를 추가하세요.

### Case C — Dataflow pipeline (including `hls::task`)
### 사례 C - 데이터 흐름 파이프라인(`hls::task` 포함)

Throughput = **max** across all stages — the slowest stage sets the rate:
처리량 = 모든 단계에서 **최대** — 가장 느린 단계에서 속도가 설정됩니다.
```
Design Throughput = max(TI_stage1, TI_stage2, ...)
```

`target_ti` is a **ceiling** that applies equally to every stage.
`target_ti`는 모든 스테이지에 동일하게 적용되는 **천장**입니다.
A stage whose budgeter-allocated Target TI exceeds the top-level target_ti will dominate throughput and cause a miss — even if that stage shows "yes" for its own allocated target.
예산 담당자가 할당한 대상 TI가 최상위 target_ti를 초과하는 단계는 처리량을 지배하고 누락을 유발합니다. 해당 단계가 자체 할당된 대상에 대해 "예"로 표시되는 경우에도 마찬가지입니다.

**Cascading strategy:**
**계단식 전략:**
1. Place the top-level pragma only.
1. 최상위 수준 pragma만 배치합니다.
2. For dataflow stages that need per-stage constraints, add per-stage pragma = top-level `target_ti` (same value, not a fraction), then cascade into that stage's loops:
2. 단계별 제약이 필요한 데이터 흐름 단계의 경우 단계별 pragma = 최상위 `target_ti`(분수가 아닌 동일한 값)를 추가한 다음 해당 단계의 루프에 계단식으로 적용합니다.
   ```
   stage_loop_II = ceil(target_ti / stage_loop_trip_count)
   ```

Print after Step 2:
2단계 이후 인쇄:
```
─────────────────────────────────────────────────────
[perf-pragma]  Step 2 — done
  ✓ Step 0  Inputs gathered
  ✓ Step 1  top-level target_ti = <N> cycles
  ✓ Step 2  Architecture: <single/sequential/dataflow>  budget allocated
  ← NEXT    Step 3 — Cascade through loops
─────────────────────────────────────────────────────
```

---

## Step 3: Cascade through loops — build the table
## 3단계: 루프를 통한 계단식 배열 - 테이블 구축

### 3a.
### Classify trip counts before cascading
### 계단식으로 연결하기 전에 이동 횟수 분류

Inspect the source for each loop.
각 루프의 소스를 검사합니다.
If any trip count is variable (runtime argument or input-dependent), **stop here** and return to the caller (`/hls-optimize` Step 1d) to run csim first.
이동 횟수가 가변적이면(런타임 인수 또는 입력에 따라 다름) **여기에서 중지**하고 호출자(`/hls-optimize` 1d단계)로 돌아가 csim을 먼저 실행합니다.
Do not guess or estimate variable trip counts.
가변적인 이동 횟수를 추측하거나 추정하지 마십시오.

| Type<br>유형 | Example<br>예 | Status<br>상태 |
|---|---|---|
| Compile-time constant<br>컴파일타임 상수 | `for (int i = 0; i < 64; i++)`<br>`for (int i = 0; i < 64; i++)` | ✓ Proceed<br>✓ 진행 |
| Fixed template parameter<br>고정 템플릿 매개변수 | `for (int i = 0; i < N; i++)` — N known at instantiation<br>`for (int i = 0; i < N; i++)` — 인스턴스화 시 알려진 N | ✓ Proceed<br>✓ 진행 |
| Runtime parameter<br>런타임 매개변수 | `for (int i = 0; i < height; i++)` — height is a function arg<br>`for (int i = 0; i < height; i++)` — 높이는 함수 인수입니다. | ✗ Run csim first<br>✗ csim을 먼저 실행하세요 |
| Input-dependent<br>입력에 따라 다름 | count driven by data<br>데이터에 따른 계산 | ✗ Run csim first<br>✗ csim을 먼저 실행하세요 |

### 3b.
### Propagate the budget
### 예산을 전파하다

Propagate the budget downward through each loop level.
각 루프 수준을 통해 예산을 하향 전파합니다.
The formula depends on loop nesting depth:
공식은 루프 중첩 깊이에 따라 달라집니다.

**General pattern:** Each nested loop divides its parent's target_ti by the parent's trip count.
**일반 패턴:** 각 중첩 루프는 상위의 target_ti를 상위의 이동 횟수로 나눕니다.

**Case 1: Single loop in function**
**사례 1: 기능의 단일 루프**
```
Loop_target_ti = Function_target_ti      (assuming negligible function overhead)
```

**Case 2: Two-level nested loops**
**사례 2: 2단계 중첩 루프**
```
Outer_loop_target_ti = Function_target_ti                    (overhead usually negligible)
Inner_loop_target_ti = Outer_loop_target_ti / outer_trip_count
```

**Case 3: Three-level nested loops**
**사례 3: 3단계 중첩 루프**
```
Outer_loop_target_ti  = Function_target_ti
Middle_loop_target_ti = Outer_loop_target_ti / outer_trip_count
Inner_loop_target_ti  = Middle_loop_target_ti / middle_trip_count
```


Fill this table using the known or profiled trip counts (example shows 2-level nesting):
알려졌거나 프로파일링된 이동 횟수를 사용하여 이 표를 채우세요(예: 2단계 중첩 표시).

| Level<br>레벨 | Formula<br>공식 | Value<br>가치 |
|---|---|---|
| Function target_ti<br>함수 target_ti | clock_hz / throughput_rate<br>clock_hz / 처리량_속도 | ? cycles<br>? 사이클 |
| Outer loop target_ti<br>외부 루프 target_ti | function_ti (overhead ≈ 0)<br>function_ti (오버헤드 ≒ 0) | ? cycles<br>? 사이클 |
| Inner loop target_ti<br>내부 루프 target_ti | outer_ti / outer_trip_count<br>외부_티/외부_여행_횟수 | ? cycles<br>? 사이클 |

### 3c.
### Detect scope types (function vs loop)
### 범위 유형 감지(함수 대 루프)

**Rule:** Only the top-level function gets a function-level pragma.
**규칙:** 최상위 수준 함수만 함수 수준 pragma를 가져옵니다.
All other pragmas target loops.
다른 모든 pragma는 루프를 대상으로 합니다.

**Detection logic:**
**탐지 논리:**

1. **Extract TOP function** from `*.cfg`:
1. **`*.cfg`에서 TOP 함수 추출**:
   ```bash
   TOP_FUNCTION=$(grep -h -m1 "^syn.top=" -- *.cfg | cut -d'=' -f2)
   ```

2. **Find loop labels** in source files:
2. 소스 파일에서 **루프 라벨 찾기**:
   ```bash
   grep -Pn "^\s*\w+:\s*for\s*\(" <source_files>
   ```
This captures patterns like:
이는 다음과 같은 패턴을 캡처합니다.
   - `Row_Loop: for (int r = 0; r < ROWS; r++)`
   - `Row_Loop: for (int r = 0; r < ROWS; r++)`
   - `Col_Loop: for (int c = 0; c < COLS; c++)`
   - `Col_Loop: for (int c = 0; c < COLS; c++)`

3. **Build enhanced table** with Type column:
3. 유형 열을 사용하여 **향상된 테이블 구축**:

| Target<br>대상          | Type<br>유형     | target_ti<br>target_ti | Pragma Position<br>프라그마 포지션      |
   |-----------------|----------|-----------|----------------------|
| <TOP_FUNCTION><br><TOP_FUNCTION>  | function<br>기능 | <value><br><값>   | Position 1 (function body)<br>위치 1(함수 본문) |
| Row_Loop<br>Row_Loop        | loop<br>루프     | <value><br><값>   | Position 2 (loop body)<br>위치 2(루프 본문)     |
| Col_Loop<br>Col_Loop        | loop<br>루프     | <value><br><값>   | Position 2 (loop body)<br>위치 2(루프 본문)     |

**Helper script** (optional): Use `../hls-scripts/detect_loops.sh` to extract loop labels:
**도우미 스크립트**(선택 사항): `../hls-scripts/Detect_loops.sh`를 사용하여 루프 레이블을 추출합니다.
```bash
#!/bin/bash
# Usage: detect_loops.sh <source_file1> <source_file2> ...
grep -Phn "^\s*(\w+):\s*for\s*\(" "$@" | awk -F: '{print $3}' | sed 's/:.*//'
```

Print after Step 3:
3단계 이후 인쇄:
```
─────────────────────────────────────────────────────
[perf-pragma]  Step 3 — done
  ✓ Step 0  Inputs gathered
  ✓ Step 1  top-level target_ti = <N> cycles
  ✓ Step 2  Architecture cascaded
  ✓ Step 3  Loop table built
  ← NEXT    Step 4 — Present table and wait for confirmation
─────────────────────────────────────────────────────
```

---

## Step 4: Present cascade table and wait for confirmation
## 4단계: 계단식 테이블 제시 및 확인 대기

Show the completed table with **Type** and **Pragma Location** columns to clearly distinguish function-level vs loop-level pragmas:
함수 수준과 루프 수준 pragma를 명확하게 구분하기 위해 **Type** 및 **Pragma Location** 열이 포함된 완성된 테이블을 표시합니다.

**Enhanced Cascade Table Format:**
**향상된 캐스케이드 테이블 형식:**

| Target<br>대상          | Type<br>유형     | target_ti<br>target_ti | Pragma Location<br>프라그마 위치                    |
|-----------------|----------|-----------|-------------------------------------|
| <TOP_FUNCTION><br><TOP_FUNCTION>  | function<br>기능 | <value><br><값>   | Position 1: inside function body<br>위치 1: 함수 본문 내부   |
| Row_Loop<br>Row_Loop        | loop<br>루프     | <value><br><값>   | Position 2: inside loop body<br>위치 2: 내부 루프 본문       |
| Col_Loop<br>Col_Loop        | loop<br>루프     | <value><br><값>   | Position 2: inside loop body<br>위치 2: 내부 루프 본문       |

**Placement guidance:**
**배치 지침:**
- **Top-level function**: Place pragma at Position 1 (top of function body):
- **최상위 함수**: 위치 1(함수 본문의 상단)에 pragma를 배치합니다.
  ```cpp
  void <TOP_FUNCTION>(...) {
      #pragma HLS performance target_ti=<value>
      // function body
  }
  ```

- **Loop targets**: Place pragma at Position 2 (top of loop body):
- **루프 대상**: 위치 2(루프 본문 상단)에 pragma를 배치합니다.
  ```cpp
  Row_Loop: for (int r = 0; r < ROWS; r++) {
      #pragma HLS performance target_ti=<value>
      // loop body
  }
  ```

Do not place any pragma until the gate above resolves (auto in demo, user-confirmed in interactive).
위의 게이트가 해결될 때까지(데모에서는 자동, 대화형에서는 사용자 확인) 어떤 pragma도 배치하지 마십시오.

Print after presenting the table:
표를 제시한 후 인쇄:
```
─────────────────────────────────────────────────────
[perf-pragma]  Step 4 Complete — Cascade Table Presented
  ✓ Step 0  Inputs gathered
  ✓ Step 1  top-level target_ti = <N> cycles
  ✓ Step 2  Architecture cascaded
  ✓ Step 3  Loop table built with Type detection
  ✓ Step 4  Cascade table presented
  ← NEXT    Step 5 — Apply pragmas to source code
─────────────────────────────────────────────────────
```

---

## Step 5: Apply performance pragmas to source code
## 5단계: 소스 코드에 성능 프라그마 적용

**This step applies ALL pragmas from the cascade table built in Steps 1-3.**
**이 단계는 1~3단계에서 구축된 계단식 테이블의 모든 pragma를 적용합니다.**

### Step 5a: Locate source file
### 5a단계: 소스 파일 찾기

Find the file containing the top-level function:
최상위 함수가 포함된 파일을 찾습니다.

```bash
TOP_FUNCTION=$(grep "^top=" *.cfg | cut -d'=' -f2)
SRC_FILE=$(grep -l "void $TOP_FUNCTION" src/*.cpp)
echo "Top function: $TOP_FUNCTION"
echo "Source file: $SRC_FILE"
```

Read the file to understand its current structure before editing.
편집하기 전에 파일을 읽어 현재 구조를 이해하세요.

### Step 5b: Apply Position 1 pragma (function-level)
### 5b단계: 위치 1 pragma 적용(함수 수준)

**Target**: Top-level function only (from cascade table where Type="function")
**대상**: 최상위 기능만(Type="function"인 캐스케이드 테이블에서)

**Location**: Immediately after the opening brace `{` of the top-level function
**위치**: 최상위 함수의 여는 중괄호 `{` 바로 뒤

**Pragma to add**:
**추가할 프라그마**:
```cpp
    #pragma HLS performance target_ti=<TOP_LEVEL_TARGET_TI>
```

**Example** (if TOP_FUNCTION=colordetect_accel, TOP_LEVEL_TARGET_TI=68863):
**예**(TOP_FUNCTION=colorDetect_accel, TOP_LEVEL_TARGET_TI=68863인 경우):

**BEFORE**:
**이전**:
```cpp
void colordetect_accel(...) {
    // Load stage
    xf::cv::Mat<XF_8UC3, HEIGHT, WIDTH, XF_NPPC1> imgInput(rows, cols);
```

**AFTER**:
**이후**:
```cpp
void colordetect_accel(...) {
    #pragma HLS performance target_ti=68863
    
    // Load stage
    xf::cv::Mat<XF_8UC3, HEIGHT, WIDTH, XF_NPPC1> imgInput(rows, cols);
```

### Step 5c: Apply Position 2 pragmas (loop-level)
### 5c단계: 위치 2 pragma 적용(루프 수준)

**Target**: All loops from cascade table where Type="loop"
**대상**: Type="loop"인 캐스케이드 테이블의 모든 루프

**Location**: Between loop label and `for` keyword
**위치**: 루프 라벨과 'for' 키워드 사이

**Pragma to add**:
**추가할 프라그마**:
```cpp
    #pragma HLS performance target_ti=<LOOP_TARGET_TI>
```

**Example** (if Row_Loop has target_ti=63):
**예**(Row_Loop에 target_ti=63이 있는 경우):

**BEFORE**:
**이전**:
```cpp
    Row_Loop: for (int r = 0; r < rows; r++) {
        // loop body
    }
```

**AFTER**:
**이후**:
```cpp
    Row_Loop:
    #pragma HLS performance target_ti=63
    for (int r = 0; r < rows; r++) {
        // loop body
    }
```

**Repeat for ALL loops** in the cascade table.
**캐스케이드 테이블의 모든 루프**에 대해 반복합니다.

### Step 5d: Verify pragmas applied correctly
### 5d단계: pragma가 올바르게 적용되었는지 확인

Read the modified source file and confirm:
수정된 소스 파일을 읽고 다음을 확인합니다.
- ✓ Position 1 pragma after function opening brace
- ✓ 함수 열기 중괄호 뒤에 1개의 pragma를 배치합니다.
- ✓ All Position 2 pragmas between loop label and `for`
- ✓ 루프 레이블과 `for` 사이의 모든 위치 2 pragma
- ✓ Syntax: `#pragma HLS performance target_ti=<number>`
- ✓ 구문: `#pragma HLS 성능 목표_ti=<번호>`
- ✓ No typos (target_ti not target_ii)
- ✓ 오타 없음(target_ii가 아닌 target_ti)
- ✓ Indentation matches surrounding code
- ✓ 들여쓰기가 주변 코드와 일치합니다.

### Step 5e: Return to caller
### 5e단계: 발신자에게 돌아가기

**DO NOT commit here.** The calling skill (architect/hls-optimize) will commit.
**여기서 커밋하지 마세요.** 호출 기술(architect/hls-optimize)이 커밋됩니다.

Return the following information to the caller:
발신자에게 다음 정보를 반환합니다.
- TOP_FUNCTION_NAME
- TOP_FUNCTION_NAME
- TOP_LEVEL_TARGET_TI
- TOP_LEVEL_TARGET_TI
- List of modified files (src/*.cpp)
- 수정된 파일 목록(src/*.cpp)
- Cascade table (for commit message)
- 캐스케이드 테이블(커밋 메시지용)

Print completion:
인쇄 완료:
```
─────────────────────────────────────────────────────
[perf-pragma]  Step 5 Complete — Pragmas Applied to Source
  Function       : $TOP_FUNCTION
    Pragma       : #pragma HLS performance target_ti=$TOP_LEVEL_TARGET_TI
    Location     : Position 1 (function body)
  
  Loops          : <N> loops
    Pragma       : target_ti values from cascade table
    Location     : Position 2 (loop bodies)
  
  Modified files : $SRC_FILE
  Status         : Pragmas applied — ready for commit by caller
─────────────────────────────────────────────────────
```

**The calling skill is responsible for**:
**호출 스킬은 다음을 담당합니다**:
- Running `git add src/*.cpp`
- `git add src/*.cpp` 실행
- Creating commit with cascade table in message
- 메시지에 캐스케이드 테이블을 사용하여 커밋 만들기
- Proceeding with next steps
- 다음 단계로 진행
