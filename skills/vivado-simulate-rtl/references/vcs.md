<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# VCS Backend
# VCS 백엔드

Use this reference for Synopsys VCS / VCS MX with a Vivado project.
Vivado 프로젝트가 포함된 Synopsys VCS/VCS MX에 대해 이 참조를 사용하세요.
The
는
canonical `export_simulation` identifier is `vcs`.
표준 `export_simulation` 식별자는 `vcs`입니다.
Help text may also mention
도움말 텍스트에서도 언급될 수 있습니다.
`vcs_mx`; do not invent a third identifier.
`vcs_mx`; 세 번째 식별자를 만들지 마세요.

## Discovery triggers
## 발견 트리거

VCS, VCS MX, `vcs`, `vlogan`, `simv`, `target_simulator VCS`,
VCS, VCS MX, `vcs`, `vlogan`, `simv`, `target_simulator VCS`,
`export_simulation -simulator vcs`.
`export_simulation -simulator vcs`.
Never select VCS because XSim failed.
XSim이 실패했기 때문에 VCS를 선택하지 마십시오.

## Prerequisites
## 전제조건

Discover or ask for:
다음을 검색하거나 요청하세요.

- `VCS_HOME` and the executable-reported version
- `VCS_HOME` 및 실행 가능한 보고 버전
- license environment (`SNPSLMD_LICENSE_FILE` or site equivalent)
- 라이센스 환경(`SNPSLMD_LICENSE_FILE` 또는 이에 상응하는 사이트)
- GNU/compiler package required by that VCS install
- 해당 VCS 설치에 필요한 GNU/컴파일러 패키지
- version-matched AMD libraries from `compile_simlib -simulator vcs`
- `compile_simlib -simulator vcs`의 버전 일치 AMD 라이브러리
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
set_property target_simulator VCS [current_project]
set_property compxlib.vcs_compiled_library_dir \
  <absolute-compiled-library-dir> [current_project]
```

Canonical IDs:
정식 ID:

- `target_simulator`: `VCS`
- `target_simulator`: `VCS`
- `export_simulation -simulator`: `vcs`
- `export_simulation -simulator`: `vcs`
- `compile_simlib -simulator`: `vcs`
- `compile_simlib -simulator`: `vcs`

Query `vcs.*` fileset properties from the installed release.
설치된 릴리스에서 `vcs.*` 파일 세트 속성을 쿼리합니다.
Do not copy Questa
Questa를 복사하지 마십시오
property names.
속성 이름.

## Execution
## 실행

```tcl
export_simulation \
  -simulator vcs \
  -of_objects $simset \
  -directory <absolute-output-dir> \
  -lib_map_path <absolute-compiled-library-dir>
```

Generated flow is typically `vlogan`/`vhdlan`, then `vcs` elaborate, then
생성된 흐름은 일반적으로 `vlogan`/`vhdlan`, 그 다음 `vcs` 정교화, 그 다음
`./simv`.
`./simv`.
Use generated scripts for mixed-language and encrypted AMD IP.
혼합 언어 및 암호화된 AMD IP에 대해 생성된 스크립트를 사용합니다.
Integrated `launch_simulation` is allowed when VCS is on the Vivado session
VCS가 Vivado 세션에 있을 때 통합 `launch_simulation`이 허용됩니다.
PATH.
경로.
Keep the run bounded; VCS DVE GUI is optional and may be absent.
실행을 제한적으로 유지하세요. VCS DVE GUI는 선택 사항이며 없을 수도 있습니다.

On Ubuntu and other hosts where `/bin/sh` is dash, VCS wrappers fail at
Ubuntu 및 '/bin/sh'가 대시인 기타 호스트에서 VCS 래퍼가 실패합니다.
elaboration with `Illegal option -h`.
'잘못된 옵션 -h'로 자세히 설명합니다.
Synopsys scripts start with
Synopsys 스크립트는 다음으로 시작합니다.
`#!/bin/sh -h`; the kernel runs dash with `-h`, and dash does not implement
`#!/bin/sh -h`; 커널은 `-h`를 사용하여 대시를 실행하고 대시는 구현하지 않습니다.
that flag.
그 깃발.
PATH trampolines and `LD_PRELOAD` cannot intercept kernel shebang
PATH 트램펄린과 `LD_PRELOAD`는 커널 shebang을 가로챌 수 없습니다.
execution.
실행.
If you cannot change the system `/bin/sh` and user namespaces are
시스템 `/bin/sh`를 변경할 수 없고 사용자 네임스페이스가 다음과 같은 경우
blocked, point `VCS_HOME` at a writable overlay: copy `$VCS_HOME/bin` (and
차단된 경우 쓰기 가능한 오버레이에서 `VCS_HOME`을 가리킵니다. `$VCS_HOME/bin`을 복사합니다(및
`$VCS_HOME/linux64/bin`) scripts whose shebang is `#!/bin/sh`, rewrite the
`$VCS_HOME/linux64/bin`) shebang이 `#!/bin/sh`인 스크립트를 다시 작성합니다.
interpreter to `/bin/bash` (keep `-h`; bash accepts it), and symlink the rest
`/bin/bash`에 대한 인터프리터(`-h` 유지; bash는 이를 허용함) 나머지는 심볼릭 링크합니다.
of the install.
설치의.
Then put the overlay `bin` first on `PATH`.
그런 다음 `PATH`에 오버레이 `bin`을 먼저 배치합니다.

Typical behavioral directory: `<project>.sim/<simset>/behav/vcs/`
일반적인 동작 디렉터리: `<project>.sim/<simset>/behav/vcs/`

## Artifacts and verdict
## 유물과 평결

Preserve `vlogan`/`vcs`/`simv` logs, generated scripts, library map, waveform
`vlogan`/`vcs`/`simv` 로그, 생성된 스크립트, 라이브러리 맵, 파형 보존
database from the generated flow (often FSDB/VPD), seed, plusargs, and defines.
생성된 흐름(종종 FSDB/VPD)의 데이터베이스, 시드, plusargs 및 정의.
Vivado's generated batch `*_simulate.do` is typically just `run <time>` and
Vivado가 생성한 배치 `*_simulate.do`는 일반적으로 `run <time>`이고
`quit`, so no waveform is written by default: report that the generated flow
`quit`, 기본적으로 파형이 기록되지 않음: 생성된 흐름을 보고합니다.
produced none rather than naming a file that does not exist, and add dumping
존재하지 않는 파일에 이름을 붙이는 대신 none을 생성하고, 덤핑을 추가합니다.
only when waveforms are needed for the diagnosis or requested.
진단을 위해 파형이 필요하거나 요청된 경우에만 가능합니다.
Require successful analysis and elaboration, a normal `simv` exit, an explicit
성공적인 분석 및 정교화, 일반적인 `simv` 종료, 명시적인 종료가 필요합니다.
`TEST_PASS`, and no unexpected error/fatal.
`TEST_PASS`이며 예상치 못한 오류/치명적 오류는 없습니다.
Waveform presence is not PASS.
파형 존재가 PASS가 아닙니다.

## Diagnostics
## 진단

License failure, missing `vcs`/`simv`, GCC/package mismatch, library map
라이센스 실패, `vcs`/`simv` 누락, GCC/패키지 불일치, 라이브러리 맵
errors, encrypted-IP compile failure, and SDF annotation errors are environment
오류, 암호화된 IP 컴파일 실패 및 SDF 주석 오류는 환경입니다.
or flow issues unless an assertion identifies the DUT.
또는 어설션이 DUT를 식별하지 않는 한 흐름 문제가 발생합니다.
Timeouts are `timeout`.
시간 초과는 'timeout'입니다.

## Official sources
## 공식 출처

- [UG900 supported simulators](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 지원 시뮬레이터](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 export_simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG900 내보내기_시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG973 compatible third-party tools](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
- [UG973 호환 타사 도구](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
