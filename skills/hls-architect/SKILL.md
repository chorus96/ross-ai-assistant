<!-- ko-translation: enabled -->
---
name: hls-architect
description: Convert input code to a multi-stage HLS dataflow, validate with the other hls skills, then hand off to optimize.
license: MIT
argument-hint: "[<throughput-target e.g. '140 FPS'>] [part=<fpga-part>] [clock=<ns>]"
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

# HLS Design Architecture
# HLS 디자인 아키텍처

> Sources: "Parallel Programming for FPGAs" (Kastner et al.) + AMD UG1399 (Vitis HLS User Guide)
> 출처: "FPGA용 병렬 프로그래밍"(Kastner 외) + AMD UG1399(Vitis HLS 사용자 가이드)

---

## Preamble — Parse Arguments and Print Summary
## 서문 — 인수 구문 분석 및 요약 인쇄

### Vitis environment
### Vitis 환경

The Step 2e g++ verification depends on `$XILINX_VITIS` to locate `ap_fixed.h`.
2e 단계 g++ 검증은 `ap_fixed.h`를 찾기 위해 `$XILINX_VITIS`에 의존합니다.

- **When called by `/hls-matlab-to-cpp`** (the normal flow): the environment was sourced
- **`/hls-matlab-to-cpp`에 의해 호출되는 경우**(일반 흐름): 환경이 소스로 제공되었습니다.
in that skill's preamble — do NOT source it again, just verify it is set.
해당 스킬의 서문에서 다시 소스를 제공하지 말고 설정되었는지 확인하세요.
- **When called standalone**: source Vitis first via `./reference/setup.md` Step 1 or ensure `$XILINX_VITIS` is set
- **독립형 호출 시**: 먼저 `./reference/setup.md`를 통해 Vitis를 소스로 지정하세요. 1단계 또는 `$XILINX_VITIS`가 설정되어 있는지 확인하세요.

Verify before proceeding:
계속하기 전에 확인하세요:
```bash
echo "XILINX_VITIS=$XILINX_VITIS" && ls $XILINX_VITIS/include/ap_fixed.h
```

### Parse arguments
### 인수 구문 분석

Parse `$ARGUMENTS` at the very start:
맨 처음부터 `$ARGUMENTS`를 구문 분석합니다.
- Everything before `part=` is the `THROUGHPUT_TARGET` (may be empty)
- `part=` 이전의 모든 내용은 `THROUGHPUT_TARGET`입니다(비어 있을 수 있음).
- `part=<value>` → `XPART`
- `part=<값>` → `XPART`
- `clock=<value>` → `CLOCK_NS`
- `시계=<값>` → `CLOCK_NS`

If `XPART` or `CLOCK_NS` are missing from `$ARGUMENTS`, ask the user for the missing values before proceeding.
`$ARGUMENTS`에 `XPART` 또는 `CLOCK_NS`가 누락된 경우 계속하기 전에 사용자에게 누락된 값을 문의하세요.

### Input Discovery — Find or Create Workspace
### 입력 검색 — 작업 공간 찾기 또는 만들기

The architect looks for an existing workspace, or creates one if needed.
건축가는 기존 작업 공간을 찾거나 필요한 경우 작업 공간을 만듭니다.
Detection is based solely on filesystem state (not context variables).
탐지는 파일 시스템 상태(컨텍스트 변수가 아님)만을 기반으로 합니다.

**Step 1: Check for existing frame_based/ workspace**
**1단계: 기존 프레임 기반/작업 공간 확인**

```bash
# Search for existing frame_based/kernel.cpp (from hls-matlab-to-cpp or previous run)
EXISTING_KERNEL=$(find . -maxdepth 3 -path "*/frame_based/kernel.cpp" -type f | head -1)

if [ -n "$EXISTING_KERNEL" ]; then
  # Found existing workspace - extract design_name from path
  design_name=$(echo "$EXISTING_KERNEL" | cut -d'/' -f2)
  MODE=existing
else
  # No workspace found - need to create one
  MODE=new
fi
```

**Step 2a: If MODE=existing (workspace found)**
**2a단계: MODE=기존(작업공간 발견)인 경우**

```bash
# Verify testbench also exists
if [ ! -f "$design_name/frame_based/testbench.cpp" ]; then
  ERROR: "Found frame_based/kernel.cpp but testbench.cpp is missing"
fi

# Print confirmation
echo "Found existing workspace: $design_name/frame_based/"
echo "  ✓ kernel.cpp"
echo "  ✓ testbench.cpp"
```

Proceed to Interface Selection.
인터페이스 선택으로 진행하세요.

**Step 2b: If MODE=new (no workspace found)**
**2b단계: MODE=new인 경우(작업공간을 찾을 수 없음)**

1. Ask user for input file paths:
1. 사용자에게 입력 파일 경로를 묻습니다.
   - "Where is your C++ kernel source file?" → `USER_KERNEL_PATH`
   - "C++ 커널 소스 파일은 어디에 있나요?" → `USER_KERNEL_PATH`
   - "Where is your testbench file?" → `USER_TESTBENCH_PATH`
   - "테스트벤치 파일은 어디에 있나요?" → `USER_TESTBENCH_PATH`

2. Ask user for design name (or derive from kernel filename):
2. 사용자에게 디자인 이름을 묻습니다(또는 커널 파일 이름에서 파생).
   - "What should the design workspace be named?" → `design_name`
   - "디자인 작업공간의 이름은 무엇으로 정해야 할까요?" → `디자인_이름`
   - Default suggestion: extract basename without extension from kernel path
   - 기본 제안: 커널 경로에서 확장명 없이 기본 이름 추출

3. Create workspace and copy files:
3. 작업공간 생성 및 파일 복사:
   ```bash
   # Create frame_based/ directory
   mkdir -p "$design_name/frame_based"
   
   # Copy user files into canonical locations
   cp "$USER_KERNEL_PATH" "$design_name/frame_based/kernel.cpp"
   cp "$USER_TESTBENCH_PATH" "$design_name/frame_based/testbench.cpp"
   ```

4. Print confirmation:
4. 인쇄 확인:
   ```
   ─────────────────────────────────────────────────────
   [architect]  New Workspace Created
     Design name    : <design_name>
     Kernel copied  : <USER_KERNEL_PATH>
                    → <design_name>/frame_based/kernel.cpp
     TB copied      : <USER_TESTBENCH_PATH>
                    → <design_name>/frame_based/testbench.cpp
   ─────────────────────────────────────────────────────
   ```

**After Input Discovery completes (both modes):**
**입력 검색이 완료된 후(두 모드 모두):**
- `design_name/frame_based/kernel.cpp` exists ← INPUT to architect
- `design_name/frame_based/kernel.cpp`가 존재합니다 ← 설계자에게 입력
- `design_name/frame_based/testbench.cpp` exists
- `design_name/frame_based/testbench.cpp`가 존재합니다.

Proceed to Interface Selection.
인터페이스 선택으로 진행하세요.

### Interface Selection
### 인터페이스 선택

**In interactive mode**: ask the user which top-level interface the kernel should use.
**대화형 모드**: 커널이 사용해야 하는 최상위 인터페이스가 무엇인지 사용자에게 묻습니다.
Present the options clearly:
옵션을 명확하게 제시하십시오.

```
[architect]  Interface Selection — choose one:

  1. m_axi          Data in/out via AXI4 burst master (arrays in DDR/HBM).
                    Best for: frame buffers, large lookup tables, batch processing.
                    load_input() reads a full frame from memory before compute starts.
                    Example: peakPicker reading xcorr[] and threshold[] arrays.

  2. axis            Data in/out via AXI4-Stream (pixel/sample streaming, no buffering).
                    Best for: real-time pipelines, video, DSP chains.
                    load_input() forwards stream samples directly into compute.
                    Example: demosaic receiving a pixel stream from a camera sensor.

  3. m_axi + axis    Mixed — one port streams in, another bursts from memory (or vice versa).
                    Best for: kernels that stream output but need a lookup table from DDR.

  4. m_axi + s_axilite  Data arrays via m_axi; scalar parameters (thresholds, sizes) via
                         AXI-Lite control register. Most common for standalone Vitis kernels.

Which interface fits your design?
```

**If `autoconfirm=true` is in `$ARGUMENTS`**: do NOT ask — infer `IFACE_TYPE`
**`autoconfirm=true`가 `$ARGUMENTS`에 있는 경우**: 묻지 말고 `IFACE_TYPE`을 추론하세요.
automatically from the top-level function signature:
최상위 함수 서명에서 자동으로:
- Pointer/array arguments (`float*`, `int[]`) with no `hls::stream` args → `m_axi`
- `hls::stream` 인수가 없는 포인터/배열 인수(`float*`, `int[]`) → `m_axi`
- `hls::stream` arguments only → `axis`
- `hls::stream` 인수만 → `축`
- Mix of pointer and `hls::stream` → `m_axi + axis`
- 포인터와 `hls::stream` 혼합 → `m_axi + 축`
- Pointer args alongside scalar-only control args → `m_axi + s_axilite`
- 스칼라 전용 제어 인수와 함께 포인터 인수 → `m_axi + s_axilite`

Print the inferred choice:
추론된 선택을 인쇄합니다.
```
[architect]  Interface inferred: m_axi  (autoconfirm; pointer args detected)
```

Store the answer as `IFACE_TYPE`.
답변을 'IFACE_TYPE'으로 저장하세요.
Use it when writing `#pragma HLS INTERFACE` pragmas and when deciding whether `load_input` / `store_output` use `hls::stream` ports or `m_axi` pointer arguments.
`#pragma HLS INTERFACE` pragma를 작성할 때와 `load_input` / `store_output`이 `hls::stream` 포트 또는 `m_axi` 포인터 인수를 사용할지 결정할 때 이를 사용하십시오.

Print confirmation:
인쇄 확인:
```
─────────────────────────────────────────────────────
[architect]  Starting — <design_name>
  Throughput target : <THROUGHPUT_TARGET or "none — will derive from II=1 floor">
  FPGA part         : <XPART>
  Clock             : <CLOCK_NS> ns
  Interface         : <IFACE_TYPE>
─────────────────────────────────────────────────────
```

---

## Directory Layout
## 디렉토리 레이아웃

After Input Discovery, the canonical workspace structure is:
입력 검색 후 정식 작업 공간 구조는 다음과 같습니다.

```
design_name/
├── frame_based/          ← INPUT (always exists after Input Discovery)
│   ├── kernel.cpp        ← Plain C++ kernel (created by hls-matlab-to-cpp OR copied in standalone mode)
│   └── testbench.cpp     ← Testbench (created by hls-matlab-to-cpp OR copied in standalone mode)
└── rearchitect/
    └── v1/               ← OUTPUT (created by architect via design-layout.md)
        ├── src/
        │   └── kernel_hls.cpp    ← HLS dataflow architecture
        ├── tb/
        │   └── testbench.cpp     ← Copied from frame_based/ unchanged
        └── hls_config.cfg         ← Generated config file
        └── vitis-comp.json        ← Generated hls component file
```

**Optional directories** (only present if called from /hls-matlab-to-cpp):
**선택적 디렉터리**(/hls-matlab-to-cpp에서 호출된 경우에만 존재함):
```
design_name/
├── golden/               ← MATLAB source files + golden bins
├── sample_based/         ← Refactor 1 (line-by-line MATLAB → C++)
```

All directories under `rearchitect/` are created via `./reference/design-layout.md` — **never create directories manually**.
`rearchitect/` 아래의 모든 디렉토리는 `./reference/design-layout.md`를 통해 생성됩니다 — **디렉토리를 수동으로 생성하지 마십시오**.

---

## Handoff Protocol
## 핸드오프 프로토콜

Every step ends with a `→ HANDOFF` block. **No step may begin until the previous step's Condition is satisfied.**
모든 단계는 '→ HANDOFF' 블록으로 끝납니다. **이전 단계의 조건이 충족될 때까지 어떤 단계도 시작할 수 없습니다.**

```
→ HANDOFF to Step X
  Produces : <artifact>
  Condition: <what must be true>
  Blocked if: <what causes a loop-back or stop>
```

---

## Flow Overview
## 흐름 개요

Print this immediately after the confirmation banner so the user can track progress:
사용자가 진행 상황을 추적할 수 있도록 확인 배너 바로 뒤에 다음을 인쇄하세요.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[architect]  Pipeline — <design_name>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Step 1   Study domain patterns (./reference/vitis-libraries.md)
  Step 2a  Draw activity timeline + choose paradigm
  Step 2b  Generate fixed top-level (load → compute → store)
  Step 2c  Decompose compute() — hierarchical or monolithic
  Step 2d  Write to rearchitect/ + pragma audit
  Step 2e  G++ functional verification
  Step 3a  Validate with hls* skills
  Step 3b  Architecture review (user approval)
  Step 3c  Write to perf_outcomes.md
  Step 3d-1  Apply performance pragmas (if throughput)
  Step 3d-2  Save baseline snapshot
  Step 3e  Hand off to /hls-optimize
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Re-print this after each step with `✓` on completed steps and `← NEXT` on the upcoming one.
완료된 단계에는 `✓`를, 다음 단계에는 `← NEXT`를 사용하여 각 단계 후에 이를 다시 인쇄하세요.

---

## Workflow
## 작업 흐름

### Step 1 / 3 — Study Domain Patterns
### 1/3단계 - 도메인 패턴 연구

```
─────────────────────────────────────────────────────
[architect]  Step 1 / 3 — Study Domain Patterns (./reference/vitis-libraries.md)
─────────────────────────────────────────────────────
```

Invoke `./reference/vitis-libraries.md` to study coding patterns from production HLS implementations before writing any code.
코드를 작성하기 전에 './reference/vitis-libraries.md'를 호출하여 프로덕션 HLS 구현의 코딩 패턴을 연구하세요.
Do NOT include or link Vitis Library headers in user designs — study, then write your own.
사용자 디자인에 Vitis Library 헤더를 포함하거나 연결하지 마십시오. 연구한 후 직접 작성하십시오.

Print before proceeding:
계속하기 전에 인쇄하세요:
```
─────────────────────────────────────────────────────
[architect]  Step 1 done — Domain Patterns
  Patterns found : <list patterns identified from ./reference/vitis-libraries.md>
  Applies to     : <how each pattern maps to this algorithm>
  Gap            : <any applicable pattern NOT found — or "none">
─────────────────────────────────────────────────────
✓ Step 1   Study domain patterns (./reference/vitis-libraries.md)
  Step 2a  Draw activity timeline + choose paradigm     ← NEXT
  Step 2b  Generate fixed top-level (load → compute → store)
  Step 2c  Decompose compute() — hierarchical or monolithic
  Step 2d  Write to rearchitect/ + pragma audit
  Step 2e  G++ functional verification
  Step 3a  Validate with hls* skills
  Step 3b  Architecture review (user approval)
  Step 3c  Write to perf_outcomes.md
  Step 3d-1  Apply performance pragmas (if throughput)
  Step 3d-2  Save baseline snapshot
  Step 3e  Hand off to /hls-optimize
```

```
→ HANDOFF to Step 2
  Produces : Domain pattern notes (in-context — coding patterns from ./reference/vitis-libraries.md)
  Condition: At least one relevant HLS pattern identified for the input algorithm
  Blocked if: No applicable pattern found → document why, proceed to Step 2 anyway
```

---

### Step 2 / 3 — Generate Architecture
### 2/3단계 - 아키텍처 생성

```
─────────────────────────────────────────────────────
[architect]  Step 2 / 3 — Generate Architecture
─────────────────────────────────────────────────────
```

Apply the **two paradigms** when converting input code to HLS:
입력 코드를 HLS로 변환할 때 **두 가지 패러다임**을 적용합니다.

| Paradigm<br>패러다임 | What it means<br>의미 | HLS mapping<br>HLS 매핑 |
|---|---|---|
| **Producer-Consumer**<br>**생산자-소비자** | N tasks, each with one responsibility, communicating through channels<br>N개의 작업은 각각 하나의 책임을 갖고 있으며 채널을 통해 통신합니다. | One `static` function per stage; top-level with `#pragma HLS DATAFLOW`<br>단계당 하나의 '정적' 기능; `#pragma HLS DATAFLOW`를 사용한 최상위 수준 |
| **Streaming Data**<br>**스트리밍 데이터** | Tasks exchange data via FIFOs (in-order, max overlap) or PIPOs (random-order, safer)<br>작업은 FIFO(순서, 최대 중첩) 또는 PIPO(무작위, 더 안전함)를 통해 데이터를 교환합니다. | `hls::stream` for sequential access; PIPO for random access; avoid wide channels<br>순차 접근을 위한 `hls::stream`; 랜덤 액세스를 위한 PIPO; 넓은 채널을 피하세요 |

**Draw the activity timeline before coding** — shows when each stage executes across multiple invocations; reveals expected parallelism and bottlenecks.
**코딩 전에 활동 타임라인 그리기** — 각 단계가 여러 호출에 걸쳐 실행되는 시기를 표시합니다. 예상되는 병렬성과 병목 현상을 보여줍니다.

> **Note:** Loop-level pipelining (`#pragma HLS PIPELINE`) is NOT applied by the architect — see Pragma Rules below.
> **참고:** 루프 수준 파이프라이닝(`#pragma HLS PIPELINE`)은 설계자가 적용하지 않습니다. 아래 Pragma 규칙을 참조하세요.
> The performance pragma handles throughput targets.
> 성능 pragma는 처리량 목표를 처리합니다.

```
→ HANDOFF to Fixed Top-Level Structure
  Produces : Activity timeline drawn; paradigm (producer-consumer / streaming) identified
  Condition: At least one activity timeline sketched showing stage overlap across invocations
  Blocked if: Timeline cannot be drawn → ask user for clarification on algorithm boundaries
```

---

#### Fixed Top-Level Structure
#### 고정된 최상위 구조

The top level **always** has exactly three functions connected by one `#pragma HLS DATAFLOW`:
최상위 레벨 **always**에는 하나의 `#pragma HLS DATAFLOW`로 연결된 정확히 세 가지 기능이 있습니다.

```
load_input()  →  compute()  →  store_output()
```

- `load_input()` — reads from AXI master port(s), writes to stream(s) or PIPO(s)
- `load_input()` — AXI 마스터 포트에서 읽고, 스트림 또는 PIPO에 씁니다.
- `compute()` — all algorithmic work; see decomposition rules below
- `compute()` — 모든 알고리즘 작업; 아래 분해 규칙을 참조하세요.
- `store_output()` — reads from stream(s) or PIPO(s), writes to AXI master port(s)
- `store_output()` — 스트림 또는 PIPO에서 읽고 AXI 마스터 포트에 씁니다.
- Channels: `hls::stream` for sequential access, local arrays (PIPO) for random access
- 채널: 순차 액세스용 `hls::stream`, 무작위 액세스용 로컬 배열(PIPO)
- All `#pragma HLS INTERFACE` pragmas on top-level function args only
- 최상위 함수 인수의 모든 `#pragma HLS INTERFACE` pragma만

**Validate the structure.** Use the `hls-dataflow` skill to confirm the `{load_input, compute, store_output}` region is canonical before proceeding.
**구조를 검증합니다.** 진행하기 전에 `hls-dataflow` 기술을 사용하여 `{load_input, Compute, store_output}` 영역이 정식인지 확인하세요.
The skill checks that the top-level matches the required three-function pattern with proper DATAFLOW pragma.
이 기술은 최상위 수준이 적절한 DATAFLOW pragma를 사용하여 필요한 3가지 기능 패턴과 일치하는지 확인합니다.

Print before proceeding:
계속하기 전에 인쇄하세요:
```
─────────────────────────────────────────────────────
[architect]  Fixed Top-Level — done
  Structure    : load_input() → compute() → store_output()
  Interface    : <IFACE_TYPE — pragmas applied>
  Channels     : <stream or PIPO — and why>
  /hls-dataflow: PASS
─────────────────────────────────────────────────────
✓ Step 1   Study domain patterns (./reference/vitis-libraries.md)
✓ Step 2a  Draw activity timeline + choose paradigm
✓ Step 2b  Generate fixed top-level (load → compute → store)
  Step 2c  Decompose compute() — hierarchical or monolithic  ← NEXT
  Step 2d  Write to rearchitect/ + pragma audit
  Step 2e  G++ functional verification
  Step 3a  Validate with hls* skills
  Step 3b  Architecture review (user approval)
  Step 3c  Write to perf_outcomes.md
  Step 3d-1  Apply performance pragmas (if throughput)
  Step 3d-2  Save baseline snapshot
  Step 3e  Hand off to /hls-optimize
```

```
→ HANDOFF to compute() Decomposition
  Produces : Top-level code (load_input + compute + store_output) with one #pragma HLS DATAFLOW
  Condition: /hls-dataflow passes on the top-level region
  Blocked if: /hls-dataflow fails → fix top-level structure; do NOT proceed to compute() Decomposition
```

---

#### compute() Decomposition
#### 계산() 분해

Analyze whether `compute()` can be internally divided into sequential sub-stages by invoking `/hls-dataflow` on the proposed decomposition.
제안된 분해에 대해 `/hls-dataflow`를 호출하여 `compute()`를 내부적으로 순차적 하위 단계로 나눌 수 있는지 분석합니다.

**If `/hls-dataflow` passes → Hierarchical compute:**
**`/hls-dataflow`가 통과하는 경우 → 계층적 계산:**
```
compute():
  #pragma HLS DATAFLOW
    compute_stage1()  →  compute_stage2()  →  …  →  compute_stageN()
```
- Decompose into N `static void compute_stageX(...)` functions, split at natural algorithmic boundaries
- N `static void Compute_stageX(...)` 함수로 분해하고 자연 알고리즘 경계에서 분할
- Add `#pragma HLS DATAFLOW` **inside** `compute()` over {compute_stage1, …, compute_stageN}
- {compute_stage1, …, Compute_stageN}에 `#pragma HLS DATAFLOW` **inside** `compute()` 추가
- N=1 requires no internal DATAFLOW — treat as monolithic
- N=1에는 내부 DATAFLOW가 필요하지 않습니다. — 모놀리식으로 처리됩니다.

**If `/hls-dataflow` fails → Monolithic compute:**
**`/hls-dataflow`가 실패하는 경우 → 모놀리식 컴퓨팅:**
- `compute()` contains sequential loops only — no internal `#pragma HLS DATAFLOW`
- `compute()`에는 순차 루프만 포함되어 있으며 내부 `#pragma HLS DATAFLOW`는 없습니다.
- Report the specific rule(s) flagged by `/hls-dataflow`
- `/hls-dataflow`에 의해 플래그가 지정된 특정 규칙을 보고합니다.

Print before proceeding:
계속하기 전에 인쇄하세요:
```
─────────────────────────────────────────────────────
[architect]  compute() Decomposition — done
  Structure    : Hierarchical (<N> stages: <names>) / Monolithic (<reason>)
  Bottleneck   : <estimated stage> (~<N> cycles)
  /hls-dataflow: PASS / FAIL (<reason if fail>)
─────────────────────────────────────────────────────
✓ Step 1   Study domain patterns (./reference/vitis-libraries.md)
✓ Step 2a  Draw activity timeline + choose paradigm
✓ Step 2b  Generate fixed top-level (load → compute → store)
✓ Step 2c  Decompose compute() — hierarchical or monolithic
  Step 2d  Write to rearchitect/ + pragma audit          ← NEXT
  Step 2e  G++ functional verification
  Step 3a  Validate with hls* skills
  Step 3b  Architecture review (user approval)
  Step 3c  Write to perf_outcomes.md
  Step 3d-1  Apply performance pragmas (if throughput)
  Step 3d-2  Save baseline snapshot
  Step 3e  Hand off to /hls-optimize
```

```
→ HANDOFF to Write to rearchitect/
  Produces : compute() implementation — hierarchical (with internal DATAFLOW) or monolithic (reason documented);
             compute() code presented to user
  Condition: /hls-dataflow result recorded; compute() code is complete
  Blocked if: /hls-dataflow fails on decomposition and no valid monolithic fallback → ask user before proceeding
```

---

#### Write to rearchitect/
#### 재설계/에 쓰기

Call `./reference/design-layout.md  design_name=<name>  stage=rearchitect_v1`
`./reference/design-layout.md design_name=<name> stage=rearchitect_v1`을 호출하세요.

This creates `design_name/rearchitect/v1/src/`, `design_name/rearchitect/v1/tb/`, a template `hls_config.cfg` and a template `vitis-comp.json`.
그러면 `design_name/rearchitect/v1/src/`, `design_name/rearchitect/v1/tb/`, `hls_config.cfg` 템플릿 및 `vitis-comp.json` 템플릿이 생성됩니다.

Write into `design_name/rearchitect/v1/`:
`design_name/rearchitect/v1/`에 작성합니다.
- `src/kernel_hls.cpp` — kernel source (load_input + compute + store_output)
- `src/kernel_hls.cpp` — 커널 소스(load_input + Compute + store_output)
- `tb/testbench.cpp`   — copied from `design_name/frame_based/testbench.cpp` unchanged (Input Discovery ensures this file exists)
- `tb/testbench.cpp` — `design_name/frame_based/testbench.cpp`에서 변경 없이 복사됨(입력 검색을 통해 이 파일이 존재하는지 확인)
- `hls_config.cfg`     — use this exact template (substituting `XPART`, `CLOCK_NS`, `TOP_FUNCTION`):
- `hls_config.cfg` — 이 정확한 템플릿을 사용합니다(`XPART`, `CLOCK_NS`, `TOP_FUNCTION`으로 대체):

```ini
# ⚠ part= MUST be at the TOP LEVEL — never write syn.part= or put part inside [hls]
# ⚠ clock= uses ns period — never write syn.clkPeriod=
part=<XPART>

[hls]
syn.top=<TOP_FUNCTION>
clock=<CLOCK_NS>
syn.file=src/kernel_hls.cpp
tb.file=tb/testbench.cpp
```

**For testbenches that use OpenCV**, add these keys — use the exact key names:
**OpenCV를 사용하는 테스트벤치의 경우** 다음 키를 추가하세요. 정확한 키 이름을 사용하세요.
```ini
tb.cflags=-I<OPENCV_INCLUDE>
csim.ldflags=-L<OPENCV_LIB> -lopencv_core -lopencv_imgcodecs -lopencv_imgproc -Wl,-rpath,<OPENCV_LIB>
```

**FORBIDDEN cfg patterns — never generate these:**
**금지된 cfg 패턴 — 절대 생성하지 마세요:**
```ini
[hls]
syn.part=xczu9eg-...     ← not a valid field; v++ ignores or rejects it
syn.clkPeriod=3.5        ← wrong field name; use clock=
```


- `vitis-comp.json`     — use this exact template (substituting `TOP_FUNCTION` `design_name`):
- `vitis-comp.json` — 이 정확한 템플릿을 사용합니다(`TOP_FUNCTION` `design_name`으로 대체):

```ini
{
   "name": <design_name>,
   "type": "HLS",
   "configuration": {
    "componentType": "HLS",
    "configFiles": ["./hls_config.cfg"],
    "work_dir": <TOP_FUNCTION>
   }
}
```


Present the generated code to the user.
생성된 코드를 사용자에게 제시합니다.

```
→ HANDOFF to Pragma Rules
  Produces : kernel_hls.cpp written to design_name/rearchitect/v1/src/; directories confirmed by ./reference/design-layout.md
  Condition: top-level has exactly one #pragma HLS DATAFLOW over {load_input, compute, store_output};
             compute() is either hierarchical (with internal DATAFLOW) or monolithic (documented reason)
  Blocked if: code cannot be written (unresolvable dependency) → document reason, ask user before proceeding
```

---

### Pragma Rules — Self-Audit Gate
### Pragma 규칙 - 자체 감사 게이트

After the code is written, **scan every pragma** before proceeding to validation.
코드가 작성된 후 유효성 검사를 진행하기 전에 **모든 pragma를 검사**하세요.

**Allowed in architect-generated code:**
**건축가가 생성한 코드에서 허용됨:**
```cpp
#pragma HLS performance target_ti=N
#pragma HLS INTERFACE   // m_axi, axis, s_axilite
#pragma HLS DATAFLOW
#pragma HLS STREAM variable=<stream_var> depth=<N>   // set FIFO depth on hls::stream channels
```

**FORBIDDEN — must not appear:**
**금지됨 — 표시되어서는 안 됩니다:**
```cpp
#pragma HLS PIPELINE
#pragma HLS UNROLL
#pragma HLS ARRAY_PARTITION
#pragma HLS ARRAY_RESHAPE
#pragma HLS INLINE
#pragma HLS FLATTEN
```

If any forbidden pragma is found → **remove it immediately** before proceeding.
금지된 pragma가 발견되면 → 진행하기 전에 **즉시 제거**하세요.

Print this confirmation before moving on:
계속 진행하기 전에 이 확인서를 인쇄하세요.
```
Pragma audit passed — code contains only: DATAFLOW, INTERFACE, performance.
```

> **Rationale:** `#pragma HLS performance` gives HLS the throughput target and lets it
> **이유:** `#pragma HLS 성능`은 HLS에 처리량 목표를 제공하고 이를 허용합니다.
> infer micro-optimizations automatically.
> 미세 최적화를 자동으로 추론합니다.
> The architect controls macro-architecture only.
> 건축가는 매크로 아키텍처만 제어합니다.
> Fine-grained pragmas are the responsibility of `/hls-optimize`.
> 세분화된 pragma는 `/hls-optimize`의 책임입니다.

Print before proceeding:
계속하기 전에 인쇄하세요:
```
─────────────────────────────────────────────────────
[architect]  Pragma audit — done
  Pragmas present : <list all #pragma HLS lines in the file>
  Forbidden found : none
─────────────────────────────────────────────────────
✓ Step 1   Study domain patterns (./reference/vitis-libraries.md)
✓ Step 2a  Draw activity timeline + choose paradigm
✓ Step 2b  Generate fixed top-level (load → compute → store)
✓ Step 2c  Decompose compute() — hierarchical or monolithic
✓ Step 2d  Write to rearchitect/ + pragma audit
  Step 2e  G++ functional verification                  ← NEXT
  Step 3a  Validate with hls* skills
  Step 3b  Architecture review (user approval)
  Step 3c  Write to perf_outcomes.md
  Step 3d-1  Apply performance pragmas (if throughput)
  Step 3d-2  Save baseline snapshot
  Step 3e  Hand off to /hls-optimize
```

```
→ HANDOFF to Step 2e
  Produces : Pragma-clean source file; audit confirmation line printed
  Condition: Zero forbidden pragmas
  Blocked if: Forbidden pragma cannot be cleanly removed → ask user before deleting
```

---

### Step 2e — G++ Functional Verification
### 2e단계 - G++ 기능 검증

```
─────────────────────────────────────────────────────
[architect]  Step 2e — G++ Functional Verification
─────────────────────────────────────────────────────
```

Before invoking the hls* validation battery, prove the rearchitected code is
hls* 검증 배터리를 호출하기 전에 재구성된 코드가 다음과 같음을 증명하십시오.
**functionally correct** by compiling with g++ and running against the MATLAB golden.
g++로 컴파일하고 MATLAB Golden에 대해 실행하여 **기능적으로 정확**합니다.
This catches regressions introduced by the architecture transformation (load_input/
이는 아키텍처 변환(load_input/
compute/store_output split, channel insertion, hierarchical decomposition) BEFORE
계산/저장_출력 분할, 채널 삽입, 계층적 분해) 이전
the expensive hls* validation cycle.
값비싼 hls* 검증 주기.

#### Compile
#### 컴파일

```bash
cd <design_name>/rearchitect/v1/

# ap_fixed/ap_int/ap_uint headers ship with Vitis at $XILINX_VITIS/include
g++ -std=c++14 \
    -I$XILINX_VITIS/include \
    -o verify_arch src/kernel_hls.cpp tb/testbench.cpp \
&& ./verify_arch
```

For OpenCV designs (when the testbench uses `cv::imread`):
OpenCV 디자인의 경우(테스트벤치가 `cv::imread`를 사용하는 경우):

```bash
g++ -std=c++14 \
    -I$XILINX_VITIS/include \
    -I${OPENCV_INCLUDE} \
    -o verify_arch src/kernel_hls.cpp tb/testbench.cpp \
    -L${OPENCV_LIB} -lopencv_core -lopencv_imgcodecs -lopencv_imgproc \
    -Wl,-rpath,${OPENCV_LIB} \
&& ./verify_arch
```

#### Pass criteria
#### 합격 기준

The executable must print `PASS` on both calls (the testbench runs the kernel
실행 파일은 두 호출 모두에서 'PASS'를 인쇄해야 합니다(테스트벤치는 커널을 실행합니다).
twice for II measurement).
II 측정의 경우 두 번).
If FAIL:
실패할 경우:

- Compare against the `frame_based/` verification — it passed there
- `frame_based/` 검증과 비교해 보세요. 통과했습니다.
- Common regressions:
- 일반적인 회귀:
  - channel/stream byte order swap in `load_input` → `store_output`
  - `load_input` → `store_output`에서 채널/스트림 바이트 순서 교환
  - `compute_stage` decomposition split a dependency
  - `compute_stage` 분해로 종속성 분할
  - pragmas were stripped that the algorithm needs (e.g., `DATAFLOW`)
  - 알고리즘에 필요한 pragma가 제거되었습니다(예: `DATAFLOW`).
- Fix `rearchitect/src/kernel_hls.cpp`; do NOT proceed to Step 3a
- `rearchitect/src/kernel_hls.cpp`를 수정하세요. 3a단계로 진행하지 마세요.

Print before proceeding:
계속하기 전에 인쇄하세요:
```
─────────────────────────────────────────────────────
[architect]  Step 2e done — G++ functional verification
  Compile      : OK / FAIL (<error if any>)
  Run result   : PASS / FAIL (<mismatch count if FAIL>)
  Status       : <ok to proceed to hls* validation / blocked>
─────────────────────────────────────────────────────
✓ Step 1   Study domain patterns (./reference/vitis-libraries.md)
✓ Step 2a  Draw activity timeline + choose paradigm
✓ Step 2b  Generate fixed top-level (load → compute → store)
✓ Step 2c  Decompose compute() — hierarchical or monolithic
✓ Step 2d  Write to rearchitect/ + pragma audit
✓ Step 2e  G++ functional verification
  Step 3a  Validate with hls* skills                    ← NEXT
  Step 3b  Architecture review (user approval)
  Step 3c  Write to perf_outcomes.md
  Step 3d-1  Apply performance pragmas (if throughput)
  Step 3d-2  Save baseline snapshot
  Step 3e  Hand off to /hls-optimize
```

```
→ HANDOFF to Step 3a (hls* validation)
  Produces : verify_arch executable, both calls PASS
  Condition: g++ compiles cleanly AND ./verify_arch prints PASS twice
  Blocked if: compile error or mismatch → fix kernel_hls.cpp before Step 3a
```

---

### Step 3 / 3 — Validate, Record, and Optimize
### 3/3단계 - 검증, 기록 및 최적화

```
─────────────────────────────────────────────────────
[architect]  Step 3 / 3 — Validate → Record → Optimize
─────────────────────────────────────────────────────
```

#### 3a — Validate with hls* Skills
#### 3a — hls* 기술로 검증

**Use all available HLS validation tools.** Run the following skills to check the rearchitected design:
**사용 가능한 모든 HLS 검증 도구를 사용합니다.** 다음 기술을 실행하여 재구성된 디자인을 확인합니다.

- `hls-synthesizable` skill: verify the code is synthesizable
- `hls-synthessible` 기술: 코드가 합성 가능한지 확인
- `hls-dataflow-infos` skill: analyze process/channel topology
- `hls-dataflow-infos` 기술: 프로세스/채널 토폴로지 분석
- `hls-array-to-stream` skill: identify array→stream optimization opportunities
- `hls-array-to-stream` 기술: 배열→스트림 최적화 기회 식별
- `hls-burst-inference` skill: check AXI burst inference eligibility
- `hls-burst-inference` 기술: AXI 버스트 추론 적격성 확인
- `hls-flattenable` skill: verify loop nest can be flattened
- `hls-평탄화 가능` 기술: 루프 중첩이 평면화될 수 있는지 확인
- `hls-stencil-pattern` skill: identify stencil pattern optimization opportunities
- `hls-stencil-pattern` 기술: 스텐실 패턴 최적화 기회 식별
- `hls-line-buffer` skill: check if line buffer pattern applies
- `hls-line-buffer` 스킬: 라인 버퍼 패턴이 적용되는지 확인

Run **all** of the above.
위의 **모두**를 실행합니다.
Fix any violations before proceeding.
계속하기 전에 위반 사항을 수정하세요.

```
╔════════════════════════════════════════════════════════════════╗
║     HLS Validation Report — <design name>                      ║
╠═════════════════════╦══════════════════════════╦═══════════════╣
║ Skill               ║ What it checks           ║ Result        ║
╠═════════════════════╬══════════════════════════╬═══════════════╣
║ hls-synthesizable   ║ Synthesizable code       ║ ✅ PASS        ║
║ hls-dataflow-infos  ║ Process/channel topology ║ ✅ PASS        ║
║ hls-array-to-stream ║ Array→stream opportunity ║ ⚠️  N/A       ║
║ hls-burst-inference ║ AXI burst inference      ║ ❌ FAIL        ║
║ hls-flattenable     ║ Loop nest flattenable    ║ ✅ PASS        ║
║ hls-stencil-pattern ║ Stencil optimization     ║ ⚠️  N/A       ║
║ hls-line-buffer     ║ Line buffer II=1 recipe  ║ ⚠️  N/A       ║
╠═════════════════════╩══════════════════════════╩═══════════════╣
║ Overall: X/7 passed   Violations: <list skills that failed>    ║
╚════════════════════════════════════════════════════════════════╝
```

**Result key:** `✅ PASS` — ok | `❌ FAIL` — fix before proceeding | `⚠️  N/A` — not applicable
**결과 키:** `✅ PASS` — 확인 | `❌ FAIL` — 진행하기 전에 수정 | `⚠️ N/A` — 해당 사항 없음

Reference skills for fixes:
수정을 위한 참조 기술:

| Violation in<br>위반 | Fix with<br>수정 |
|---|---|
| `hls-synthesizable`<br>`hls 합성 가능` | `/hls-synthesizable`<br>`/hls-합성 가능`|
| `hls-dataflow`<br>`hls-데이터 흐름` | `/hls-dataflow`<br>`/hls-데이터플로우`|
| `hls-burst-inference`<br>`hls-버스트 추론` | `/hls-burst-inference`<br>`/hls-버스트-추론`|
| `hls-flattenable`<br>`hls-평탄화 가능` | `/hls-flattenable`<br>`/hls-평탄화 가능`|
| `hls-line-buffer`<br>`hls-라인 버퍼` | `/hls-line-buffer` → re-check all 8 steps<br>`/hls-line-buffer` → 8단계 모두 다시 확인 |

If any `❌ FAIL` → fix violation, re-run that specific skill, reprint the table; repeat until clean.
`❌ FAIL` → 위반 수정, 특정 스킬을 다시 실행하고 테이블을 다시 인쇄하세요. 깨끗해질 때까지 반복하세요.

---

#### 3b — Architecture Review Gate
#### 3b — 아키텍처 검토 게이트

Check `$ARGUMENTS` for `autoconfirm=true`.
`autoconfirm=true`에 대해 `$ARGUMENTS`를 확인하세요.
If present, skip the approval wait and auto-proceed.
있는 경우 승인 대기를 건너뛰고 자동으로 진행합니다.

Present the architecture summary:
아키텍처 요약을 제시합니다.

```
─────────────────────────────────────────────────────
[architect]  Architecture Review — <design_name>
─────────────────────────────────────────────────────
  Top-level  : load_input() → compute() → store_output()
  Interface  : <IFACE_TYPE>
  compute()  : <Hierarchical — N stages>  OR  <Monolithic — reason>
  Bottleneck : <stage_name>  (~N cycles)
  Est. rate  : ~X FPS @ Y MHz
  hls* checks: all ✅ PASS
─────────────────────────────────────────────────────
✓ Step 1   Study domain patterns (./reference/vitis-libraries.md)
✓ Step 2a  Draw activity timeline + choose paradigm
✓ Step 2b  Generate fixed top-level (load → compute → store)
✓ Step 2c  Decompose compute() — hierarchical or monolithic
✓ Step 2d  Write to rearchitect/ + pragma audit
✓ Step 2e  G++ functional verification
✓ Step 3a  Validate with hls* skills
  Step 3b  Architecture review                           ← HERE
  Step 3c  Write to perf_outcomes.md
  Step 3d-1  Apply performance pragmas (if throughput)
  Step 3d-2  Save baseline snapshot
  Step 3e  Hand off to /hls-optimize
```

**If `autoconfirm=true` is in `$ARGUMENTS`**: print `[architect] autoconfirm=true — auto-proceeding` and continue immediately.
**`autoconfirm=true`가 `$ARGUMENTS`에 있는 경우**: `[architect] autoconfirm=true — auto-proceeding`을 인쇄하고 즉시 계속합니다.
Do NOT wait for input.
입력을 기다리지 마십시오.

**Otherwise**: ask the user:
**그렇지 않은 경우**: 사용자에게 다음과 같이 질문하세요.

> **Does this architecture match your expectations?**
> **이 아키텍처가 귀하의 기대와 일치합니까?**
> - Reply **yes** to proceed to perf_outcomes.md and /hls-optimize
> - perf_outcomes.md 및 /hls-optimize로 진행하려면 **예**라고 답하세요.
> - Reply with feedback to revise the architecture
> - 아키텍처 수정을 위해 피드백을 보내주세요.

Wait for an explicit "yes" before continuing.
계속하기 전에 명시적인 "예"를 기다리십시오.
If the user gives feedback:
사용자가 피드백을 제공하는 경우:
1. Acknowledge — summarise what will change
1. 승인 — 변경 사항을 요약합니다.
2. Return to Step 2 (Fixed Top-Level Structure or compute() Decomposition as appropriate)
2. 2단계(고정된 최상위 구조 또는 적절하게 계산() 분해)로 돌아가기
3. Re-run Step 3a validation on the revised code
3. 수정된 코드에 대해 3a단계 검증을 다시 실행하세요.
4. Re-present this review gate
4. 이 리뷰 게이트를 다시 제시하세요.
5. Repeat until the user says yes
5. 사용자가 '예'라고 답할 때까지 반복합니다.

If the user gives feedback:
사용자가 피드백을 제공하는 경우:
1. Acknowledge — summarise what will change
1. 승인 — 변경 사항을 요약합니다.
2. Return to Step 2 (Fixed Top-Level Structure or compute() Decomposition as appropriate)
2. 2단계(고정된 최상위 구조 또는 적절하게 계산() 분해)로 돌아가기
3. Re-run Step 3a validation on the revised code
3. 수정된 코드에 대해 3a단계 검증을 다시 실행하세요.
4. Re-present this review gate
4. 이 리뷰 게이트를 다시 제시하세요.
5. Repeat until the user says yes
5. 사용자가 '예'라고 답할 때까지 반복합니다.

```
→ HANDOFF to Write Architecture to perf_outcomes.md
  Produces : Explicit "yes" approval from user
  Condition: User has confirmed the architecture matches expectations
  Blocked if: User gives revision feedback → loop back to Step 2 → Step 3a → re-present review
```

---

#### 3c — Write Architecture to perf_outcomes.md
#### 3c — perf_outcomes.md에 아키텍처 쓰기

Write the design configuration to a repo-local memory file.
디자인 구성을 repo-local 메모리 파일에 씁니다.
Resolve the repo root first so the path works regardless of where the skill was invoked:
기술이 호출된 위치에 관계없이 경로가 작동하도록 먼저 저장소 루트를 확인합니다.

```bash
REPO_ROOT="$(git -C . rev-parse --show-toplevel 2>/dev/null || pwd)"
PERF_LOG="$REPO_ROOT/.claude/memory/perf_outcomes.md"
mkdir -p "$REPO_ROOT/.claude/memory"
```

Then:
그런 다음:

**If `$PERF_LOG` does not exist** — create it with the full template (header, All Designs Summary table, Design section).
**`$PERF_LOG`가 존재하지 않는 경우** — 전체 템플릿(헤더, 모든 디자인 요약 테이블, 디자인 섹션)을 사용하여 생성합니다.
**If `$PERF_LOG` exists** — append a new Design section; add one row to the All Designs Summary table.
**`$PERF_LOG`가 존재하는 경우** — 새 디자인 섹션을 추가합니다. 모든 설계 요약 테이블에 행 하나를 추가합니다.
Never overwrite existing content.
기존 콘텐츠를 덮어쓰지 마세요.

```markdown
## Design: <name>

### Configuration

| Field | Value |
|---|---|
| Description | <one-line description of what the kernel does> |
| Part | <XPART from hls_config.cfg> |
| Clock | <CLOCK_NS> ns = <MHz> MHz |
| Target | <throughput target if given by user, else "TBD"> |
| Architecture | Hierarchical (compute internally staged)  OR  Monolithic (compute atomic) |
| Working dir | <path to design directory> |
| Source file | <path to kernel source file> |
| Build command | `make csynth`  or  `python run.py csynth` |

### Architecture

#### Stage Breakdown

| Stage | Function | Input | Output | Loop trip count | Est. cycles |
|---|---|---|---|---|---|
| 1 | load_input()     | <m_axi port> | stream / PIPO | <N> | <N> |
| 2 | compute()        | stream / PIPO | stream / PIPO | <N> | <N> |
| N | store_output()   | stream / PIPO | <m_axi port>  | <N> | <N> |

#### compute() Internal Stages (if hierarchical)

| Sub-stage | Function | Input | Output | Loop trip count | Est. cycles |
|---|---|---|---|---|---|
| 1 | compute_stage1() | stream / PIPO | stream / PIPO | <N> | <N> |
| … | compute_stage2() | … | … | … | … |

#### Bottleneck

| Bottleneck stage | Est. cycles | Est. throughput @ <MHz> MHz |
|---|---|---|
| <stage_name> | <N> cycles | ~<X> FPS  /  ~<Y> Msps |

### Target TI Calculation

(To be filled in by /hls-optimize Step 1b after throughput target is confirmed)

### Attempt History

| # | target_ti | Achieved TI | Achieved Rate | Outcome |
|---|---|---|---|---|
| — | — | — | — | Architecture written by /hls-architect — optimization not started |
```

---

#### 3d-1 — Apply Performance Pragmas (if throughput target)
#### 3d-1 - 성능 Pragma 적용(처리량이 목표인 경우)

**IF THROUGHPUT_TARGET is EMPTY**: Skip to Step 3d-2
**THROUGHPUT_TARGET이 비어 있는 경우**: 3d-2단계로 건너뛰세요.

**IF THROUGHPUT_TARGET is NON-EMPTY**:
**THROUGHPUT_TARGET이 비어 있지 않은 경우**:

Call `/hls-perf-pragma` to:
`/hls-perf-pragma`를 호출하여 다음을 수행합니다.
1. Calculate target_ti from throughput
1. 처리량에서 target_ti 계산
2. Cascade through architecture/loops
2. 아키텍처/루프를 통한 캐스케이드
3. Apply pragmas to source (Position 1: top-level, Position 2: all loops)
3. 소스에 pragma 적용(위치 1: 최상위 수준, 위치 2: 모든 루프)
4. Record cascade table in perf_outcomes.md
4. perf_outcomes.md에 계단식 테이블 기록

```bash
cd <design_name>/rearchitect/v1/
/hls-perf-pragma $THROUGHPUT_TARGET
```

hls-perf-pragma will:
hls-perf-pragma는 다음을 수행합니다.
- Modify src/*.cpp (add pragmas)
- src/*.cpp 수정(pragma 추가)
- Write cascade table to perf_outcomes.md
- perf_outcomes.md에 캐스케이드 테이블 쓰기

**DO NOT git commit yet** — commit happens after baseline snapshot.
**아직 git commit하지 마세요** — 커밋은 기본 스냅샷 이후에 발생합니다.

Print:
인쇄:
```
─────────────────────────────────────────────────────
[architect]  Step 3d-1 ✓ Pragmas Applied
  Throughput     : $THROUGHPUT_TARGET
  Target TI      : <value> cycles
  Applied to     : Top-level function + <N> loops
  Recorded in    : perf_outcomes.md
─────────────────────────────────────────────────────
```
#### 3d-2 — Save Baseline Snapshot
#### 3d-2 — 기준 스냅샷 저장

Before handing off to `/hls-optimize`, save a clean copy of the architect-generated code:
`/hls-optimize`로 전달하기 전에 건축가가 생성한 코드의 깨끗한 복사본을 저장하세요.

```bash
cd <design_name>
cp -r rearchitect/v1 architect_baseline
echo "Baseline saved to architect_baseline/"
```

This preserves the architecture-stage output before `/hls-optimize` begins iterating, allowing you to:
이렇게 하면 `/hls-optimize`가 반복을 시작하기 전에 아키텍처 단계 출력이 보존되어 다음을 수행할 수 있습니다.
- Compare optimize iterations against the architect baseline
- 설계 기준과 최적화 반복 비교
- Revert to the clean architecture if optimization goes off-track
- 최적화가 궤도를 벗어나면 클린 아키텍처로 되돌리기
- Archive the macro-architecture separately from micro-optimizations
- 마이크로 아키텍처와 별도로 매크로 아키텍처를 보관합니다.

Print confirmation:
인쇄 확인:
```
─────────────────────────────────────────────────────
[architect]  Baseline snapshot saved
  Source      : <design_name>/rearchitect/v1/
  Snapshot    : <design_name>/architect_baseline/
  Contents    : src/*.cpp, tb/*.cpp, hls_config.cfg, vitis-comp.json
─────────────────────────────────────────────────────
```

---

#### 3e — Hand Off to Optimize
#### 3e — 최적화를 위해 핸드오프

```
─────────────────────────────────────────────────────
[architect]  Handing off to /hls-optimize
─────────────────────────────────────────────────────
✓ Step 1   Study domain patterns
✓ Step 2a  Draw activity timeline + choose paradigm
✓ Step 2b  Generate fixed top-level (load → compute → store)
✓ Step 2c  Decompose compute() — hierarchical or monolithic
✓ Step 2d  Write to rearchitect/ + pragma audit
✓ Step 2e  G++ functional verification
✓ Step 3a  Validate with hls* skills
✓ Step 3b  Architecture review (user approval)
✓ Step 3c  Write to perf_outcomes.md
✓ Step 3d-1  Apply performance pragmas (if throughput)
✓ Step 3d-2  Save baseline snapshot
  Step 3e  Hand off to /hls-optimize                       ← NOW
```

Invoke `/hls-optimize` passing `THROUGHPUT_TARGET` parsed from `$ARGUMENTS`:
`$ARGUMENTS`에서 구문 분석된 `THROUGHPUT_TARGET`을 전달하는 `/hls-optimize`를 호출합니다.

**If THROUGHPUT_TARGET is non-empty:**
**THROUGHPUT_TARGET이 비어 있지 않은 경우:**
```
/hls-optimize <THROUGHPUT_TARGET>
```

**If THROUGHPUT_TARGET is empty** — ask the user:
**THROUGHPUT_TARGET이 비어 있는 경우** — 사용자에게 다음을 물어보세요.

> "What is your throughput target? (e.g.
> "처리량 목표는 무엇입니까? (예:
> 140 FPS, 500 Msps, 1 GFLOPS)"
> 140FPS, 500Msps, 1GFLOPS)"

Wait for the response, then invoke:
응답을 기다린 후 다음을 호출합니다.
```
/hls-optimize <user-provided target>
```

> **Note on pragma scope:** `/hls-optimize` may internally use `#pragma HLS PIPELINE`, `UNROLL`, `ARRAY_PARTITION`, etc. — the Pragma Rules restriction above applies **only to architect-generated code**, not to the optimize skill.
> **pragma 범위에 대한 참고 사항:** `/hls-optimize`는 내부적으로 `#pragma HLS PIPELINE`, `UNROLL`, `ARRAY_PARTITION` 등을 사용할 수 있습니다. — 위의 Pragma 규칙 제한은 **아키텍트 생성 코드**에만 적용되며 최적화 기술에는 적용되지 않습니다.

The optimize skill runs: hls-perf-pragma → csim → csynth → iterate → cosim.
최적화 기술은 hls-perf-pragma → csim → csynth → iterate → cosim을 실행합니다.

---

## Reference
## 참고자료

### §1 — AMD UG1399 Prescription (Checklist)
### §1 — AMD UG1399 처방(체크리스트)

> "Software written for CPUs and software written for FPGAs is fundamentally different.
> "CPU용으로 작성된 소프트웨어와 FPGA용으로 작성된 소프트웨어는 근본적으로 다릅니다.
> Embrace this."
> 이것을 받아들여라."

1. **Establish verification first** — testbench with golden reference before any optimization
1. **먼저 검증 설정** — 최적화 전에 골든 참조가 포함된 테스트벤치
2. **Focus on macro-architecture first** — model with producer-consumer paradigm
2. **먼저 매크로 아키텍처에 초점** — 생산자-소비자 패러다임을 갖춘 모델
3. **Draw the activity timeline** — identify parallelism and bottlenecks across multiple invocations
3. **활동 타임라인 그리기** — 여러 호출에서 병렬성과 병목 현상을 식별합니다.
4. **Only code after macro-architecture is set** — no premature micro-optimization
4. **매크로 아키텍처가 설정된 이후의 코드만** — 조기 마이크로 최적화 없음
5. **HLS infers task-level parallelism only from function calls** — concurrent blocks must be separate functions
5. **HLS는 함수 호출에서만 작업 수준 병렬성을 추론합니다** — 동시 블록은 별도의 함수여야 합니다
6. **Decompose into small modular components** — smaller components can be replicated for parallelism
6. **소형 모듈식 구성 요소로 분해** — 병렬 처리를 위해 더 작은 구성 요소를 복제할 수 있습니다.
7. **Aim for a single loop nest per function** — simplifies throughput measurement
7. **기능당 단일 루프 중첩을 목표로 합니다** — 처리량 측정을 단순화합니다.
8. **Avoid wide channels** — decompose into narrower ones
8. **넓은 채널은 피하세요** - 더 좁은 채널로 분해하세요
9. **Avoid large inlined functions** — complex control paths harm tool QoR
9. **큰 인라인 함수를 피하세요** — 복잡한 제어 경로는 도구 QoR에 해를 끼칩니다.
10. **Size streams correctly** — FIFO depth must prevent deadlock
10. **올바른 스트림 크기** — FIFO 깊이는 교착 상태를 방지해야 합니다.
11. **Use HLS reports to guide optimization** — never optimize blind
11. **HLS 보고서를 사용하여 최적화 안내** — 절대 맹목적으로 최적화하지 마세요
