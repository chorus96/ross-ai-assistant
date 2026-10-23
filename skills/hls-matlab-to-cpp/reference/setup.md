<!-- ko-translation: enabled -->
---
description: Set up the matlab-to-cpp environment — verify Vitis, OpenCV, MATLAB. Delegates to hls-architect/setup for Vitis+OpenCV, then adds MATLAB on top.
argument-hint: "[mode=tooling-only]   # tooling-only skips design discovery"
---
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Skill: setup (matlab-to-cpp)
# 기술: 설정(matlab-to-cpp)

Prepare the environment for a MATLAB-to-C++ design session.
MATLAB-C++ 설계 세션을 위한 환경을 준비합니다.
This setup:
이 설정은 다음과 같습니다.
1. **Delegates to `../hls-architect/reference/setup.md`** for Vitis + OpenCV + design discovery
1. **Vitis + OpenCV + 디자인 검색에 대해서는 `../hls-architect/reference/setup.md`에 위임합니다
2. **Adds MATLAB setup** on top
2. **맨 위에 MATLAB 설정 추가**

Run this:
이것을 실행하세요:
- **From `/matlab-to-cpp`** at the very start, with `mode=tooling-only` (no design exists yet)
- **처음에는 `/matlab-to-cpp`**에서 `mode=tooling-only`를 사용합니다(아직 디자인이 없음).
- **From `/csim`, `/csynth`, `/cosim` after matlab-to-cpp** with no args (full setup including design discovery)
- **인수 없이 matlab-to-cpp 이후 `/csim`, `/csynth`, `/cosim`에서(디자인 검색을 포함한 전체 설정)

---

## Step 0: Delegate to hls-architect setup
## 0단계: hls-architect 설정에 위임

Call `../hls-architect/reference/setup.md` to handle Vitis, OpenCV, and design discovery.
Vitis, OpenCV 및 디자인 검색을 처리하려면 `../hls-architect/reference/setup.md`를 호출하세요.

Pass through the `mode=tooling-only` argument if present in `$ARGUMENTS`.
`$ARGUMENTS`에 있는 경우 `mode=tooling-only` 인수를 전달합니다.

```bash
# Conceptually:
# Call ../hls-architect/reference/setup.md with same arguments
# This gets us: Vitis (Step 1), Design (Step 2), Build (Step 3), OpenCV (Step 4)
```

Print at the start:
시작 시 인쇄:
```
─────────────────────────────────────────────────────
[matlab-to-cpp/setup]  Step 0 — delegating to hls-architect/setup
  → Calling ../hls-architect/reference/setup.md
  → This will verify: Vitis, OpenCV, Design (if not tooling-only)
─────────────────────────────────────────────────────
```

**Wait for hls-architect/setup to complete.** Then continue to Step 1 below.
**hls-architect/setup이 완료될 때까지 기다립니다.** 그런 다음 아래 1단계를 계속 진행합니다.

After Step 0 completes, print:
0단계가 완료되면 다음을 인쇄합니다.
```
─────────────────────────────────────────────────────
[matlab-to-cpp/setup]  Step 0 — done
  ✓ hls-architect/setup completed
  ← NEXT    Step 1 — MATLAB executable check
─────────────────────────────────────────────────────
```

---

## Step 1: MATLAB executable check
## 1단계: MATLAB 실행 파일 검사

MATLAB is required by `/matlab-to-cpp` to run `.m` files and dump `matlab_input.bin` and `matlab_golden.bin` (the goldens every C++ refactor verifies against).
MATLAB은 `/matlab-to-cpp`에서 `.m` 파일을 실행하고 `matlab_input.bin` 및 `matlab_golden.bin`(모든 C++ 리팩터링에서 확인하는 골든)을 덤프하는 데 필요합니다.

If the design has a `golden/` directory with `matlab_*.bin` files already present, mark this step `N/A` and skip.
설계에 `matlab_*.bin` 파일이 이미 있는 `golden/` 디렉토리가 있는 경우 이 단계를 `N/A`로 표시하고 건너뜁니다.

Otherwise resolve the MATLAB binary in this order:
그렇지 않으면 다음 순서로 MATLAB 바이너리를 해결합니다.

1. **Already in env?** If `$MATLAB_BIN` is set and executes, reuse it.
1. **이미 env에 있습니까?** `$MATLAB_BIN`이 설정되어 실행되면 재사용하세요.
2. **On PATH?** `which matlab` — if found, set `MATLAB_BIN=$(which matlab)`.
2. **PATH에?** `which matlab` — 발견된 경우 `MATLAB_BIN=$(which matlab)`을 설정합니다.
3. **Otherwise prompt the user** with AskUserQuestion:
3. **그렇지 않으면 AskUserQuestion을 사용하여 사용자에게 메시지를 표시**합니다.
   - "Path to MATLAB executable (e.g.
   - "MATLAB 실행 파일의 경로(예:
   - `/usr/local/MATLAB/R2024a/bin/matlab`):"
   - `/usr/local/MATLAB/R2024a/bin/matlab`):"

Then `export MATLAB_BIN=<resolved-path>`.
그런 다음 `MATLAB_BIN=<해결 경로>`를 내보냅니다.

Verify the binary launches headless:
바이너리가 헤드리스로 실행되는지 확인합니다.

```bash
"$MATLAB_BIN" -batch "disp('matlab-ok')" -nodesktop -nosplash 2>&1 | tail -3
```

Output must contain `matlab-ok`.
출력에는 `matlab-ok`가 포함되어야 합니다.
If MATLAB hangs at a license prompt, the user has a license problem — surface that to them and stop.
라이선스 프롬프트에서 MATLAB이 중단되면 사용자에게 라이선스 문제가 있는 것입니다. 이를 알리고 중지하십시오.

`$MATLAB_BIN` is inherited by downstream matlab-to-cpp steps — never re-resolve downstream.
`$MATLAB_BIN`은 다운스트림 matlab-to-cpp 단계에 의해 상속됩니다. 다운스트림을 다시 해석하지 마십시오.

> **Tip for repeat users:** add `export MATLAB_BIN=<path>` to your shell rc so you skip the prompt every session.
> **반복 사용자를 위한 팁:** 쉘 rc에 `export MATLAB_BIN=<path>`를 추가하면 매 세션마다 프롬프트를 건너뛸 수 있습니다.

---

## Final summary print
## 최종 요약 인쇄

After all steps complete, print:
모든 단계가 완료되면 다음을 인쇄합니다.

```
─────────────────────────────────────────────────────
[matlab-to-cpp/setup]  done
  ✓ Step 0  Delegated  : hls-architect/setup completed
            ├─ Vitis   : <version>  (<path to vitis-run>)
            ├─ OpenCV  : <OPENCV_INCLUDE>   (or "N/A")
            ├─ Design  : top=<TOP_FUNCTION>  cfg=<CONFIG_FILE>   (or "skipped — tooling-only")
            └─ Build   : Build commands ready           (or "skipped — tooling-only")
  ✓ Step 1  MATLAB     : <MATLAB_BIN>   (or "N/A — golden/ already populated")
─────────────────────────────────────────────────────
```

**Environment variables exported:**
**내보낸 환경 변수:**
- `$XILINX_VITIS` — from hls-architect/setup Step 1
- `$XILINX_VITIS` — hls-architect/setup 1단계에서
- `$OPENCV_INCLUDE`, `$OPENCV_LIB` — from hls-architect/setup Step 4 (if OpenCV needed)
- `$OPENCV_INCLUDE`, `$OPENCV_LIB` — hls-architect/setup 4단계에서(OpenCV가 필요한 경우)
- `$MATLAB_BIN` — from this setup Step 1
- `$MATLAB_BIN` — 이 설정에서 1단계
