<!-- ko-translation: enabled -->
---
name: hls-matlab-to-cpp
description: Convert MATLAB sample-based code to plain C++ frame-based loops — analyze algorithm, generate C++ that compiles with g++, verify against MATLAB golden, then hand off to /hls-architect for HLS dataflow architecture.
license: MIT
argument-hint: <matlab-file.m> [part=<fpga-part>] [clock=<ns>] [throughput=<target>]
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

# MATLAB to C++ Conversion
# MATLAB에서 C++로 변환

Convert MATLAB sample-based algorithms into frame-based plain C++ that compiles with g++, then hand off to `/hls-architect` for multi-stage HLS dataflow architecture.
MATLAB 샘플 기반 알고리즘을 g++로 컴파일되는 프레임 기반 일반 C++로 변환한 다음 다단계 HLS 데이터 흐름 아키텍처를 위해 '/hls-architect'로 전달합니다.

---

## Preamble — Verify Tooling + Capture Required Parameters
## 서문 - 툴링 확인 + 필수 매개변수 캡처

### Step 0 — Run ./reference/setup.md in tooling-only mode
### 0단계 - 도구 전용 모드에서 ./reference/setup.md 실행

All required tooling (Vitis HLS, MATLAB, OpenCV) is verified by the `./reference/setup.md` skill.
필요한 모든 도구(Vitis HLS, MATLAB, OpenCV)는 `./reference/setup.md` 기술로 확인됩니다.
At this point in the flow there is no design directory yet, so call `./reference/setup.md` with
흐름의 이 시점에는 아직 디자인 디렉터리가 없으므로 `./reference/setup.md`를 호출하세요.
`mode=tooling-only` — design discovery and build commands (Steps 2 & 3 of ./reference/setup.md)
`mode=tooling-only` — 디자인 검색 및 빌드 명령(./reference/setup.md의 2단계 및 3단계)
are skipped.
건너뜁니다.

```
./reference/setup.md mode=tooling-only
```

The `./reference/setup.md` skill will:
`./reference/setup.md` 스킬은 다음을 수행합니다.
- Source Vitis (and export `$XILINX_VITIS`) — Step 1
- Vitis 소스(및 `$XILINX_VITIS` 내보내기) — 1단계
- Detect & verify OpenCV install if the testbench will use `cv::imread` — Step 4
- 테스트벤치가 `cv::imread`를 사용할 경우 OpenCV 설치 감지 및 확인 — 4단계
- Locate MATLAB and export `$MATLAB_BIN` — Step 5
- MATLAB을 찾아 `$MATLAB_BIN` 내보내기 — 5단계

If `./reference/setup.md` cannot resolve a tool (e.g.
`./reference/setup.md`가 도구를 확인할 수 없는 경우(예:
MATLAB not on PATH, OpenCV lib path missing),
MATLAB이 PATH에 없음, OpenCV lib 경로 누락),
it prompts the user with `AskUserQuestion`. **Do not proceed until `./reference/setup.md` reports
사용자에게 'AskUserQuestion'이라는 메시지를 표시합니다. **`./reference/setup.md`가 보고될 때까지 진행하지 마세요.
all three checks passed (or marked N/A).**
세 가지 검사가 모두 통과되었습니다(또는 N/A로 표시됨).**

After `./reference/setup.md` returns, the following env vars are guaranteed to be set:
`./reference/setup.md`가 반환된 후 다음 환경 변수가 설정되는 것이 보장됩니다.

| Env var<br>환경변수          | Set by<br>설정자         | Used by<br>사용처                                   |
|------------------|----------------|-------------------------------------------|
| `$XILINX_VITIS`<br>`$XILINX_VITIS`  | `./reference/setup.md` Step 1<br>`./reference/setup.md` 1단계| g++ compile commands in Step 2 & Step 4<br>2단계 및 4단계의 g++ 컴파일 명령   |
| `$OPENCV_INCLUDE`<br>`$OPENCV_INCLUDE`| `./reference/setup.md` Step 4<br>`./reference/setup.md` 4단계| g++ `-I` flag for testbench compile<br>테스트벤치 컴파일을 위한 g++ `-I` 플래그       |
| `$OPENCV_LIB`<br>`$OPENCV_LIB`    | `./reference/setup.md` Step 4<br>`./reference/setup.md` 4단계| g++ `-L` flag for testbench compile<br>테스트벤치 컴파일을 위한 g++ `-L` 플래그       |
| `$MATLAB_BIN`<br>`$MATLAB_BIN`    | `./reference/setup.md` Step 5<br>`./reference/setup.md` 5단계| Step 0 — running the user's `.m` files<br>0단계 - 사용자의 `.m` 파일 실행    |

Downstream skills (`/hls-architect`, `/hls-optimize`, `/csim`, `/csynth`, `/cosim`) inherit
다운스트림 기술(`/hls-architect`, `/hls-optimize`, `/csim`, `/csynth`, `/cosim`) 상속
this environment — **never re-verify or re-source**.
이 환경 — **다시 확인하거나 리소스를 확보하지 마세요**.

### Capture pipeline parameters
### 파이프라인 매개변수 캡처

Parse parameters from `$ARGUMENTS`:
`$ARGUMENTS`에서 매개변수를 구문 분석합니다.

**Expected format:**
**예상 형식:**
```
/matlab-to-cpp <file.m> [part=<fpga-part>] [clock=<ns>] [throughput=<target>]
```

**Examples:**
**예:**
```
/matlab-to-cpp rgbEdgeDetector.m part=xczu9eg-ffvb1156-2-e clock=3.3 throughput=4400fps
/matlab-to-cpp demosaic.m part=xczu9eg-ffvb1156-2-e clock=3.5
/matlab-to-cpp filter.m
```

**Parse logic:**
**분석 논리:**
1. Extract MATLAB file path (first non-key=value argument)
1. MATLAB 파일 경로 추출(키=값이 아닌 첫 번째 인수)
2. Extract `part=...` → store as `XPART`
2. `part=...` 추출 → `XPART`로 저장
3. Extract `clock=...` → store as `CLOCK_NS`
3. `clock=...` 추출 → `CLOCK_NS`로 저장
4. Extract `throughput=...` → store as `THROUGHPUT_TARGET`
4. `throughput=...` 추출 → `THROUGHPUT_TARGET`로 저장

**No silent defaults.** If any of `<matlab-file>`, `part=`, `clock=`, or `throughput=` is missing from `$ARGUMENTS`, prompt the user with `AskUserQuestion`:
**자동 기본값은 없습니다.** `<matlab-file>`, `part=`, `clock=` 또는 `throughput=` 중 하나라도 `$ARGUMENTS`에서 누락된 경우 사용자에게 `AskUserQuestion` 메시지를 표시합니다.

| Missing arg<br>인수 누락 | Question header<br>질문 헤더 | Question text<br>질문 텍스트 |
|---|---|---|
| matlab file<br>MATLAB 파일        | "MATLAB script"<br>"MATLAB 스크립트" | "Which MATLAB script should I convert? (path to `.m` file)"<br>"어떤 MATLAB 스크립트를 변환해야 합니까?(`.m` 파일 경로)" |
| `part=`<br>`부분=`            | "FPGA part"<br>"FPGA 부분"     | "FPGA part to target? (e.g. `xczu9eg-ffvb1156-2-e`)"<br>"대상으로 삼을 FPGA 부분? (예: `xczu9eg-ffvb1156-2-e`)" |
| `clock=`<br>`시계=`           | "Clock period"<br>"시계 기간"  | "Clock period in ns? (e.g. `3.3` for ~303 MHz)"<br>"클럭 주기는 ns 단위입니까?(예: ~303MHz의 경우 `3.3`)" |
| `throughput=`<br>`처리량=`      | "Throughput"<br>"처리량"    | "Throughput target? (e.g. `4400 FPS`, `500 Msps`, `1 GFLOPS`)"<br>"처리량 목표는 무엇입니까? (예: '4400 FPS', '500 Msps', '1 GFLOPS')" |

**Why no defaults:** silent defaults (`xczu9eg`, `3.5 ns`) used to mask user typos and propagate the wrong values into `/hls-architect` and `/hls-optimize`.
**기본값이 없는 이유:** 사용자 오타를 가리고 잘못된 값을 `/hls-architect` 및 `/hls-optimize`에 전파하는 데 사용되는 자동 기본값(`xczu9eg`, `3.5 ns`)입니다.
The prompt forces an explicit choice and surfaces parsing failures immediately.
프롬프트는 명시적인 선택을 강제하고 구문 분석 실패를 즉시 표시합니다.

After all four are resolved, print confirmation:
네 가지 문제가 모두 해결되면 인쇄 확인이 이루어집니다.
```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Pipeline parameters
  Source     : <matlab-file>
  Part       : <XPART>
  Clock      : <CLOCK_NS> ns  (≈ <MHz> MHz)
  Throughput : <THROUGHPUT_TARGET>
─────────────────────────────────────────────────────
```
  
## Flow Overview
## 흐름 개요

Print this at the very start so the user can track progress throughout:
사용자가 전체 진행 상황을 추적할 수 있도록 맨 처음에 이것을 인쇄하십시오.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[matlab-to-cpp]  Pipeline — <design_name>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Step 0   Run MATLAB simulation → save golden I/O
  Step 0b  Range instrumentation → sim-measured types
  Step 1   Analyze MATLAB algorithm
  Step 2   Generate refactor_1 (plain C++) + verify
  Step 3   Generate refactor_2 (frame-based C++)
  Step 4   Hand off to /hls-architect → /hls-optimize
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Re-print this overview after each step completes with a `✓` next to the completed step so the user always knows where the flow is.
각 단계가 완료된 후 이 개요를 다시 인쇄하고 완료된 단계 옆에 `✓`를 표시하여 사용자가 항상 흐름의 위치를 알 수 있도록 하세요.

---

## Design Workspace
## 디자인 작업공간

Before running Step 0, derive `design_name` from the MATLAB script filename (snake_case, no `.m` extension).
0단계를 실행하기 전에 MATLAB 스크립트 파일 이름(snake_case, `.m` 확장자 없음)에서 `design_name`을 파생시킵니다.

Call `./reference/design-layout.md` to orient the workspace:
작업공간 방향을 지정하려면 `./reference/design-layout.md`를 호출하세요.
```
./reference/design-layout.md  design_name=<name>  stage=show
```
This prints the full expected directory tree without creating anything.
그러면 아무것도 생성하지 않고 예상되는 전체 디렉토리 트리가 인쇄됩니다.
Confirm the tree with the user, then proceed — each step calls `./reference/design-layout.md` to create its directory before writing files.
사용자와 함께 트리를 확인한 다음 진행하세요. 각 단계에서는 파일을 쓰기 전에 './reference/design-layout.md'를 호출하여 디렉터리를 만듭니다.

---

## Step 0 / 4 — Run MATLAB Simulation (Golden Reference)
## 0/4단계 - MATLAB 시뮬레이션 실행(골든 참조)

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 0 / 4 — Run MATLAB Simulation (Golden Reference)
─────────────────────────────────────────────────────
```

Before converting anything, run the MATLAB simulation to capture golden inputs and outputs.
무엇이든 변환하기 전에 MATLAB 시뮬레이션을 실행하여 골든 입력 및 출력을 캡처하십시오.
These are used in Step 2 to verify the generated C++.
이는 생성된 C++를 확인하기 위해 2단계에서 사용됩니다.

### Launch MATLAB CLI
### MATLAB CLI 실행

Use `$MATLAB_BIN` — exported by `./reference/setup.md` Step 5.
`$MATLAB_BIN` 사용 - `./reference/setup.md`로 내보냄 5단계.
Do NOT hardcode a path here; that breaks for any user other than the one whose path was baked in.
여기에 경로를 하드코딩하지 마세요. 경로가 구운 사용자 이외의 모든 사용자에 대해서는 중단됩니다.

```bash
"$MATLAB_BIN" -nodisplay -nodesktop
```

If `$MATLAB_BIN` is empty at this point, `./reference/setup.md mode=tooling-only` was not run (or it failed) — go back and re-run it before continuing.
이 시점에서 `$MATLAB_BIN`이 비어 있으면 `./reference/setup.md mode=tooling-only`가 실행되지 않은 것입니다(또는 실패한 것입니다). 계속하기 전에 돌아가서 다시 실행하십시오.

### Run the Simulation
### 시뮬레이션 실행

Once the MATLAB CLI is open, run the `*_runme.m` script in the design directory:
MATLAB CLI가 열리면 디자인 디렉터리에서 `*_runme.m` 스크립트를 실행합니다.

```matlab
>> run('path/to/design_runme.m')
```

**Important:**
**중요:**
- Run **only the simulation section** — do **not** run any MtoHDL Coder or code generation sections
- **시뮬레이션 섹션만** 실행합니다. MtoHDL Coder 또는 코드 생성 섹션은 실행하지 **않습니다**
- If the runme script contains both, stop before the HDL/HLS coder calls
- runme 스크립트에 둘 다 포함되어 있으면 HDL/HLS 코더가 호출하기 전에 중지하십시오.

### Save Inputs and Outputs
### 입력 및 출력 저장

Save both the **input data** (stimulus) and **output data** (golden reference) so the C++ testbench can use the same files:
C++ 테스트벤치가 동일한 파일을 사용할 수 있도록 **입력 데이터**(자극)와 **출력 데이터**(골든 참조)를 모두 저장합니다.

```matlab
% Save input
fid = fopen('matlab_input.bin', 'wb');
fwrite(fid, input_array, 'uint8');    % match the actual data type
fclose(fid);

% Save golden output
fid = fopen('matlab_golden.bin', 'wb');
fwrite(fid, output_array, 'uint8');
fclose(fid);
```

Keep note of: output dimensions, data type, and border pixels that are undefined (e.g., warmup edges for filter kernels).
정의되지 않은 출력 차원, 데이터 유형 및 경계 픽셀(예: 필터 커널의 워밍업 가장자리)을 기록해 두십시오.

### Write to golden/
### 황금/에 쓰기

Call `./reference/design-layout.md  design_name=<name>  stage=golden`
`./reference/design-layout.md design_name=<name> stage=golden`을 호출하세요.

Then copy into `design_name/golden/`:
그런 다음 `design_name/golden/`에 복사합니다.
- All MATLAB source files (`*.m`) from the input design path
- 입력 설계 경로의 모든 MATLAB 소스 파일(`*.m`)
- `matlab_input.bin` — saved stimulus
- `matlab_input.bin` — 저장된 자극
- `matlab_golden.bin` — saved golden reference output
- `matlab_golden.bin` — 저장된 골든 참조 출력

Print before proceeding:
계속하기 전에 인쇄하세요:
```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 0 done — MATLAB Simulation
  Input saved  : golden/matlab_input.bin   (<N> bytes, <dtype>)
  Output saved : golden/matlab_golden.bin  (<N> bytes, <dtype>)
  Dimensions   : <ROWS × COLS or signal length>
  Border/warmup: <KSIZE/2 rows+cols invalid / N/A>
─────────────────────────────────────────────────────
✓ Step 0   Run MATLAB simulation → save golden I/O
  Step 0b  Range instrumentation → sim-measured types   ← NEXT
  Step 1   Analyze MATLAB algorithm
  Step 2   Generate refactor_1 (plain C++) + verify
  Step 3   Generate refactor_2 (frame-based C++)
  Step 4   Hand off to /hls-architect → /hls-optimize
```

---

## Step 0b — Range Instrumentation
## 0b단계 - 범위 계측

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 0b — Range Instrumentation
  0b.0  Detect design type (A: script / B: function-based)
  0b.1  Generate instrumented MATLAB files
  0b.2  Re-run MATLAB, collect RANGE lines
  0b.2b Precision sweep — measure F end-to-end vs golden
  0b.3  Build range table (filter constants/masks)
  0b.4  Derive W and I for each variable
  0b.4b Select Q and O modes (closest to MATLAB double)
  0b.4c Report quantization error (from the 0b.2b measurement)
  0b.4d Size fixed-point multiply intermediates
  0b.5  Present table → user accept / refine loop
─────────────────────────────────────────────────────
```

Capture **min/max of every intermediate variable** from the MATLAB simulation.
MATLAB 시뮬레이션에서 **모든 중간 변수의 최소/최대**를 캡처합니다.
This drives exact-bitwidth type selection in Step 2 — tight widths reduce area and improve achievable II.
이는 2단계에서 정확한 비트폭 유형 선택을 유도합니다. 좁은 폭은 면적을 줄이고 달성 가능한 II를 향상시킵니다.
Without this, type selection falls back to conservative static analysis.
이것이 없으면 유형 선택은 보수적인 정적 분석으로 대체됩니다.

### 0.
### Detect design type
### 디자인 유형 감지

Examine the design directory:
디자인 디렉터리를 살펴보세요.

- **Type A — script-based:** the top-level runme calls no sub-function `.m` files; all computation is inline.
- **유형 A — 스크립트 기반:** 최상위 Runme은 하위 함수 `.m` 파일을 호출하지 않습니다. 모든 계산은 인라인입니다.
- The MATLAB `who` command can see all variables after the script runs.
- MATLAB `who` 명령은 스크립트가 실행된 후 모든 변수를 볼 수 있습니다.
- **Type B — function-based:** the design has multiple `.m` files that start with the `function` keyword.
- **유형 B — 함수 기반:** 디자인에는 `function` 키워드로 시작하는 `.m` 파일이 여러 개 있습니다.
- Sub-function internals are invisible to `who` — only the function's return values appear in the caller's workspace.
- 하위 함수 내부는 '누구'에게 보이지 않습니다. 호출자의 작업 공간에는 함수의 반환 값만 나타납니다.
- These require per-function instrumentation.
- 이를 위해서는 기능별 계측이 필요합니다.

**Detection rule:** scan the design directory for `.m` files.
**탐지 규칙:** 디자인 디렉터리에서 '.m' 파일을 검색합니다.
If more than one file begins with `function` (excluding the runme/tb scripts), it is **Type B**.
두 개 이상의 파일이 'function'으로 시작하는 경우(runme/tb 스크립트 제외) **유형 B**입니다.

### 1.
### Generate instrumented MATLAB files
### 계측된 MATLAB 파일 생성

#### Type A — append snippet to runme
#### 유형 A — runme에 스니펫 추가

Add this block to the **end** of the `*_runme.m` script, after the main algorithm has already executed.
기본 알고리즘이 이미 실행된 후 `*_runme.m` 스크립트의 **끝**에 이 블록을 추가하세요.
Do not modify the algorithm itself.
알고리즘 자체를 수정하지 마십시오.

```matlab
% --- Range instrumentation — append after main algorithm ---
fprintf('\n=== RANGE INSTRUMENTATION ===\n');
vars = who;
for k = 1:length(vars)
    v = eval(vars{k});
    if isnumeric(v) && ~isempty(v) && numel(v) > 1
        vmin   = double(min(v(:)));
        vmax   = double(max(v(:)));
        is_int = all(v(:) == floor(v(:)));
        fprintf('RANGE %-30s  min=%12.4f  max=%12.4f  integer=%d\n', ...
                vars{k}, vmin, vmax, double(is_int));
    end
end
fprintf('=== END RANGE INSTRUMENTATION ===\n');
```

#### Type B — auto-generate instrumented sub-functions
#### 유형 B — 계측된 하위 기능 자동 생성

Use the two scripts shipped with this repo under `./hls-scripts/`:
`./hls-scripts/` 아래에 이 저장소와 함께 제공된 두 개의 스크립트를 사용하십시오.

```bash
SCRIPTS_DIR="./hls-scripts"
```

Both `gen_all_instr.py` and `gen_matlab_instr.py` ship with the repo — no external download required.
`gen_all_instr.py`와 `gen_matlab_instr.py` 모두 저장소와 함께 제공됩니다. 외부 다운로드가 필요하지 않습니다.

**Step 1: Generate `*_instr.m` for every sub-function:**
**1단계: 모든 하위 기능에 대해 `*_instr.m` 생성:**

```bash
python3 $SCRIPTS_DIR/gen_all_instr.py --dir <design_dir>
```

This scans the design directory, finds all files starting with `function`, skips `*_instr`, `*_runme`, `*_tb`, `*_range`, `*_init`, and generates `<name>_instr.m` for each.
이는 디자인 디렉토리를 스캔하고 `function`으로 시작하는 모든 파일을 찾고 `*_instr`, `*_runme`, `*_tb`, `*_range`, `*_init`를 건너뛰고 각각에 대해 `<name>_instr.m`을 생성합니다.
The prefix embedded in every `RANGE` line equals the filename stem (e.g., `hb2_fir_i.m` → prefix `hb2_fir_i`).
모든 `RANGE` 줄에 포함된 접두사는 파일 이름 어간과 같습니다(예: `hb2_fir_i.m` → 접두사 `hb2_fir_i`).

Print the summary output from `gen_all_instr.py` — it lists every generated file and its tracked variable count.
`gen_all_instr.py`에서 요약 출력을 인쇄합니다. 여기에는 생성된 모든 파일과 추적된 변수 개수가 나열됩니다.

**Step 2: Create or update an instrumented runner.**
**2단계: 계측된 실행기를 생성하거나 업데이트합니다.**

Create `<design_name>_full_instr_runme.m` in the design directory that:
다음과 같은 디자인 디렉터리에 `<design_name>_full_instr_runme.m`을 만듭니다.
1. Calls `clear functions` at the top to reset all persistent variables
1. 모든 영구 변수를 재설정하려면 상단의 `clear 함수`를 호출합니다.
2. Calls the `*_instr` versions of every sub-function (not the originals)
2. 모든 하위 함수의 `*_instr` 버전을 호출합니다(원본이 아님)
3. Uses the same stimulus and parameters as the original runme
3. 원래 runme과 동일한 자극 및 매개변수를 사용합니다.
4. Prints any top-level coefficient/LUT ranges directly via `fprintf('RANGE ...')` before the pipeline loop
4. 파이프라인 루프 전에 `fprintf('RANGE ...')`를 통해 직접 최상위 계수/LUT 범위를 인쇄합니다.

The RANGE lines are emitted by each `*_instr.m` function automatically at the end of their execution — no additional snippet is needed in the runner.
RANGE 줄은 실행이 끝나면 각 `*_instr.m` 함수에 의해 자동으로 내보내집니다. 실행기에는 추가 스니펫이 필요하지 않습니다.

> **Note on loop-internal constants:** `gen_all_instr.py` tracks ALL named assignments inside the outer loop, including constants like `bit_start = 21` or `offset = fi(1025,...)`.
> **루프 내부 상수에 대한 참고 사항:** `gen_all_instr.py`는 `bit_start = 21` 또는 `offset = fi(1025,...)`와 같은 상수를 포함하여 외부 루프 내부의 모든 명명된 할당을 추적합니다.
> These will appear in the RANGE output with trivial ranges (e.g., `[21,21]`).
> 이는 사소한 범위(예: `[21,21]`)와 함께 RANGE 출력에 나타납니다.
> Ignore them when building the type table — they are not signal-path variables.
> 유형 테이블을 작성할 때 이를 무시하십시오. 이는 신호 경로 변수가 아닙니다.

> **Note on output_direct temps:** assignments like `cout(i) = expr_with_no_named_intermediate` are captured via injected temp variables named `rng_<array>_N`.
> **output_direct 임시에 대한 참고사항:** `cout(i) = expr_with_no_named_intermediate`와 같은 할당은 `rng_<array>_N`이라는 이름의 주입된 임시 변수를 통해 캡처됩니다.
> These appear as `RANGE prefix_rng_<array>_N` and represent the actual output signal value — keep them in the type table.
> 이는 `RANGE prefix_rng_<배열>_N`으로 나타나며 실제 출력 신호 값을 나타냅니다. 유형 테이블에 보관하세요.

### 2.
### Re-run the simulation
### 시뮬레이션을 다시 실행하세요.

**STOP — do not proceed until MATLAB has actually been re-run.**
**중지 — MATLAB이 실제로 다시 실행될 때까지 진행하지 마십시오.**

The only valid source for the range table is the `RANGE` lines printed to the MATLAB console during this re-run.
범위 테이블의 유일한 유효한 소스는 이 재실행 중에 MATLAB 콘솔에 인쇄된 `RANGE` 라인입니다.
Any other source is forbidden:
다른 소스는 금지됩니다.

| Forbidden source<br>금지된 소스 | Why it fails<br>실패하는 이유 |
|---|---|
| Prior simulation output<br>사전 시뮬레이션 출력 | Does not contain loop-internal variables<br>루프 내부 변수를 포함하지 않습니다. |
| Code inspection / static analysis<br>코드 검사/정적 분석 | Cannot see runtime values or conditional variables<br>런타임 값이나 조건 변수를 볼 수 없습니다. |
| Type inference from input ranges<br>입력 범위에서 유형 추론 | Underestimates multiply/accumulate intermediates<br>중간값을 곱하거나 누적하는 것을 과소평가합니다. |
| Memory of a previous run<br>이전 실행의 기억 | Stale — types may have changed<br>오래됨 - 유형이 변경되었을 수 있음 |

Run the script:
스크립트를 실행합니다:

```bash
# Type A
/path/to/matlab -nodisplay -nodesktop -r "run('path/to/design_runme.m'); exit"

# Type B
/path/to/matlab -nodisplay -nodesktop -r "run('path/to/design_full_instr_runme.m'); exit"
```

**How to verify:** the output must contain `RANGE` lines — one per tracked variable per function.
**확인 방법:** 출력에는 함수당 추적 변수당 하나씩 'RANGE' 줄이 포함되어야 합니다.
For Type B, lines arrive interleaved as each `*_instr` function finishes its loop; collect all of them.
유형 B의 경우 각 `*_instr` 함수가 루프를 완료할 때 행이 인터리브되어 도착합니다. 모두 모아보세요.

Collect every line beginning with `RANGE` from the output.
출력에서 `RANGE`로 시작하는 모든 줄을 수집합니다.

### 2b.
### Precision sweep — measure F
### 정밀 스윕 - F 측정

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 0b.2b — Precision sweep
  Tolerance : 1% (skill default — see 0b.4c thresholds)
─────────────────────────────────────────────────────
```

The `RANGE` lines just collected answer **how big** each variable gets.
방금 수집된 'RANGE' 줄은 각 변수가 얼마나 커지는지에 대한 답을 제공합니다.
They cannot answer
그들은 대답할 수 없다
**how fine** it needs to be.
**얼마나 괜찮아** 그래야 할까요?
`F` is measured here, by running the algorithm at a series of
여기서 'F'는 일련의 알고리즘을 실행하여 측정됩니다.
fractional-bit widths and comparing the **final output** against the golden from Step 0.
분수 비트 너비를 계산하고 **최종 출력**을 0단계의 황금값과 비교합니다.

**Why this is measured and not derived.** `F` is a property of the algorithm and its input,
**이것이 측정되었으나 도출되지 않은 이유.** 'F'는 알고리즘 및 해당 입력의 속성입니다.
not of a variable's values.
변수의 값이 아닙니다.
Same tolerance, same peak magnitude, wildly different answers:
동일한 허용 오차, 동일한 피크 크기, 매우 다른 답변:

| Structure<br>구조 | Input<br>입력 | F for 1%<br>1%에 대한 F |
|---|---|---|
| 16-tap averaging FIR<br>16탭 평균 FIR | white<br>흰색 | 10 |
| adjacent-sample difference<br>인접 표본 차이 | white<br>흰색 | 6 |
| adjacent-sample difference<br>인접 표본 차이 | oversampled (smooth)<br>오버샘플링됨(부드러움) | **16** |
| single-pole IIR (`a=0.995`)<br>단극 IIR(`a=0.995`) | white<br>흰색 | 8 |

Rows 2 and 3 are the *same code* on the *same amplitude* — a 10-bit swing caused only by
행 2와 3은 *동일한 진폭*의 *동일한 코드*입니다.
how correlated the input samples are.
입력 샘플의 상관 관계.
Nothing in a `min`/`max` pair can see that.
`min`/`max` 쌍의 어떤 것도 이를 볼 수 없습니다.
Averaging
평균화
attenuates the signal while error stays put; differencing cancels the signal while errors
오류가 지속되는 동안 신호를 감쇠합니다. 차이점은 오류가 발생하는 동안 신호를 취소합니다.
add; recursion accumulates error across iterations.
추가; 재귀는 반복을 통해 오류를 누적합니다.
Only running it measures the result.
실행만 하면 결과가 측정됩니다.

#### Generate the quantized variants
#### 양자화된 변형 생성

```bash
# one file per design function; qF is a runtime argument, so one file serves the whole sweep
python hls-scripts/gen_matlab_instr.py <src>.m <src>_q.m <prefix> --quantize
```

This emits `<func>_q(..., qF)` with `x = q_(x, qF);` after every fractional assignment,
이것은 매 분수 할당 후에 `x = q_(x, qF);`와 함께 `<func>_q(..., qF)`를 방출합니다.
where `q_ = @(x,F) round(x .* 2^F) ./ 2^F` — round-to-nearest, matching `AP_RND_CONV`.
여기서 `q_ = @(x,F) round(x .* 2^F) ./ 2^F` — 가장 가까운 값으로 반올림하여 `AP_RND_CONV`와 일치합니다.
Variables detected as integer-valued are left alone (they are already on an exact grid).
정수 값으로 감지된 변수는 그대로 유지됩니다(이미 정확한 그리드에 있음).

#### Sweep driver
#### 스윕 드라이버

```matlab
% ── 0b.2b precision sweep ──────────────────────────────────────────────────
TOL    = 0.01;            % 1% — skill default, same number as the 0b.4c gate
F_LIST = 6:2:24;

ref  = <run the design unmodified, doubles>;      % or load golden/matlab_golden.bin
ref  = double(ref(:));
peak = max(abs(ref));
is_int_golden = all(ref == floor(ref));           % uint8/uint16 golden → bit-exact is reachable

fprintf('\nSWEEP   F   max_abs_err      rel_err   mismatch    SQNR(dB)  status\n');
fprintf('SWEEP  ─────────────────────────────────────────────────────────────\n');

F_pick = NaN;
for F = F_LIST
    out = double(reshape(<run the design with _q variants, passing qF = F>, [], 1));
    e   = out - ref;
    max_abs_err = max(abs(e));
    rel_err     = max_abs_err / peak;
    mismatch    = sum(e ~= 0);
    sqnr        = 20*log10(norm(ref) / max(norm(e), eps));

    ok = rel_err < TOL;
    if is_int_golden, ok = ok && (mismatch == 0); end   % integer golden ⇒ demand bit-exact
    if ok && isnan(F_pick), F_pick = F; end

    status = '';
    if ok, status = 'PASS'; end
    fprintf('SWEEP  %3d  %12.4e  %9.4f%%  %9d  %9.1f  %s\n', ...
            F, max_abs_err, 100*rel_err, mismatch, sqnr, status);
end
fprintf('SWEEP  smallest F meeting %.0f%% tolerance: %d\n', 100*TOL, F_pick);
```

#### Reading the result
#### 결과 읽기

```
SWEEP   F   max_abs_err      rel_err   mismatch    SQNR(dB)  status
SWEEP  ─────────────────────────────────────────────────────────────
SWEEP    6    1.9531e-02     4.5139%      4081       26.9
SWEEP    8    4.8828e-03     1.0723%      4079       39.0
SWEEP   10    1.2207e-03     0.2831%      4064       51.0  PASS
SWEEP   12    3.0518e-04     0.0654%      3971       63.0  PASS
SWEEP   14    7.6294e-05     0.0170%      3585       75.1  PASS
SWEEP  smallest F meeting 1% tolerance: 10
```

- **`F_pick`** is the answer.
- **`F_pick`**이 답입니다.
- Carry it into 0b.4 as `F`, giving `W = I + F`.
- 이를 `F`로 0b.4에 전달하여 `W = I + F`를 제공합니다.
- **SQNR rises ~6 dB per bit.** If it does not, quantization is not what limits accuracy —
- **SQNR은 비트당 ~6dB 상승합니다.** 그렇지 않은 경우 양자화는 정확도를 제한하지 않습니다.
suspect an algorithmic difference, not a width problem.
너비 문제가 아니라 알고리즘 차이가 의심됩니다.
- **`rel_err` must fall monotonically.** If it plateaus, adding bits will not help; the
- **`rel_err`은 단조롭게 감소해야 합니다.** 정체되면 비트를 추가해도 도움이 되지 않습니다. 는
error is structural (an `Inf`/`NaN` path, a saturating clamp, a divide by a small number).
오류는 구조적입니다(`Inf`/`NaN` 경로, 포화 클램프, 작은 숫자로 나누기).
- **Never report only `F_pick`.** The whole table goes to the user at 0b.5 so the accuracy
- **'F_pick'만 보고하지 마세요.** 전체 테이블은 0b.5에서 사용자에게 전달되므로 정확성이 높아집니다.
vs width trade is theirs to make, not the skill's.
대 폭 거래는 기술이 아닌 그들의 몫입니다.

**Margin.** `F_pick` is the *minimum* that met tolerance on *this* stimulus.
**마진.** `F_pick`은 *이* 자극에 대한 허용 오차를 충족하는 *최소*입니다.
Apply the same
동일하게 적용
reasoning as the Margin rule in 0b.4 and offer `F_pick + 2` as the safe default.
0b.4의 마진 규칙으로 추론하고 'F_pick + 2'를 안전한 기본값으로 제공합니다.

**If no F in the list passes:** do not silently extend the sweep forever.
**목록에 F가 통과되지 않은 경우:** 자동으로 스윕을 영원히 연장하지 마세요.
Extend once to
한 번 연장하여
`F_LIST = 6:2:40` and re-run.
`F_LIST = 6:2:40`을 입력하고 다시 실행하세요.
If it still fails, the design is not width-limited — stop and
그래도 실패하면 디자인의 폭이 제한되지 않는 것입니다. 중지하고
report that to the user rather than widening blindly.
맹목적으로 확대하는 대신 이를 사용자에게 보고합니다.

### 3.
### Build the range table
### 범위 테이블 작성

**First — print every `RANGE` line exactly as captured, unfiltered:**
**먼저 — 모든 `RANGE` 줄을 필터링되지 않고 캡처된 그대로 정확하게 인쇄합니다.**

```
RANGE <var1>   min=...  max=...  integer=...
RANGE <var2>   min=...  max=...  integer=...
...
```

Do not omit any line.
어떤 줄도 생략하지 마십시오.
This is the raw capture — show it all before making any decisions.
이것은 원시 캡처입니다. 결정을 내리기 전에 모든 것을 보여주세요.

**Second — classify each variable:**
**두 번째 — 각 변수를 분류합니다.**

| Variable<br>변수 | Min<br>최소 | Max<br>맥스 | Keep / Filter<br>유지/필터링 | Reason if filtered<br>필터링된 이유 |
|---|---|---|---|---|
| `prefix_acc`<br>`접두사_acc` | -1.2 | 1.1 | keep<br>유지하다 | signal-path accumulator<br>신호 경로 누산기 |
| `prefix_rng_cout_1`<br>`prefix_rng_cout_1` | -1.0 | 1.0 | keep<br>유지하다 | output_direct temp — actual output value<br>output_direct temp — 실제 출력 값 |
| `prefix_bit_start`<br>`접두사_비트_시작` | 21 | 21 | filter<br>필터 | loop-internal constant (trivial range)<br>루프 내부 상수(사소한 범위) |
| `prefix_bit_end`<br>`접두사_비트_끝` | 32 | 32 | filter<br>필터 | loop-internal constant (trivial range)<br>루프 내부 상수(사소한 범위) |
| `prefix_offset`<br>`접두사_오프셋` | 1025 | 1025 | filter<br>필터 | loop-internal constant (trivial range)<br>루프 내부 상수(사소한 범위) |

**Filter rules:**
**필터 규칙:**
- **Loop-internal constants**: min == max (or range is trivially the literal value) — these are assigned inside the loop but never change.
- **루프 내부 상수**: min == max(또는 range는 간단히 리터럴 값임) — 이는 루프 내부에 할당되지만 변경되지 않습니다.
- Filter them — they have no useful range information for HLS type sizing.
- 필터링하세요. HLS 유형 크기 조정에 유용한 범위 정보가 없습니다.
- **Type A scalars**: loop counters, config constants, boolean scalars — filter.
- **유형 A 스칼라**: 루프 카운터, 구성 상수, 부울 스칼라 — 필터.
- **Keep everything else**: signal-path variables, accumulators, intermediate products, output_direct temps (`rng_*`).
- **다른 모든 항목 유지**: 신호 경로 변수, 누산기, 중간 생성물, 출력_직접 온도(`rng_*`).

Build the type derivation table from the **kept** rows only:
**kept** 행에서만 유형 파생 테이블을 작성하십시오.

| Variable<br>변수 | Min<br>최소 | Max<br>맥스 | All-integer?<br>모든 정수? | Chosen HLS type<br>선택된 HLS 유형 |
|---|---|---|---|---|
| `im1_pre`<br>`im1_pre` | -219 | 1107 | yes<br>응 | `ap_int<12>`<br>`ap_int<12>` |
| `s_horiz`<br>`s_horiz` | 0 | 510 | yes<br>응 | `ap_uint<9>`<br>`ap_uint<9>` |
| `coeff_acc`<br>`coeff_acc` | -0.125 | 0.5 | **no** | `ap_fixed<4,1>`<br>`ap_고정<4,1>` |

### 4.
### Derive HLS type for each variable
### 각 변수에 대한 HLS 유형 도출

Types come from the **observed range table**, not from reading the MATLAB source.
유형은 MATLAB 소스를 읽는 것이 아니라 **관찰된 범위 테이블**에서 가져옵니다.
MATLAB
MATLAB
computes everything in double, so the instrumented ranges are the only evidence of what a
모든 것을 이중으로 계산하므로 계측된 범위는
variable actually needs.
변수가 실제로 필요합니다.

| Observed range<br>관찰된 범위 | Type<br>유형 |
|---|---|
| all integers, min ≥ 0<br>모든 정수, 최소 ≥ 0 | `ap_uint<N>`<br>`ap_uint<N>` |
| all integers, min < 0<br>모든 정수, 최소 < 0 | `ap_int<N>`<br>`ap_int<N>` |
| **any** fractional value<br>**임의** 분수 값 | `ap_fixed<W,I>`<br>`ap_fixed<W,I>` |

Any fractional value at all means `ap_fixed`.
분수 값은 'ap_fixed'를 의미합니다.
Using `ap_int` where MATLAB used a double
MATLAB이 double을 사용한 `ap_int` 사용
breaks bit-exactness — the property this entire flow exists to preserve.
비트 정확성을 깨뜨립니다. 이 전체 흐름이 보존하기 위해 존재하는 속성입니다.

#### Integer bits `I` — one formula for all three types
#### 정수 비트 `I` — 세 가지 유형 모두에 대한 하나의 공식

`I` is fixed entirely by observed magnitude.
'I'는 관측된 크기에 의해 완전히 고정됩니다.
Compute it the same way regardless of which
어느 쪽이든 상관없이 같은 방식으로 계산합니다.
of the three types you end up choosing:
결국 세 가지 유형 중 하나를 선택하게 됩니다.

```
A = max(|min|, |max|)                            peak magnitude, from the range table

if A == 0:                                       degenerate — see guard below
    I = 1
else:
    I = floor(log2(A)) + 1 + (1 if min < 0 else 0)   the +1 for sign only when min < 0
```

- `ap_uint<N>` / `ap_int<N>` — no fractional bits, so `N = I`
- `ap_uint<N>` / `ap_int<N>` — 소수 비트가 없으므로 `N = I`
- `ap_fixed<W,I>` / `ap_ufixed<W,I>` — `I` is this value, and `W = I + F`
- `ap_fixed<W,I>` / `ap_ufixed<W,I>` — `I`가 이 값이고, `W = I + F`

**`I` may be zero or negative — that is legal and frequently correct.** A signal whose peak
**`I`는 0이거나 음수일 수 있습니다. 이는 합법적이며 올바른 경우가 많습니다.**
is below 1.0 needs no integer bits at all; a peak of `4.97e-3` gives `I = -7`, placing the
1.0 미만이면 정수 비트가 전혀 필요하지 않습니다. '4.97e-3'의 피크는 'I = -7'을 제공하여
binary point outside the word so that all `W` bits are fractional.
모든 `W` 비트가 분수가 되도록 단어 외부의 이진 소수점입니다.

This matters for precision, not area: **`F = W - I`**, so every integer bit allocated to
이는 면적이 아니라 정밀도에 중요합니다. **`F = W - I`**, 따라서 모든 정수 비트는 다음에 할당됩니다.
range the signal never reaches is a fractional bit you do not get.
신호가 도달하지 못하는 범위는 얻을 수 없는 분수 비트입니다.
Holding `W = 34`, an
'W = 34'를 유지하면
`I` of `1` leaves `F = 33` (LSB `1.16e-10`); the correct `I` of `-7` leaves `F = 41`
`1`의 `I`는 `F = 33`(LSB `1.16e-10`)을 남깁니다. '-7'의 올바른 'I'는 'F = 41'이 됩니다.
(LSB `4.55e-13`) — same width, 8 more bits of resolution.
(LSB `4.55e-13`) — 동일한 너비, 8비트 이상의 해상도.

> **Guard — `A = 0`.** If the variable is all zeros across the whole simulation — a signal
> **가드 - `A = 0`.** 전체 시뮬레이션에서 변수가 모두 0인 경우 - 신호
> that never activates on this stimulus, a branch that is never taken, a coefficient array
> 이 자극에 대해 결코 활성화되지 않는 가지, 결코 취해지지 않는 가지, 계수 배열
> that is genuinely zero, or a constant `0` fed in as input — then `min = max = 0`, so
> 이는 실제로 0이거나 상수 `0`이 입력으로 입력된 경우 `min = max = 0`이므로
> `A = 0` and `log2(0) = -inf`.
> `A = 0` 및 `log2(0) = -inf`.
> The formula explodes: `I` comes out `-inf`, and any width
> 공식은 폭발합니다: `I`는 `-inf`로 나오고 모든 너비는
> derived from it is garbage.
> 그것에서 파생된 것은 쓰레기다.
>
> Special-case it before evaluating the log: **`if A == 0 → I = 1`** (a lone sign bit, or
> 로그를 평가하기 전에 특수한 경우: **`A == 0 → I = 1`**(고독한 부호 비트 또는
> a single integer bit if unsigned).
> 부호가 없는 경우 단일 정수 비트).
> One bit is enough to represent zero exactly, and the
> 1비트는 0을 정확하게 표현하는 데 충분하며,
> type stays legal so downstream arithmetic and `W = I + F` still work.
> 유형은 합법적이므로 다운스트림 산술 및 `W = I + F`는 계속 작동합니다.
>
> **But treat it as a signal, not just a number.** `A = 0` almost always means the
> **그러나 단순한 숫자가 아닌 신호로 취급하십시오.** `A = 0`은 거의 항상
> instrumentation never exercised that variable.
> 계측에서는 해당 변수를 실행하지 않았습니다.
> Before accepting `I = 1`, check which case
> `I = 1`을 수락하기 전에 어떤 경우인지 확인하세요.
> you are in:
> 당신은 다음 위치에 있습니다:
>
> | Cause | What to do |
> | 원인 | 해야 할 일 |
> |---|---|
> | Stimulus never drives this path (dead branch, unused mode) | Range is **not trustworthy** — the variable is unsized, not zero-sized.
> | 자극은 이 경로를 구동하지 않습니다(데드 브랜치, 사용되지 않는 모드) | 범위가 **신뢰할 수 없음** — 변수의 크기가 0이 아니라 크기가 지정되지 않았습니다.
> Re-run 0b.2 with stimulus that exercises it, or carry it forward marked unverified. |
> 0b.2를 실행하는 자극으로 다시 실행하거나 확인되지 않은 것으로 표시하여 앞으로 이동합니다. |
> | Variable is genuinely and permanently zero | `I = 1` is correct.
> | 변수는 실제로 영구적으로 0입니다 | '나=1'이 맞습니다.
> Better: fold the constant away — it is not a signal, it is a literal. |
> 더 나은 방법: 상수를 접는 것입니다. 이는 신호가 아니라 리터럴입니다. |
> | Assignment executed but the value happened to be 0 every time | Same as row 1 — the stimulus is too narrow.
> | 할당이 실행되었지만 매번 값이 0이 되었습니다 | 1행과 동일 - 자극이 너무 좁습니다.
> Widen it. |
> 넓히세요. |

> **Related degenerate case — `min = Inf`, `max = -Inf`.** This is *not* `A = 0`, and it is
> **관련 퇴화 사례 — `min = Inf`, `max = -Inf`.** 이는 `A = 0`이 *아님*이며,
> important not to confuse the two.
> 이 둘을 혼동하지 않는 것이 중요합니다.
> The generated tracker seeds `mn_x = inf; mx_x = -inf;`
> 생성된 추적기 시드 `mn_x = inf; mx_x = -inf;`
> before the loop, so a variable whose tracking line **never executed** prints:
> 루프 앞에 있으므로 추적 라인이 **절대 실행되지** 않는 변수는 다음을 인쇄합니다.
>
> ```
> RANGE  prefix_myvar    min=           Inf  max=          -Inf  integer=0
> RANGE prefix_myvar min= Inf 최대= -Inf 정수=0
> ```
>
> `A = max(|Inf|, |-Inf|) = Inf`, which blows up in the opposite direction.
> `A = max(|Inf|, |-Inf|) = Inf`, 반대 방향으로 폭발합니다.
> Causes: the
> 원인:
> assignment sits in a branch the stimulus never took, or the injection landed inside a
> 과제는 자극이 전혀 받아들여지지 않은 가지에 있거나 주사가
> `...` line continuation and silently broke. **Never size a type from an `Inf` row** — it
> ``...' 줄이 계속되고 조용히 끊어졌습니다. **`Inf` 행에서 유형의 크기를 조정하지 마십시오** —
> is an instrumentation result, not a range measurement.
> 범위 측정이 아닌 계측 결과입니다.
> Fix or re-stimulate, then re-run.
> 수정하거나 다시 자극한 후 다시 실행하세요.
>
> The two are distinguishable straight from the RANGE line, so use that:
> 두 개는 RANGE 라인과 구별 가능하므로 다음을 사용하십시오.
> `min=0, max=0` → assignment ran, value was always zero.
> `min=0, max=0` → 할당이 실행되었으며 값은 항상 0이었습니다.
> `min=Inf, max=-Inf` → assignment
> `min=Inf, max=-Inf` → 할당
> never ran at all.
> 전혀 뛰지 않았습니다.

Report every `A = 0` and every `Inf` variable explicitly in the 0b.5 table with the cause
원인과 함께 0b.5 테이블의 모든 `A = 0` 및 모든 `Inf` 변수를 명시적으로 보고합니다.
identified — never let one pass through silently as a normal sized row.
식별됨 - 일반 크기의 행처럼 자동으로 통과하지 않도록 하세요.

> **Exactness:** the rule is exact except when `min` is an exact negative power of two
> **정확성:** 'min'이 정확히 2의 음수인 경우를 제외하면 규칙은 정확합니다.
> (`[-128, 127]` → 9 instead of 8).
> (`[-128, 127]` → 8 대신 9).
> One bit conservative there, never unsafe — it can
> 약간 보수적이며 결코 안전하지 않습니다.
> over-allocate but never under-allocate, so `AP_WRAP` stays safe by construction.
> 과도하게 할당하되 과소 할당하지 마십시오. 따라서 `AP_WRAP`은 구성에 따라 안전하게 유지됩니다.

#### ap_fixed<W,I> sizing
#### ap_fixed<W,I> 크기 조정

**Formula:**
**공식:**
```
A = max(|min|, |max|)
I = floor(log2(A)) + 1 + (1 if min < 0 else 0)   (integer bits incl. sign; may be <= 0)
F = F_pick                                       measured in 0b.2b — never derived here
W = I + F
Example: range [-0.125, 0.5] → A = 0.5, min < 0
         I = floor(log2(0.5)) + 1 + 1 = -1 + 2 = 1
         F = 3  (from the 0b.2b sweep)  →  ap_fixed<4, 1>
```

> **`F` is never computed from the range table.** It is `F_pick` from the 0b.2b sweep.
> **'F'는 범위 테이블에서 계산되지 않습니다.** 0b.2b 스윕의 'F_pick'입니다.
> There is no valid formula for `F` in terms of `min`/`max`, for three reasons:
> 다음 세 가지 이유로 `min`/`max` 측면에서 `F`에 대한 유효한 공식이 없습니다.
>
> 1. **The information is not there.** A rule over "the values a variable took" needs every
> 1. **정보가 없습니다.** "변수가 취하는 값"에 대한 규칙은
> value; the `RANGE` lines carry exactly two numbers per variable.
> 가치; `RANGE` 줄은 변수당 정확히 두 개의 숫자를 전달합니다.
> 2. **The question often has no answer.** `0.7` and `0.1` have no finite binary expansion —
> 2. **질문에 답이 없는 경우가 많습니다.** `0.7`과 `0.1`에는 유한 이진 확장이 없습니다.
> no `F` represents them exactly, so "bits needed to be exact" is undefined for most data.
> 'F'는 이를 정확하게 나타내지 않으므로 "정확해야 하는 비트"는 대부분의 데이터에 대해 정의되지 않습니다.
> 3. **`F` is a property of the algorithm, not the variable.** Quantization error propagates:
> 3. **`F`는 변수가 아닌 알고리즘의 속성입니다.** 양자화 오류는 다음과 같이 전파됩니다.
> averaging attenuates signal while error stays put, differencing cancels signal while
> 평균화는 오류가 있는 동안 신호를 감쇠시키고, 차분은 신호를 취소하는 반면
> errors add, recursion accumulates.
> 오류가 추가되고 재귀가 누적됩니다.
> Two designs with identical range tables need
> 동일한 범위 테이블을 가진 두 가지 디자인이 필요합니다.
> different `F`.
> 'F'가 달라요.
> See the measured table in 0b.2b — the same code on the same amplitude
> 0b.2b의 측정된 표를 참조하십시오 - 동일한 진폭의 동일한 코드
> ranged from `F = 6` to `F = 16` purely with input correlation.
> 순전히 입력 상관관계에 따라 'F = 6'에서 'F = 16'까지의 범위를 갖습니다.
>
> `min`/`max` answer `I` exactly.
> `min`/`max`는 `I`에 정확하게 대답합니다.
> They cannot answer `F` at all.
> 그들은 'F'라고 전혀 대답할 수 없습니다.
> `W = I + F` joins the two.
> `W = I + F`는 둘을 결합합니다.

**Common ranges:**
**공통 범위:**
```
[0,   1.0]  → ap_ufixed<12, 1>   A=1.0, unsigned: floor(0)+1     = 1 int, 11 frac
[-4.0, 4.0] → ap_fixed <16, 4>   A=4.0, signed:   floor(2)+1+1   = 4 int, 12 frac
[0,   6.0]  → ap_ufixed<16, 3>   A=6.0, unsigned: floor(2.58)+1  = 3 int, 13 frac
```

Note how the third row is unsigned (`min = 0`) — spending a sign bit it never uses would
세 번째 행이 어떻게 부호가 없는지(`min = 0`) 참고하세요. 전혀 사용하지 않는 부호 비트를 사용하면
cost a fractional bit.
약간의 비용이 듭니다.
And if the first row is *measured* at `[0.027, 0.992]` rather than
그리고 첫 번째 행이 아닌 `[0.027, 0.992]`에서 *측정*된 경우
declared as `[0, 1.0]`, then `A = 0.992` and `I = floor(-0.012) + 1 = 0` →
`[0, 1.0]`으로 선언된 다음 `A = 0.992` 및 `I = Floor(-0.012) + 1 = 0` →
`ap_ufixed<12, 0>`, range `[0, 1)`, all 12 bits fractional.
`ap_ufixed<12, 0>`, 범위 `[0, 1)`, 모두 12비트 분수.
That is the correct answer, not
그게 정답이 아니라
a bug — but see the **Margin rule** below before committing to it.
버그 — 하지만 커밋하기 전에 아래의 **마진 규칙**을 참조하세요.

**CRITICAL - Do NOT "optimize" by converting to integers:**
**중요 - 정수로 변환하여 "최적화"하지 마십시오:**

Given a MATLAB weighted sum `y = 0.25*a + 0.5*b + 0.25*c` with observed range
관찰된 범위를 갖는 MATLAB 가중 합 `y = 0.25*a + 0.5*b + 0.25*c`가 주어지면
`[0.027, 0.992]`:

❌ **WRONG:**
❌ **틀림:**
```cpp
// "I'll convert to uint8 for hardware efficiency"
ap_uint<8> y = (64*a + 128*b + 64*c) >> 8;  // [0, 255] range
// ❌ This changes the numeric domain → different values → mismatches
```

✅ **CORRECT:**
✅ **맞음:**
```cpp
// Match MATLAB's domain
ap_ufixed<12,1> y = 0.25*a + 0.5*b + 0.25*c;  // [0, 1] range
// ✅ Same numeric domain → same values → bit-exact match
```

**Why this matters:**
**이것이 중요한 이유:**
- Rescaling [0,1] to [0,255] changes ALL downstream computations
- [0,1]에서 [0,255]로 크기를 조정하면 모든 다운스트림 계산이 변경됩니다.
- Any nonlinear step (`sqrt`, compare, saturate) gives **different results** in a different domain
- 모든 비선형 단계(`sqrt`, 비교, 포화)는 다른 영역에서 **다른 결과**를 제공합니다.
- Constants calibrated for [0,1] won't work in the [0,255] domain
- [0,1]에 대해 보정된 상수는 [0,255] 도메인에서 작동하지 않습니다.
- Result: mismatches in csim, failed verification
- 결과: csim 불일치, 확인 실패

**Rule:** If MATLAB uses floating-point, C++ uses `ap_fixed`.
**규칙:** MATLAB이 부동 소수점을 사용하는 경우 C++에서는 `ap_fixed`를 사용합니다.
Period.
기간.

---

> **Margin rule:** Add 1 extra bit beyond strict minimum when observed range comes from a single small test image.
> **여백 규칙:** 관찰된 범위가 하나의 작은 테스트 이미지에서 나온 경우 엄격한 최소값 이상으로 1비트를 추가합니다.
> Larger production images may push extremes slightly further.
> 더 큰 프로덕션 이미지는 극단을 약간 더 밀어붙일 수 있습니다.

### 4b.
### Select Q (quantization) and O (overflow) modes
### Q(양자화) 및 O(오버플로) 모드 선택

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 0b.4b — Q/O mode selection
─────────────────────────────────────────────────────
```

The full fixed-point type is `ap_fixed<W, I, Q, O>`.
전체 고정 소수점 유형은 `ap_fixed<W, I, Q, O>`입니다.
Default is `AP_TRN, AP_WRAP`.
기본값은 'AP_TRN, AP_WRAP'입니다.
Select Q and O to minimise deviation from MATLAB's double-precision result.
MATLAB의 배정밀도 결과로부터의 편차를 최소화하려면 Q와 O를 선택하십시오.

Reference — all valid modes (from `ap_decl.h`):
참조 — 모든 유효한 모드(`ap_decl.h`에서):

| Q mode<br>Q 모드 | Behaviour<br>행동 |
|---|---|
| `AP_TRN`<br>`AP_TRN` | Truncation toward −∞ (default)<br>−무한 방향으로 잘림(기본값) |
| `AP_TRN_ZERO`<br>`AP_TRN_ZERO` | Truncation toward zero<br>0을 향한 잘림 |
| `AP_RND`<br>`AP_RND` | Round toward +∞<br>+무한대 방향으로 반올림 |
| `AP_RND_ZERO`<br>`AP_RND_ZERO` | Round toward zero<br>0을 향해 반올림 |
| `AP_RND_MIN_INF`<br>`AP_RND_MIN_INF` | Round toward −∞<br>-무한대 방향으로 반올림 |
| `AP_RND_INF`<br>`AP_RND_INF` | Round away from zero (matches MATLAB `round()`)<br>0에서 멀어지는 방향으로 반올림(MATLAB `round()`와 일치) |
| `AP_RND_CONV`<br>`AP_RND_CONV` | Convergent / banker's rounding — round half to even (matches IEEE 754, MATLAB's default double arithmetic)<br>수렴/은행가 반올림 - 반올림하여 짝수로 반올림(MATLAB의 기본 이중 산술인 IEEE 754와 일치) |

| O mode<br>O 모드 | Behaviour<br>행동 |
|---|---|
| `AP_WRAP`<br>`AP_WRAP` | Wrap-around (default)<br>랩어라운드(기본값) |
| `AP_SAT`<br>`AP_SAT` | Saturate at ±max<br>±max에서 포화 |
| `AP_SAT_ZERO`<br>`AP_SAT_ZERO` | Saturate to zero on overflow<br>오버플로 시 0으로 포화 |
| `AP_SAT_SYM`<br>`AP_SAT_SYM` | Symmetrical saturation<br>대칭 채도 |
| `AP_WRAP_SM`<br>`AP_WRAP_SM` | Sign-magnitude wrap<br>부호 크기 랩 |

#### Overflow mode O — always derive from the range table
#### 오버플로 모드 O - 항상 범위 테이블에서 파생됩니다.

| Condition<br>상태 | O mode<br>O 모드 | Reason<br>이유 |
|---|---|---|
| Range table proves I bits cover min/max<br>범위 테이블은 I 비트가 최소/최대를 포괄함을 증명합니다. | `AP_WRAP` (default)<br>`AP_WRAP`(기본값) | Overflow never occurs → no saturation hardware, no overhead<br>오버플로가 발생하지 않음 → 포화 하드웨어 없음, 오버헤드 없음 |
| Intermediate result can temporarily exceed I bits<br>중간 결과는 일시적으로 I 비트를 초과할 수 있습니다. | Widen I instead<br>대신 I를 넓혀라 | AP_SAT masks bugs — fix the width, do not hide the overflow<br>AP_SAT 마스크 버그 — 너비를 수정하고 오버플로를 숨기지 마세요. |
| Final output boundary (e.g. clip to [0, 255])<br>최종 출력 경계(예: [0, 255]로 자르기) | Explicit `if/else` in C++<br>C++의 명시적 `if/else` | More readable than AP_SAT and visible in code<br>AP_SAT보다 읽기 쉽고 코드에서 볼 수 있음 |

**Rule: size I correctly from the range table and use AP_WRAP.
**규칙: 범위 테이블에서 I 크기를 올바르게 지정하고 AP_WRAP을 사용합니다.
Never use AP_SAT to compensate for an undersized I.**
크기가 작은 I를 보완하기 위해 AP_SAT를 사용하지 마십시오.**

#### Quantization mode Q — derive from how MATLAB computes each variable
#### 양자화 모드 Q — MATLAB이 각 변수를 계산하는 방법에서 파생됩니다.

MATLAB computes in IEEE 754 double precision.
MATLAB은 IEEE 754 배정밀도로 계산합니다.
Q determines how the HLS type handles fractional bits that are dropped when assigning to a narrower fixed-point type.
Q는 HLS 유형이 더 좁은 고정 소수점 유형에 할당할 때 삭제되는 소수 비트를 처리하는 방법을 결정합니다.

Inspect the MATLAB expression that produces each variable:
각 변수를 생성하는 MATLAB 표현식을 검사합니다.

| MATLAB operation<br>MATLAB 연산 | Q mode<br>Q 모드 | Reason<br>이유 |
|---|---|---|
| General arithmetic (`+`, `-`, `*`, `/N`)<br>일반 산술(`+`, `-`, `*`, `/N`) | `AP_RND_CONV`<br>`AP_RND_CONV` | IEEE 754 default is round-half-to-even — AP_RND_CONV matches exactly<br>IEEE 754 기본값은 반올림 - AP_RND_CONV가 정확히 일치함 |
| Power-of-2 divide then store (`/8`, `/16`)<br>2의 거듭제곱을 나눈 후 저장(`/8`, `/16`) | `AP_RND_CONV`<br>`AP_RND_CONV` | MATLAB rounds the double result before storing; plain `>> N` in C++ truncates (AP_TRN) — use AP_RND_CONV to recover 1 LSB accuracy<br>MATLAB은 저장하기 전에 double 결과를 반올림합니다. C++ 잘림의 일반 `>> N`(AP_TRN) - AP_RND_CONV를 사용하여 1 LSB 정확도 복구 |
| Explicit `floor(x)`<br>명시적 `floor(x)` | `AP_TRN`<br>`AP_TRN` | MATLAB truncates toward −∞; AP_TRN does the same<br>MATLAB은 -무한대 방향으로 자릅니다. AP_TRN도 마찬가지입니다. |
| Explicit `round(x)`<br>명시적 `round(x)` | `AP_RND_INF`<br>`AP_RND_INF` | MATLAB rounds half away from zero; AP_RND_INF matches<br>MATLAB은 0에서 반올림합니다. AP_RND_INF 일치 |
| Explicit `fix(x)` / `int(x)`<br>명시적 `fix(x)` / `int(x)` | `AP_TRN_ZERO`<br>`AP_TRN_ZERO` | MATLAB truncates toward zero; AP_TRN_ZERO matches<br>MATLAB은 0을 향해 자릅니다. AP_TRN_ZERO 일치 |
| Explicit `ceil(x)`<br>명시적 `ceil(x)` | implement as `floor(x) + 1` or negate-floor-negate in C++<br>`floor(x) + 1`로 구현하거나 C++에서 negate-floor-negate | No direct HLS mode<br>직접 HLS 모드 없음 |
| Pure integer arithmetic (no fractional bits drop)<br>순수 정수 산술(소수 비트 드롭 없음) | `AP_TRN` (default)<br>`AP_TRN`(기본값) | Q is irrelevant when F=0; leave at default<br>F=0일 때 Q는 관련이 없습니다. 기본값으로 두세요 |

**Default when unsure:** `AP_RND_CONV` — it is the IEEE 754-compatible choice and closest to MATLAB's double-precision arithmetic.
**확실하지 않은 경우 기본값:** `AP_RND_CONV` — IEEE 754 호환 선택이며 MATLAB의 배정밀도 연산에 가장 가깝습니다.
It costs one extra adder per assignment versus `AP_TRN`.
`AP_TRN`에 비해 할당당 하나의 추가 가산기가 필요합니다.

> **Note:** `AP_RND` (round toward +∞) is **not** the same as round-to-nearest.
> **참고:** `AP_RND`(+무한대 방향으로 반올림)는 가장 가까운 반올림과 **같지** 않습니다.
> It biases positive results upward.
> 이는 긍정적인 결과를 위쪽으로 편향시킵니다.
> Do not use it as a substitute for `AP_RND_CONV`.
> 'AP_RND_CONV' 대신 사용하지 마세요.

#### Full type examples
#### 풀타입 예시

```cpp
// xcorr: fractional, general MATLAB arithmetic → AP_RND_CONV, sized from range table
// peak is 4.97e-3, so I = floor(log2(4.97e-3)) + 1 = -8 + 1 = -7  → all 34 bits fractional
ap_ufixed<34, -7, AP_RND_CONV, AP_WRAP>  xcorr_val;

// accumulator: fractional intermediate — widen I to avoid overflow
ap_fixed<20, 4, AP_RND_CONV, AP_WRAP>    acc;

// result of explicit floor() in MATLAB
ap_fixed<16, 8, AP_TRN, AP_WRAP>         floored_val;

// result of explicit round() in MATLAB
ap_fixed<16, 8, AP_RND_INF, AP_WRAP>     rounded_val;

// index: unsigned integer — Q/O irrelevant
ap_uint<13>                               idx;

// final output clamp [0, 255] — done explicitly, not via AP_SAT
ap_uint<8> out_px;
if      (result < 0)   out_px = 0;
else if (result > 255) out_px = 255;
else                   out_px = (ap_uint<8>)result;
```

Update the range table to include the full type with Q and O:
Q 및 O가 포함된 전체 유형을 포함하도록 범위 테이블을 업데이트합니다.

| Variable<br>변수 | Min<br>최소 | Max<br>맥스 | All-integer?<br>모든 정수? | HLS type (full)<br>HLS 유형(전체) |
|---|---|---|---|---|
| `xcorr`<br>'xcorr' | 3.14e-10 | 4.97e-3 | no | `ap_ufixed<34,-7,AP_RND_CONV,AP_WRAP>`<br>`ap_ufixed<34,-7,AP_RND_CONV,AP_WRAP>` |
| `s_horiz`<br>`s_horiz` | 0 | 510 | yes<br>응 | `ap_uint<9>`<br>`ap_uint<9>` |
| `coeff_acc`<br>`coeff_acc` | -0.125 | 0.5 | no | `ap_fixed<4,1,AP_RND_CONV,AP_WRAP>`<br>`ap_고정<4,1,AP_RND_CONV,AP_WRAP>` |

### 4c.
### Report quantization error
### 양자화 오류 보고

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 0b.4c — Quantization error check
  Source : measured 0b.2b sweep (not predicted)
─────────────────────────────────────────────────────
```

For every fractional variable, report the grid the chosen type sits on:
모든 분수 변수에 대해 선택한 유형이 위치한 그리드를 보고합니다.

```
F            = W - I                          (fractional bits, = F_pick from 0b.2b)
LSB          = 2^(-F)
max_q_error  = 0.5 × LSB   (AP_RND_CONV)
             = 1.0 × LSB   (AP_TRN / AP_TRN_ZERO)
```

| Variable<br>변수 | HLS type<br>HLS 유형 | LSB<br>LSB | Max q-error<br>최대 q-오류 |
|---|---|---|---|
| `xcorr`<br>'xcorr' | `ap_ufixed<34,-7,AP_RND_CONV,AP_WRAP>`<br>`ap_ufixed<34,-7,AP_RND_CONV,AP_WRAP>` | 4.55e-13 | 2.27e-13 |
| `coeff_acc`<br>`coeff_acc` | `ap_fixed<4,1,AP_RND_CONV,AP_WRAP>`<br>`ap_고정<4,1,AP_RND_CONV,AP_WRAP>` | 0.125 | 0.0625 |

These are **descriptive, not a verdict.** They say how coarse each variable's grid is.
이는 **설명이 아닌 평결**입니다. 이는 각 변수의 그리드가 얼마나 거친지를 나타냅니다.
They
그들은
do not say whether the design is accurate — error propagates, so a coarse grid on a variable
설계가 정확한지 여부를 말하지 마십시오. 오류가 전파되므로 변수에 대한 거친 그리드가 표시됩니다.
whose error cancels downstream is fine, and a fine grid on one that feeds a subtraction is
오류가 다운스트림을 취소하는 것은 괜찮고 빼기를 제공하는 미세한 그리드는
not.
아닙니다.
The accuracy verdict is the measured 0b.2b sweep, and only that.
정확도 판정은 측정된 0b.2b 스윕입니다.

#### The gate
#### 문

Carry the sweep table forward verbatim.
스윕 테이블을 그대로 전달합니다.
This is what decides:
이것이 결정하는 것입니다:

| F | max_abs_err<br>최대_abs_err | rel_err<br>rel_err | mismatch<br>불일치 | SQNR (dB)<br>SQNR(dB) | Status<br>상태 |
|---|---|---|---|---|---|
| 6 | 1.9531e-02 | 4.5139% | 4081 | 26.9 | ✗ |
| 8 | 4.8828e-03 | 1.0723% | 4079 | 39.0 | ✗ |
| 10 | 1.2207e-03 | 0.2831% | 4064 | 51.0 | ✓ |
| 12 | 3.0518e-04 | 0.0654% | 3971 | 63.0 | ✓ |
| 14 | 7.6294e-05 | 0.0170% | 3585 | 75.1 | ✓ |

**Thresholds** — applied to the sweep's measured `rel_err`, error at the design output
**임계값** - 스윕의 측정된 `rel_err`에 적용되며 설계 출력 오류입니다.
relative to peak output:
피크 출력 대비:

| Measured rel_err<br>측정된 rel_err | Action<br>액션 |
|---|---|
| < 1% | ✓ acceptable<br>✓ 허용됨 |
| 1% – 10% | ⚠ present to the user with the adjacent rows so they can trade width for accuracy<br>⚠ 사용자가 정확성을 위해 너비를 교환할 수 있도록 인접한 행을 사용자에게 제공합니다. |
| > 10% | ✗ do not offer this row as a default<br>✗ 이 행을 기본값으로 제공하지 마세요. |

`F_pick` is the smallest ✓ row.
`F_pick`은 가장 작은 ✓ 행입니다.
Recommend `F_pick + 2` as the default and let the user choose
'F_pick + 2'를 기본값으로 권장하고 사용자가 선택하도록 합니다.
— see 0b.5.
— 0b.5를 참조하세요.

> **Do not compute `rel_err` from `LSB / |signal_min|`.** The obvious-looking metric
> **`LSB / |signal_min|`에서 `rel_err`을 계산하지 마세요.** 명백해 보이는 측정항목
> `max_q_error / |signal_min|` — error relative to the smallest observed sample — is wrong
> `max_q_error / |signal_min|` - 관찰된 가장 작은 샘플과 관련된 오류 - 잘못되었습니다.
> and was removed from this step.
> 이 단계에서 제거되었습니다.
> It grades a fixed-point type on *relative* precision, which
> *상대* 정밀도에 따라 고정 소수점 유형의 등급을 매깁니다.
> only floating-point provides:
> 부동 소수점만이 다음을 제공합니다:
>
> | | precision model | error at a small sample |
> | | 정밀 모델 | 작은 샘플의 오류 |
> |---|---|---|
> | `ap_fixed` | uniform **absolute** grid — `LSB` is the same everywhere | grows without bound as the sample shrinks |
> | `ap_fixed` | 균일한 **절대** 그리드 — `LSB`는 모든 곳에서 동일합니다 | 샘플이 줄어들면서 한계 없이 증가함 |
> | `float`/`double` | uniform **relative** — the exponent rescales | constant fraction of the sample |
> | `플로트`/`더블` | 균일 **상대적** — 지수 크기 조정 | 샘플의 일정한 부분 |
>
> A correctly-sized fixed-point type fails that test, and the failure says nothing about
> 올바른 크기의 고정 소수점 유형은 해당 테스트에 실패하며 실패는
> whether the design is accurate.
> 디자인이 정확한지.
> Concretely, the generated tracker records **every**
> 구체적으로 생성된 추적기는 **모든** 것을 기록합니다.
> assignment including initialization, so `acc = 0;` puts `min = 0` in the range table for
> 초기화를 포함한 할당이므로 `acc = 0;`은 범위 테이블에 `min = 0`을 넣습니다.
> essentially every accumulator in every design.
> 기본적으로 모든 설계의 모든 누산기.
> `max_q_error / 0 = Inf` → every accumulator
> `max_q_error / 0 = Inf` → 모든 누산기
> fails unconditionally, and any "add `ceil(log2(rel_err/0.01))` bits" remedy asks for
> 무조건 실패하며 "`ceil(log2(rel_err/0.01))` 비트 추가" 해결 방법이 필요합니다.
> infinitely many.
> 무한히 많습니다.
> The same happens for any bipolar signal crossing zero and any image with
> 0을 교차하는 양극성 신호와 다음과 같은 이미지에 대해서도 마찬가지입니다.
> a black pixel.
> 검은색 픽셀.
> Measure the output; do not predict from the range table.
> 출력을 측정합니다. 범위 테이블에서 예측하지 마십시오.

#### Width ceiling
#### 폭 천장

| Limit<br>한도 | Value<br>가치 | Nature<br>자연 |
|---|---|---|
| `ap_[u]int` / `ap_[u]fixed` `W`<br>`ap_[u]int` / `ap_[u]고정` `W` | **1024 bits** default<br>**1024비트** 기본값 | hard, but raisable to 4096 via `#define AP_INT_MAX_W` before including `ap_int.h`<br>어렵지만 `ap_int.h`를 포함하기 전에 `#define AP_INT_MAX_W`를 통해 4096으로 인상할 수 있습니다. |
| Native/DSP-friendly operators<br>네이티브/DSP 친화적인 운영자 | ~64 bits<br>~64비트 | **QoR advisory, not a limit** — beyond this, operators decompose into multiple DSPs and carry chains, costing area and Fmax<br>**QoR 자문, 제한 없음** — 이 이상으로 운영자는 여러 DSP 및 캐리 체인, 비용 영역 및 Fmax로 분해됩니다. |

Source: UG1399, *Overview of Arbitrary Precision Integer Data Types*.
출처: UG1399, *임의 정밀도 정수 데이터 유형 개요*.
If a design genuinely
디자인이 진심이라면
needs `W > 64`, that is legal and synthesizes — flag the QoR cost to the user, do not refuse
'W > 64'가 필요합니다. 즉, 합법적이고 합성됩니다. QoR 비용을 사용자에게 표시하고 거부하지 마세요.
the width.
너비.

---

### 4d.
### Fixed-point multiplication intermediates
### 고정 소수점 곱셈 중간체

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 0b.4d — Multiply intermediate sizing
─────────────────────────────────────────────────────
```

Fixed-point multiplication changes the value range: `a × b` produces a result with `W_a + W_b` total bits and `I_a + I_b` integer bits.
고정 소수점 곱셈은 값 범위를 변경합니다. `a × b`는 `W_a + W_b` 총 비트와 `I_a + I_b` 정수 비트로 결과를 생성합니다.
If the result is immediately assigned to a narrower type, fractional bits are silently dropped — MATLAB never loses these bits (it stays in double throughout).
결과가 더 좁은 유형에 즉시 할당되면 소수 비트가 자동으로 삭제됩니다. MATLAB은 이러한 비트를 잃지 않습니다(전체적으로 이중으로 유지됨).

**Rule: any intermediate that is the result of a multiply must be declared at the full product width before any narrowing.**
**규칙: 곱셈의 결과인 모든 중간체는 축소되기 전에 전체 제품 너비로 선언되어야 합니다.**

For every multiply in the MATLAB algorithm, identify the operands and declare the intermediate:
MATLAB 알고리즘의 모든 곱셈에 대해 피연산자를 식별하고 중간체를 선언합니다.

```cpp
// MATLAB: result = a * b
// a: ap_fixed<Wa, Ia, Qa, Oa>
// b: ap_fixed<Wb, Ib, Qb, Ob>

ap_fixed<Wa+Wb, Ia+Ib, AP_RND_CONV, AP_WRAP> prod = a * b;

// Only then narrow — after the full-precision product is captured:
ap_fixed<W_out, I_out, AP_RND_CONV, AP_WRAP> result = prod;
```

For the peakPicker case — comparing `xcorr[i] >= threshold[i]`:
peakPicker 사례의 경우 — `xcorr[i] >= 임계값[i]` 비교:
- This is a comparison, not a multiply — no intermediate needed
- 이것은 곱셈이 아닌 비교입니다. 중간이 필요하지 않습니다.
- If an accumulation or scale factor is added later (e.g.
- 나중에 누적 또는 축척 계수가 추가되는 경우(예:
- `xcorr * weight`), declare `prod` at full width first
- `xcorr * Weight`), 먼저 전체 너비로 `prod`를 선언합니다.

Include a multiply-intermediate column in the range table where applicable:
해당하는 경우 범위 테이블에 곱중중간 열을 포함합니다.

| Multiply<br>곱하기 | Operand types<br>피연산자 유형 | Intermediate type<br>중급형 | Output type<br>출력 유형 |
|---|---|---|---|
| `a * b` | `ap_fixed<16,4>` × `ap_fixed<16,4>`<br>`ap_고정<16,4>` × `ap_고정<16,4>` | `ap_fixed<32,8,AP_RND_CONV,AP_WRAP>`<br>`ap_고정<32,8,AP_RND_CONV,AP_WRAP>` | `ap_fixed<16,4,AP_RND_CONV,AP_WRAP>`<br>`ap_고정<16,4,AP_RND_CONV,AP_WRAP>` |

---

### 5.
### Present to user and validate
### 사용자에게 제시하고 유효성을 검사합니다.

Print the completed table.
완성된 테이블을 인쇄합니다.
`I` is derived from the measured range; `F` comes from the
'I'는 측정된 범위에서 파생됩니다. 'F'는 다음에서 유래합니다.
measured sweep.
측정된 스윕.
Show both sources so the user can see which number came from where:
사용자가 어떤 번호가 어디에서 왔는지 확인할 수 있도록 두 소스를 모두 표시합니다.

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 0b — Types + Measured Accuracy
  Variables instrumented : <N>
  F source : 0b.2b sweep @ F=<F_pick>   Tolerance : 1%

  Variable         Sim range              HLS type                              LSB        Max q-err
  ──────────────────────────────────────────────────────────────────────────────────────────────────
  xcorr            [3.14e-10, 4.97e-3]   ap_ufixed<34,-7,AP_RND_CONV,AP_WRAP>  4.55e-13   2.27e-13
  threshold        [1.53e-5,  4.40e-3]   ap_ufixed<34,-7,AP_RND_CONV,AP_WRAP>  4.55e-13   2.27e-13
  ...

  End-to-end accuracy vs golden (measured, 0b.2b):
   F    max_abs_err     rel_err   mismatch   SQNR(dB)
   6     1.9531e-02     4.5139%       4081       26.9
   8     4.8828e-03     1.0723%       4079       39.0
  10     1.2207e-03     0.2831%       4064       51.0   ← smallest meeting 1%
  12     3.0518e-04     0.0654%       3971       63.0
  14     7.6294e-05     0.0170%       3585       75.1

  Multiply intermediates:
  <a> × <b>  →  ap_fixed<Wa+Wb, Ia+Ib, AP_RND_CONV, AP_WRAP>   (full product width)

─────────────────────────────────────────────────────
```

Then ask the user to pick a row, using **AskUserQuestion**.
그런 다음 **AskUserQuestion**을 사용하여 사용자에게 행을 선택하도록 요청합니다.
Every option must be a real row
모든 옵션은 실제 행이어야 합니다.
of the sweep, labelled with its measured consequence — never an abstract accept/refine:
측정된 결과로 라벨이 붙은 스윕 - 결코 추상적인 승인/정제가 아닙니다.

```
Which precision do you want to build at?

  [F=12]  +2 bits margin — rel_err 0.065%, SQNR 63 dB     (recommended)
  [F=10]  minimum meeting 1% — rel_err 0.283%, SQNR 51 dB
  [F=14]  rel_err 0.017%, SQNR 75 dB — costs 2 more bits per variable
  [F=8]   below tolerance: rel_err 1.07% — only if width is critical
```

**Rules for building the options:**
**옵션 구성 규칙:**

1. **Recommend `F_pick + 2`**, not `F_pick`.
1. **`F_pick`이 아닌 `F_pick + 2`**를 권장합니다.
1. `F_pick` is the minimum that passed on *this*
1. `F_pick`은 *this*를 전달한 최소값입니다.
stimulus; two bits of margin costs little and absorbs a wider input distribution.
자극; 2비트의 마진은 비용이 거의 들지 않으며 더 넓은 입력 분포를 흡수합니다.
Same
동일
reasoning as the Margin rule for `I` in 0b.4.
0b.4의 'I'에 대한 여백 규칙으로 추론합니다.
2. **Offer `F_pick` itself** as the tight option.
2. **'F_pick' 자체를 제공** 타이트 옵션으로 제공합니다.
3. **Offer one row above** so the user can buy accuracy if they want it.
3. **위에 한 행을 제공**하면 사용자가 원하는 경우 정확성을 구입할 수 있습니다.
4. **Offer one row below tolerance** only when it is close (1–10%), labelled as such.
4. **공차보다 낮은 한 행을 제공** 허용 범위가 가까울 때만(1~10%) 라벨이 붙어 있습니다.
4. Never
4. 절대로
offer a `> 10%` row.
`>10%` 행을 제공합니다.
5. **If the golden is integer-valued** (uint8/uint16 image), state bit-exactness explicitly —
5. **골든이 정수 값인 경우**(uint8/uint16 이미지) 비트 정확성을 명시적으로 명시합니다.
`mismatch = 0` is a stronger and more meaningful claim than any percentage:
'불일치 = 0'은 어떤 비율보다 더 강력하고 의미 있는 주장입니다.
`[F=14] bit-exact vs golden — 0 of 262144 pixels differ`.
`[F=14] 비트 정확 vs 황금 — 262144픽셀 중 0이 다릅니다`.
6. **Never present a row the sweep did not actually run.** If the user wants a width not in
6. **스윕이 실제로 실행되지 않은 행을 표시하지 마십시오.** 사용자가 너비가 아닌 너비를 원하는 경우
`F_LIST`, extend `F_LIST` and re-run 0b.2b — do not interpolate.
`F_LIST`, `F_LIST`를 확장하고 0b.2b를 다시 실행합니다. 보간하지 마세요.

Wait for the user's answer, then set `F = <chosen>` and `W = I + F` for every fractional
사용자의 답변을 기다린 후 모든 분수에 대해 `F = <chosen>` 및 `W = I + F`를 설정합니다.
variable. **Refining is cheap now** — it is re-reading a row of a table that already exists,
변수. **정제 비용이 이제 저렴해졌습니다** — 이미 존재하는 테이블의 행을 다시 읽는 것입니다.
not a Step 0 re-run.
0단계 재실행이 아닙니다.
Only return to 0b.2b if the user asks for an `F` outside the swept list.
사용자가 스위프 목록 외부에서 'F'를 요청하는 경우에만 0b.2b로 돌아갑니다.

Once a row is chosen:
행이 선택되면 다음을 수행합니다.

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 0b done — Range Instrumentation
✓ Step 0   Run MATLAB simulation → save golden I/O
✓ Step 0b  Range instrumentation → sim-measured types
  Step 1   Analyze MATLAB algorithm                     ← NEXT
  Step 2   Generate refactor_1 (plain C++) + verify
  Step 3   Generate refactor_2 (frame-based C++)
  Step 4   Hand off to /hls-architect → /hls-optimize
─────────────────────────────────────────────────────
```

Save the accepted range table in memory — Step 2 references it for every intermediate type decision.
허용된 범위 테이블을 메모리에 저장합니다. 2단계에서는 모든 중간 유형 결정에 대해 이를 참조합니다.

---

## Step 1 / 4 — Analyze the MATLAB Code
## 1/4단계 - MATLAB 코드 분석

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 1 / 4 — Analyze MATLAB Code
─────────────────────────────────────────────────────
```

Read the MATLAB code and extract:
MATLAB 코드를 읽고 다음을 추출합니다.

1. **Algorithm intent** — what does it compute? (filter, transform, color space, etc.)
1. **알고리즘 의도** — 무엇을 계산하나요? (필터, 변환, 색상 공간 등)
2. **Inputs and outputs** — types, dimensions, value ranges
2. **입력 및 출력** — 유형, 차원, 값 범위
3. **Sample-based operations** — identify every operation that works on a single pixel/sample at a time vs. entire frame
3. **샘플 기반 작업** — 전체 프레임과 동시에 단일 픽셀/샘플에서 작동하는 모든 작업을 식별합니다.
4. **Neighborhood access** — does it use `imfilter`, `conv2`, sliding windows, or index expressions like `A(i-1:i+1, j-1:j+1)`?
4. **이웃 액세스** — `imfilter`, `conv2`, 슬라이딩 윈도우 또는 `A(i-1:i+1, j-1:j+1)`와 같은 인덱스 표현식을 사용합니까?
5. **Filter coefficients** — are they fixed at compile time or runtime parameters?
5. **필터 계수** - 컴파일 타임에 고정되나요, 아니면 런타임 매개변수에 고정되나요?
6. **Control flow** — loops, conditionals, switch/case based on pixel position (Bayer phase, color channel, etc.)
6. **제어 흐름** — 루프, 조건문, 픽셀 위치 기반 스위치/케이스(베이어 위상, 색상 채널 등)

---

Print before proceeding:
계속하기 전에 인쇄하세요:
```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 1 done — MATLAB Analysis
  Algorithm    : <what it computes>
  Input        : <type, dimensions>
  Output       : <type, dimensions>
  Operations   : <sample-based ops identified>
  Neighborhood : <sliding window / none>
  Coefficients : <fixed at compile time / runtime>
  Assumptions  : <any assumptions made>
  Any gaps     : <MATLAB constructs with no direct HLS equivalent — or "none">
─────────────────────────────────────────────────────
✓ Step 0   Run MATLAB simulation → save golden I/O
✓ Step 0b  Range instrumentation → sim-measured types
✓ Step 1   Analyze MATLAB algorithm
  Step 2   Generate refactor_1 (plain C++) + verify    ← NEXT
  Step 3   Generate refactor_2 (frame-based C++)
  Step 4   Hand off to /hls-architect → /hls-optimize
```

## Step 2 / 4 — Generate `refactor_1` (Plain C++) + Testbench, Verify Against MATLAB
## 2/4 단계 - `refactor_1`(일반 C++) + 테스트벤치 생성, MATLAB과 비교하여 확인

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 2 / 4 — Generate refactor_1 (Plain C++) + Verify
─────────────────────────────────────────────────────
```

Before writing HLS code, generate **`refactor_1`** — a plain C++ implementation (no HLS pragmas, no `ap_int`, no streams) — and a testbench that mirrors the MATLAB simulation exactly.
HLS 코드를 작성하기 전에 일반 C++ 구현(HLS pragma 없음, `ap_int` 없음, 스트림 없음)인 **`refactor_1`**과 MATLAB 시뮬레이션을 정확하게 반영하는 테스트 벤치를 생성하십시오.
Verify correctness first.
먼저 정확성을 확인하세요.

### `refactor_1` — Plain C++ File
### `refactor_1` — 일반 C++ 파일

- Translate the MATLAB algorithm to HLS C++ using types chosen from the **Step 0b simulation range table**.
- **0b단계 시뮬레이션 범위 표**에서 선택한 유형을 사용하여 MATLAB 알고리즘을 HLS C++로 변환합니다.
- For each intermediate variable, the simulation result drives the choice among exactly three candidates:
- 각 중간 변수에 대해 시뮬레이션 결과는 정확히 세 가지 후보 중에서 선택을 유도합니다.
  - **`ap_uint<N>`** — simulation shows min ≥ 0 and all values are integer
  - **`ap_uint<N>`** — 시뮬레이션에서는 min ≥ 0이고 모든 값은 정수입니다.
  - **`ap_int<N>`** — simulation shows min < 0 and all values are integer
  - **`ap_int<N>`** — 시뮬레이션에서는 min < 0이고 모든 값은 정수입니다.
  - **`ap_fixed<W,I>`** — simulation shows fractional values, or the MATLAB path uses `double`/`float` (which are forbidden in synthesis and must be replaced)
  - **`ap_fixed<W,I>`** — 시뮬레이션이 분수 값을 표시하거나 MATLAB 경로가 `double`/`float`(합성에서 금지되어 대체되어야 함)을 사용합니다.
  - For variables not captured by the range table (scalars, loop counters): apply the same three-way choice using static analysis of the MATLAB code
  - 범위 테이블(스칼라, 루프 카운터)에 의해 캡처되지 않은 변수의 경우: MATLAB 코드의 정적 분석을 사용하여 동일한 3방향 선택 적용

#### Generate Typedefs from Range Table
#### 범위 테이블에서 Typedef 생성

For **every variable** in the accepted Step 0b range table, generate a typedef:
허용되는 0b 단계 범위 테이블의 **모든 변수**에 대해 typedef를 생성합니다.

```cpp
// Generate one typedef per unique type from range table
typedef ap_ufixed<W1, I1, AP_RND_CONV, AP_WRAP> input_t;
typedef ap_fixed<W2, I2, AP_RND_CONV, AP_WRAP>  intermediate_t;
typedef ap_ufixed<W3, I3, AP_RND_CONV, AP_WRAP> output_t;
// ... one typedef for each variable in the range table
```

**Naming convention**:
**명명 규칙**:
- Use the MATLAB variable name as the suffix: `<var>_t`
- MATLAB 변수 이름을 접미사로 사용하십시오: `<var>_t`
- For shared ranges (variables with identical min/max), use one typedef with a descriptive name
- 공유 범위(최소/최대가 동일한 변수)의 경우 설명이 포함된 이름이 있는 하나의 typedef를 사용하세요.

#### Apply Types to All Variables
#### 모든 변수에 유형 적용

Use the generated typedefs for:
생성된 typedef를 다음 용도로 사용하세요.
1. **Output arrays**: Must match the derived output types from Step 0b
1. **출력 배열**: 0b단계에서 파생된 출력 유형과 일치해야 합니다.
2. **Intermediate variables**: Loop-internal calculations use the range-table types
2. **중간 변수**: 루프 내부 계산에서는 범위 테이블 유형을 사용합니다.
3. **Multiply products**: Declare at full product width before narrowing
3. **제품 곱하기**: 범위를 좁히기 전에 전체 제품 너비로 선언

**✅ CORRECT**: Use derived types from range table:
**✅ 정확함**: 범위 테이블에서 파생된 유형을 사용하세요.

```cpp
static intermediate_t buffer[FRAME_PIXELS];
static result_t       results[FRAME_PIXELS];

void my_kernel(input_t input_arr[FRAME_PIXELS],
               output_t output_arr[FRAME_PIXELS])  // ← use derived output_t, not float
{
    for (int i = 0; i < FRAME_PIXELS; i++) {
        intermediate_t temp = buffer[i] * buffer[i];  // ← use typed intermediate
        output_arr[i] = process(temp);
    }
}
```

**❌ WRONG**: Don't fall back to float for fixed-point variables:
**❌ 잘못됨**: 고정 소수점 변수에 대해 부동 소수점으로 대체하지 마세요.

```cpp
// ❌ Defeats the purpose of Step 0b range instrumentation
float output_arr[FRAME_PIXELS];
float intermediate = ...;
```

- Use the same logic, same coefficients, same normalization as the MATLAB code
- MATLAB 코드와 동일한 논리, 동일한 계수, 동일한 정규화를 사용합니다.
- No HLS pragma constructs yet — this is a functional translation, but with HLS-compatible types
- 아직 HLS pragma 구성이 없습니다. 이는 기능적 변환이지만 HLS 호환 유형이 있습니다.
- **Always use row-major scan order** (row outer, col inner) — even if the MATLAB code uses column-major.
- **항상 행 주요 스캔 순서를 사용합니다**(행 외부, 열 내부) — MATLAB 코드가 열 주요를 사용하는 경우에도 마찬가지입니다.
- Row-major is the HLS standard for image processing.
- Row-major는 이미지 처리를 위한 HLS 표준입니다.
- **Output dimensions must match input dimensions exactly** — read `ROWS` and `COLS` from the input image; do NOT add extra rows or columns for border padding.
- **출력 차원은 입력 차원과 정확히 일치해야 합니다** — 입력 이미지에서 `ROWS` 및 `COLS`를 읽습니다. 테두리 패딩을 위해 추가 행이나 열을 추가하지 마세요.
- If the algorithm has a warmup border (e.g., filter kernels), exclude those border pixels from the error comparison — do not change the output array size.
- 알고리즘에 준비 경계(예: 필터 커널)가 있는 경우 오류 비교에서 해당 경계 픽셀을 제외합니다. 출력 배열 크기를 변경하지 마십시오.

```cpp
// Row-major flat loop — always use this pattern
for (int k = 0; k < ROWS * COLS; k++) {
    int row = k / COLS;
    int col = k % COLS;
    ...
}
```

> **Pattern-string indices:** a MATLAB expression like `find(str == 'r') - 1` must be
> **패턴 문자열 인덱스:** `find(str == 'r') - 1`과 같은 MATLAB 표현식은 다음과 같아야 합니다.
> resolved outside the kernel — pre-encode it as an integer and pass it in as an
> 커널 외부에서 해결됨 - 정수로 사전 인코딩하고
> `ap_uint<2>` parameter.
> `ap_uint<2>` 매개변수.
> Do not search a character array inside the kernel; it
> 커널 내부에서 문자 배열을 검색하지 마십시오. 그것
> synthesises, but it is wasted logic and the wrong interface.
> 합성이지만 논리 낭비이고 인터페이스가 잘못되었습니다.

### Testbench
### 테스트벤치

The testbench must mirror the MATLAB `*_runme.m` structure — same input data, same comparison against the golden output.
테스트벤치는 MATLAB `*_runme.m` 구조를 미러링해야 합니다. 즉, 동일한 입력 데이터, 골든 출력과 동일한 비교가 필요합니다.

**Shape mismatch warning:** MATLAB stores arrays in **column-major** order; C++ iterates in **row-major** order.
**모양 불일치 경고:** MATLAB은 **열 중심** 순서로 배열을 저장합니다. C++는 **행 우선** 순서로 반복합니다.
A direct flat index comparison will fail even on a correct implementation.
올바른 구현에서도 직접 플랫 인덱스 비교는 실패합니다.
The comparison must map between the two orderings:
비교는 두 순서 간에 매핑되어야 합니다.

```
MATLAB flat index (column-major): j = col * ROWS + row
C++    flat index (row-major):    i = row * COLS + col
```

```cpp
int main() {
    // 1. Load input — MATLAB saves in column-major order.
    //    Reorder to row-major for C++ kernel.
    load_bin("matlab_input.bin", matlab_in, FRAME_PIXELS);  // column-major
    for (int row = 0; row < ROWS; row++)
        for (int col = 0; col < COLS; col++)
            in_buf[row * COLS + col] = matlab_in[col * ROWS + row];

    // 2. Load MATLAB golden output (column-major)
    load_bin("matlab_golden.bin", golden, OUT_PIXELS);

    // 3. Run the C++ function TWICE. This testbench is carried forward
    //    unchanged to /hls-architect, where cosim needs back-to-back calls
    //    to report an Initiation Interval. g++ measures no II here.
    int total_mismatches = 0;
    for (int call = 1; call <= 2; call++) {
        my_kernel(in_buf, out_buf);

        // 4. Compare: map C++ row-major index to MATLAB column-major index.
        //    Compare in double. NEVER cast the kernel output to int — that
        //    truncates ap_fixed fractional bits and hides sub-LSB error,
        //    which is exactly the error the 0b.2b sweep was sizing.
        int mismatches = 0;
        for (int row = 0; row < ROWS; row++) {
            for (int col = 0; col < COLS; col++) {
                int cpp_idx    = row * COLS + col;   // C++ row-major
                int matlab_idx = col * ROWS + row;   // MATLAB column-major
                double got = out_buf[cpp_idx].to_double();   // ap_uint/ap_int/ap_fixed all define to_double()
                double expect = (double)golden[matlab_idx];
                if (fabs(got - expect) > TOLERANCE) {
                    printf("Call %d MISMATCH (%d,%d): got %.6f expected %.6f\n",
                           call, row, col, got, expect);
                    mismatches++;
                }
            }
        }
        printf("Call %d: %s\n", call, mismatches == 0 ? "PASS" : "FAIL");
        total_mismatches += mismatches;
    }

    // 5. Collapse to 0/1. The shell masks exit codes to 8 bits, so returning
    //    the raw count makes any multiple of 256 mismatches look like success.
    printf("TOTAL mismatches: %d\n", total_mismatches);
    return total_mismatches ? 1 : 0;
}
```

> **Note:** This testbench carries forward unchanged into the HLS design.
> **참고:** 이 테스트 벤치는 HLS 설계에 변경 없이 적용됩니다.
> `/hls-architect` uses the same testbench — only the kernel implementation changes, not the stimulus or comparison logic.
> `/hls-architect`는 동일한 테스트벤치를 사용합니다. 자극이나 비교 논리가 아닌 커널 구현만 변경됩니다.

### Image I/O Detection — PNG/Image Read/Write
### 이미지 I/O 감지 - PNG/이미지 읽기/쓰기

**MANDATORY CHECK during Step 1:** Scan the MATLAB code for image I/O:
**1단계 중 필수 확인:** 이미지 I/O에 대한 MATLAB 코드 스캔:

```matlab
% Detection patterns:
imread()
imwrite()
imshow()
```

**Detection result determines testbench I/O:**
**탐지 결과에 따라 테스트벤치 I/O가 결정됩니다.**

| MATLAB has<br>MATLAB은 | C++ testbench must<br>C++ 테스트벤치는 다음과 같습니다. |
|---|---|
| `imread()` + `imwrite()`<br>`imread()` + `imwrite()` | Read PNG input + write PNG output (OpenCV)<br>PNG 입력 읽기 + PNG 출력 쓰기(OpenCV) |
| `imread()` only<br>`imread()`만 | Read PNG input + save `.bin` golden + write PNG output<br>PNG 입력 읽기 + `.bin` 골든 저장 + PNG 출력 쓰기 |
| Neither<br>둘 다 | Binary `.bin` I/O only<br>바이너리 `.bin` I/O 전용 |

**Rule: If MATLAB writes visual output (imwrite/imshow), C++ must write PNG for side-by-side comparison.**
**규칙: MATLAB이 시각적 출력(imwrite/imshow)을 작성하는 경우 C++에서는 병렬 비교를 위해 PNG를 작성해야 합니다.**

---

### PNG I/O Implementation (when detected)
### PNG I/O 구현(감지된 경우)

When image I/O is detected, the testbench must do **BOTH**:
이미지 I/O가 감지되면 테스트벤치는 **두 가지**를 모두 수행해야 합니다.
1. Binary `.bin` comparison (numerical verification)
1. 바이너리 `.bin` 비교(수치 검증)
2. PNG write (visual comparison)
2. PNG 쓰기(시각적 비교)

Start from the testbench above and add the four pieces below.
위의 테스트벤치에서 시작하여 아래의 4개 부분을 추가하세요.
The load, the two-call
부하, 두 가지 호출
loop, the comparison and the return are **unchanged** — do not fork a second copy of them.
루프, 비교 및 반환은 **변경되지 않습니다** — 두 번째 복사본을 포크하지 마세요.

```cpp
#include <opencv2/opencv.hpp>

// (a) Before the loads: resolve golden paths up front and build absolute paths.
//     Downstream tools run the testbench from their own build directory, so a
//     relative "../golden" only works for the local g++ run.
const char *GOLDEN_DIR = getenv("GOLDEN_PATH");
if (!GOLDEN_DIR) GOLDEN_DIR = "../golden";
char input_bin[512], golden_bin[512], input_png_path[512];
snprintf(input_bin,      sizeof(input_bin),      "%s/matlab_input.bin",  GOLDEN_DIR);
snprintf(golden_bin,     sizeof(golden_bin),     "%s/matlab_golden.bin", GOLDEN_DIR);
snprintf(input_png_path, sizeof(input_png_path), "%s/<input>.png",       GOLDEN_DIR);
// then load_bin(input_bin, ...) and load_bin(golden_bin, ...) instead of bare filenames

// (b) Optional: load the input PNG so you can eyeball it next to the output.
cv::Mat input_png = cv::imread(input_png_path, cv::IMREAD_GRAYSCALE);
if (input_png.empty())
    printf("Warning: input PNG not found — proceeding with binary-only input\n");

// (c) Inside the two-call loop, after the comparison: write the C++ output as PNG.
cv::Mat out_png(ROWS, COLS, CV_8UC1);
for (int row = 0; row < ROWS; row++)
    for (int col = 0; col < COLS; col++)
        out_png.at<uint8_t>(row, col) = (uint8_t)out_buf[row * COLS + col];
char filename[256];
snprintf(filename, sizeof(filename), "cpp_output_call%d.png", call);
cv::imwrite(filename, out_png);
printf("           (visual): %s written\n", filename);

// (d) After the loop, before the return: name the three files to compare.
printf("Visual comparison — open side by side:\n");
printf("  Input      : %s\n", input_png_path);
printf("  MATLAB gold: %s/<golden>.png\n", GOLDEN_DIR);
printf("  C++ output : cpp_output_call2.png (written to the current directory)\n");
```

**Hybrid testbench rules:**
**하이브리드 테스트벤치 규칙:**
- **Binary verification:** the `.bin` comparison catches precision errors
- **바이너리 확인:** `.bin` 비교는 정밀도 오류를 포착합니다.
- **Visual output:** the PNG catches algorithm errors (shifts, channel swaps, wrong Bayer phase)
- **시각적 출력:** PNG가 알고리즘 오류(시프트, 채널 스왑, 잘못된 Bayer 단계)를 포착합니다.
- **Both must pass:** binary PASS *and* visual inspection
- **둘 다 통과해야 합니다:** 바이너리 PASS *및* 육안 검사
- Use `cv::IMREAD_GRAYSCALE` or `cv::IMREAD_COLOR` to match MATLAB
- MATLAB과 일치시키려면 `cv::IMREAD_GRAYSCALE` 또는 `cv::IMREAD_COLOR`를 사용하십시오.
- For RGB output, use `CV_8UC3` and interleave channels correctly
- RGB 출력의 경우 'CV_8UC3'을 사용하고 채널을 올바르게 인터리브하세요.
- Write PNG for **both calls** (call1 and call2) to verify consistency
- 일관성을 확인하기 위해 **두 호출**(call1 및 call2)에 대해 PNG를 작성합니다.
- **Resolve every input path absolutely at generation time** — the testbench is reused by downstream tools that run it from a different working directory
- **생성 시 모든 입력 경로를 절대적으로 해결** — 테스트벤치는 다른 작업 디렉터리에서 실행하는 다운스트림 도구에 의해 재사용됩니다.

### Verify
### 확인

```bash
# ap_fixed/ap_int/ap_uint headers ship with Vitis at $XILINX_VITIS/include
g++ -std=c++14 -I$XILINX_VITIS/include -o verify testbench.cpp kernel.cpp && ./verify
```

Must print `PASS` before proceeding.
계속하기 전에 'PASS'를 인쇄해야 합니다.

### Write to sample_based/
### Sample_based/에 쓰기

Call `./reference/design-layout.md  design_name=<name>  stage=sample_based`
`./reference/design-layout.md design_name=<name> stage=sample_based`를 호출하세요.

Write into `design_name/sample_based/`:
`design_name/sample_based/`에 작성합니다.
- `kernel.cpp`    — the refactor_1 plain C++ kernel
- `kernel.cpp` — refactor_1 일반 C++ 커널
- `testbench.cpp` — the testbench (loads `.bin` files from `../golden/`)
- `testbench.cpp` — 테스트벤치(`../golden/`에서 `.bin` 파일 로드)

If it fails:
실패하는 경우:

> **First, rule out width.** The 0b.2b sweep already ran this algorithm in MATLAB at the
> **먼저 너비를 배제합니다.** 0b.2b 스윕은 이미 MATLAB에서 이 알고리즘을 실행했습니다.
> chosen `F` and measured the result against the same golden.
> 'F'를 선택하고 동일한 황금색에 대해 결과를 측정했습니다.
> If that row passed, the widths
> 해당 행이 통과된 경우 너비는
> are sufficient — **the failure is not a precision problem, and widening types will not fix
> 충분합니다 — **실패는 정밀도 문제가 아니며 확장 유형은 수정되지 않습니다.
> it.** Look for a behavioural difference instead.
> it.** 대신 행동의 차이를 찾아보세요.
> Widening in response to a csim mismatch is
> csim 불일치에 대한 응답으로 확대하는 것은
> the most common wasted debug cycle in this flow: it changes the numbers, sometimes moves
> 이 흐름에서 가장 일반적으로 낭비되는 디버그 주기: 숫자를 변경하고 때로는 이동합니다.
> the mismatch, and never addresses the cause.
> 일치하지 않으며 원인을 해결하지 않습니다.
>
> Confirm the direction before spending time: if the C++ error is far larger than the sweep's
> 시간을 보내기 전에 방향을 확인하십시오. C++ 오류가 스윕 오류보다 훨씬 큰 경우
> `max_abs_err` for that `F`, it is structural.
> 'F'에 대한 'max_abs_err'은 구조적입니다.
> If it is the same order, re-check `Q` mode
> 같은 순서라면 `Q` 모드를 다시 확인하세요
> (`AP_TRN` vs `AP_RND_CONV`) and multiply-intermediate widths from 0b.4d.
> (`AP_TRN` 대 `AP_RND_CONV`) 및 0b.4d의 중간 너비를 곱합니다.

- Check coefficient scaling (`/8` → `>> 3`, `/16` → `>> 4`)
- 계수 스케일링 확인(`/8` → `>> 3`, `/16` → `>> 4`)
- Check rounding vs truncation differences
- 반올림과 잘림 차이 확인
- Exclude warmup border pixels from comparison for filter kernels (KSIZE/2 rows + cols)
- 필터 커널 비교에서 준비 경계 픽셀 제외(KSIZE/2행 + 열)
- Check column-major (MATLAB) vs row-major (C++) index mapping — a transposed comparison
- 주요 열(MATLAB) 대 행 주요(C++) 인덱스 매핑 확인 - 전치된 비교
fails everywhere except on a square, symmetric image
정사각형 대칭 이미지를 제외한 모든 곳에서 실패합니다.
- Check that a multiply intermediate was declared at full product width before narrowing (0b.4d)
- 축소하기 전에(0b.4d) 전체 제품 너비에서 곱셈 중간이 선언되었는지 확인하세요.

---

Print before proceeding:
계속하기 전에 인쇄하세요:
```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 2 done — refactor_1
  Data types   : <variable: sim range [min, max] → ap_uint<N> / ap_int<N> / ap_fixed<W,I> — list each>
  Border/warmup: <warmup = N rows/cols skipped in comparison / N/A>
  Index mapping: <row-major ↔ column-major note if applicable / N/A>
  Verify result: PASS / FAIL (<mismatch count if fail>)
─────────────────────────────────────────────────────
✓ Step 0   Run MATLAB simulation → save golden I/O
✓ Step 0b  Range instrumentation → sim-measured types
✓ Step 1   Analyze MATLAB algorithm
✓ Step 2   Generate refactor_1 (plain C++) + verify
  Step 3   Generate refactor_2 (frame-based C++)        ← NEXT
  Step 4   Hand off to /hls-architect → /hls-optimize
```

## Step 3 / 4 — Generate `refactor_2`: Sample-Based to Frame-Based Conversion
## 3/4단계 - `refactor_2` 생성: 샘플 기반에서 프레임 기반으로의 변환

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 3 / 4 — Generate refactor_2 (Frame-Based C++)
─────────────────────────────────────────────────────
```

Starting from `refactor_1`, convert to **`refactor_2`** — a frame-based C++ file that iterates pixel-by-pixel in raster scan order.
`refactor_1`부터 시작하여 래스터 스캔 순서에 따라 픽셀 단위로 반복하는 프레임 기반 C++ 파일인 **`refactor_2`**로 변환합니다.
This is the file handed to `/hls-architect` in Step 4.
4단계에서 `/hls-architect`에 전달된 파일입니다.

MATLAB code is inherently **sample-based**: it operates on entire arrays/matrices at once using vectorized operations.
MATLAB 코드는 본질적으로 **샘플 기반**입니다. 즉, 벡터화된 연산을 사용하여 전체 배열/행렬에서 동시에 작동합니다.
HLS requires **frame-based** code: iterate over pixels one at a time in raster scan order.
HLS에는 **프레임 기반** 코드가 필요합니다. 래스터 스캔 순서에 따라 한 번에 하나씩 픽셀을 반복합니다.

> **Image size rule:** The frame-based kernel must produce exactly `ROWS × COLS` output pixels — the same dimensions as the input.
> **이미지 크기 규칙:** 프레임 기반 커널은 정확히 'ROWS × COLS' 출력 픽셀(입력과 동일한 크기)을 생성해야 합니다.
> Do NOT increase the output array size to accommodate border warmup.
> 테두리 준비를 수용하기 위해 출력 배열 크기를 늘리지 마십시오.
> Border pixels (e.g., first `KSIZE/2` rows and columns for a filter) are invalid and must be skipped in the testbench comparison, not padded into the output.
> 테두리 픽셀(예: 필터의 첫 번째 `KSIZE/2` 행과 열)은 유효하지 않으며 테스트벤치 비교에서 건너뛰어야 하며 출력에 채워지지 않아야 합니다.

> **Type consistency rule:** refactor_2 uses **the exact same types** as refactor_1.
> **유형 일관성 규칙:** refactor_2는 refactor_1과 **완전히 동일한 유형**을 사용합니다.
> The only difference is the loop structure (flat pixel iteration instead of nested row/col loops). **Do NOT change types** when converting sample-based to frame-based.
> 유일한 차이점은 루프 구조입니다(중첩된 행/열 루프 대신 플랫 픽셀 반복). **샘플 기반을 프레임 기반으로 변환할 때 유형을 변경하지 마세요**.
> If refactor_1 uses `output_t` for output arrays, refactor_2 must also use `output_t`.
> refactor_1이 출력 배열에 'output_t'를 사용하는 경우 refactor_2도 'output_t'를 사용해야 합니다.
> Copy the typedef block verbatim from refactor_1 to refactor_2.
> refactor_1에서 refactor_2로 typedef 블록을 그대로 복사합니다.

### How to Convert
### 변환 방법

Step 3 restructures array-level MATLAB operations into pixel-level C++ loops.
3단계에서는 배열 수준 MATLAB 작업을 픽셀 수준 C++ 루프로 재구성합니다.

**Do this:**
**이렇게 하세요:**
- Access a neighbourhood with clamped 2D indices: `buf[clamp(row±1) * COLS + clamp(col±1)]`
- 고정된 2D 인덱스가 있는 이웃에 액세스: `buf[clamp(row±1) * COLS + 클램프(col±1)]`
- Keep a full-frame intermediate buffer (`buf[ROWS*COLS]`) between stages
- 단계 사이에 전체 프레임 중간 버퍼(`buf[ROWS*COLS]`)를 유지합니다.
- The structure should be: sequential loops over a shared buffer, NOT a single-pass pipeline
- 구조는 다음과 같아야 합니다. 단일 패스 파이프라인이 아닌 공유 버퍼에 대한 순차 루프

Worked example — MATLAB `ndgrid` + `bitget` parity masks (Bayer phase, colour channel
실제 사례 — MATLAB `ndgrid` + `bitget` 패리티 마스크(바이엘 위상, 색상 채널
selection) become row/col counters derived from the flat loop index:
선택)은 플랫 루프 인덱스에서 파생된 행/열 카운터가 됩니다.

```matlab
% MATLAB
[y, x] = ndgrid(0:ROWS-1, 0:COLS-1);
mask00 = and(not(bitget(y,1)), not(bitget(x,1)));
```
```cpp
// C++ — track row/col from the loop counter. bitget(v,1) is the LSB (1-indexed).
int  row    = k / COLS;
int  col    = k % COLS;
bool lsb_y  = (row & 1);
bool lsb_x  = (col & 1);
bool mask00 = !lsb_y && !lsb_x;
```

**Not this (causes shift bugs):**
**이것은 아닙니다(교대 버그 발생):**
- Attempting to fuse consecutive stages into a single pass
- 연속된 단계를 단일 패스로 융합하려는 시도
- `#pragma HLS` of any kind — pragmas belong to /hls-optimize, not here
- 모든 종류의 `#pragma HLS` — pragma는 여기가 아닌 /hls-optimize에 속합니다.
- Re-associating filter arithmetic (pre-summing symmetric taps, folding coefficients
- 필터 연산 재연관(대칭 탭 미리 합산, 계수 접기)
into shifts) — it changes intermediate widths and breaks bit-exactness against the golden
교대로) - 중간 너비를 변경하고 황금색에 대한 비트 정확성을 깨뜨립니다.

---

### Write to frame_based/
### Frame_based/에 쓰기

Call `./reference/design-layout.md  design_name=<name>  stage=frame_based`
`./reference/design-layout.md design_name=<name> stage=frame_based`를 호출하세요.

Write into `design_name/frame_based/`:
`design_name/frame_based/`에 작성합니다.
- `kernel.cpp`    — the refactor_2 frame-based C++ kernel
- `kernel.cpp` — refactor_2 프레임 기반 C++ 커널
- `testbench.cpp` — same testbench as `sample_based/` (unchanged — same inputs, same golden comparison)
- `testbench.cpp` — `sample_based/`와 동일한 테스트벤치(변경되지 않음 — 동일한 입력, 동일한 황금 비교)

### Verify refactor_2 against MATLAB golden
### MATLAB Golden에 대해 refactor_2 확인

Compile and run immediately after writing the files — same testbench, same golden `.bin` files:
파일을 작성한 후 즉시 컴파일하고 실행합니다. 동일한 테스트벤치, 동일한 골든 `.bin` 파일입니다.

```bash
cd design_name/frame_based/
# ap_fixed/ap_int/ap_uint headers ship with Vitis at $XILINX_VITIS/include
g++ -std=c++14 -I$XILINX_VITIS/include -o verify_frame kernel.cpp testbench.cpp && ./verify_frame
```

For OpenCV designs, add the include and link flags:
OpenCV 디자인의 경우 포함 및 링크 플래그를 추가합니다.

```bash
g++ -std=c++14 \
    -I$XILINX_VITIS/include \
    -I${OPENCV_INCLUDE} \
    -o verify_frame kernel.cpp testbench.cpp \
    -L${OPENCV_LIB} -lopencv_core -lopencv_imgcodecs -lopencv_imgproc \
    -Wl,-rpath,${OPENCV_LIB} \
&& ./verify_frame
```

Must print `PASS` before proceeding to Step 4.
4단계로 진행하기 전에 'PASS'를 인쇄해야 합니다.
If it fails:
실패하는 경우:
- A raster-order bug (scan direction, row/col swap) shows up here but not in refactor_1 — fix the loop structure
- 래스터 순서 버그(스캔 방향, 행/열 교환)가 여기에 표시되지만 refactor_1에는 표시되지 않습니다. 루프 구조 수정
- A warmup border difference (filter edge pixels) — confirm the testbench skips the same border as in Step 2
- 준비 경계 차이(필터 가장자리 픽셀) - 테스트 벤치가 2단계에서와 동일한 경계를 건너뛰는지 확인합니다.
- A type narrowing issue introduced during frame-based rewrite — check `ap_int`/`ap_fixed` intermediate widths match refactor_1
- 프레임 기반 재작성 중에 유형 축소 문제가 발생했습니다. `ap_int`/`ap_fixed` 중간 너비가 일치하는지 확인하세요 refactor_1

---

Print before proceeding:
계속하기 전에 인쇄하세요:
```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 3 done — refactor_2 (frame-based)
  Key changes  : <vectorised→loop, 2D→1D scan, imfilter→explicit neighbourhood loop, etc.>
  HLS types    : <ap_uint/ap_int/ap_fixed changes from refactor_1>
  Verify       : PASS / FAIL (<mismatch count if fail>)
  Handing off  : frame_based/kernel.cpp → /hls-architect
─────────────────────────────────────────────────────
✓ Step 0   Run MATLAB simulation → save golden I/O
✓ Step 0b  Range instrumentation → sim-measured types
✓ Step 1   Analyze MATLAB algorithm
✓ Step 2   Generate refactor_1 (plain C++) + verify
✓ Step 3   Generate refactor_2 (frame-based C++)
  Step 4   Hand off to /hls-architect → /hls-optimize           ← NEXT
```

## Step 4 / 4 — Call `/hls-architect`
## 4/4단계 — `/hls-architect` 호출

```
─────────────────────────────────────────────────────
[matlab-to-cpp]  Step 4 / 4 — Handing off to /hls-architect
─────────────────────────────────────────────────────
```

Once the plain C++ passes verification (Step 2), hand off to `/hls-architect`.
일반 C++가 확인을 통과하면(2단계) `/hls-architect`에 전달합니다.

Always pass `THROUGHPUT_TARGET`, `XPART`, and `CLOCK_NS` collected in the Preamble:
항상 프리앰블에 수집된 `THROUGHPUT_TARGET`, `XPART` 및 `CLOCK_NS`를 전달합니다.

```
/hls-architect <THROUGHPUT_TARGET>  part=<XPART>  clock=<CLOCK_NS>
```

If `THROUGHPUT_TARGET` is empty, pass an empty string — `/hls-architect` will derive the best achievable target from II=1 floor and pass it to `/hls-optimize` automatically:
`THROUGHPUT_TARGET`이 비어 있으면 빈 문자열을 전달합니다. `/hls-architect`는 II=1 층에서 달성 가능한 최상의 목표를 파생하여 `/hls-optimize`에 자동으로 전달합니다.

```
/hls-architect  part=<XPART>  clock=<CLOCK_NS>
```

Provide as context:
컨텍스트로 제공:
- `design_name` — the workspace root (architect uses this to call `./reference/design-layout.md`)
- `design_name` — 작업 공간 루트(건축가는 이를 사용하여 `./reference/design-layout.md`를 호출합니다)
- `design_name/frame_based/kernel.cpp` — the frame-based C++ kernel (input to architect)
- `design_name/frame_based/kernel.cpp` — 프레임 기반 C++ 커널(설계자에게 입력)
- `design_name/frame_based/testbench.cpp` — the testbench (carried forward unchanged)
- `design_name/frame_based/testbench.cpp` — 테스트벤치(변경되지 않은 상태로 전달됨)
- The identified stages and their responsibilities (from Step 3)
- 식별된 단계 및 해당 책임(3단계)
- Input/output types, frame dimensions, filter coefficients
- 입력/출력 유형, 프레임 크기, 필터 계수

`/hls-architect` will:
`/hls-architect`는 다음을 수행합니다.
- Convert the plain C++ into multi-stage HLS dataflow C++
- 일반 C++를 다단계 HLS 데이터 흐름 C++로 변환
- Reuse the **same testbench** from Step 2 (same inputs, same MATLAB golden comparison)
- 2단계의 **동일한 테스트벤치** 재사용(동일한 입력, 동일한 MATLAB 최종 비교)
- Run the full hls* validation battery
- 전체 hls* 검증 배터리 실행
- Run csim — a PASS confirms the HLS C++ matches the MATLAB golden reference
- csim 실행 — PASS는 HLS C++가 MATLAB 골든 참조와 일치하는지 확인합니다.
- Hand off to `/hls-optimize <throughput target>` (passing the target through directly)
- `/hls-optimize <처리량 목표>`에 전달(대상을 직접 전달)
