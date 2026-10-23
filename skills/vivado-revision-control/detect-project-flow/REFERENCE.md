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
name: detect-project-flow
이름: 감지 프로젝트 흐름
description: Step 1 — detects Vivado project flow type (Standard, DFX, Segmented Config, IPI BDC).
설명: 1단계 — Vivado 프로젝트 흐름 유형(Standard, DFX, Segmented Config, IPI BDC)을 감지합니다.
---

# Detect Project Flow (Step 1)
# 프로젝트 흐름 감지(1단계)

**Source file:** `helper-procedures/helper_scripts.tcl`
**소스 파일:** `helper-procedures/helper_scripts.tcl`
**Proc name:** `detect_project_flow`
**프록 이름:** `Detect_project_flow`

## Procedure
## 절차

```tcl
set flow_info [detect_project_flow]
```

Returns a dictionary with these keys:
다음 키가 포함된 사전을 반환합니다.

| Key<br>열쇠 | Type<br>유형 | Description<br>설명 |
|-----|------|-------------|
| `project_name`<br>`프로젝트_이름` | string<br>문자열 | Project name<br>프로젝트 이름 |
| `device`<br>`장치` | string<br>문자열 | Part number<br>부품 번호 |
| `flow_type`<br>`흐름_유형` | string<br>문자열 | "Standard", "DFX (Partial Reconfiguration)", "Segmented Configuration", "DFX + Segmented Configuration", with optional "+ IPI BDC"<br>"표준", "DFX(부분 재구성)", "세그먼트 구성", "DFX + 세그먼트 구성", 옵션 "+ IPI BDC" 포함 |
| `is_dfx`<br>`is_dfx` | boolean<br>부울 | 1 if PR_FLOW enabled or reconfigurable cells found<br>PR_FLOW가 활성화되었거나 재구성 가능한 셀이 발견된 경우 1 |
| `is_segmented_config`<br>`is_segmented_config` | boolean<br>부울 | 1 if SEGMENTED_CONFIGURATION property set<br>SEGMENTED_CONFIGURATION 속성이 설정된 경우 1 |
| `is_versal`<br>`is_versal` | boolean<br>부울 | 1 if Versal device<br>Versal 장치인 경우 1 |
| `is_ipi_bdc`<br>`is_ipi_bdc` | boolean<br>부울 | 1 if Block Design Containers detected<br>블록 디자인 컨테이너가 감지된 경우 1 |
| `bdc_count`<br>`bdc_count` | integer<br>정수 | Number of BDC instances<br>BDC 인스턴스 수 |
| `bd_file_count`<br>`bd_file_count` | integer<br>정수 | Number of Block Design files<br>블록 디자인 파일 수 |
| `pblock_count`<br>`pblock_count` | integer<br>정수 | Number of Pblocks<br>P블록 수 |

## Detection Logic
## 탐지 논리

**DFX detection** checks (any one sufficient):
**DFX 감지** 확인(하나라도 충분함):
- `PR_FLOW` project property is true
- `PR_FLOW` 프로젝트 속성이 true입니다.
- Cells with `HD.RECONFIGURABLE == true` exist AND Pblocks exist
- 'HD.RECONFIGURABLE == true'인 셀이 존재하고 Pblock이 존재합니다.

**Segmented Config detection**:
**분할된 구성 감지**:
- `SEGMENTED_CONFIGURATION` project property is true
- `SEGMENTED_CONFIGURATION` 프로젝트 속성이 true입니다.
- Versal Gen2 devices (xcve2*, xcvp2*, xcvm2*) have implicit enablement
- Versal Gen2 장치(xcve2*, xcvp2*, xcvm2*)에는 암시적 활성화가 있습니다.

**IPI BDC detection**: Opens each Block Design and checks cells for
**IPI BDC 감지**: 각 블록 디자인을 열고 셀을 확인합니다.
`CONFIG.ACTIVE_SYNTH_BD` or `CONFIG.LIST_SYNTH_BD` properties — these indicate
`CONFIG.ACTIVE_SYNTH_BD` 또는 `CONFIG.LIST_SYNTH_BD` 속성 - 이는 다음을 나타냅니다.
a Block Design Container (BD instantiated inside another BD).
블록 디자인 컨테이너(다른 BD 내부에 인스턴스화된 BD).

## Edge Cases
## 엣지 케이스

- **No synthesized netlist needed** — detection works on the source project
- **합성된 넷리스트가 필요하지 않습니다** — 소스 프로젝트에서 감지 작업이 수행됩니다.
- **Versal Gen2 implicit Segmented Config** — Gen2 devices always have it,
- **Versal Gen2 암시적 세그먼트 구성** — Gen2 장치에는 항상 이 기능이 있습니다.
even without the explicit property.
명시적인 속성이 없어도 마찬가지입니다.
The procedure does NOT set this
절차에서는 이것을 설정하지 않습니다.
implicitly; it only checks the property.
암시적으로; 속성만 확인합니다.
You may want to note Gen2 status
Gen2 상태를 기록해 두는 것이 좋습니다.
in your documentation.
귀하의 문서에.
- **Empty project** — returns "Standard" with zeroed counts; proceed with caution
- **빈 프로젝트** — 개수가 0인 "표준"을 반환합니다. 조심해서 진행하세요
- **`get_cells` returns empty pre-synthesis** — DFX detection falls back to
- **`get_cells`는 빈 사전 합성을 반환합니다** — DFX 감지는 다음으로 대체됩니다.
`PR_FLOW` property alone
`PR_FLOW` 속성만

## When to Use vivado_doc_search
## vivado_doc_search를 사용해야 하는 경우

If the project uses an unfamiliar device family, use `vivado_doc_search` to
프로젝트가 익숙하지 않은 장치 제품군을 사용하는 경우 `vivado_doc_search`를 사용하여
look up the part number and confirm whether it's Versal Gen1 or Gen2.
부품 번호를 찾아 Versal Gen1인지 Gen2인지 확인하세요.
