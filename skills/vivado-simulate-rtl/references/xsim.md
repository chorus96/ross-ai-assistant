<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# XSim Backend
# XSim 백엔드

Use this reference for AMD Vivado Simulator.
AMD Vivado Simulator에 대해 이 참조를 사용하세요.
XSim is the default backend and
XSim은 기본 백엔드이며
ships with Vivado.
Vivado와 함께 배송됩니다.

## Contents
## 목차

- Project-mode flow
- 프로젝트 모드 흐름
- Non-project flow
- 비프로젝트 흐름
- Simulation modes
- 시뮬레이션 모드
- Batch control and artifacts
- 일괄 제어 및 아티팩트
- Coverage handoff
- 커버리지 핸드오프

## Project-mode flow
## 프로젝트 모드 흐름

Inspect the simulation set before changing it:
시뮬레이션 세트를 변경하기 전에 검사하십시오.

```tcl
set simset [get_filesets <simset>]
get_property TOP $simset
get_files -of_objects $simset
update_compile_order -fileset $simset
set_property target_simulator XSim [current_project]
```

Set a bounded runtime on the simulation fileset when the testbench does not
테스트벤치가 실행되지 않는 경우 시뮬레이션 파일 세트에 제한된 런타임을 설정합니다.
terminate itself:
자체 종료:

```tcl
set_property xsim.simulate.runtime 1us $simset
```

Use a duration justified by the clock, reset, protocol latency, and test plan;
시계, 재설정, 프로토콜 대기 시간 및 테스트 계획에 따라 정당화되는 기간을 사용하십시오.
do not copy `1us` blindly.
맹목적으로 '1us'를 복사하지 마세요.

Launch behavioral simulation:
행동 시뮬레이션 실행:

```tcl
catch {close_sim -force}
launch_simulation -simset $simset -mode behavioral
```

`launch_simulation` blocks while XSim launches.
XSim이 시작되는 동안 `launch_simulation`이 차단됩니다.
Give
주다
`Vivado:vivado_execute` a suitable timeout.
`Vivado:vivado_execute`는 적절한 시간 초과입니다.
If it returns a running handoff,
실행 중인 핸드오프를 반환하는 경우,
poll `Vivado:vivado_status` before issuing another Tcl command.
다른 Tcl 명령을 실행하기 전에 `Vivado:vivado_status`를 폴링하세요.

## Simulation modes
## 시뮬레이션 모드

Use only a mode supported by the completed design stage:
완료된 설계 단계에서 지원되는 모드만 사용하십시오.

```tcl
# Behavioral RTL
launch_simulation -simset $simset -mode behavioral

# Post-synthesis functional
launch_simulation -simset $simset -mode post-synthesis -type functional

# Post-implementation functional
launch_simulation -simset $simset -mode post-implementation -type functional

# Post-implementation timing with SDF
launch_simulation -simset $simset -mode post-implementation -type timing
```

For post-synthesis or post-implementation modes, verify the corresponding run
합성 후 또는 구현 후 모드의 경우 해당 실행을 확인하세요.
completed successfully.
성공적으로 완료되었습니다.
Timing simulation supplements static timing analysis;
타이밍 시뮬레이션은 정적 타이밍 분석을 보완합니다.
it does not replace it.
그것은 그것을 대체하지 않습니다.

## Non-project flow
## 비프로젝트 흐름

Prefer project mode when an `.xpr` exists because Vivado owns source order, IP
Vivado가 소스 순서, IP를 소유하므로 `.xpr`이 존재할 때 프로젝트 모드를 선호합니다.
models, libraries, defines, and include paths.
모델, 라이브러리, 정의 및 경로 포함.
For an explicit non-project
명시적인 비프로젝트의 경우
flow, use the generated scripts from `export_simulation`, or run the XSim
흐름을 사용하려면 'export_simulation'에서 생성된 스크립트를 사용하거나 XSim을 실행하세요.
stages:
단계:

1. `xvlog` / `xvhdl` — analyze HDL into libraries
1. `xvlog` / `xvhdl` — HDL을 라이브러리로 분석
2. `xelab` — elaborate a snapshot
2. `xelab` — 스냅샷을 정교하게 만듭니다.
3. `xsim` — run the snapshot
3. `xsim` — 스냅샷 실행

Do not synthesize command lines from memory for IP-heavy or mixed-language
IP가 많거나 혼합 언어에 대해 메모리에서 명령줄을 합성하지 마세요.
designs.
디자인.
Create a temporary in-memory/project representation and use:
임시 메모리 내/프로젝트 표현을 만들고 다음을 사용합니다.

```tcl
export_simulation -simulator xsim -directory <absolute-output-dir>
```

Run the generated compile, elaborate, and simulate scripts outside the
생성된 컴파일, 정교화 및 시뮬레이션 스크립트를 외부에서 실행합니다.
single-threaded Vivado Tcl session.
단일 스레드 Vivado Tcl 세션.
Preserve the generated scripts with the
생성된 스크립트를 다음과 같이 보존합니다.
result.
결과.

## Batch control
## 일괄 제어

Self-checking testbenches should:
자체 점검 테스트벤치는 다음을 수행해야 합니다.

- print one terminal `TEST_PASS` marker only after all checks pass;
- 모든 검사가 통과된 후에만 하나의 터미널 'TEST_PASS' 마커를 인쇄합니다.
- use `$fatal` or equivalent for failed checks;
- 실패한 검사에는 `$fatal` 또는 이와 동등한 것을 사용하십시오.
- call `$finish`;
- `$finish`를 호출하세요.
- include a watchdog that fails rather than hanging.
- 중단되지 않고 실패하는 감시 장치를 포함합니다.

When a complete batch waveform is explicitly required and design size permits,
완전한 배치 파형이 명시적으로 필요하고 설계 크기가 허용되는 경우,
enable project-mode logging before launch:
시작하기 전에 프로젝트 모드 로깅을 활성화합니다.

```tcl
set_property xsim.simulate.log_all_signals true $simset
set_property xsim.simulate.wdb <absolute-path>/<target>.wdb $simset
```

For a focused capture, leave `xsim.simulate.runtime` empty so launch loads the
집중 캡처의 경우 `xsim.simulate.runtime`을 비워 두어 실행 시
snapshot without advancing, then log selected objects before running:
진행하지 않고 스냅샷을 만든 다음 실행하기 전에 선택한 개체를 기록합니다.

```tcl
set_property xsim.simulate.runtime {} $simset
launch_simulation -simset $simset -mode behavioral
log_wave [get_objects <clock-reset-and-interface-signals>]
log_wave [get_objects <expected-observed-and-status-signals>]
run <bounded-duration>
create_wave_config <target>_FAIL
add_wave <focused-signals>
```

Run `log_wave` before advancing simulation; it cannot reconstruct earlier
시뮬레이션을 진행하기 전에 'log_wave'를 실행하세요. 더 일찍 재구성할 수는 없습니다.
transitions.
전환.
`add_wave` controls display, while `log_wave` controls WDB
`add_wave`는 디스플레이를 제어하고 `log_wave`는 WDB를 제어합니다.
capture.
캡처.
Do not recursively log every signal by default.
기본적으로 모든 신호를 재귀적으로 기록하지 마세요.
Large wave databases
대규모 파도 데이터베이스
slow simulation and obscure the causal signals.
시뮬레이션이 느려지고 인과적인 신호가 모호해집니다.

## Logs and artifacts
## 로그 및 아티팩트

The usual behavioral project-mode directory is:
일반적인 동작 프로젝트 모드 디렉터리는 다음과 같습니다.

```text
<project>.sim/<simset>/behav/xsim/
```

Common artifacts:
일반적인 아티팩트:

- `xvlog.log` / `xvhdl.log` — analysis
- `xvlog.log` / `xvhdl.log` — 분석
- `xelab.log` — elaboration
- `xelab.log` — 정교화
- `simulate.log` — runtime transcript
- `simulate.log` — 런타임 기록
- `<snapshot>.wdb` — waveform database
- `<스냅샷>.wdb` — 파형 데이터베이스
- generated shell/batch scripts when scripts-only export is used
- 스크립트 전용 내보내기가 사용될 때 생성된 셸/배치 스크립트

Paths differ for post-synthesis and post-implementation modes.
합성 후 모드와 구현 후 모드의 경로는 다릅니다.
Discover the
발견하다
actual directory after launch; do not hard-code `behav/xsim` for every mode.
실행 후 실제 디렉토리; 모든 모드에 대해 'behav/xsim'을 하드 코딩하지 마세요.

Read the complete relevant log when deciding PASS/FAIL.
PASS/FAIL을 결정할 때 전체 관련 로그를 읽으십시오.
A truncated MCP log
잘린 MCP 로그
tail is navigation evidence, not a verdict.
꼬리는 항법 증거이지 평결이 아닙니다.
Vivado can return Tcl exit code
Vivado는 Tcl 종료 코드를 반환할 수 있습니다.
zero from `launch_simulation` even when XSim reports a runtime `Fatal`, so the
XSim이 런타임 `Fatal`을 보고하는 경우에도 `launch_simulation`에서 0이 발생하므로
transcript and explicit terminal marker are mandatory verdict inputs.
성적 증명서 및 명시적 종결 표시는 필수 평결 입력입니다.

## Coverage handoff
## 커버리지 핸드오프

For XSim code coverage, use the coverage-closure reference linked directly
XSim 코드 적용 범위의 경우 직접 연결된 적용 범위 폐쇄 참조를 사용하세요.
from `SKILL.md`.
'SKILL.md'에서.
Coverage must be enabled before elaboration, then written and
설명하기 전에 적용 범위를 활성화한 다음 서면으로 작성해야 합니다.
exported after the run.
실행 후 내보냈습니다.

## Official sources
## 공식 출처

- [UG900 Logic Simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation)
- [UG900 로직 시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation)
- [UG900 batch simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Running-the-Vivado-Simulator-in-Batch-Mode)
- [UG900 배치 시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Running-the-Vivado-Simulator-in-Batch-Mode)
- [UG900 post-synthesis and post-implementation simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Running-Post-Synthesis-and-Post-Implementation-Simulations)
- [UG900 합성 후 및 구현 후 시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Running-Post-Synesis-and-Post-Implementation-Simulations)
