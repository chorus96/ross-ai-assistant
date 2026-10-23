<!-- ko-translation: enabled -->
---
name: hls-cosim-report
description: 'Get Vitis HLS cosimulation report information for a given HLS component.'
license: MIT
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

# hls-cosim-report
# hls-cosim-보고서

Collect **Vitis HLS cosimulation** artifacts from an HLS component directory and emit a single JSON object.
HLS 구성 요소 디렉터리에서 **Vitis HLS 공동 시뮬레이션** 아티팩트를 수집하고 단일 JSON 개체를 내보냅니다.

## When to use
## 언제 사용하나요?

After cosimulation has completed, use this skill to inspect performance/resource estimates, dataflow process and channel infos for a component.
공동 시뮬레이션이 완료된 후 이 기술을 사용하여 구성 요소에 대한 성능/리소스 추정, 데이터 흐름 프로세스 및 채널 정보를 검사합니다.

**Prerequisite:** the component must contain `hls/.autopilot/db/process_info.csv`.
**전제 조건:** 구성 요소에는 `hls/.autopilot/db/process_info.csv`가 포함되어 있어야 합니다.

## Inputs
## 입력

- `component_location` (string, required) — absolute path to the Vitis HLS component directory (the folder containing `vitis-comp.json`).
- `comComponent_location`(문자열, 필수) — Vitis HLS 구성 요소 디렉터리(`vitis-comp.json`이 포함된 폴더)의 절대 경로입니다.
  - If the user specifies a component_location: use the user-specified component_location.
  - 사용자가 구성 요소 위치를 지정하는 경우: 사용자가 지정한 구성 요소 위치를 사용합니다.
  - If the user does not specify component_location:
  - 사용자가 component_location을 지정하지 않은 경우:
    - If inside Vitis Unified IDE: Call the `getActiveComponentLocation` tool to get the component_location.
    - Vitis Unified IDE 내부인 경우: `getActiveComponentLocation` 도구를 호출하여 component_location을 가져옵니다.
    - If outside Vitis Unified IDE: Request the user to provide component_location.
    - Vitis Unified IDE 외부인 경우: 사용자에게 component_location을 제공하도록 요청합니다.

## Output
## 출력

- `stdout`: JSON (pretty-printed)
- `stdout`: JSON(예쁘게 인쇄됨)
  - Top-level: a JSON object.
  - 최상위 수준: JSON 개체.
  - The JSON object includes:
  - JSON 객체에는 다음이 포함됩니다.
    - `coSimulationReportContent`
    - `coSimulationReportContent`
    - `dataflowProcessAndChannelContent`
    - `dataflowProcessAndChannelContent`

## How to invoke
## 호출 방법

Two equivalent methods — same JSON shape.
두 가지 동등한 방법 — 동일한 JSON 모양.

### 1.
### Inside the Vitis Unified IDE (preferred)
### Vitis Unified IDE 내부(선호)

Call the `getHLSCoSimulationInfo` tool if available.
가능한 경우 `getHLSCoSimulationInfo` 도구를 호출하세요.
It uses the currently selected project.
현재 선택된 프로젝트를 사용합니다.

### 2.
### Outside Vitis Unified IDE (fallback)
### Vitis 통합 IDE 외부(대체)

Must `cd` into `./hls-scripts/` first.
먼저 `./hls-scripts/`에 `cd`를 입력해야 합니다.

```bash
cd ./hls-scripts
./run.sh <component_location>
```

Script source: [run.sh](./hls-scripts/run.sh)
스크립트 소스: [run.sh](./hls-scripts/run.sh)
