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
name: dfx-revision-control
이름: dfx-revision-control
description: DFX (Partial Reconfiguration) specific guidance — RM ordering, pblocks, DCPs, pr_verify, Git LFS.
설명: DFX(부분 재구성) 관련 지침 — RM 주문, pblocks, DCP, pr_verify, Git LFS.
---

# DFX Revision Control
# DFX 개정 관리

Read this when Step 1 detects `is_dfx == 1` in flow_info.
1단계에서 flow_info에서 `is_dfx == 1`을 감지하면 이 내용을 읽어보세요.

## Key Requirements
## 주요 요구 사항

### 1.
### RM Sourcing Order in build.tcl
### build.tcl의 RM 소싱 순서

In DFX projects using Block Design Containers (BDC), the static design's BDCs
블록 디자인 컨테이너(BDC)를 사용하는 DFX 프로젝트에서 정적 디자인의 BDC
reference Reconfigurable Module (RM) Block Designs by name.
이름으로 재구성 가능 모듈(RM) 블록 설계를 참조하세요.
The RMs must exist
RM이 존재해야 합니다.
before the static BD is sourced.
정적 BD가 소싱되기 전에.

**Correct order in build.tcl Step 8:**
**build.tcl의 올바른 순서 8단계:**
1. Source all RM BD scripts (e.g., rp1rm1.tcl, rp1rm2.tcl, rp2rm1.tcl)
1. 모든 RM BD 스크립트 소스(예: rp1rm1.tcl, rp1rm2.tcl, rp2rm1.tcl)
2. Source the static design BD script (e.g., design_1.tcl)
2. 정적 디자인 BD 스크립트 소스(예: design_1.tcl)

If the order is wrong:
순서가 잘못된 경우:
```
ERROR: [BD 41-1279] Block Container 'rp1_container' is referencing
an instance 'rp1rm1' that does not exist in the design.
```

After `generate_build_script` runs, review the BD sourcing section and manually
`generate_build_script` 실행 후 BD 소싱 섹션을 검토하고 수동으로
reorder if needed.
필요한 경우 다시 주문하세요.
A simple heuristic: files with "rm" in the name go first.
간단한 경험적 방법: 이름에 "rm"이 있는 파일이 먼저 옵니다.

### 2.
### Pblocks Constraint File
### Pblocks 제약 파일

Pblocks define the physical FPGA regions for reconfigurable partitions.
P블록은 재구성 가능한 파티션에 대한 물리적 FPGA 영역을 정의합니다.
The
는
`export_all_sources` procedure searches for XDC files containing "pblock" in
`export_all_sources` 프로시저는 "pblock"이 포함된 XDC 파일을 검색합니다.
the name or content.
이름이나 내용.
Verify the exported pblocks.xdc contains:
내보낸 pblocks.xdc에 다음이 포함되어 있는지 확인하세요.
- `create_pblock` commands
- `create_pblock` 명령
- `add_cells_to_pblock` assignments
- `add_cells_to_pblock` 할당
- `resize_pblock` region definitions
- `resize_pblock` 영역 정의
- `IS_RECONFIGURABLE true` property
- `IS_RECONFIGURABLE true` 속성

### 3.
### Locked Static DCP
### 잠긴 정적 DCP

After implementation, the locked static DCP is needed for `pr_verify` and
구현 후에는 'pr_verify'에 잠긴 정적 DCP가 필요하며
multi-RM implementation.
다중 RM 구현.
Export it to Sources/Checkpoints/:
소스/체크포인트/로 내보내기:
```tcl
write_checkpoint -force RevisionControl/Sources/Checkpoints/static_locked.dcp
```

### 4.
### Git LFS for DCPs
### DCP용 Git LFS

DCP files are large binaries (often 50-500MB).
DCP 파일은 대용량 바이너리(종종 50-500MB)입니다.
Track them with Git LFS:
Git LFS로 추적하세요.
```bash
git lfs install
git lfs track "*.dcp"
git add .gitattributes
```

### 5. pr_verify Validation
### 5. pr_verify 검증

After rebuilding from build.tcl, validate the DFX configuration:
build.tcl에서 다시 빌드한 후 DFX 구성을 검증합니다.
```tcl
pr_verify -full_check
```

The `helper_scripts.tcl` includes a `pr_verify` utility procedure that checks
`helper_scripts.tcl`에는 다음을 확인하는 `pr_verify` 유틸리티 프로시저가 포함되어 있습니다.
PR_FLOW, reconfigurable cells, and Pblocks.
PR_FLOW, 재구성 가능한 셀 및 Pblock.

## When to Use vivado_doc_search
## vivado_doc_search를 사용해야 하는 경우

- For `pr_verify` command syntax and options
- `pr_verify` 명령 구문 및 옵션의 경우
- For `HD.RECONFIGURABLE` property behavior
- `HD.RECONFIGURABLE` 속성 동작의 경우
- For Block Design Container workflow documentation (UG994)
- 블록 디자인 컨테이너 워크플로 문서(UG994)
- For DFX tutorial and flow details (UG896)
- DFX 튜토리얼 및 흐름 세부정보(UG896)
