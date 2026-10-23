<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# Multi-Violation Timing Methodology — Quick Start Prompts
# 다중 위반 타이밍 방법론 - 빠른 시작 프롬프트

The example ships as source only (RTL + Tcl).
예제는 소스로만 제공됩니다(RTL + Tcl).
Build and synthesize first,
먼저 빌드하고 합성하고,
then run methodology checks.
그런 다음 방법론 점검을 실행하십시오.

## Step 1 — Create and synthesize the project
## 1단계 - 프로젝트 생성 및 합성

Either run it yourself:
직접 실행해 보세요.

```bash
cd input
vivado -mode batch -source create_project.tcl
# then:
#   launch_runs synth_1 -jobs 8
#   wait_on_run synth_1
```

or ask the agent:
또는 상담원에게 문의하세요.

```
Open Vivado, source input/create_project.tcl, then synthesize the design
(launch_runs synth_1, wait_on_run synth_1).
```

## Step 2 — Run methodology checks and resolve violations
## 2단계 - 방법론 검사 실행 및 위반 해결

```
Use /timing-methodology-checks to run report_methodology on the synthesized
design. The constraints file has intentional errors (duplicate clock, missing
clock, wrong false_path target, missing clock groups). Identify all violations,
prioritize by GROUP, and resolve them. Generate the resolution report.
```
