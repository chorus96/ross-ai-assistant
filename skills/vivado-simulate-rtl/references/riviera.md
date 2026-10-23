<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Riviera-PRO Backend
# Riviera-PRO 백엔드

Use this reference for Aldec Riviera-PRO with a Vivado project.
Vivado 프로젝트와 함께 Aldec Riviera-PRO에 대해 이 참조를 사용하세요.
Do not reuse an
재사용하지 마십시오.
Active-HDL install or library tree.
Active-HDL 설치 또는 라이브러리 트리.
The canonical `export_simulation`
정식 `export_simulation`
identifier is `riviera`.
식별자는 '리비에라'입니다.

## Discovery triggers
## 발견 트리거

Riviera-PRO, Riviera, Aldec `vsim` from a Riviera tree, `ALDEC_PATH`,
Riviera-PRO, Riviera, Riviera 트리의 Aldec `vsim`, `ALDEC_PATH`,
`target_simulator Riviera`, `export_simulation -simulator riviera`.
`target_simulator Riviera`, `export_simulation -simulator riviera`.

## Prerequisites
## 전제조건

Discover or ask for:
다음을 검색하거나 요청하세요.

- Riviera installation/`ALDEC_PATH` and the executable-reported version
- Riviera 설치/`ALDEC_PATH` 및 실행 가능한 보고 버전
- license environment (`ALDEC_LICENSE_FILE` or site equivalent)
- 라이센스 환경(`ALDEC_LICENSE_FILE` 또는 이에 상응하는 사이트)
- version-matched AMD libraries from `compile_simlib -simulator riviera`
- `compile_simlib -simulator riviera`의 버전 일치 AMD 라이브러리
- Red Hat Linux or Windows host supported by UG973 for the Vivado release
- Vivado 릴리스용 UG973이 지원하는 Red Hat Linux 또는 Windows 호스트
- project, simulation set, top, mode, and finite runtime
- 프로젝트, 시뮬레이션 세트, 상단, 모드 및 유한 런타임

Do not guess paths.
경로를 추측하지 마십시오.
Do not run `compile_simlib` without approval.
승인 없이 `compile_simlib`를 실행하지 마세요.

## Vivado integration
## 비바도 통합

```tcl
set simset [get_filesets <simset>]
update_compile_order -fileset $simset
set_property target_simulator Riviera [current_project]
set_property compxlib.riviera_compiled_library_dir \
  <absolute-compiled-library-dir> [current_project]
```

Canonical IDs:
정식 ID:

- `target_simulator`: `Riviera`
- `target_simulator`: `리비에라`
- `export_simulation -simulator`: `riviera`
- `export_simulation -simulator`: `리비에라`
- `compile_simlib -simulator`: `riviera`
- `compile_simlib -simulator`: `리비에라`

Query `riviera.*` fileset properties.
`riviera.*` 파일 세트 속성을 쿼리합니다.
Do not copy Active-HDL or Questa
Active-HDL 또는 Questa를 복사하지 마십시오.
prefixes.
접두사.

## Execution
## 실행

```tcl
export_simulation \
  -simulator riviera \
  -of_objects $simset \
  -directory <absolute-output-dir> \
  -lib_map_path <absolute-compiled-library-dir>
```

Run generated compile/elaborate/simulate scripts in order.
생성된 컴파일/정교/시뮬레이션 스크립트를 순서대로 실행합니다.
Integrated
통합
`launch_simulation` is allowed when Riviera is on PATH.
Riviera가 PATH에 있으면 `launch_simulation`이 허용됩니다.
Bound the run; do not
실행을 바운드; 하지 마십시오
use `run -all` without a watchdog.
워치독 없이 `run -all`을 사용하세요.

Typical behavioral directory: `<project>.sim/<simset>/behav/riviera/`
일반적인 동작 디렉터리: `<project>.sim/<simset>/behav/riviera/`

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

License failure, picking ModelSim/Questa `vsim` instead of Riviera, stale
라이센스 실패, Riviera 대신 ModelSim/Questa `vsim` 선택, 오래된 문제
CLIBS, encrypted-IP warnings that become errors, and timeout: classify as
CLIBS, 오류가 되는 암호화된 IP 경고 및 시간 초과:로 분류
environment/`timeout` unless a checker identifies the DUT.
환경/'timeout' 검사기가 DUT를 식별하지 않는 한.

## Official sources
## 공식 출처

- [UG900 supported simulators](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 지원 시뮬레이터](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 export_simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG900 내보내기_시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG973 compatible third-party tools](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
- [UG973 호환 타사 도구](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
