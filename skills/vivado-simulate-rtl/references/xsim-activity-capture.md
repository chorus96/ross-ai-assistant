<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# XSim Activity Capture
# XSim 활동 캡처

Use this reference for scoped SAIF power activity, general VCD export, and
범위가 지정된 SAIF 전력 활동, 일반 VCD 내보내기 및
portable sub-design stimulus capture with `generate_vcd_ports` and
`generate_vcd_ports`를 사용한 휴대용 하위 설계 자극 캡처 및
`create_testbench`.
'create_testbench'.

## Contents
## 목차

- Shared preflight
- 공유 프리플라이트
- SAIF power activity
- SAIF 전력 활동
- General VCD capture
- 일반 VCD 캡처
- Portable stimulus capture
- 휴대용 자극 캡처
- Verification and reporting
- 검증 및 보고

## 1.
## Shared preflight
## 공유 프리플라이트

Record before launch:
출시 전 기록:

```text
Purpose: power activity | interchange VCD | portable sub-design stimulus
Simulation set / top / mode:
Source or DUT hierarchy:
Warm-up interval:
Measurement interval:
Signals or ports:
Expected functional result:
Artifact paths:
```

Require a self-checking test and bounded runtime.
자체 점검 테스트와 제한된 런타임이 필요합니다.
Activity artifacts supplement
활동 아티팩트 보충
the transcript; their existence is not functional PASS.
성적표; 그 존재는 기능적 PASS가 아닙니다.

Use `-debug typical` or the installed project property's equivalent because
`-debug traditional` 또는 설치된 프로젝트 속성과 동등한 것을 사용하십시오.
SAIF and hierarchy access depend on elaboration visibility.
SAIF 및 계층 구조 액세스는 정교화 가시성에 따라 달라집니다.
Query and preserve
쿼리 및 보존
existing properties:
기존 속성:

```tcl
set simset [get_filesets <simset>]
list_property $simset
get_property xsim.elaborate.debug_level $simset
get_property xsim.simulate.xsim.more_options $simset
```

Prefer a focused hierarchy and explicit objects.
집중된 계층 구조와 명시적인 개체를 선호합니다.
Recursive whole-design
재귀적 전체 디자인
activity capture can produce large files and slow simulation.
활동 캡처로 인해 대용량 파일이 생성되고 시뮬레이션이 느려질 수 있습니다.

## 2.
## SAIF power activity
## SAIF 전력 활동

Correctness stimulus and representative power stimulus are not automatically
정확성자극과 대표전력자극은 자동으로 발생하지 않음
the same.
동일합니다.
Choose a workload reflecting expected operating modes, duty cycles,
예상되는 작동 모드, 듀티 사이클,
data patterns, idle periods, and clock/reset behavior.
데이터 패턴, 유휴 기간 및 클럭/리셋 동작.
Record why it is
그 이유를 기록해 보세요.
representative.
대표.

Separate:
별도:

1. reset and initialization;
1. 재설정 및 초기화;
2. warm-up to stable operation;
2. 안정적인 작동을 위한 워밍업;
3. measured activity window;
3. 측정된 활동 창;
4. post-window functional checks.
4. 창 후 기능 점검.

For an exact measurement window, load the snapshot without auto-running,
정확한 측정 창을 위해 자동 실행 없이 스냅샷을 로드하고,
advance through warm-up, then capture only the intended scope:
워밍업을 통해 진행한 다음 의도한 범위만 캡처합니다.

```tcl
set_property xsim.simulate.runtime {} $simset
launch_simulation -simset $simset -mode behavioral

run <warm-up-duration>
open_saif <absolute-path>/activity.saif
log_saif [get_objects <dut-scope>/*]
run <measurement-duration>
close_saif

run <bounded-remainder>
```

`open_saif` must precede `log_saif`; `close_saif` must run before closing the
`open_saif`는 `log_saif`보다 앞에 와야 합니다. `close_saif`는 닫기 전에 실행되어야 합니다.
simulation.
시뮬레이션.
If the testbench calls `$finish` at the window boundary, preserve
테스트벤치가 창 경계에서 '$finish'를 호출하면 다음을 유지합니다.
`-onfinish stop` in `xsim.simulate.xsim.more_options` so the Tcl session can
`xsim.simulate.xsim.more_options`의 `-onfinish stop`을 사용하여 Tcl 세션이
close the SAIF handle and save other in-memory artifacts.
SAIF 핸들을 닫고 다른 메모리 내 아티팩트를 저장합니다.

For broad project-property capture, Vivado exposes:
광범위한 프로젝트 속성 캡처를 위해 Vivado는 다음을 노출합니다.

```tcl
set_property xsim.simulate.saif <absolute-path>/activity.saif $simset
set_property xsim.simulate.saif_scope <dut-hierarchy> $simset
```

Use property discovery for release-sensitive capitalization and verify the
릴리스에 민감한 대문자 사용을 위해 속성 검색을 사용하고
readback.
다시 읽어보기.
Prefer explicit Tcl windowing when warm-up must be excluded.
워밍업을 제외해야 하는 경우 명시적인 Tcl 기간 설정을 선호합니다.

### SAIF artifact checks
### SAIF 아티팩트 검사

Require:
요구 사항:

- file exists and is non-empty;
- 파일이 존재하며 비어 있지 않습니다.
- header and timescale are present;
- 헤더와 기간이 존재합니다.
- expected DUT hierarchy is present;
- 예상되는 DUT 계층 구조가 존재합니다.
- requested ports/signals have activity;
- 요청된 포트/신호에 활동이 있습니다.
- measurement start/end and workload are recorded;
- 측정 시작/종료 및 작업량이 기록됩니다.
- test transcript still satisfies the functional contract.
- 테스트 기록은 여전히 ​​기능 계약을 충족합니다.

The simulation skill may hand the SAIF to `read_saif`/`report_power`, but must
시뮬레이션 기술은 SAIF를 `read_saif`/`report_power`에 전달할 수 있지만 반드시
report hierarchy matching and unmatched nets.
계층 구조 일치 및 일치하지 않는 네트를 보고합니다.
Do not claim power signoff:
전원 사인오프를 요구하지 마십시오:
workload representativeness, netlist correlation, process/voltage/temperature,
워크로드 대표성, 넷리스트 상관관계, 프로세스/전압/온도,
clock activity, and the downstream power model remain separate evidence.
클럭 활동과 다운스트림 전력 모델은 별도의 증거로 남아 있습니다.

Post-implementation timing activity is preferable when realistic glitches are
현실적인 결함이 있는 경우 구현 후 타이밍 활동이 바람직합니다.
needed and the design supports that mode.
필요하며 디자인이 해당 모드를 지원합니다.
Vivado timing netlists are Verilog;
Vivado 타이밍 넷리스트는 Verilog입니다.
VHDL-only RTL activity has additional internal-signal limitations.
VHDL 전용 RTL 활동에는 추가적인 내부 신호 제한이 있습니다.

## 3.
## General VCD capture
## 일반 VCD 캡처

Use VCD for portable signal activity, external analysis, or exchange with a
휴대용 신호 활동, 외부 분석 또는 교환을 위해 VCD를 사용하십시오.
tool that does not read WDB.
WDB를 읽지 않는 도구입니다.
WDB remains preferable for native XSim debug.
기본 XSim 디버그에는 WDB가 여전히 선호됩니다.

Example bounded capture:
제한된 캡처 예:

```tcl
set_property xsim.simulate.runtime {} $simset
launch_simulation -simset $simset -mode behavioral

run <warm-up-duration>
open_vcd <absolute-path>/activity.vcd
log_vcd [get_objects {<clock> <reset> <controls> <data> <outputs>}]
run <measurement-duration>
close_vcd

run <bounded-remainder>
```

Use `start_vcd`, `stop_vcd`, `flush_vcd`, `checkpoint_vcd`, or `limit_vcd`
`start_vcd`, `stop_vcd`, `flush_vcd`, `checkpoint_vcd` 또는 `limit_vcd`를 사용하세요.
only when needed for a long capture.
긴 캡처에 필요한 경우에만.
Always close the handle.
항상 손잡이를 닫으십시오.
Do not assume
가정하지 마십시오
objects in the Wave window are automatically in the VCD; pass them explicitly
Wave 창의 개체는 자동으로 VCD에 포함됩니다. 명시적으로 전달하세요
to `log_vcd`.
`log_vcd`로.

### VCD artifact checks
### VCD 아티팩트 검사

Parse or inspect:
구문 분석 또는 검사:

- `$date`, `$version`, `$timescale`, scopes, and variable declarations;
- `$date`, `$version`, `$timescale`, 범위 및 변수 선언;
- exact requested object set;
- 정확한 요청 객체 세트;
- first and last timestamps;
- 첫 번째 및 마지막 타임스탬프
- expected transitions and final values;
- 예상 전환 및 최종 값;
- absence of unintended recursive hierarchy when focused capture was required.
- 집중 캡처가 필요할 때 의도하지 않은 재귀 계층 구조가 없습니다.

Verify values against the test contract and transcript.
테스트 계약 및 성적표와 비교하여 값을 확인합니다.
A syntactically valid
구문적으로 유효한
VCD can still describe a failing simulation.
VCD는 여전히 실패한 시뮬레이션을 설명할 수 있습니다.

## 4.
## Portable sub-design stimulus capture
## 휴대용 하위 설계 자극 캡처

Use this only when a working parent simulation can exercise a selected
작업 중인 상위 시뮬레이션이 선택한 항목을 실행할 수 있는 경우에만 이 옵션을 사용하십시오.
sub-design and the goal is to reproduce its port activity independently.
하위 설계이며 목표는 포트 활동을 독립적으로 재현하는 것입니다.
It is not a replacement for deriving tests from requirements.
이는 요구사항에서 테스트를 도출하는 것을 대체하지 않습니다.

### Capture source behavior
### 소스 동작 캡처

1. Run the original self-checking simulation and prove its verdict.
1. 원래의 자체 점검 시뮬레이션을 실행하고 그 결과를 증명하십시오.
2. Select the exact hierarchical instance.
2. 정확한 계층적 인스턴스를 선택합니다.
3. Start port capture before the behavior of interest.
3. 관심 있는 동작이 나타나기 전에 포트 캡처를 시작하십시오.
4. Run a bounded interval.
4. 제한된 간격을 실행합니다.
5. Close the special port VCD handle.
5. 특수 포트 VCD 핸들을 닫습니다.

```tcl
launch_simulation -simset <source-simset> -mode behavioral
generate_vcd_ports {/<source-top>/<selected-instance>}
run <capture-duration>
close_vcd -ports
```

Vivado writes a port-activity VCD in the active simulation run location.
Vivado는 활성 시뮬레이션 실행 위치에 포트 활동 VCD를 작성합니다.
Discover the returned/generated path; do not assume a working-directory name.
반환/생성된 경로를 검색합니다. 작업 디렉토리 이름을 가정하지 마십시오.

### Generate an isolated testbench
### 격리된 테스트벤치 생성

Create a new simulation set so the source test remains unchanged:
소스 테스트가 변경되지 않도록 새 시뮬레이션 세트를 만듭니다.

```tcl
create_testbench \
  -name <generated-top> \
  -add_to_simset <new-simset> \
  -set_as_top
update_compile_order -fileset [get_filesets <new-simset>]
```

`create_testbench` generates a simulator-independent SystemVerilog wrapper,
`create_testbench`는 시뮬레이터 독립적인 SystemVerilog 래퍼를 생성합니다.
DUT instantiation, declarations, and stimulus include based on the captured
캡처된 데이터를 기반으로 DUT 인스턴스화, 선언 및 자극이 포함됩니다.
ports.
포트.
Discover and preserve every generated file.
생성된 모든 파일을 검색하고 보존하세요.

Before launch, generate or inspect the replay's compile/elaborate command and
시작하기 전에 리플레이의 컴파일/정교 명령을 생성하거나 검사하고
compare every injected option with the installed `xelab -help`.
삽입된 모든 옵션을 설치된 `xelab -help`와 비교하세요.
In a Vivado
비바도에서
2026.1 smoke test, `create_testbench` emitted the obsolete
2026.1 스모크 테스트, 'create_testbench'에서 더 이상 사용되지 않는 항목을 방출함
`-ignore_localparam_override` option, which that release rejects.
해당 릴리스에서는 거부되는 `-ignore_localparam_override` 옵션입니다.
If observed:
관찰된 경우:

1. preserve the original generated script and failure log;
1. 원래 생성된 스크립트와 실패 로그를 보존합니다.
2. confirm that installed `xelab -help` exposes
2. 설치된 `xelab -help`가 노출되는지 확인하세요.
`-suppress_localparam_override_error`;
`-suppress_localparam_override_error`;
3. replace only the obsolete option in the generated replay script;
3. 생성된 재생 스크립트에서 사용되지 않는 옵션만 교체합니다.
4. rerun elaboration and simulation;
4. 정교화 및 시뮬레이션을 다시 실행합니다.
5. report the workaround as generated-flow remediation, not a DUT change.
5. 해결 방법을 DUT 변경이 아닌 생성된 흐름 교정으로 보고하십시오.

Do not blindly apply this substitution to other releases.
이 대체물을 다른 릴리스에 맹목적으로 적용하지 마십시오.

### Convert replay into verification
### 재생을 검증으로 전환

Treat the generated output as replay scaffolding until all are added:
모두 추가될 때까지 생성된 출력을 재생 스캐폴딩으로 처리합니다.

- checks for expected output values/events;
- 예상되는 출력 값/이벤트를 확인합니다.
- protocol legality checks where applicable;
- 해당되는 경우 프로토콜 합법성 확인;
- watchdog and bounded completion;
- 워치독 및 제한된 완료;
- exactly one terminal PASS marker;
- 정확히 하나의 터미널 PASS 마커;
- failure messages with expected/observed values and time;
- 예상/관찰된 값과 시간이 포함된 실패 메시지
- documented assumptions about clocks, reset, bidirectional ports, unknowns,
- 시계, 재설정, 양방향 포트, 알 수 없는 항목에 대한 문서화된 가정
timing, parameters, and external models.
타이밍, 매개변수 및 외부 모델.

Do not overwrite the source testbench.
소스 테스트벤치를 덮어쓰지 마십시오.
Make verification-only edits in the new
새 항목에서 확인 전용 편집을 수행합니다.
simulation set, launch it independently, and compare source versus replay:
시뮬레이션 세트를 독립적으로 실행하고 소스와 재생을 비교합니다.

```text
Captured interval:
Input sequence:
Expected output sequence/final state:
Source verdict:
Replay verdict:
Differences:
```

Call the artifact portable only after it runs successfully with the intended
의도한 대로 성공적으로 실행된 후에만 이식 가능한 아티팩트를 호출하십시오.
simulator flow.
시뮬레이터 흐름.
Port-level replay does not prove portability across language
포트 수준 재생은 언어 간 이식성을 입증하지 않습니다.
semantics, timing models, encrypted IP, or unsupported simulators.
의미 체계, 타이밍 모델, 암호화된 IP 또는 지원되지 않는 시뮬레이터.

## 5.
## Result report
## 결과보고

```text
Capture purpose:
Backend / mode / top:
Source scope and objects:
Warm-up and measurement interval:
Functional verdict:
SAIF path / size / hierarchy:
VCD path / size / declared objects / time range:
Generated testbench and stimulus files:
Source versus replay result:
Downstream power match (if run):
Limitations / unverified assumptions:
```

## Official sources
## 공식 출처

- [UG900 SAIF dumping](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Dumping-SAIF-for-Power-Analysis)
- [UG900 SAIF 덤핑](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Dumping-SAIF-for-Power-Analytic)
- [UG900 VCD dumping](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Dumping-VCD-for-Simulation)
- [UG900 VCD 덤핑](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Dumping-VCD-for-Simulation)
- [UG900 generate_vcd_ports](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/generate_vcd_ports)
- [UG900 generate_vcd_ports](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/generate_vcd_ports)
- [UG900 create_testbench](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/create_testbench)
- [UG900 create_testbench](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/create_testbench)
- [UG900 power analysis using XSim](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Power-Analysis-Using-Vivado-Simulator)
- [Xsim을 사용한 UG900 전력 분석](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Power-Analytic-Using-Vivado-Simulator)
