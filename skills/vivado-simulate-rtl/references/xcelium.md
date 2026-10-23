<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Xcelium Backend
# Xcelium 백엔드

Use this reference for Cadence Xcelium Parallel Simulator with a Vivado
Vivado가 포함된 Cadence Xcelium 병렬 시뮬레이터에 대해 이 참조를 사용하세요.
project.
프로젝트.
The canonical `export_simulation` identifier is `xcelium`.
표준 `export_simulation` 식별자는 `xcelium`입니다.

## Discovery triggers
## 발견 트리거

Xcelium, `xrun`, `xmvlog`, `CDS_INST_DIR`, `target_simulator Xcelium`,
Xcelium, `xrun`, `xmvlog`, `CDS_INST_DIR`, `target_simulator Xcelium`,
`export_simulation -simulator xcelium`.
`export_simulation -simulator xcelium`.
Do not select Xcelium as a silent
Xcelium을 무음으로 선택하지 마십시오.
fallback.
대체.

## Prerequisites
## 전제조건

Discover or ask for:
다음을 검색하거나 요청하세요.

- `CDS_INST_DIR` and the executable-reported `xrun` version
- `CDS_INST_DIR` 및 실행 가능 보고 `xrun` 버전
- license environment (`CDS_LIC_FILE` / `CDS_LIC_WAN` or site equivalent)
- 라이센스 환경(`CDS_LIC_FILE` / `CDS_LIC_WAN` 또는 이에 상응하는 사이트)
- version-matched AMD libraries from `compile_simlib -simulator xcelium`
- `compile_simlib -simulator xcelium`의 버전 일치 AMD 라이브러리
- Linux host supported by the installed Vivado/UG973 pair
- 설치된 Vivado/UG973 쌍이 지원하는 Linux 호스트
- project, simulation set, top, mode, and finite runtime
- 프로젝트, 시뮬레이션 세트, 상단, 모드 및 유한 런타임

Current official Vivado integration is Linux-only.
현재 공식 Vivado 통합은 Linux 전용입니다.
Do not guess paths.
경로를 추측하지 마십시오.
Do not
하지 마십시오
run `compile_simlib` without approval.
승인 없이 `compile_simlib`를 실행합니다.

## Vivado integration
## 비바도 통합

```tcl
set simset [get_filesets <simset>]
update_compile_order -fileset $simset
set_property target_simulator Xcelium [current_project]
set_property compxlib.xcelium_compiled_library_dir \
  <absolute-compiled-library-dir> [current_project]
```

Canonical IDs:
정식 ID:

- `target_simulator`: `Xcelium`
- `target_simulator`: `Xcelium`
- `export_simulation -simulator`: `xcelium`
- `export_simulation -simulator`: `xcelium`
- `compile_simlib -simulator`: `xcelium`
- `compile_simlib -simulator`: `xcelium`

`export_simulation -single_step` is Xcelium-specific.
`export_simulation -single_step`은 Xcelium에만 해당됩니다.
Query `xcelium.*`
`xcelium.*` 쿼리
fileset properties from the installed release.
설치된 릴리스의 파일 세트 속성.

## Execution
## 실행

```tcl
export_simulation \
  -simulator xcelium \
  -of_objects $simset \
  -directory <absolute-output-dir> \
  -lib_map_path <absolute-compiled-library-dir>
```

Use the generated `xrun`/multi-step scripts.
생성된 `xrun`/다단계 스크립트를 사용하세요.
Do not invent `xrun` command
`xrun` 명령을 만들지 마세요
lines for IP-heavy designs.
IP가 많은 설계를 위한 라인입니다.
Integrated `launch_simulation` is allowed when
통합 `launch_simulation`은 다음과 같은 경우에 허용됩니다.
Xcelium is on PATH.
Xcelium이 PATH에 있습니다.
Bound the runtime; prefer batch over GUI for unattended
런타임을 바인딩합니다. 무인 작업에는 GUI보다 일괄 처리를 선호합니다.
runs.
달린다.

Typical behavioral directory: `<project>.sim/<simset>/behav/xcelium/`
일반적인 동작 디렉터리: `<project>.sim/<simset>/behav/xcelium/`

## Artifacts and verdict
## 유물과 평결

Preserve `xrun` logs, generated scripts, library mapping, waveform/SHM from
'xrun' 로그, 생성된 스크립트, 라이브러리 매핑, 파형/SHM을 보존합니다.
the generated flow, seed, plusargs, and defines.
생성된 흐름, 시드, plusargs 및 정의.
Require compile/elaborate
컴파일/정교 필요
success, a normal `xrun` exit, an explicit `TEST_PASS`, and no unexpected
성공, 일반적인 `xrun` 종료, 명시적인 `TEST_PASS`, 예상치 못한 결과 없음
`*E` / `*F` fatals.
`*E` / `*F`는 치명적입니다.
A waveform database is not functional PASS.
파형 데이터베이스가 작동하지 않습니다. PASS.

## Diagnostics
## 진단

License failure, missing `xrun`, INCA/Xcelium library path errors, encrypted
라이선스 실패, 'xrun' 누락, INCA/Xcelium 라이브러리 경로 오류, 암호화됨
IP rejection, and timeout are environment/`timeout` unless an assertion names
IP 거부 및 시간 초과는 어설션 이름이 아닌 한 환경/`시간 초과`입니다.
the DUT mismatch.
DUT 불일치.

In timing mode, `xmsdfc` errors followed by `*W,SDFCNC ... skipping
타이밍 모드에서 `xmsdfc` 오류에 이어 `*W,SDFCNC ... 건너뛰기
annotation` or `This SDF System Task will be Ignored` mean the netlist ran
주석' 또는 '이 SDF 시스템 작업은 무시됩니다'는 넷리스트가 실행되었음을 의미합니다.
with zero delay: the run is not a timing result even if it prints
지연 없음: 인쇄하더라도 실행은 타이밍 결과가 아닙니다.
`TEST_PASS`.
`테스트_패스`.
See `diagnostics.md` (post-implementation timing).
`diagnostics.md`(구현 후 타이밍)를 참조하세요.

## Official sources
## 공식 출처

- [UG900 supported simulators](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 지원 시뮬레이터](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 export_simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG900 내보내기_시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG973 compatible third-party tools](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
- [UG973 호환 타사 도구](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
