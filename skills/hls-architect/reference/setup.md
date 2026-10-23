<!-- ko-translation: enabled -->
---
description: Set up the HLS design environment — verify Vitis, OpenCV, discover design config, detect build system. Run once at the start of any hls-architect, csim, csynth, cosim, or optimize session. For MATLAB setup, use matlab-to-cpp/reference/setup.md instead.
argument-hint: "[mode=tooling-only]   # tooling-only skips design discovery (use when no design exists yet)"
---
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Skill: setup
# 스킬 : 설정

Prepare the environment for an HLS design session.
HLS 설계 세션을 위한 환경을 준비합니다.
Run this:
이것을 실행하세요:
- **From `/hls-architect`, `/hls-run-flow`, `/hls-optimize`** with no args (full setup including design discovery)
- **인수 없이 `/hls-architect`, `/hls-run-flow`, `/hls-optimize`**에서(디자인 검색을 포함한 전체 설정)
- **From other contexts** with `mode=tooling-only` (only Vitis / OpenCV checks, skip design discovery)
- **다른 상황에서** `mode=tooling-only` 사용(Vitis/OpenCV 검사만, 디자인 검색 건너뛰기)

**Note:** This setup does NOT include MATLAB.
**참고:** 이 설정에는 MATLAB이 포함되지 않습니다.
If you need MATLAB (for `/hls-matlab-to-cpp`), use `../hls-matlab-to-cpp/reference/setup.md` instead — it delegates to this file and adds MATLAB on top.
MATLAB(`/hls-matlab-to-cpp`용)이 필요한 경우 대신 `../hls-matlab-to-cpp/reference/setup.md`를 사용하십시오. 그러면 이 파일에 위임되고 MATLAB이 맨 위에 추가됩니다.

If `$ARGUMENTS` contains `mode=tooling-only`, **skip Step 2 (design discovery) and Step 3 (build commands)**.
`$ARGUMENTS`에 `mode=tooling-only`가 포함된 경우 **2단계(설계 검색)와 3단계(빌드 명령)**를 건너뛰세요**.
Only Steps 1 and 4 run.
1단계와 4단계만 실행됩니다.

Print at the start:
시작 시 인쇄:
```
─────────────────────────────────────────────────────
[hls-architect/setup]  required tooling
  Step 1  — Verify Vitis HLS         (sources Vitis, exports $XILINX_VITIS)
  Step 4  — Verify OpenCV            (only if testbench uses cv::imread)
[hls-architect/setup]  design configuration  (skipped in tooling-only mode)
  Step 2  — Discover the design
  Step 3  — Build commands
─────────────────────────────────────────────────────
```

---

## Step 1: Verify Vitis environment
## 1단계: Vitis 환경 확인

```bash
which vitis-run
```

If not found, check if the environment variable XILINX_VITIS is set correctly and `vitis-run` is on PATH.
찾을 수 없는 경우 환경 변수 XILINX_VITIS가 올바르게 설정되어 있고 'vitis-run'이 PATH에 있는지 확인하세요.
If not, ask the user to set XILINX_VITIS to their Vitis installation directory and ensure `vitis-run` is accessible.
그렇지 않은 경우 사용자에게 XILINX_VITIS를 Vitis 설치 디렉터리로 설정하고 'vitis-run'에 액세스할 수 있는지 확인하도록 요청하세요.

Do not proceed until `vitis-run` is on PATH.
'vitis-run'이 PATH에 있을 때까지 진행하지 마세요.

Print after Step 1:
1단계 이후 인쇄:
```
─────────────────────────────────────────────────────
[hls-architect/setup]  Step 1 — done
  ✓ Step 1  Vitis <version> on PATH  (<path to vitis-run>)
  ← NEXT    Step 2 — Discover the design
─────────────────────────────────────────────────────
```

---

## Step 2: Discover the design
## 2단계: 디자인 발견

Ask the user for the design directory path.
사용자에게 디자인 디렉터리 경로를 물어보세요.
Once provided, find the HLS config file (name varies by design):
제공되면 HLS 구성 파일을 찾습니다(이름은 디자인에 따라 다름).

```bash
grep -rl "^\[hls\]" *.cfg 2>/dev/null | head -1
```

Store the result as `<config_file>`.
결과를 `<config_file>`로 저장합니다.
Then read the key settings from it:
그런 다음 주요 설정을 읽으십시오.

```bash
grep -E "^(syn\.top|clock|part|syn\.file)" <config_file>
```

Extract and hold for the rest of the session:
나머지 세션 동안 추출하여 보관합니다.
- `CONFIG_FILE` — the detected config filename (e.g.
- `CONFIG_FILE` — 감지된 구성 파일 이름(예:
- `PerfPragma.cfg`, `hls_config.cfg`)
- `PerfPragma.cfg`, `hls_config.cfg`)
- `TOP_FUNCTION` — top-level HLS function name, if missing use default 'top-function'
- `TOP_FUNCTION` — 최상위 HLS 함수 이름, 누락된 경우 기본값 'top-function' 사용
- `CLOCK_NS` — clock period in ns
- `CLOCK_NS` — ns 단위의 클럭 주기
- `XPART` — FPGA part
- `XPART` — FPGA 부분
- `SRC_FILES` — all `syn.file=` entries
- `SRC_FILES` — 모든 `syn.file=` 항목

Print after Step 2:
2단계 이후 인쇄:
```
─────────────────────────────────────────────────────
[hls-architect/setup]  Step 2 — done
  ✓ Step 1  Vitis on PATH
  ✓ Step 2  top=<TOP_FUNCTION>  cfg=<CONFIG_FILE>  part=<XPART>  clock=<CLOCK_NS> ns
  ← NEXT    Step 3 — Build commands
─────────────────────────────────────────────────────
```

---

## Step 3: Build commands
## 3단계: 빌드 명령

The canonical commands for all HLS flows are:
모든 HLS 흐름에 대한 표준 명령은 다음과 같습니다.

| Step<br>단계 | Command<br>명령 |
|---|---|
| C Simulation<br>C 시뮬레이션 | `vitis-run --mode hls --csim --config <config_file> --work_dir <TOP_FUNCTION>`<br>`vitis-run --mode hls --csim --config <config_file> --work_dir <TOP_FUNCTION>` |
| C Synthesis<br>C 합성 | `v++ --compile --mode hls --config <config_file> --work_dir <TOP_FUNCTION>`<br>`v++ --compile --mode hls --config <config_file> --work_dir <TOP_FUNCTION>` |
| Co-Simulation<br>공동 시뮬레이션 | `vitis-run --mode hls --cosim --config <config_file> --work_dir <TOP_FUNCTION>`<br>`vitis-run --mode hls --cosim --config <config_file> --work_dir <TOP_FUNCTION>` |

**Do not use** `vitis -s run.py`, `make csim`, `make csynth`, or `make cosim` — these either spawn a vitis-server (which hangs) or are not available in all designs.
**사용하지 마십시오** `vitis -s run.py`, `make csim`, `make csynth` 또는 `make cosim` — 이러한 것들은 vitis 서버를 생성하거나(멈추는) 모든 디자인에서 사용할 수 없습니다.

Reports land under `hls/hls/syn/report/` (csynth) and `hls/hls/csim/report/` (csim).
보고서는 `hls/hls/syn/report/`(csynth) 및 `hls/hls/csim/report/`(csim) 아래에 있습니다.

Print after Step 3:
3단계 이후 인쇄:
```
─────────────────────────────────────────────────────
[hls-architect/setup]  Step 3 — done
  ✓ Step 1  Vitis on PATH
  ✓ Step 2  Design discovered
  ✓ Step 3  Build commands ready (vitis-run / v++)
  ← NEXT    Step 4 — OpenCV dependency check
─────────────────────────────────────────────────────
```

---

## Step 4: OpenCV dependency check
## 4단계: OpenCV 종속성 확인

Detect whether OpenCV is even needed:
OpenCV가 필요한지 여부를 감지합니다.

```bash
grep -rl "opencv\|cv\.h\|highgui\|imgproc\|imgcodecs" . --include="*.cpp" --include="*.hpp" --include="*.h" 2>/dev/null | head -5
```

**If no matches → mark this step `N/A` and skip the rest of Step 4.**
**일치하는 항목이 없으면 → 이 단계를 '해당 사항 없음'으로 표시하고 나머지 4단계를 건너뜁니다.**

**If matches exist**, OpenCV is required.
**일치하는 항목이 있는 경우** OpenCV가 필요합니다.
Resolve paths in this order:
다음 순서로 경로를 확인합니다.

1. **Already in env?** If `$OPENCV_INCLUDE` and `$OPENCV_LIB` are set and the headers exist, reuse them.
1. **이미 환경에 있습니까?** `$OPENCV_INCLUDE` 및 `$OPENCV_LIB`가 설정되어 있고 헤더가 존재하는 경우 재사용합니다.
2. **Otherwise prompt the user** with AskUserQuestion.
2. **그렇지 않으면 AskUserQuestion을 사용하여 사용자에게 메시지를 표시합니다**.
2. Two questions:
2. 두 가지 질문:
   - "Path to OpenCV include directory (containing `opencv2/` subfolder):"
   - "OpenCV 포함 디렉터리 경로(`opencv2/` 하위 폴더 포함):"
   - "Path to OpenCV lib directory (containing `libopencv_core.so`):"
   - "OpenCV lib 디렉터리 경로(`libopencv_core.so` 포함):"

Then `export OPENCV_INCLUDE=<answer>` and `export OPENCV_LIB=<answer>`.
그런 다음 `OPENCV_INCLUDE=<답변>` 내보내기 및 `OPENCV_LIB=<답변>` 내보내기를 수행합니다.

> **Tip for repeat users:** add `export OPENCV_INCLUDE=…` / `export OPENCV_LIB=…` to your shell rc so you skip the prompt every session.
> **반복 사용자를 위한 팁:** 쉘 rc에 `export OPENCV_INCLUDE=…` / `export OPENCV_LIB=…`를 추가하면 매 세션마다 프롬프트를 건너뛸 수 있습니다.

In all paths, finally:
모든 경로에서 마지막으로 다음을 수행합니다.

```bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$OPENCV_LIB
```

Verify the headers and `libopencv_core.so` are reachable:
헤더와 `libopencv_core.so`에 연결할 수 있는지 확인하세요.

```bash
ls "$OPENCV_INCLUDE/opencv2/core.hpp"
ls "$OPENCV_LIB"/libopencv_core.so*
```

These env vars are inherited by every downstream skill (`/hls-optimize`, `/csim`, `/csynth`) — never re-export downstream.
이러한 환경 변수는 모든 다운스트림 기술(`/hls-optimize`, `/csim`, `/csynth`)에 상속되며 다운스트림을 다시 내보내지 않습니다.

Also verify that `<config_file>` (the detected HLS config, if Step 2 ran) references `$(OPENCV_INCLUDE)` / `$(OPENCV_LIB)` — if the build system doesn't pick them up automatically, the compile step will fail with missing headers.
또한 `<config_file>`(2단계가 실행된 경우 감지된 HLS 구성)이 `$(OPENCV_INCLUDE)` / `$(OPENCV_LIB)`를 참조하는지 확인합니다. 빌드 시스템이 이를 자동으로 선택하지 않으면 헤더가 누락되어 컴파일 단계가 실패합니다.

---

## Final summary print
## 최종 요약 인쇄

After all required steps complete, print:
모든 필수 단계가 완료되면 다음을 인쇄하세요.

```
─────────────────────────────────────────────────────
[hls-architect/setup]  done
  ✓ Step 1  Vitis HLS  : <version>  (<path to vitis-run>)
  ✓ Step 4  OpenCV     : <OPENCV_INCLUDE>   (or "N/A — testbench has no OpenCV")
  ✓ Step 2  Design     : top=<TOP_FUNCTION>  cfg=<CONFIG_FILE>  part=<XPART>  clock=<CLOCK_NS> ns   (skipped if tooling-only)
  ✓ Step 3  Build      : Build commands ready                                              (skipped if tooling-only)
─────────────────────────────────────────────────────
```

**Note:** For MATLAB setup, see `../hls-matlab-to-cpp/reference/setup.md`.
**참고:** MATLAB 설정은 `../hls-matlab-to-cpp/reference/setup.md`를 참조하세요.
