<!-- ko-translation: enabled -->
---
name: hls-run-flow
description:  Run HLS Flow including C Simulation (csim), C Synthesis (csynth), C/RTL Co-Simulation (cosim), Implementation (impl).
license: MIT
argument-hint: < hls config path $CONFIG >
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

# Skill: hls-run-flow
# 스킬: hls-run-flow

## Description
## 설명

This skill runs one of the four HLS flows based on the keyword provided by the user:
이 기술은 사용자가 제공한 키워드를 기반으로 네 가지 HLS 흐름 중 하나를 실행합니다.

| User keyword<br>사용자 키워드 | Flow triggered<br>흐름이 트리거됨 |
|---|---|
| `run csim`<br>`csim 실행` | C Simulation<br>C 시뮬레이션 |
| `run csynth`<br>`신신스 실행` | C Synthesis<br>C 합성 |
| `run cosim`<br>'코심 실행' | C/RTL Co-Simulation<br>C/RTL 공동 시뮬레이션 |
| `run impl`<br>`impl 실행` | Implementation<br>구현 |

**Resolving `$CONFIG` and `$WORK_DIR` before execution:**
**실행 전 `$CONFIG` 및 `$WORK_DIR` 해결:**

1. **Locate the HLS config file (`$CONFIG`):**
1. **HLS 구성 파일(`$CONFIG`)을 찾습니다.**
   - If the user provides a config path explicitly, use it.
   - 사용자가 구성 경로를 명시적으로 제공한 경우 이를 사용합니다.
   - Otherwise, search the workspace for a `.cfg` file (e.g., `hls_config.cfg`) and use the first one found.
   - 그렇지 않으면 작업공간에서 `.cfg` 파일(예: `hls_config.cfg`)을 검색하고 발견된 첫 번째 파일을 사용하세요.
   - `$CONFIG` should be a path relative to the workspace root.
   - `$CONFIG`는 작업공간 루트에 상대적인 경로여야 합니다.

2. **Resolve `$WORK_DIR`:**
2. **`$WORK_DIR` 해결:**
   - If the user provides `$WORK_DIR` explicitly, use it directly.
   - 사용자가 `$WORK_DIR`을 명시적으로 제공한 경우 직접 사용하세요.
   - Otherwise, read the config file and extract the value of `syn.top` (e.g., `syn.top=top_module_name`), then use that as `$WORK_DIR`.
   - 그렇지 않은 경우에는 구성 파일을 읽고 `syn.top` 값(예: `syn.top=top_module_name`)을 추출한 다음 이를 `$WORK_DIR`로 사용하세요.
   - This matches the directory HLS creates for its outputs (e.g., `top_module_name/hls/`).
   - 이는 HLS가 출력을 위해 생성하는 디렉터리(예: `top_module_name/hls/`)와 일치합니다.
   - Do **not** use `.` or `hls` or the workspace root as `$WORK_DIR`.
   - `.` 또는 `hls`를 사용하거나 작업공간 루트를 `$WORK_DIR`로 사용하지 **마세요**.

**Prerequisite checks (enforced before execution):**
**전제조건 확인(실행 전 시행):**

- Before running **cosim**, verify that csynth has already been executed by checking whether the csynth output directory (`$WORK_DIR/hls/syn/`) exists.
- **cosim**을 실행하기 전에 csynth 출력 디렉터리(`$WORK_DIR/hls/syn/`)가 존재하는지 확인하여 csynth가 이미 실행되었는지 확인하세요.
- If it does not exist, stop and inform the user:
- 존재하지 않는 경우 중지하고 사용자에게 다음을 알립니다.
  > "C Synthesis (csynth) must be completed before running Co-Simulation.
  > "Co-Simulation을 실행하기 전에 C 합성(csynth)이 완료되어야 합니다.
  > Please run csynth first."
  > 먼저 csynth를 실행해 주세요."
- Before running **cosim**, always perform a `cosim.argv` preflight check in `$CONFIG`:
- **cosim**을 실행하기 전에 항상 `$CONFIG`에서 `cosim.argv` 실행 전 검사를 수행하세요.
  1. Parse `cosim.argv` from `$CONFIG`.
  1. `$CONFIG`에서 `cosim.argv`를 구문 분석합니다.
  2. If `cosim.argv` exists and its value is non-empty, continue.
  2. `cosim.argv`가 존재하고 해당 값이 비어 있지 않으면 계속하세요.
  3. If `cosim.argv` is missing or empty, try fallback from `csim.argv`.
  3. `cosim.argv`가 누락되었거나 비어 있는 경우 `csim.argv`에서 대체를 시도하세요.
  4. If `csim.argv` exists and its value is non-empty, write the same value to `cosim.argv` in `$CONFIG`, save the file, and continue.
  4. `csim.argv`가 존재하고 해당 값이 비어 있지 않으면 `$CONFIG`의 `cosim.argv`에 동일한 값을 쓰고 파일을 저장한 후 계속합니다.
  5. If both are missing or empty, continue. (some HLS versions do not require `cosim.argv` to be set, so we allow it to be empty).
  5. 둘 다 누락되었거나 비어 있으면 계속하십시오. (일부 HLS 버전에서는 `cosim.argv` 설정이 필요하지 않으므로 비워두는 것이 허용됩니다.)

- Before running **impl**, verify that cosim has already been executed by checking whether the cosim output directory (`$WORK_DIR/hls/sim/`) exists.
- **impl**을 실행하기 전에 cosim 출력 디렉터리(`$WORK_DIR/hls/sim/`)가 있는지 확인하여 cosim이 이미 실행되었는지 확인하세요.
- If it does not exist, stop and inform the user:
- 존재하지 않는 경우 중지하고 사용자에게 다음을 알립니다.
  > "C/RTL Co-Simulation (cosim) must be completed before running Implementation.
  > "구현을 실행하기 전에 C/RTL Co-Simulation(cosim)을 완료해야 합니다.
  > Please run cosim first."
  > 먼저 cosim을 실행해 주세요."

If the user's input does not match any of the four keywords above, ask the user to clarify which flow they want to run.
사용자 입력이 위의 4개 키워드 중 어느 것과도 일치하지 않는 경우 사용자에게 어떤 흐름을 실행하고 싶은지 명확히 해달라고 요청하세요.

**Execution method selection:**
**실행 방법 선택:**

- Detect whether the skill is running inside the **Vitis Unified IDE** by checking if the `runHlsActiveComponentByType` function is available in the current environment.
- 현재 환경에서 `runHlsActiveComponentByType` 함수를 사용할 수 있는지 확인하여 **Vitis Unified IDE** 내에서 스킬이 실행 중인지 감지합니다.
- **If inside Vitis Unified IDE** (function is available): prefer calling `runHlsActiveComponentByType` directly.
- **Vitis Unified IDE 내부의 경우**(기능 사용 가능): `runHlsActiveComponentByType`을 직접 호출하는 것이 좋습니다.
- **If outside Vitis Unified IDE** (e.g., plain terminal): fall back to the corresponding `bash` command.
- **Vitis Unified IDE 외부**(예: 일반 터미널): 해당 'bash' 명령으로 돌아갑니다.

## How to run
## 달리는 방법

### Run C Simulation (csim)
### C 시뮬레이션 실행(csim)

**If inside Vitis Unified IDE** (preferred): call `runHlsActiveComponentByType` with type `"0"`.
**Vitis Unified IDE 내부에 있는 경우**(권장): `"0"` 유형으로 `runHlsActiveComponentByType`을 호출합니다.

**If outside Vitis Unified IDE** (fallback):
**Vitis Unified IDE 외부인 경우**(대체):

```bash
vitis-run --mode hls --config $CONFIG --work_dir $WORK_DIR --csim
```


### Run C Synthesis (csynth)
### C 합성(csynth) 실행

**If inside Vitis Unified IDE** (preferred): call `runHlsActiveComponentByType` with type `"1"`.
**Vitis Unified IDE 내부에 있는 경우**(권장): `"1"` 유형으로 `runHlsActiveComponentByType`을 호출합니다.

**If outside Vitis Unified IDE** (fallback):
**Vitis Unified IDE 외부인 경우**(대체):

```bash
v++ --compile --mode hls --config $CONFIG --work_dir $WORK_DIR
```


### Run C/RTL Co-Simulation (cosim)
### C/RTL 공동 시뮬레이션 실행(cosim)

Preflight (mandatory): complete the **cosim.argv preflight check** in the prerequisite section before launching cosim.
실행 전(필수): cosim을 시작하기 전에 전제 조건 섹션에서 **cosim.argv 실행 전 확인**을 완료하세요.

**If inside Vitis Unified IDE** (preferred): call `runHlsActiveComponentByType` with type `"2"`.
**Vitis Unified IDE 내부에 있는 경우**(권장): `"2"` 유형으로 `runHlsActiveComponentByType`을 호출합니다.

**If outside Vitis Unified IDE** (fallback):
**Vitis Unified IDE 외부인 경우**(대체):

```bash
vitis-run --mode hls --config $CONFIG --work_dir $WORK_DIR --cosim
```


### Run Implementation (impl)
### 구현 실행(impl)

**If inside Vitis Unified IDE** (preferred): call `runHlsActiveComponentByType` with type `"3"`.
**Vitis Unified IDE 내부에 있는 경우**(권장): `"3"` 유형으로 `runHlsActiveComponentByType`을 호출합니다.

**If outside Vitis Unified IDE** (fallback):
**Vitis Unified IDE 외부인 경우**(대체):

```bash
vitis-run --mode hls --config $CONFIG --work_dir $WORK_DIR --impl
```
