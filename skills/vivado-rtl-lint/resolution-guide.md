<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Resolution Guide Integration
# 해결 가이드 통합

This file explains the resolution guide system and how to use it when processing
해결 가이드 시스템과 처리 시 사용 방법을 설명하는 파일입니다.
linter violations.
린터 위반.

---

## Overview
## 개요

Resolution guides are pre-written, validated fix templates for known violation types.
해결 가이드는 알려진 위반 유형에 대해 미리 작성되고 검증된 수정 템플릿입니다.
They live in `resolution/<RULE-ID>.md` and provide:
`solution/<RULE-ID>.md`에 있으며 다음을 제공합니다.

- **Consistency** — Same violation type always gets the same fix pattern
- **일관성** — 동일한 위반 유형은 항상 동일한 수정 패턴을 갖습니다.
- **Accuracy** — Fixes validated against IEEE standards and Xilinx documentation
- **정확성** — IEEE 표준 및 Xilinx 문서에 대해 검증된 수정 사항
- **Completeness** — Includes rationale, verification, and documentation references
- **완전성** — 근거, 확인 및 문서 참조를 포함합니다.
- **Speed** — Pre-written templates reduce analysis time
- **속도** — 미리 작성된 템플릿으로 분석 시간 단축

---

## Available Guides
## 사용 가능한 가이드

| Rule ID<br>규칙 ID    | Description<br>설명 |
|------------|-------------|
| ASSIGN-1<br>할당-1   | Width mismatch in assignments<br>할당의 너비 불일치 |
| ASSIGN-2<br>할당-2   | Signed/unsigned arithmetic mixing<br>부호 있는/부호 없는 산술 혼합 |
| ASSIGN-3<br>할당-3   | Shift overflow (right/left shift exceeding operand width)<br>시프트 오버플로(피연산자 너비를 초과하는 오른쪽/왼쪽 시프트) |
| ASSIGN-5<br>할당-5   | Unassigned signal usage<br>할당되지 않은 신호 사용 |
| ASSIGN-6<br>할당-6   | Signal assigned but never used<br>신호가 할당되었지만 사용되지 않았습니다. |
| ASSIGN-7<br>할당-7   | Multiple assignments to same signal bit<br>동일한 신호 비트에 대한 다중 할당 |
| ASSIGN-8<br>할당-8   | Array dimension mismatch in comparison<br>비교 시 배열 차원 불일치 |
| ASSIGN-9<br>ASSIGN-9   | Unassigned output port bits<br>할당되지 않은 출력 포트 비트 |
| ASSIGN-10<br>할당-10  | Unused I/O port bits<br>사용되지 않은 I/O 포트 비트 |
| ASSIGN-12<br>할당-12  | Unconnected/tri-state port connections<br>연결되지 않은/3상태 포트 연결 |
| ASSIGN-14<br>할당-14  | Duplicate case branches<br>중복된 사례 분기 |
| INFER-1<br>추론-1    | Latch inference (unintended combinational latch)<br>래치 추론(의도하지 않은 조합 래치) |
| INFER-2<br>추론-2    | Incomplete case statement (missing default)<br>불완전한 사례 설명(기본값 누락) |
| INFER-3<br>추론-3    | Case equality (`===`) converted to `==` for synthesis<br>대소문자 동일(`===`)은 합성을 위해 `==`로 변환됩니다. |
| CLOCK-1<br>시계-1    | Mixed clock edges (posedge/negedge)<br>혼합 클록 에지(posedge/negedge) |
| QOR-1<br>QOR-1      | Arithmetic precision loss preventing operator merging<br>연산자 병합을 방지하는 산술 정밀도 손실 |
| RESET-1<br>리셋-1    | Multiple async reset/set on same register<br>동일한 레지스터에 대한 다중 비동기 재설정/설정 |
| RESET-2<br>리셋-2    | Incomplete async reset coverage<br>불완전한 비동기 재설정 적용 범위 |
| RESET-3<br>리셋-3    | Incomplete sync reset / enable driven by sync reset<br>불완전한 동기화 재설정/동기화 재설정으로 인한 활성화 |

*Check `resolution/` directory for the latest list — more guides are added as violations
*최신 목록은 `solution/` 디렉토리를 확인하세요. 더 많은 가이드가 위반 사항으로 추가되었습니다.
are encountered and documented.*
발견되어 문서화되었습니다.*

---

## Mandatory Workflow (Per Violation)
## 필수 워크플로(위반당)

```
1. Detect violation in lint_report.rpt (e.g., ASSIGN-3)
2. Check for guide: resolution/ASSIGN-3.md
3. If guide EXISTS:
   a. Load complete guide with read_file
   b. Follow fix recommendations exactly as documented
   c. Use provided code templates (replace placeholders with actual design objects)
   d. Include the rationale from the guide in your analysis
   e. Reference the documentation links provided
4. If guide does NOT exist:
   a. Use vivado_doc_search + general knowledge
   b. Note in report: "No resolution guide available for [RULE-ID]"
```

---

## Example Workflow — ASSIGN-3
## 워크플로우 예 — ASSIGN-3

```
Step 1: Parse lint_report.rpt
   Found: ASSIGN-3 violation at top.v:21

Step 2: Load resolution guide
   read_file → resolution/ASSIGN-3.md

Step 3: Extract violation specifics
   - Signal: a[12:0] (13 bits)
   - Operator: >> (right shift)
   - Shift amount: 30
   - File: top.v, Line: 21

Step 4: Select fix from guide
   Fix Option 1: Correct shift parameter (SHIFTSIZE = 30 → 2)

Step 5: Generate report section using guide template
   - Include issue description from guide
   - Show before/after code using diff syntax
   - Add rationale from guide
   - Reference UG901 documentation

Step 6: Output formatted markdown
   See "Example Agent Output Template" in ASSIGN-3.md
```

---

## Naming Convention for New Guides
## 새 가이드의 명명 규칙

- File: `resolution/<RULE-ID>.md` (e.g., `TIMING-18.md`, `CDC-1.md`, `SYNTH-8.md`)
- 파일: `해상도/<RULE-ID>.md`(예: `TIMING-18.md`, `CDC-1.md`, `SYNTH-8.md`)
- Required sections: Problem Statement, Detection Pattern, Standard Resolution, Fix Template
- 필수 섹션: 문제 설명, 감지 패턴, 표준 해결, 수정 템플릿
