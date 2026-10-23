<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# RTL Lint Multi-Violation — Quick Start Prompts
# RTL 린트 다중 위반 - 빠른 시작 프롬프트

The example ships as a single RTL source file with intentional lint violations.
예제는 의도적인 Lint 위반이 포함된 단일 RTL 소스 파일로 제공됩니다.
No project creation is needed — the skill runs `synth_design -lint` directly on
프로젝트 생성이 필요하지 않습니다. 기술은 `synth_design -lint`를 직접 실행합니다.
the Verilog file.
Verilog 파일.

## Step 1 — Run RTL lint on the design
## 1단계 - 디자인에서 RTL Lint 실행

```
Use /rtl-lint to lint the file input/src/lint_violation_top.v targeting part
xcvc1902-vsva2197-2MP-e-S. The design has intentional violations across
multiple categories (ASSIGN, INFER, CLOCK). Detect all violations, generate
the lint report, and propose fixes for each one.
```
