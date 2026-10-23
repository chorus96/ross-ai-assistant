<!-- ko-translation: enabled -->
---
name: hls-dataflow-infos
description: 'Get Vitis HLS dataflow process/channel informations for a given HLS component.'
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

# hls-dataflow-infos
# hls-데이터 흐름-정보

This skill collects **Vitis HLS dataflow** information (processes + channels) from an HLS component directory.
이 스킬은 HLS 구성 요소 디렉터리에서 **Vitis HLS 데이터 흐름** 정보(프로세스 + 채널)를 수집합니다.

It is implemented by calling the script:
이는 스크립트를 호출하여 구현됩니다.

- [run.sh](./hls-scripts/run.sh)
- [run.sh](./hls-scripts/run.sh)

That script:
해당 스크립트는 다음과 같습니다.
- takes exactly **one argument**: `<component_location>`
- 정확히 **하나의 인수**를 취합니다: `<comComponent_location>`
- prints a JSON array describing all detected dataflow modules, including per-process and per-channel fields
- 프로세스별 및 채널별 필드를 포함하여 감지된 모든 데이터 흐름 모듈을 설명하는 JSON 배열을 인쇄합니다.

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
  - Top-level: a JSON array; each element corresponds to one dataflow module.
  - 최상위 수준: JSON 배열 각 요소는 하나의 데이터 흐름 모듈에 해당합니다.

## How to invoke
## 호출 방법

Must `cd` into `./hls-scripts/` first.
먼저 `./hls-scripts/`에 `cd`를 입력해야 합니다.

```bash
cd ./hls-scripts
./run.sh <component_location>
```
