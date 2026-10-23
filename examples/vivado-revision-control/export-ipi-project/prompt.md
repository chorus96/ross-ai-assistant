<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# Export Mixed-Source IPI Project — Quick Start Prompts
# 혼합 소스 IPI 프로젝트 내보내기 - 빠른 시작 프롬프트

The example ships as source only (RTL + Tcl).
예제는 소스로만 제공됩니다(RTL + Tcl).
Create the project first, then
먼저 프로젝트를 생성한 다음
run the revision-control skill on it.
그것에 개정 제어 기술을 실행하십시오.

## Step 1 — Create the project from source
## 1단계 - 소스에서 프로젝트 생성

Either run it yourself:
직접 실행해 보세요.

```bash
cd input
vivado -mode batch -source create_project.tcl
```

or ask the agent:
또는 상담원에게 문의하세요.

```
Open Vivado and source input/create_project.tcl to create the project.
```

## Step 2 — Export sources and generate a portable build script
## 2단계 - 소스 내보내기 및 이식 가능한 빌드 스크립트 생성

```
Use /vivado-revision-control to make this project portable for version control.
Export all sources (RTL, XDC, IP, Block Design) and generate a self-contained
build.tcl that can recreate the project from scratch on any machine.
```

## Step 3 — Verify the generated build script
## 3단계 - 생성된 빌드 스크립트 확인

```
Close the current project. Then run the generated build.tcl in a fresh
Vivado session to verify it recreates the project correctly. Confirm the
top module, IP, and Block Design are all present.
```
