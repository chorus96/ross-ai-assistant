<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

---
name: capture-project-settings
이름: 캡처-프로젝트-설정
description: Step 4 — captures all non-default project properties for reproducible builds.
설명: 4단계 - 재현 가능한 빌드를 위해 기본이 아닌 모든 프로젝트 속성을 캡처합니다.
---

# Capture Project Settings (Step 4)
# 프로젝트 설정 캡처(4단계)

**Source file:** `helper-procedures/helper_scripts.tcl`
**소스 파일:** `helper-procedures/helper_scripts.tcl`
**Proc name:** `capture_project_settings`
**프록 이름:** `capture_project_settings`

## Procedure
## 절차

```tcl
capture_project_settings "RevisionControl/Scripts/project_settings.tcl"
```

**Parameters:**
**매개변수:**
- `output_file` (string) — path for the generated settings file
- `output_file`(문자열) — 생성된 설정 파일의 경로

**Returns:** the output file path.
**반환:** 출력 파일 경로.

## What Gets Captured
## 캡처되는 내용

| Category<br>카테고리 | Properties<br>속성 |
|----------|-----------|
| Project<br>프로젝트 | PART, BOARD_PART, TARGET_LANGUAGE, PR_FLOW, SEGMENTED_CONFIGURATION<br>부품, BOARD_PART, TARGET_LANGUAGE, PR_FLOW, SEGMENTED_CONFIGURATION |
| Source fileset<br>소스 파일 세트 | TOP, VERILOG_DEFINE<br>탑, VERILOG_DEFINE |
| Synthesis run<br>합성 실행 | STEPS.SYNTH_DESIGN.ARGS<br>STEPS.SYNTH_DESIGN.ARGS |
| Implementation run<br>구현 실행 | STEPS.PLACE_DESIGN.ARGS, STEPS.ROUTE_DESIGN.ARGS<br>STEPS.PLACE_DESIGN.ARGS, STEPS.ROUTE_DESIGN.ARGS |
| Hook scripts<br>후크 스크립트 | STEPS.*.TCL.PRE, STEPS.*.TCL.POST (synth and impl runs)<br>STEPS.*.TCL.PRE, STEPS.*.TCL.POST(synth 및 impl 실행) |

The output is a series of `set_property` commands that can be sourced to
출력은 소스로 제공될 수 있는 일련의 `set_property` 명령입니다.
restore the project configuration.
프로젝트 구성을 복원합니다.

## Why This Step Matters
## 이 단계가 중요한 이유

Settings like PR_FLOW and SEGMENTED_CONFIGURATION are invisible in the source
PR_FLOW 및 SEGMENTED_CONFIGURATION과 같은 설정은 소스에 표시되지 않습니다.
files but critical for correct synthesis and implementation.
파일이지만 올바른 합성 및 구현에 중요합니다.
If they're missing
누락된 경우
during rebuild, DFX projects won't enable partial reconfiguration and Versal
재구축 중에 DFX 프로젝트는 부분 재구성 및 Versal을 활성화하지 않습니다.
projects won't generate dual PDI files.
프로젝트는 이중 PDI 파일을 생성하지 않습니다.

## Edge Cases
## 엣지 케이스

- **Default values are excluded** — only non-default properties are written,
- **기본값은 제외** — 기본이 아닌 속성만 기록됩니다.
keeping the file minimal
파일을 최소한으로 유지
- **Run after all configuration** — if you capture settings mid-setup, you'll
- **모든 구성 후 실행** - 설정 중간에 설정을 캡처하면
miss properties set later
나중에 설정된 속성 누락
- **Multiple implementation runs** — all impl runs are captured (impl_1, child runs for DFX)
- **다중 구현 실행** — 모든 impl 실행이 캡처됩니다(impl_1, DFX의 하위 실행).
- **IP_REPO_PATHS** — for Remote/Mixed source scenarios, ensure IP_REPO_PATHS
- **IP_REPO_PATHS** — 원격/혼합 소스 시나리오의 경우 IP_REPO_PATHS를 확인하세요.
is set before capture so it gets recorded
캡처 전에 설정되어 녹화됩니다.

## When to Use vivado_doc_search
## vivado_doc_search를 사용해야 하는 경우

If a property seems missing from the captured file, use `vivado_doc_search` to
캡처된 파일에서 속성이 누락된 것 같으면 `vivado_doc_search`를 사용하여
look up the property name format (e.g., `STEPS.SYNTH_DESIGN.ARGS.DIRECTIVE`
속성 이름 형식을 찾아보세요(예: `STEPS.SYNTH_DESIGN.ARGS.DIRECTIVE`
vs.
`STEPS.SYNTH_DESIGN.ARGS.MORE OPTIONS`).
`STEPS.SYNTH_DESIGN.ARGS.추가 옵션`).
