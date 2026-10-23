<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ModelSim Backend
# ModelSim 백엔드

Use this reference for Siemens EDA ModelSim with a Vivado project.
Vivado 프로젝트와 함께 Siemens EDA ModelSim에 대해 이 참조를 사용하세요.
Keep
유지
ModelSim distinct from Questa even though the command family overlaps.
ModelSim은 명령 계열이 겹치더라도 Questa와 다릅니다.
The
는
canonical `export_simulation` identifier is `modelsim`.
표준 `export_simulation` 식별자는 `modelsim`입니다.

## Discovery triggers
## 발견 트리거

ModelSim, ModelSim DE, `vsim` from a ModelSim install, `target_simulator
ModelSim, ModelSim DE, ModelSim 설치의 `vsim`, `target_simulator
ModelSim`, `export_simulation -simulator modelsim`.
ModelSim`, `export_simulation -simulator modelsim`.
Do not select ModelSim
ModelSim을 선택하지 마십시오.
because Questa is missing.
Questa가 없기 때문입니다.

## Prerequisites
## 전제조건

Discover or ask for:
다음을 검색하거나 요청하세요.

- ModelSim installation/`MODEL_TECH` and the executable-reported version
- ModelSim 설치/`MODEL_TECH` 및 실행 가능 보고 버전
- license environment (`MGLS_LICENSE_FILE` or `LM_LICENSE_FILE` as used on site)
- 라이센스 환경(사이트에서 사용되는 `MGLS_LICENSE_FILE` 또는 `LM_LICENSE_FILE`)
- version-matched AMD libraries from `compile_simlib -simulator modelsim`
- `compile_simlib -simulator modelsim`의 버전 일치 AMD 라이브러리
- a UG973-supported host for the installed Vivado release
- 설치된 Vivado 릴리스를 위한 UG973 지원 호스트
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
set_property target_simulator ModelSim [current_project]
set_property compxlib.modelsim_compiled_library_dir \
  <absolute-compiled-library-dir> [current_project]
```

Query properties after setting them.
속성을 설정한 후 쿼리합니다.
Canonical IDs:
정식 ID:

- `target_simulator`: `ModelSim`
- `target_simulator`: `ModelSim`
- `export_simulation -simulator`: `modelsim`
- `export_simulation -simulator`: `modelsim`
- `compile_simlib -simulator`: `modelsim`
- `compile_simlib -simulator`: `modelsim`

Query `list_property $simset` for `modelsim.*` runtime and log options.
`modelsim.*` 런타임 및 로그 옵션에 대해서는 `list_property $simset`을 쿼리하세요.
Do not
하지 마십시오
copy `questa*` prefixes.
`questa*` 접두어를 복사하세요.

## Execution
## 실행

Prefer Vivado-generated scripts:
Vivado가 생성한 스크립트를 선호하세요:

```tcl
export_simulation \
  -simulator modelsim \
  -of_objects $simset \
  -directory <absolute-output-dir> \
  -lib_map_path <absolute-compiled-library-dir>
```

Or `launch_simulation -scripts_only` / integrated `launch_simulation` when the
또는 `launch_simulation -scripts_only` / 통합된 `launch_simulation`
Vivado session has ModelSim on PATH.
Vivado 세션에는 PATH에 ModelSim이 있습니다.
Run compile, then elaborate/optimize, then
컴파일을 실행한 다음 정교화/최적화한 다음
simulate.
시뮬레이션.
Do not replace generated scripts with handwritten `vlog`/`vsim` for
생성된 스크립트를 손으로 쓴 `vlog`/`vsim`으로 바꾸지 마세요.
IP, mixed-language, encrypted, or netlist work.
IP, 혼합 언어, 암호화 또는 넷리스트 작업.

Typical behavioral directory: `<project>.sim/<simset>/behav/modelsim/`
일반적인 동작 디렉터리: `<project>.sim/<simset>/behav/modelsim/`

## Artifacts and verdict
## 유물과 평결

Preserve transcript/log, generated scripts, library map/`modelsim.ini`, WLF,
기록/로그, 생성된 스크립트, 라이브러리 맵/`modelsim.ini`, WLF를 보존합니다.
seed, plusargs, generics, and defines.
시드, plusargs, 제네릭 및 정의.
Require compile/elaborate success, a
컴파일/정교한 성공이 필요합니다.
normal simulator exit, an explicit `TEST_PASS` (or framework equivalent), and
일반 시뮬레이터 종료, 명시적인 `TEST_PASS`(또는 이에 상응하는 프레임워크) 및
no unexpected `** Error` / `** Fatal`.
예상치 못한 `** 오류` / `** 치명적`이 없습니다.
A WLF is not functional PASS.
WLF는 기능적 PASS가 아닙니다.

## Diagnostics
## 진단

License checkout failure, missing `vsim`, wrong ModelSim vs Questa install,
라이센스 체크아웃 실패, 'vsim' 누락, 잘못된 ModelSim 대 Questa 설치,
stale `compile_simlib` output, encrypted-IP rejection, and timeout: classify as
오래된 `compile_simlib` 출력, 암호화된 IP 거부 및 시간 초과: 다음으로 분류
environment unless the transcript shows a design or testbench assertion.
녹취록에 디자인 또는 테스트벤치 주장이 표시되지 않는 한 환경입니다.

## Official sources
## 공식 출처

- [UG900 supported simulators](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 지원 시뮬레이터](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 export_simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG900 내보내기_시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG973 compatible third-party tools](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
- [UG973 호환 타사 도구](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
