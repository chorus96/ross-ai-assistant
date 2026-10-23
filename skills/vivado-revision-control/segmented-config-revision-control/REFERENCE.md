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
name: segmented-config-revision-control
이름: 세그먼트 구성-개정-제어
description: Versal Segmented Configuration guidance — property capture, dual PDI generation, NoC export.
설명: Versal 세그먼트 구성 지침 — 속성 캡처, 듀얼 PDI 생성, NoC 내보내기.
---

# Segmented Configuration Revision Control
# 분할된 구성 개정 제어

Read this when Step 1 detects `is_segmented_config == 1` in flow_info.
1단계에서 flow_info에서 `is_segmented_config == 1`을 감지하면 이 내용을 읽어보세요.

## Key Concept
## 핵심 개념

Segmented Configuration generates dual PDI files on Versal devices:
세그먼트 구성은 Versal 장치에서 이중 PDI 파일을 생성합니다.
- **boot.pdi** — Platform Management Controller (PMC) + PS boot
- **boot.pdi** — 플랫폼 관리 컨트롤러(PMC) + PS 부팅
- **pld.pdi** — Programmable Logic design
- **pld.pdi** — 프로그래밍 가능 논리 설계

This enables independent PL updates without full device reconfiguration.
이를 통해 전체 장치를 재구성하지 않고도 독립적인 PL 업데이트가 가능합니다.

## Versal Device Families
## Versal 장치 제품군

| Family<br>가족 | Part Pattern<br>부품 패턴 | Segmented Config<br>분할된 구성 | Explicit Property Needed?<br>명시적 속성이 필요합니까? |
|--------|-------------|-----------------|--------------------------|
| Versal Premium<br>베르살 프리미엄 | xcve*<br>xcve* | Optional<br>선택사항 | Yes<br>예 |
| Versal Prime<br>버살 프라임 | xcvp*<br>xcvp* | Optional<br>선택사항 | Yes<br>예 |
| Versal AI Micro<br>Versal AI 마이크로 | xcvm*<br>xcvm* | Optional<br>선택사항 | Yes<br>예 |
| Versal Premium Gen2<br>Versal 프리미엄 Gen2 | xcve2*<br>xcve2* | Always ON<br>항상 켜짐 | No (implicit)<br>아니요(암시적) |
| Versal Prime Gen2<br>버살 프라임 Gen2 | xcvp2*<br>xcvp2* | Always ON<br>항상 켜짐 | No (implicit)<br>아니요(암시적) |
| Versal AI Micro Gen2<br>Versal AI 마이크로 Gen2 | xcvm2*<br>xcvm2* | Always ON<br>항상 켜짐 | No (implicit)<br>아니요(암시적) |

## Key Requirements
## 주요 요구 사항

### 1.
### SEGMENTED_CONFIGURATION Property
### SEGMENTED_CONFIGURATION 속성

This property MUST be captured in project_settings.tcl and restored during
이 속성은 project_settings.tcl에서 캡처되어야 하며 도중에 복원되어야 합니다.
rebuild.
재건축.
Without it, only a single PDI is generated.
이것이 없으면 단일 PDI만 생성됩니다.

The `capture_project_settings` procedure captures this automatically when set.
`capture_project_settings` 프로시저는 설정 시 이를 자동으로 캡처합니다.
For Gen2 devices, the property may not be explicitly set but the feature is
Gen2 장치의 경우 속성이 명시적으로 설정되지 않을 수 있지만 기능은 다음과 같습니다.
implicitly enabled — document this in your project README.
암시적으로 활성화됨 - 프로젝트 README에 이를 문서화하세요.

### 2.
### NoC Solution Export (Optional)
### NoC 솔루션 내보내기(선택 사항)

For designs using Network-on-Chip, the `export_all_sources` procedure searches
Network-on-Chip을 사용하는 설계의 경우 `export_all_sources` 프로시저가 검색됩니다.
for `.ncr` files in common locations.
공통 위치에 있는 `.ncr` 파일의 경우.
If none exist at export time, they can
내보내기 시 존재하지 않는 경우 다음을 수행할 수 있습니다.
be generated post-implementation:
구현 후 생성됩니다.
```tcl
write_noc_solution -force RevisionControl/Sources/NoC/noc_solution.ncr
```

Most IPI designs embed NoC configuration in the CIPS IP, so separate .ncr
대부분의 IPI 설계에는 CIPS IP에 NoC 구성이 포함되어 있으므로 .ncr을 분리합니다.
export is usually not required.
일반적으로 내보내기는 필요하지 않습니다.

### 3.
### Verification After Rebuild
### 재구축 후 확인

After recreating the project from build.tcl, verify:
build.tcl에서 프로젝트를 다시 생성한 후 다음을 확인하세요.
1. `get_property SEGMENTED_CONFIGURATION [current_project]` returns true
1. `get_property SEGMENTED_CONFIGURATION [현재_프로젝트]`가 true를 반환합니다.
2. After bitstream generation: both boot.pdi and pld.pdi are created
2. 비트스트림 생성 후: boot.pdi 및 pld.pdi가 모두 생성됩니다.

## When to Use vivado_doc_search
## vivado_doc_search를 사용해야 하는 경우

- For SEGMENTED_CONFIGURATION property details
- SEGMENTED_CONFIGURATION 속성 세부정보
- For `write_noc_solution` command syntax
- `write_noc_solution` 명령 구문의 경우
- For dual PDI generation flow (UG1281)
- 듀얼 PDI 생성 흐름용(UG1281)
- For Versal device family identification
- Versal 장치 제품군 식별용
