<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Questa Backend
# 퀘스타 백엔드

Use this reference for Siemens EDA Questa Advanced Simulator with a Vivado
Vivado가 포함된 Siemens EDA Questa Advanced Simulator에 대해 이 참조를 사용하세요.
project.
프로젝트.
Vivado owns source ordering, IP models, netlist/SDF generation, and
Vivado는 소스 주문, IP 모델, 넷리스트/SDF 생성 및
simulation-script export; Questa performs compile, optimize/elaborate, and run.
시뮬레이션 스크립트 내보내기; Questa는 컴파일, 최적화/세부화 및 실행을 수행합니다.
The canonical `export_simulation` identifier is `questa`.
표준 `export_simulation` 식별자는 `questa`입니다.

## Contents
## 목차

- Prerequisites
- 전제조건
- Vivado project setup
- 비바도 프로젝트 셋업
- Integrated and exported-script flows
- 통합 및 내보낸 스크립트 흐름
- Simulation modes
- 시뮬레이션 모드
- Verdict and artifacts
- 평결 및 아티팩트
- Common failures
- 일반적인 실패

## Prerequisites
## 전제조건

Discover or ask for:
다음을 검색하거나 요청하세요.

- Questa installation/bin path and executable version
- Questa 설치/bin 경로 및 실행 버전
- license environment used by the site
- 사이트에서 사용하는 라이센스 환경
- version-matched AMD libraries previously produced by `compile_simlib`
- 이전에 `compile_simlib`에 의해 생성된 버전과 일치하는 AMD 라이브러리
- host platform supported by the installed Vivado release
- 설치된 Vivado 릴리스에서 지원되는 호스트 플랫폼
- project, simulation set, top, mode, and finite runtime
- 프로젝트, 시뮬레이션 세트, 상단, 모드 및 유한 런타임

Do not guess installation or license paths.
설치 또는 라이센스 경로를 추측하지 마십시오.
Do not automatically launch a
자동으로 시작하지 않음
long `compile_simlib` job merely because libraries are missing.
긴 `compile_simlib` 작업은 단지 라이브러리가 없기 때문입니다.
Report the
보고하다
required command and destination, then ask before running it.
필요한 명령과 대상을 확인한 다음 실행하기 전에 물어보십시오.

Validate the actual version from the executable.
실행 파일에서 실제 버전의 유효성을 검사합니다.
Directory names are not
디렉토리 이름은
authoritative.
권위 있는.

## Project setup
## 프로젝트 설정

In one Vivado MCP session:
하나의 Vivado MCP 세션에서:

```tcl
set simset [get_filesets <simset>]
get_property TOP $simset
get_files -of_objects $simset
update_compile_order -fileset $simset
set_property target_simulator Questa [current_project]
set_property compxlib.questa_compiled_library_dir \
  <absolute-compiled-library-dir> [current_project]
```

Query the properties after setting them.
속성을 설정한 후 쿼리합니다.
Confirm the compiled-library directory
컴파일된 라이브러리 디렉터리 확인
exists and contains the mappings expected by the generated scripts.
존재하며 생성된 스크립트에서 예상하는 매핑을 포함합니다.

If libraries must be built, verify the installed release syntax first:
라이브러리를 빌드해야 하는 경우 먼저 설치된 릴리스 구문을 확인하십시오.

```tcl
compile_simlib -simulator questa -directory <absolute-output-dir>
```

Treat library output as simulator- and Vivado-version-specific.
라이브러리 출력을 시뮬레이터 및 Vivado 버전별로 처리합니다.
Preserve the
보존하다
compile log and do not reuse a stale library merely because the directory
로그를 컴파일하고 디렉토리가 너무 나쁘다는 이유만으로 오래된 라이브러리를 재사용하지 마십시오.
exists.
존재합니다.

## Choose an execution path
## 실행 경로 선택

### Integrated launch
### 통합 출시

Use when the configured Vivado session can launch Questa in the target
구성된 Vivado 세션이 대상에서 Questa를 시작할 수 있는 경우 사용
environment:
환경:

```tcl
set qprops [list_property $simset]
set qlog [lsearch -inline -nocase $qprops {questa*.simulate.log_all_signals}]
set qrun [lsearch -inline -nocase $qprops {questa*.simulate.runtime}]
if {$qlog eq "" || $qrun eq ""} {error "Questa simulation properties not found"}
set_property $qlog true $simset
set_property $qrun <bounded-duration> $simset
catch {close_sim -force}
launch_simulation -simset $simset -mode behavioral
```

This is blocking.
이것이 차단입니다.
Use a bounded testbench/runtime and an appropriate MCP
제한된 테스트벤치/런타임 및 적절한 MCP 사용
timeout.
시간 초과.
Avoid GUI launch for unattended runs.
무인 실행을 위한 GUI 실행을 피하세요.
Full logging can be expensive;
전체 로깅에는 비용이 많이 들 수 있습니다.
use a focused custom `.do` file for large designs.
대규모 디자인에는 집중된 사용자 정의 '.do' 파일을 사용하세요.
Querying is required because
쿼리가 필요하기 때문에
Vivado releases have used both `QUESTA.*` and `QUESTASIM.*` property prefixes.
Vivado 릴리스에서는 'QUESTA.*' 및 'QUESTASIM.*' 속성 접두사를 모두 사용했습니다.

### Exported scripts
### 내보낸 스크립트

Prefer exported scripts for CI, remote shells, reproducible batch runs, or when
CI, 원격 셸, 재현 가능한 배치 실행을 위해 내보낸 스크립트를 선호합니다.
Vivado should not remain blocked:
Vivado는 차단된 상태로 남아 있어서는 안 됩니다.

```tcl
export_simulation \
  -simulator questa \
  -of_objects $simset \
  -directory <absolute-output-dir>
```

Alternatively, Vivado supports:
또는 Vivado는 다음을 지원합니다.

```tcl
launch_simulation -simset $simset -scripts_only
```

Use the generated compile/elaborate/simulate scripts.
생성된 컴파일/정교/시뮬레이트 스크립트를 사용합니다.
Do not replace them with
다음으로 교체하지 마십시오.
handwritten `vlog`/`vcom`/`vopt`/`vsim` commands for IP-heavy,
IP가 많은 경우 직접 작성한 `vlog`/`vcom`/`vopt`/`vsim` 명령,
mixed-language, encrypted, or netlist simulations.
혼합 언어, 암호화 또는 넷리스트 시뮬레이션.

Before execution:
실행 전:

1. inspect the generated README/script usage;
1. 생성된 README/스크립트 사용법을 검사합니다.
2. set the site-provided Questa and license environment in the same shell;
2. 사이트에서 제공하는 Questa와 라이센스 환경을 동일한 쉘에 설정합니다.
3. confirm the generated library mapping points to the validated directory;
3. 생성된 라이브러리 매핑이 검증된 디렉터리를 가리키는지 확인합니다.
4. configure a bounded run and transcript/WLF output;
4. 제한된 실행 및 기록/WLF 출력을 구성합니다.
5. execute compile, elaborate, then simulate in order.
5. 컴파일, 정교화, 시뮬레이션을 순서대로 실행합니다.

## Simulation modes
## 시뮬레이션 모드

Vivado can generate or launch:
Vivado는 다음을 생성하거나 시작할 수 있습니다.

```tcl
# Behavioral
launch_simulation -simset $simset -mode behavioral

# Post-synthesis functional
launch_simulation -simset $simset -mode post-synthesis -type functional

# Post-implementation functional
launch_simulation -simset $simset -mode post-implementation -type functional

# Post-implementation timing
launch_simulation -simset $simset -mode post-implementation -type timing
```

For script export, set the required design stage and mode according to the
스크립트 내보내기의 경우 요구되는 디자인 단계와 모드를 다음과 같이 설정합니다.
installed `export_simulation -help` output.
`export_simulation -help` 출력을 설치했습니다.
Verify netlist and SDF generation
넷리스트 및 SDF 생성 확인
for timing mode.
타이밍 모드의 경우.
Do not hand-annotate an arbitrary SDF onto a mismatched
일치하지 않는 SDF에 임의의 SDF를 직접 주석으로 추가하지 마십시오.
netlist.
넷리스트.

## Batch termination and waveform
## 일괄 종료 및 파형

The testbench should emit an explicit PASS marker, fail checks with `$fatal` or
테스트벤치는 명시적인 PASS 마커를 내보내야 하며 `$fatal`을 사용한 실패 확인 또는
the verification framework's error mechanism, call `$finish`, and include a
검증 프레임워크의 오류 메커니즘, `$finish`를 호출하고
watchdog.
감시견.

For a direct Questa batch command generated by Vivado, preserve:
Vivado에 의해 생성된 직접적인 Questa 배치 명령의 경우 다음을 유지하십시오.

- transcript/log
- 성적표/로그
- generated compile/elaborate/simulate scripts and `.do` files
- 생성된 컴파일/정교/시뮬레이트 스크립트 및 `.do` 파일
- `modelsim.ini` or generated library map
- `modelsim.ini` 또는 생성된 라이브러리 맵
- WLF waveform database
- WLF 파형 데이터베이스
- seed, plusargs, generics, and defines
- 시드, plusargs, 제네릭 및 정의

The usual project-mode behavioral directory is:
일반적인 프로젝트 모드 동작 디렉터리는 다음과 같습니다.

```text
<project>.sim/<simset>/behav/questa/
```

Discover the actual run directory from the generated scripts or launch output;
생성된 스크립트 또는 실행 출력에서 실제 실행 디렉터리를 검색합니다.
post-synthesis and post-implementation paths differ.
합성 후 경로와 구현 후 경로가 다릅니다.
Verify that the transcript
성적표 확인
and WLF exist before reporting them.
WLF는 보고하기 전에 존재합니다.

If adding a custom run `.do`, keep it minimal and layer it on generated scripts:
사용자 정의 실행 `.do`를 추가하는 경우 이를 최소화하고 생성된 스크립트에 계층화합니다.

```tcl
log -r /*
run <bounded-duration>
quit -code 0
```

Do not use `run -all` unless the testbench has a proven watchdog and terminal
테스트벤치에 검증된 워치독과 터미널이 없으면 'run -all'을 사용하지 마세요.
finish.
마무리.

## Verdict
## 평결

Require:
요구 사항:

- all generated compile and optimize/elaboration stages succeeded;
- 생성된 모든 컴파일 및 최적화/정교 단계가 성공했습니다.
- the simulator exited normally;
- 시뮬레이터가 정상적으로 종료되었습니다.
- the required PASS marker or framework result exists;
- 필수 PASS 마커 또는 프레임워크 결과가 존재합니다.
- no unexpected `** Error`, `** Fatal`, failed assertion, timeout, or nonzero
- 예상치 못한 `** 오류`, `** 치명적`, 어설션 실패, 시간 초과 또는 0이 아닌 값 없음
test count exists.
테스트 횟수가 존재합니다.

A WLF file proves signals were recorded, not that behavior passed.
WLF 파일은 동작이 전달된 것이 아니라 신호가 기록되었음을 증명합니다.

## Coverage scope
## 적용범위

This skill's iterative code-coverage closure workflow is XSim-specific.
이 기술의 반복적인 코드 적용 범위 폐쇄 작업 흐름은 XSim에만 해당됩니다.
Do not
하지 마십시오
apply `write_xsim_coverage` or `export_xsim_coverage` to Questa.
Questa에 `write_xsim_coverage` 또는 `export_xsim_coverage`를 적용합니다.
Questa native
퀘스타 원주민
coverage can be added as a separate backend capability only after its license,
해당 라이선스 이후에만 별도의 백엔드 기능으로 적용 범위를 추가할 수 있습니다.
commands, database format, merge flow, and evaluations are documented and
명령, 데이터베이스 형식, 병합 흐름 및 평가가 문서화되어 있으며
tested.
테스트되었습니다.

## Common failures
## 일반적인 실패

- **`Unable to checkout a license` / `License checkout failed`**: preserve the
- **`라이센스를 체크아웃할 수 없습니다` / `라이센스 체크아웃 실패`**:
feature name and vendor message; escalate to the site's license setup.
기능 이름 및 공급업체 메시지 사이트의 라이센스 설정으로 에스컬레이션하세요.
- **`Library ... not found` / `Failed to open design unit`**: distinguish a
- **`라이브러리 ... 찾을 수 없음` / `디자인 단위를 열지 못했습니다`**: 구별
missing mapping from stale or incomplete `compile_simlib` output by checking
확인하여 오래되었거나 불완전한 `compile_simlib` 출력에서 매핑이 누락되었습니다.
the generated map and library compile log.
생성된 맵 및 라이브러리 컴파일 로그.
- **`vsim`/`vopt` executable not found or wrong version**: report the
- **`vsim`/`vopt` 실행 파일을 찾을 수 없거나 잘못된 버전**: 보고
executable-resolved path/version; do not infer it from the install directory.
실행 파일 확인 경로/버전; 설치 디렉터리에서 추론하지 마세요.
- **Encrypted IP failure**: confirm the supported simulator/version and
- **암호화된 IP 실패**: 지원되는 시뮬레이터/버전을 확인하고
generated Vivado scripts.
Vivado 스크립트를 생성했습니다.
- **Mixed-language binding failure**: use Vivado compile order and generated
- **혼합 언어 바인딩 실패**: Vivado 컴파일 순서를 사용하여 생성됨
scripts; inspect library assignment and top language.
스크립트; 라이브러리 할당 및 상위 언어를 검사합니다.
- **Optimization removed signals**: capture interface evidence first; add only
- **최적화 제거 신호**: 인터페이스 증거를 먼저 캡처합니다. 추가만
justified debug visibility options.
정당한 디버그 가시성 옵션.
- **SDF annotation failure**: verify netlist/SDF pair, hierarchy, mode, and
- **SDF 주석 실패**: 넷리스트/SDF 쌍, 계층 구조, 모드 및
implementation freshness.
구현 신선도.
`vopt` `syntax error, unexpected $end` in the SDF
SDF의 `vopt` `구문 오류, 예상치 못한 $end`
points at an incomplete Vivado-written SDF; follow the SDF-writer check in
불완전한 Vivado 작성 SDF를 지적합니다. SDF 작가 체크인을 따르세요
`diagnostics.md` (post-implementation timing).
`diagnostics.md`(구현 후 타이밍).
A timing PASS also requires
타이밍 PASS에는 다음 사항도 필요합니다.
`(vsim-3587) SDF Backannotation Successfully Completed`.
`(vsim-3587) SDF 역주석이 성공적으로 완료되었습니다.`.
- **Run hangs**: stop at the configured timeout, preserve transcript, and
- **실행 중단**: 구성된 시간 초과 시 중지하고 기록을 보존하며
classify as timeout; do not repeatedly relaunch.
시간 초과로 분류합니다. 반복해서 다시 시작하지 마십시오.

## Official sources
## 공식 출처

- [UG900 supported simulators](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 지원 시뮬레이터](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 third-party simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Simulating-with-Third-Party-Simulators)
- [UG900 타사 시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Simlating-with-Third-Party-Simulators)
- [UG900 export_simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG900 내보내기_시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG973 compatible third-party tools](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
- [UG973 호환 타사 도구](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
