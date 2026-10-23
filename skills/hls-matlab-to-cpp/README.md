<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# AMD Vitis HLS™ — LLM Skills for Algorithm → RTL
# AMD Vitis HLS™ — 알고리즘을 위한 LLM 기술 → RTL

A collection of [Claude Code](https://claude.com/product/claude-code) skills that take an algorithm written in **MATLAB** or **C++** and walk it all the way to **synthesizable, optimized RTL** for AMD FPGAs using **Vitis HLS**.
**MATLAB** 또는 **C++**로 작성된 알고리즘을 가져와 **Vitis HLS**를 사용하여 AMD FPGA용 **합성 가능하고 최적화된 RTL**까지 진행하는 [Claude Code](https://claude.com/product/claude-code) 기술 모음입니다.

Skills live under `.claude/skills/`.
기술은 `.claude/skills/`에 있습니다.
Each one owns a single, well-defined stage of the flow.
각각은 흐름의 잘 정의된 단일 단계를 소유합니다.

> **First time here?** Read top-to-bottom — sections are in the order you will actually use the skills.
> **여기 처음 오시나요?** 위에서 아래로 읽으세요. 섹션은 실제로 기술을 사용하는 순서대로 되어 있습니다.

---

## Software Requirements
## 소프트웨어 요구사항

### Required Software
### 필수 소프트웨어

- **Claude Code**: Required to invoke skills
- **클로드 코드**: 스킬 발동에 필요
  
- **AMD Vitis HLS**: Version 26.1 or 25.2
- **AMD Vitis HLS**: 버전 26.1 또는 25.2
  - Verification command: `which vitis-run`
  - 확인 명령: `which vitis-run`
  
- **MATLAB**: 2025b (for `/hls-matlab-to-cpp` workflow)
- **MATLAB**: 2025b(`/hls-matlab-to-cpp` 워크플로용)
  - Required for running `.m` scripts and generating golden reference files
  - `.m` 스크립트를 실행하고 골든 참조 파일을 생성하는 데 필요합니다.
  
- **OpenCV**: Version 4.x (conditional - only if testbench uses OpenCV)
- **OpenCV**: 버전 4.x(조건부 - 테스트벤치가 OpenCV를 사용하는 경우에만)
  - Installation guide: https://adaptivesupport.amd.com/s/article/Vitis-Libraries-Compiling-and-Installing-OpenCV?language=en_US
  - 설치 가이드: https://adaptivesupport.amd.com/s/article/Vitis-Libraries-Compiling-and-Installing-OpenCV?언어=en_US

### Environment Setup
### 환경설정

**Set environment variables in your shell or make them permanent**
**셸에서 환경 변수를 설정하거나 영구적으로 설정**

Export in your shell **before** launching Claude Code (recommended):
Claude Code를 실행하기 **전에** 셸에서 내보내기(권장):
```bash
# In your terminal (bash/csh), BEFORE launching Claude Code:
export OPENCV_INCLUDE=/path/to/opencv/install/include/opencv4
export OPENCV_LIB=/path/to/opencv/install/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$OPENCV_LIB
export MATLAB_BIN=/path/to/matlab/bin/matlab

# Then launch Claude Code from THIS shell
claude-code
```

Claude Code inherits environment variables from the shell it was launched from.
Claude Code는 실행된 셸에서 환경 변수를 상속합니다.

**Or make it permanent** (add to `.bashrc` or `.cshrc`):
**또는 영구적으로 만드세요**(`.bashrc` 또는 `.cshrc`에 추가):
```bash
# Add to ~/.bashrc (bash) or ~/.cshrc (csh):
export OPENCV_INCLUDE=/path/to/opencv/install/include/opencv4
export OPENCV_LIB=/path/to/opencv/install/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$OPENCV_LIB
export MATLAB_BIN=/path/to/matlab/bin/matlab
```

Then every new shell (and Claude Code launched from it) will have these variables.
그러면 모든 새 셸(및 여기에서 시작된 Claude Code)에는 이러한 변수가 있습니다.

**What [`/setup`](.claude/skills/setup/SKILL.md) does**:
**[`/setup`](.claude/skills/setup/SKILL.md)의 기능**:
- **Vitis HLS**: Verifies `$XILINX_VITIS` is set and `vitis-run` is on `$PATH` (source your Vitis installation's `settings64.sh` beforehand if not already set)
- **Vitis HLS**: `$XILINX_VITIS`가 설정되어 있고 `vitis-run`이 `$PATH`에 있는지 확인합니다(아직 설정되지 않은 경우 미리 Vitis 설치의 `settings64.sh`를 소스로 제공).
- **OpenCV**: Uses `$OPENCV_INCLUDE` and `$OPENCV_LIB` from your environment
- **OpenCV**: 사용자 환경에서 `$OPENCV_INCLUDE` 및 `$OPENCV_LIB`를 사용합니다.
- **MATLAB**: Uses `$MATLAB_BIN` from your environment or finds `matlab` on `$PATH`
- **MATLAB**: 사용자 환경에서 `$MATLAB_BIN`을 사용하거나 `$PATH`에서 `matlab`을 찾습니다.

---

## Run Your Design
## 디자인 실행

### Option 1: MATLAB → HLS C++
### 옵션 1: MATLAB → HLS C++

Start with MATLAB algorithm and go all the way to optimized RTL.
MATLAB 알고리즘으로 시작하여 최적화된 RTL까지 가십시오.

In Claude Code, `cd` to the folder containing your `<design_name>.m` and its testbench, then:
Claude Code에서 `<design_name>.m`과 해당 테스트벤치가 포함된 폴더로 `cd`한 후 다음을 수행합니다.

```
/hls-matlab-to-cpp  <design_name>  <throughput_target>  part=<fpga_part>  clock=<ns>
```

Where:
어디에:
- `<design_name>` — the `.m` filename **without** the extension (e.g.
- `<디자인_이름>` — 확장자가 **없는** `.m` 파일 이름(예:
- `rgbEdgeDetector`, not `rgbEdgeDetector.m`)
- `rgbEdgeDetector.m`이 아닌 `rgbEdgeDetector`)
- `<throughput_target>` — `<N> FPS` for image kernels, `<N> Msps` for sample-rate DSP, etc.
- `<throughput_target>` — 이미지 커널의 경우 `<N> FPS`, 샘플링 속도 DSP의 경우 `<N> Msps` 등
- `<fpga_part>` — full Xilinx part string (find via `lsdev -p` or AMD docs)
- `<fpga_part>` — 전체 Xilinx 부분 문자열(`lsdev -p` 또는 AMD 문서를 통해 찾기)
- `<ns>` — clock period in nanoseconds (e.g.
- `<ns>` — 나노초 단위의 클록 주기(예:
- `3.3` ≈ 303 MHz)
- '3.3' ≒ 303MHz)

**Example:**
**예:**
```
/hls-matlab-to-cpp  rgbEdgeDetector  4400 FPS  part=xczu9eg-ffvb1156-2-e  clock=3.3
```

This runs the entire pipeline: [`/setup`](.claude/skills/setup/SKILL.md) → [`/hls-matlab-to-cpp`](.claude/skills/hls-matlab-to-cpp/SKILL.md) → [`/architect`](.claude/skills/architect/SKILL.md) → [`/hls-optimize`](.claude/skills/hls-optimize/SKILL.md).
이는 전체 파이프라인을 실행합니다: [`/setup`](.claude/skills/setup/SKILL.md) → [`/hls-matlab-to-cpp`](.claude/skills/hls-matlab-to-cpp/SKILL.md) → [`/architect`](.claude/skills/architect/SKILL.md) → [`/hls-optimize`](.claude/skills/hls-optimize/SKILL.md).

---

### Option 2: C++ → HLS C++
### 옵션 2: C++ → HLS C++

Start with existing C++ code (kernel + testbench) and optimize to meet performance targets.
기존 C++ 코드(커널 + 테스트벤치)로 시작하여 성능 목표를 충족하도록 최적화합니다.

In Claude Code, `cd` to the folder containing your `kernel.cpp`, `kernel.hpp`, and `main.cpp`:
Claude Code에서 `kernel.cpp`, `kernel.hpp` 및 `main.cpp`가 포함된 폴더로 `cd`합니다.

```
/hls-optimize  <optimization_criteria>
```

**Optimization criteria:** Natural language describing your goal
**최적화 기준:** 목표를 설명하는 자연어

Examples:
예:
- `/hls-optimize minimize latency`
- `/hls-최적화 대기 시간 최소화`
- `/hls-optimize minimize DSP usage while meeting timing`
- `/hls-optimize 타이밍을 맞추면서 DSP 사용량 최소화`
- `/hls-optimize maximize throughput (minimize II)`
- `/hls-optimize 처리량 최대화(최소화 II)`
- `/hls-optimize reduce LUT count below 5000`
- `/hls-optimize LUT 수를 5000 미만으로 줄입니다`

**Matrix Multiplication Example:**
**행렬 곱셈 예:**

```bash
cd hls-intro-matmul
```

Then in Claude Code:
그런 다음 클로드 코드에서:
```
/hls-optimize make it 8x faster than baseline
```

This runs: [`/hls-optimize`](.claude/skills/hls-optimize/SKILL.md).
이는 [`/hls-optimize`](.claude/skills/hls-optimize/SKILL.md)를 실행합니다.

The skill will:
기술은 다음과 같습니다.
1. Analyze your `kernel.cpp` (simple triple-nested matrix multiply)
1. `kernel.cpp` 분석(간단한 삼중 중첩 행렬 곱셈)
2. Generate HLS DATAFLOW architecture
2. HLS DATAFLOW 아키텍처 생성
3. Optimize pragmas to meet the criteria (8x speedup)
3. 기준을 충족하도록 pragma 최적화(8배 속도 향상)
4. Produce RTL ready for integration
4. 통합 준비가 완료된 RTL 생성

### What success looks like
### 성공은 어떤 모습인가

Expect ~30 minutes for a clean end-to-end run on a small kernel.
작은 커널에서 완전한 엔드투엔드 실행을 위해서는 약 30분 정도가 소요됩니다.
At each stage the skill prints a banner; the final `/hls-optimize` summary should look like:
각 단계에서 스킬은 배너를 인쇄합니다. 최종 `/hls-optimize` 요약은 다음과 같습니다.

```
─────────────────────────────────────────────────────
[optimize]  done
  ✓ /perf-pragma  target_ti = 68847 cycles  (4400 FPS @ 303 MHz)
  ✓ /csim         PASS — max abs error = 0
  ✓ /csynth       II = 1   latency = 68843 cycles   target_ti met
  ✓ /cosim        post-RTL latency = 68847 cycles
  ✓ Resources     LUT 12,453   FF 9,872   BRAM 14   DSP 21
─────────────────────────────────────────────────────
```

If the printed `target_ti met = no` after `/hls-optimize`, the skill loops back into pragma tuning automatically; you'll see multiple `/csim → /csynth` cycles with the chosen pragma diff between iterations.
`/hls-optimize` 뒤에 `target_ti met = no`가 인쇄되면 기술은 자동으로 pragma 조정으로 다시 돌아갑니다. 반복 사이에 선택된 pragma diff가 포함된 여러 `/csim → /csynth` 주기가 표시됩니다.

---

## The big picture
## 큰 그림

```
  ┌────────────┐  /hls-matlab-to-cpp  ┌──────────────┐  /architect   ┌──────────────────┐  /hls-optimize   ┌──────────────────┐
  │  MATLAB    │ ───────────────► │  frame_based │ ────────────► │  rearchitect/v1  │ ───────────► │   optimized RTL  │
  │ algorithm  │                  │  kernel.cpp  │               │   (load/compute  │              │  (target_ti met, │
  │     .m     │                  │   + tb.cpp   │               │   /store + DF)   │              │   pragmas tuned) │
  └────────────┘                  └──────────────┘               └──────────────────┘              └──────────────────┘
```

Three handoffs, each owned by one skill:
각각 하나의 기술이 소유하는 세 가지 핸드오프:

| Handoff<br>핸드오프 | Skill<br>스킬 | Input<br>입력 | Output<br>출력 |
|---|---|---|---|
| **MATLAB → C++**<br>**MATLAB → C++** | `/hls-matlab-to-cpp`<br>`/hls-matlab-to-cpp` | `algorithm.m` + golden test<br>`algorithm.m` + 골든 테스트 | `frame_based/kernel.cpp` (functionally identical, fixed-point types)<br>`frame_based/kernel.cpp`(기능적으로 동일한 고정 소수점 유형) |
| **C++ → HLS macro-architecture**<br>**C++ → HLS 매크로 아키텍처** | `/architect`<br>`/건축가` | `frame_based/kernel.cpp`<br>`frame_based/kernel.cpp` | `rearchitect/v1/kernel_hls.cpp` (load → compute → store, DATAFLOW)<br>`rearchitect/v1/kernel_hls.cpp`(로드 → 컴퓨팅 → 저장, DATAFLOW) |
| **Macro-arch → optimized RTL**<br>**매크로아치 → 최적화된 RTL** | `/hls-optimize`<br>`/hls-최적화` | `kernel_hls.cpp` + throughput target<br>`kernel_hls.cpp` + 처리량 목표 | Pragma-tuned source + RTL meeting `target_ti`<br>Pragma 조정 소스 + RTL 회의 `target_ti` |

---

## Stage 1 — MATLAB → C++   (`/hls-matlab-to-cpp`)
## 1단계 — MATLAB → C++(`/hls-matlab-to-cpp`)

Converts a sample-based MATLAB algorithm to frame-based C++ with **bit-exact fixed-point types**.
샘플 기반 MATLAB 알고리즘을 **정확한 비트 고정 소수점 유형**을 사용하여 프레임 기반 C++로 변환합니다.

```
  Input (your files)              /hls-matlab-to-cpp does this              Output tree
  ──────────────────              ──────────────────────                ───────────
  ┌──────────────┐                ┌─────────────────────────┐           ┌──────────────────┐
  │ algorithm.m  │                │ 1. Run MATLAB → capture │           │ design/golden/        │
  │ testbench.m  │ ─────────────► │    matlab_input.bin     │ ────────► │ design/sample_based/  │
  │              │                │    matlab_golden.bin    │           │ design/frame_based/   │
  └──────────────┘                │ 2. Refactor 1: sample-  │           └──────────────────┘
                                  │    based C++ (line-by- │
                                  │    line port)          │
                                  │ 3. Range analysis →    │
                                  │    ap_fixed<W,I> types │
                                  │ 4. Refactor 2: frame-  │
                                  │    based C++           │
                                  │ 5. Verify vs golden    │
                                  └─────────────────────────┘
```

**Why two refactors?** The sample-based step preserves MATLAB semantics so divergence is caught early.
**리팩터링이 두 개여야 하는 이유는 무엇입니까?** 샘플 기반 단계에서는 MATLAB 의미 체계가 보존되므로 발산이 조기에 포착됩니다.
The frame-based step is the shape HLS expects.
프레임 기반 단계는 HLS가 기대하는 형태입니다.

**Key artifact:** `frame_based/kernel.cpp` — passes the same golden as the MATLAB.
**주요 아티팩트:** `frame_based/kernel.cpp` — MATLAB과 동일한 골든을 전달합니다.

---

## Stage 2 — C++ → HLS macro-architecture   (`/architect`)
## 2단계 — C++ → HLS 매크로 아키텍처(`/architect`)

Converts the frame-based C++ into a producer-consumer HLS dataflow.
프레임 기반 C++를 생산자-소비자 HLS 데이터 흐름으로 변환합니다.
The architect chooses **macro structure only** — pragmas like `PIPELINE`/`UNROLL` are deferred to `/hls-optimize`.
설계자는 **매크로 구조만** 선택합니다. `PIPELINE`/`UNROLL`과 같은 pragma는 `/hls-optimize`로 연기됩니다.

```
  ┌─────────────────────────────── #pragma HLS DATAFLOW ────────────────────────────────┐
  │                                                                                     │
  │   ┌──────────────┐         ┌──────────────┐          ┌──────────────┐               │
  │   │ load_input() │ stream  │   compute()  │ stream   │ store_output()│              │
  │   │              │ ──────► │              │ ───────► │               │              │
  │   │ m_axi → strm │         │  algorithmic │          │  strm → m_axi │              │
  │   └──────────────┘         │     work     │          └──────────────┘               │
  │                            └──────┬───────┘                                         │
  │                                   │                                                 │
  │                       (if decomposable, hierarchical:)                              │
  │                                   ▼                                                 │
  │   ┌────────────── inner #pragma HLS DATAFLOW ──────────────┐                        │
  │   │  compute_stage1 ──► compute_stage2 ──► … ──► stageN    │                        │
  │   └──────────────────────────────────────────────────────────┘                      │
  └─────────────────────────────────────────────────────────────────────────────────────┘
```

**The architect is allowed to write only:** `DATAFLOW`, `INTERFACE`, `STREAM depth=N`, `performance target_ti=N`.
**설계자는 쓰기만 허용됩니다:** `DATAFLOW`, `INTERFACE`, `STREAM 깊이=N`, `performance target_ti=N`.
**It must NOT write:** `PIPELINE`, `UNROLL`, `ARRAY_PARTITION`, `ARRAY_RESHAPE`, `INLINE`, `FLATTEN`.
**다음과 같이 쓰면 안 됩니다.** `PIPELINE`, `UNROLL`, `ARRAY_PARTITION`, `ARRAY_RESHAPE`, `INLINE`, `FLATTEN`.
Those belong to `/hls-optimize`.
이는 `/hls-optimize`에 속합니다.

**Key artifact:** `rearchitect/v1/src/kernel_hls.cpp` + `hls_config.cfg`.
**주요 아티팩트:** `rearchitect/v1/src/kernel_hls.cpp` + `hls_config.cfg`.
After this stage the architect saves a snapshot to `architect_baseline/` before handing off.
이 단계가 끝나면 건축가는 전달하기 전에 'architect_baseline/'에 스냅샷을 저장합니다.

---

## Stage 3 — Optimization loop   (`/hls-optimize`)
## 3단계 — 최적화 루프(`/hls-optimize`)

Closes the loop between **what you target** and **what HLS achieves**.
**목표로 삼는 것**과 **HLS가 달성하는 것** 사이의 루프를 닫습니다.
Iterates pragmas until throughput target is met.
처리량 목표가 충족될 때까지 pragma를 반복합니다.

```
  throughput target (e.g. 140 FPS)
              │
              ▼
  ┌──────────────────────┐
  │  /perf-pragma        │  compute target_ti cascade
  │  top + per-loop      │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │  /csim               │  verify functional + profile loop trip counts
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │  /csynth             │  → RTL + latency/II/resource report
  └──────────┬───────────┘
             │
       target met?
        │        │
       no        yes
        │         │
        ▼         ▼
  ┌─────────┐   ┌──────────┐    ┌──────────┐
  │ tune 1  │   │ /cosim   │ ─► │ /impl    │ ─► done
  │ pragma  │   │ true RTL │    │ P&R real │
  │ (loop ▲)│   │ latency  │    │   Fmax   │
  └────┬────┘   └──────────┘    └──────────┘
       └──────────► back to /csim
```

Each iteration changes **one thing** so cause-and-effect stays clean.
반복할 때마다 **한 가지**가 변경되므로 원인과 결과가 깨끗하게 유지됩니다.
Outcomes recorded in `perf_outcomes.md`.
결과는 `perf_outcomes.md`에 기록됩니다.

---

## Directory layout per design
## 디자인별 디렉토리 레이아웃

```
design_name/
├── golden/               ← MATLAB sources + matlab_input.bin + matlab_golden.bin
├── sample_based/         ← refactor 1: line-by-line C++ port
├── frame_based/          ← refactor 2: frame-based C++ (HLS-shaped)
└── rearchitect/
    └── v1/
        ├── src/                 ← kernel_hls.cpp (load → compute → store)
        ├── tb/                  ← testbench.cpp (copied from frame_based/)
        ├── hls_config.cfg
        └── architect_baseline/  ← snapshot saved before /hls-optimize
```

`design-layout` skill manages this tree — never `mkdir` manually.
'design-layout' 스킬은 이 트리를 관리합니다. 절대로 수동으로 'mkdir'을 사용하지 마세요.

---

## Skill catalog
## 스킬 카탈로그

| Skill<br>스킬 | Purpose<br>목적 |
|---|---|
| [`/setup`](.claude/skills/setup/SKILL.md)<br>[`/setup`](.claude/skills/setup/SKILL.md) | Verify Vitis / MATLAB / OpenCV; discover design + build commands<br>Vitis / MATLAB / OpenCV 확인; 디자인 + 빌드 명령 검색 |
| [`/hls-matlab-to-cpp`](.claude/skills/hls-matlab-to-cpp/SKILL.md)<br>[`/hls-matlab-to-cpp`](.claude/skills/hls-matlab-to-cpp/SKILL.md) | MATLAB → frame-based C++ with `ap_fixed` types<br>MATLAB → `ap_fixed` 유형을 사용하는 프레임 기반 C++ |
| [`/design-layout`](.claude/skills/design-layout/SKILL.md)<br>[`/design-layout`](.claude/skills/design-layout/SKILL.md) | Create canonical directory tree per stage (`golden/`, `frame_based/`, `rearchitect/v1/` …)<br>단계별 표준 디렉터리 트리 생성(`golden/`, `frame_based/`, `rearchitect/v1/` …) |
| [`/architect`](.claude/skills/architect/SKILL.md)<br>[`/architect`](.claude/skills/architect/SKILL.md) | Frame-based C++ → HLS dataflow macro-architecture<br>프레임 기반 C++ → HLS 데이터 흐름 매크로 아키텍처 |
| [`/perf-pragma`](.claude/skills/perf-pragma/SKILL.md)<br>[`/perf-pragma`](.claude/skills/perf-pragma/SKILL.md) | Cascade `target_ti` from top-level → loops<br>최상위 레벨에서 `target_ti` 캐스케이드 → 루프 |
| [`/hls-optimize`](.claude/skills/hls-optimize/SKILL.md)<br>[`/hls-optimize`](.claude/skills/hls-optimize/SKILL.md) | Iterate pragmas until throughput target met<br>처리량 목표가 충족될 때까지 pragma를 반복합니다. |
| [`/vitis-libraries`](.claude/skills/vitis-libraries/SKILL.md)<br>[`/vitis-libraries`](.claude/skills/vitis-libraries/SKILL.md) | Study production HLS coding patterns before writing<br>작성하기 전에 제작 HLS 코딩 패턴을 연구하십시오. |
| [`/hls-line-buffer`](.claude/skills/hls-line-buffer/SKILL.md)<br>[`/hls-line-buffer`](.claude/skills/hls-line-buffer/SKILL.md) | Stencil II=1 line-buffer recipe<br>스텐실 II=1 라인 버퍼 레시피 |

**Validation skills (called by `/architect` to check the generated code):**
**검증 기술(생성된 코드를 확인하기 위해 `/architect`에 의해 호출됨):**

- [`/hls-dataflow`](.claude/skills/hls-dataflow/SKILL.md) — DATAFLOW canonical-form check
- [`/hls-dataflow`](.claude/skills/hls-dataflow/SKILL.md) — DATAFLOW 표준 형식 확인
- [`/hls-synthesizable`](.claude/skills/hls-synthesizable/SKILL.md) — guards against unsynthesizable C++
- [`/hls-synthessible`](.claude/skills/hls-synthessible/SKILL.md) — 합성할 수 없는 C++로부터 보호합니다.
- [`/hls-burst-inference`](.claude/skills/hls-burst-inference/SKILL.md) — verify AXI bursts will be inferred
- [`/hls-burst-inference`](.claude/skills/hls-burst-inference/SKILL.md) — AXI 버스트가 추론되는지 확인
- [`/hls-flattenable`](.claude/skills/hls-flattenable/SKILL.md) — loop-nest flatten eligibility
- [`/hls-plattenable`](.claude/skills/hls-plattenable/SKILL.md) — 루프 중첩 평면화 적격성
- [`/hls-array-to-stream`](.claude/skills/hls-array-to-stream/SKILL.md) — opportunities to convert PIPO → stream
- [`/hls-array-to-stream`](.claude/skills/hls-array-to-stream/SKILL.md) — PIPO → 스트림으로 변환할 수 있는 기회
- [`/hls-stencil-pattern`](.claude/skills/hls-stencil-pattern/SKILL.md) — stencil-shape recognition
- [`/hls-stencil-pattern`](.claude/skills/hls-stencil-pattern/SKILL.md) — 스텐실 모양 인식

---

## Example Designs
## 예시 설계

End-to-end examples under this repo:
이 저장소의 엔드 투 엔드 예:

**MATLAB → HLS Examples:**
**MATLAB → HLS 예:**
- **[`test-demosaic/`](test-demosaic/)** — Bayer demosaicing (Malvar et al. linear interpolation)
- **[`test-demosaic/`](test-demosaic/)** — 바이엘 디모자이싱(Malvar 외 선형 보간법)
- **[`test-edge-detection/`](test-edge-detection/)** — RGB edge detection (Sobel-style, 3 modes)
- **[`test-edge-Detection/`](test-edge-Detection/)** — RGB 가장자리 감지(Sobel 스타일, 3가지 모드)

**C++ → HLS Examples:**
**C++ → HLS 예:**
- **[`hls-intro-matmul/`](hls-intro-matmul/)** — Matrix multiplication (simple triple-nested loop baseline)
- **[`hls-intro-matmul/`](hls-intro-matmul/)** — 행렬 곱셈(간단한 삼중 중첩 루프 기준선)

Each has its own README with the algorithm description and how to drive the LLM flow against it.
각각에는 알고리즘 설명과 이에 대해 LLM 흐름을 구동하는 방법이 포함된 자체 README가 있습니다.

---

---

## License
## 라이센스

MIT.
MIT.
Copyright © 2026 AMD.
저작권 © 2026 AMD.
