<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Active-HDL Backend
# Active-HDL 백엔드

Use this reference for Aldec Active-HDL with a Vivado project.
Vivado 프로젝트와 함께 Aldec Active-HDL에 대해 이 참조를 사용하세요.
Official Vivado
공식 비바도
integration is Windows-only.
통합은 Windows에만 해당됩니다.
Do not treat Riviera-PRO as a substitute.
Riviera-PRO를 대체물로 취급하지 마십시오.
The
는
canonical `export_simulation` identifier is `activehdl`.
표준 `export_simulation` 식별자는 `activehdl`입니다.

## Discovery triggers
## 발견 트리거

Active-HDL, `activehdl`, `target_simulator ActiveHDL`,
Active-HDL, `activehdl`, `target_simulator ActiveHDL`,
`export_simulation -simulator activehdl`.
`export_simulation -simulator activehdl`.
Never select Active-HDL on Linux.
Linux에서는 Active-HDL을 선택하지 마십시오.

## Prerequisites
## 전제조건

Discover or ask for:
다음을 검색하거나 요청하세요.

- Active-HDL Windows install and executable-reported version
- Active-HDL Windows 설치 및 실행 가능 보고 버전
- license environment (`ALDEC_LICENSE_FILE` or site equivalent)
- 라이센스 환경(`ALDEC_LICENSE_FILE` 또는 이에 상응하는 사이트)
- version-matched AMD libraries from `compile_simlib -simulator activehdl`
- `compile_simlib -simulator activehdl`의 버전 일치 AMD 라이브러리
- Windows host listed in UG973 for the installed Vivado release
- 설치된 Vivado 릴리스에 대해 UG973에 나열된 Windows 호스트
- project, simulation set, top, mode, and finite runtime
- 프로젝트, 시뮬레이션 세트, 상단, 모드 및 유한 런타임

Stop with an unsupported-platform rejection on Linux before changing
변경하기 전에 Linux에서 지원되지 않는 플랫폼 거부를 중지하세요.
`target_simulator`.
`target_simulator`.
Do not guess paths.
경로를 추측하지 마십시오.
Do not run `compile_simlib` without
없이 `compile_simlib`을 실행하지 마십시오.
approval.
승인.

## Vivado integration
## 비바도 통합

```tcl
set simset [get_filesets <simset>]
update_compile_order -fileset $simset
set_property target_simulator ActiveHDL [current_project]
set_property compxlib.activehdl_compiled_library_dir \
  <absolute-compiled-library-dir> [current_project]
```

Canonical IDs:
정식 ID:

- `target_simulator`: `ActiveHDL`
- `target_simulator`: `ActiveHDL`
- `export_simulation -simulator`: `activehdl`
- `export_simulation -simulator`: `activehdl`
- `compile_simlib -simulator`: `activehdl`
- `compile_simlib -simulator`: `activehdl`

Query `activehdl.*` fileset properties.
`activehdl.*` 파일 세트 속성을 쿼리합니다.
Qualification of Active-HDL is often
Active-HDL의 자격은 종종
RTL-only; do not claim SystemC or every encrypted-IP flow without checking the
RTL 전용; 확인하지 않고 SystemC 또는 모든 암호화된 IP 흐름을 요청하지 마십시오.
current UG900/UG973 notes.
현재 UG900/UG973 노트.

## Execution
## 실행

```tcl
export_simulation \
  -simulator activehdl \
  -of_objects $simset \
  -directory <absolute-output-dir> \
  -lib_map_path <absolute-compiled-library-dir>
```

On Windows, run the generated `.bat` compile/elaborate/simulate scripts.
Windows에서는 생성된 `.bat` 컴파일/정교/시뮬레이션 스크립트를 실행합니다.
Integrated `launch_simulation` is allowed when Active-HDL is configured in
Active-HDL이 구성된 경우 통합 'launch_simulation'이 허용됩니다.
Vivado.
비바도.
Bound the runtime.
런타임을 바인딩했습니다.

Typical behavioral directory: `<project>.sim/<simset>/behav/activehdl/`
일반적인 동작 디렉터리: `<project>.sim/<simset>/behav/activehdl/`

## Artifacts and verdict
## 유물과 평결

Preserve transcript, generated scripts, library map, waveform from the
사본, 생성된 스크립트, 라이브러리 맵, 파형을 보존합니다.
generated flow, seed, plusargs, and defines.
생성된 흐름, 시드, plusargs 및 정의.
Require compile/elaborate
컴파일/정교 필요
success, a normal simulator exit, an explicit `TEST_PASS`, and no unexpected
성공, 정상적인 시뮬레이터 종료, 명시적인 `TEST_PASS`, 예상치 못한 결과 없음
error/fatal.
오류/치명적.
Waveform presence is not PASS.
파형 존재가 PASS가 아닙니다.

## Diagnostics
## 진단

Linux host, missing Windows install, license failure, Riviera/Active-HDL
Linux 호스트, Windows 설치 누락, 라이센스 오류, Riviera/Active-HDL
library mix-up, and timeout: environment or unsupported-platform.
라이브러리 혼합 및 시간 초과: 환경 또는 지원되지 않는 플랫폼.
Design
디자인
failures still require the first assertion and simulation time.
실패에는 여전히 첫 번째 어설션 및 시뮬레이션 시간이 필요합니다.

## Official sources
## 공식 출처

- [UG900 supported simulators](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 지원 시뮬레이터](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 export_simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG900 내보내기_시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG973 compatible third-party tools](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
- [UG973 호환 타사 도구](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
