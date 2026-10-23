<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# XSim Advanced Verification
# XSim 고급 검증

Use this reference for UVM, constrained-random regressions, SystemVerilog
UVM, 제한된 무작위 회귀, SystemVerilog에 이 참조를 사용하십시오.
assertions, and functional coverage.
주장 및 기능적 적용 범위.
Use
사용
[xsim-coverage-closure.md](xsim-coverage-closure.md) for the shared coverage
공유 커버리지의 경우 [xsim-coverage-closure.md](xsim-coverage-closure.md)
database and closure workflow.
데이터베이스 및 클로저 워크플로우.

## Contents
## 목차

- Preflight
- 비행 전
- UVM
- UVM
- Seeded random regression
- 시드된 무작위 회귀
- Assertion-aware verdicts
- 주장 인식 판정
- Functional coverage
- 기능적 범위
- Run manifests and merging
- 매니페스트 실행 및 병합

## 1.
## Preflight
## 비행 전

Record:
기록:

```text
Vivado/XSim version:
Simulation set and top:
UVM version and selected test:
Seed list and plusargs:
Assertion constructs and expected failures:
Functional covergroups and closure targets:
Runtime/watchdog:
Artifact root:
```

Query the installed release before adding options:
옵션을 추가하기 전에 설치된 릴리스를 쿼리하십시오.

```tcl
set simset [get_filesets <simset>]
list_property $simset
get_property xsim.compile.xvlog.more_options $simset
get_property xsim.elaborate.xelab.more_options $simset
get_property xsim.simulate.xsim.more_options $simset
```

XSim implements subsets of SystemVerilog assertions, constraints, and
XSim은 SystemVerilog 어설션, 제약 조건 및
functional coverage.
기능적 범위.
Check the current UG900 **Test Bench Feature** table when
현재 UG900 **테스트 벤치 기능** 표를 확인하세요.
a construct fails analysis or elaboration.
구성이 분석이나 정교화에 실패합니다.
Do not rewrite an unsupported
지원되지 않는 것을 다시 작성하지 마십시오
checker into a weaker checker merely to make it compile.
단지 컴파일하기 위해 검사기를 더 약한 검사기로 바꾸는 것입니다.

Reject options that hide verification evidence:
확인 증거를 숨기는 옵션을 거부합니다.

- `-ignore_assertions`
- `-ignore_assertions`
- `-ignore_coverage`
- `-ignore_coverage`
- `-ignore_feature assertion`
- `-ignore_feature 어설션`
- `-ignore_feature coverage`
- `-ignore_feature 보도`
- error/fatal severity downgrade options
- 오류/치명적 심각도 다운그레이드 옵션

If the project already contains any of these, report them and classify the run
프로젝트에 이미 이러한 항목이 포함된 경우 이를 보고하고 실행을 분류하세요.
as `INCONCLUSIVE` unless the verification plan explicitly requires them.
검증 계획에서 명시적으로 요구하지 않는 한 'INCONCLUSIVE'로 처리됩니다.

## 2.
## UVM
## UVM

Vivado-integrated XSim provides a precompiled UVM 1.2 library by default.
Vivado 통합 XSim은 기본적으로 사전 컴파일된 UVM 1.2 라이브러리를 제공합니다.
Do
not install or compile a second UVM library for an integrated project run.
통합 프로젝트 실행을 위해 두 번째 UVM 라이브러리를 설치하거나 컴파일하지 마십시오.

For standalone XSim, pass `-L uvm` to both `xvlog` and `xelab`.
독립 실행형 XSim의 경우 `-L uvm`을 `xvlog`와 `xelab`에 모두 전달합니다.
UVM 1.1 is
UVM 1.1은
available only when explicitly required; pass `-uvm_version 1.1` during
명시적으로 필요한 경우에만 사용할 수 있습니다. 도중 `-uvm_version 1.1`을 전달합니다.
compile and elaboration.
컴파일하고 정교하게 만듭니다.
Preserve existing options when setting project
프로젝트 설정 시 기존 옵션 유지
properties:
속성:

```tcl
set copt [get_property xsim.compile.xvlog.more_options $simset]
set eopt [get_property xsim.elaborate.xelab.more_options $simset]
set_property -name XSIM.COMPILE.XVLOG.MORE_OPTIONS \
  -value [string trim "$copt -uvm_version 1.1"] -objects $simset
set_property -name XSIM.ELABORATE.XELAB.MORE_OPTIONS \
  -value [string trim "$eopt -uvm_version 1.1"] -objects $simset
```

Do not add the 1.1 switches for the normal 1.2 flow.
일반 1.2 흐름에는 1.1 스위치를 추가하지 마십시오.

Select one test explicitly in each run.
각 실행에서 명시적으로 하나의 테스트를 선택합니다.
XSim exposes simulator plusargs through
XSim은 다음을 통해 시뮬레이터 plusargs를 노출합니다.
`-testplusarg`; verify the installed syntax with `xsim -help` and preserve the
`-testplusarg`; `xsim -help`를 사용하여 설치된 구문을 확인하고
existing simulation options.
기존 시뮬레이션 옵션.
The effective argument must be visible to UVM as
유효 인수는 UVM에 다음과 같이 표시되어야 합니다.
`+UVM_TESTNAME=<test>`.
`+UVM_TESTNAME=<테스트>`.

A UVM PASS requires:
UVM 패스에는 다음이 필요합니다.

1. the intended test name appears in the transcript;
1. 의도한 테스트 이름이 성적표에 나타납니다.
2. every required phase and scoreboard completes;
2. 모든 필수 단계와 점수판이 완료됩니다.
3. the explicit terminal PASS contract appears;
3. 명시적인 터미널 PASS 계약이 나타납니다.
4. the UVM report summary has zero `UVM_ERROR` and zero `UVM_FATAL`;
4. UVM 보고서 요약에는 'UVM_ERROR'이 0이고 'UVM_FATAL'이 0입니다.
5. no unexpected HDL assertion failure or timeout occurs.
5. 예기치 않은 HDL 어설션 실패나 시간 초과가 발생하지 않습니다.

Make the terminal scoreboard message include the expected and observed final
터미널 점수판 메시지에 예상 및 관찰된 최종 결과가 포함되도록 만듭니다.
state; a generic "scoreboard passed" string is weaker evidence.
상태; 일반적인 "점수판 통과" 문자열은 약한 증거입니다.

`UVM_INFO` output, dropped objections, or normal simulator exit alone are not
`UVM_INFO` 출력, 이의제기 삭제 또는 일반 시뮬레이터 종료만으로는 문제가 되지 않습니다.
PASS.
통과.
Preserve the complete transcript because an early UVM error can precede
초기 UVM 오류가 선행될 수 있으므로 전체 기록을 보존합니다.
a clean-looking final message.
깔끔한 최종 메시지.

## 3.
## Seeded random regression
## 시드된 무작위 회귀

Use a declared finite seed list.
선언된 유한 시드 목록을 사용하십시오.
Deterministic default regressions and wider
결정론적 기본 회귀 및 더 넓은 범위
random exploration are different jobs; do not silently replace one with the
무작위 탐색은 다른 직업입니다. 자동으로 하나를 다음으로 대체하지 마십시오.
other.
기타.

For each seed:
각 씨앗에 대해:

1. create a unique run directory and manifest;
1. 고유한 실행 디렉터리와 매니페스트를 생성합니다.
2. preserve existing XSim options;
2. 기존 XSim 옵션을 유지합니다.
3. set exactly one `-sv_seed <integer>`;
3. 정확히 하나의 `-sv_seed <integer>`를 설정합니다.
4. set the selected test and other plusargs;
4. 선택한 테스트 및 기타 plusargs를 설정합니다.
5. close the prior XSim process and launch a fresh run with the same elaborated
5. 이전 XSim 프로세스를 닫고 동일한 정교한 작업으로 새로 실행을 시작합니다.
design and bounded runtime;
디자인 및 제한된 런타임;
6. classify the transcript and assertion/UVM summary;
6. 성적표와 주장/UVM 요약을 분류합니다.
7. persist waveform and coverage before closing;
7. 닫기 전에 파형 및 적용 범위를 유지합니다.
8. never reuse the run directory for a retry.
8. 재시도를 위해 실행 디렉터리를 재사용하지 마세요.

When regression coverage is requested, enable a uniquely named database for
회귀 적용 범위가 요청되면 고유하게 명명된 데이터베이스를 활성화합니다.
each passing seed.
지나가는 씨앗 하나하나.
Record its exact path and byte size.
정확한 경로와 바이트 크기를 기록하십시오.
Failed-seed coverage is
실패한 시드 적용 범위는 다음과 같습니다.
diagnostic only and stays out of the passing merge.
진단 전용이며 통과 병합에서 벗어나 있습니다.

Example option construction:
옵션 구성 예시:

```tcl
set old [get_property xsim.simulate.xsim.more_options $simset]
regsub -all {(^|[[:space:]])-sv_seed[[:space:]]+[^[:space:]]+} \
  $old {} cleaned
set runopts [string trim \
  "$cleaned -sv_seed <seed> -testplusarg UVM_TESTNAME=<test>"]
set_property -name XSIM.SIMULATE.XSIM.MORE_OPTIONS \
  -value $runopts -objects $simset
catch {close_sim -force}
launch_simulation -simset $simset -mode behavioral
```

Restore the original property after the regression.
회귀 후에 원래 속성을 복원합니다.
Do not accumulate stale
오래된 것을 축적하지 마십시오
seeds or test names across runs.
실행 전반에 걸쳐 시드 또는 테스트 이름.
Do not rely on `relaunch_sim` alone after
이후 `relaunch_sim`에만 의존하지 마세요.
changing `-sv_seed`; a live simulator can retain its prior runtime options.
`-sv_seed` 변경; 라이브 시뮬레이터는 이전 런타임 옵션을 유지할 수 있습니다.
Confirm the effective seed in the generated XSim command and run manifest.
생성된 XSim 명령에서 유효한 시드를 확인하고 매니페스트를 실행합니다.

On failure:
실패 시:

- preserve the failing transcript and focused WDB separately;
- 실패한 기록과 집중된 WDB를 별도로 보존합니다.
- do not merge its coverage into the passing-regression signoff database;
- 해당 적용 범위를 통과-회귀 사인오프 데이터베이스에 병합하지 마십시오.
- rerun the exact test, seed, plusargs, sources, parameters, and XSim version;
- 정확한 테스트, 시드, plusargs, 소스, 매개변수 및 XSim 버전을 다시 실행합니다.
- require the same random choice and first causal failure;
- 동일한 무작위 선택과 첫 번째 인과 실패가 필요합니다.
- call the failure reproducible only after that replay.
- 해당 재생 후에만 오류 재현 가능을 호출합니다.

## 4.
## Assertion-aware verification
## 어설션 인식 검증

Inventory immediate assertions, concurrent assertions, and UVM/reporting
인벤토리 즉시 주장, 동시 주장 및 UVM/보고
checks separately.
별도로 확인합니다.
Record which failures are expected by a negative test.
부정적인 테스트로 예상되는 실패를 기록합니다.

When concurrent assertion pass accounting is requested, preserve the existing
동시 어설션 통과 계산이 요청되면 기존을 유지합니다.
elaboration options and add:
정교화 옵션을 선택하고 다음을 추가하세요.

```tcl
set eopt [get_property xsim.elaborate.xelab.more_options $simset]
set_property -name XSIM.ELABORATE.XELAB.MORE_OPTIONS \
  -value [string trim "$eopt -report_assertion_pass"] -objects $simset
```

Remove or restore the added option after the run.
실행 후 추가된 옵션을 제거하거나 복원하세요.
Re-elaborate after changing
변경 후 다시 작성
elaboration options; a restart of the old snapshot is insufficient.
정교화 옵션; 이전 스냅샷을 다시 시작하는 것만으로는 충분하지 않습니다.

Read the complete transcript and report:
전체 내용을 읽고 보고하세요.

```text
Assertion label / source:
Kind: immediate | concurrent | UVM/checker
Status: pass | fail | disabled | unsupported | not observed
First failure time:
Expected by test plan: yes | no
Message and sampled context:
```

Persist this classification as an assertion-summary artifact and list the
이 분류를 주장 요약 아티팩트로 유지하고
transcript and focused WDB paths with byte sizes.
바이트 크기가 포함된 사본 및 집중 WDB 경로.

An expected negative-test assertion is evidence that the checker fired, but
예상된 부정 테스트 주장은 검사기가 실행되었다는 증거이지만
the simulation's functional verdict is still `EXPECTED_FAIL`, not ordinary
시뮬레이션의 기능적 판정은 여전히 평범하지 않은 'EXPECTED_FAIL'입니다.
PASS.
통과.
Unexpected assertion failure is `FAIL`.
예상치 못한 어설션 실패는 'FAIL'입니다.
A compile-time unsupported
지원되지 않는 컴파일 타임
construct is an environment/capability result, not an assertion pass.
구성은 어설션 전달이 아닌 환경/기능 결과입니다.

Never:
절대:

- infer all assertions passed from the absence of printed pass messages;
- 인쇄된 전달 메시지가 없으면 전달된 모든 주장을 추론합니다.
- count a disabled or vacuous property as exercised without explicit evidence;
- 명시적인 증거 없이 장애가 있거나 비어 있는 재산을 행사한 것으로 간주합니다.
- downgrade severity or ignore assertions to continue a regression;
- 회귀를 계속하려면 심각도를 낮추거나 어설션을 무시하세요.
- edit the DUT when the user requested diagnosis only.
- 사용자가 진단만 요청한 경우 DUT를 편집합니다.

## 5.
## Functional coverage
## 기능적 범위

Functional coverage is test-plan coverage, not source-code coverage.
기능적 적용 범위는 소스 코드 적용 범위가 아닌 테스트 계획 적용 범위입니다.
Define
정의
covergroups, coverpoints, bins, transitions, and crosses from requirements.
커버그룹, 커버포인트, 저장소, 전환 및 요구사항 교차.
Record the exact target bins before changing stimulus.
자극을 변경하기 전에 정확한 목표 상자를 기록하십시오.

XSim creates a coverage database when functional coverage is present.
XSim은 기능적 적용 범위가 있을 때 적용 범위 데이터베이스를 생성합니다.
Use
사용
unique `-cov_db_dir` and `-cov_db_name` values, or their installed project
고유한 `-cov_db_dir` 및 `-cov_db_name` 값 또는 설치된 프로젝트
property equivalents, for every run.
모든 실행에 대해 등가물 속성.
`write_xsim_coverage` and
`write_xsim_coverage` 및
`export_xsim_coverage` persist and report both functional and code coverage.
`export_xsim_coverage`는 기능과 코드 적용 범위를 모두 유지하고 보고합니다.

For each uncovered target:
발견된 각 대상에 대해 다음을 수행합니다.

```text
Covergroup / instance:
Coverpoint or cross:
Uncovered bin:
Requirement:
Legal stimulus:
Observable checker:
Reachability:
```

Closure loop:
폐쇄 루프:

1. run the unchanged self-checking baseline;
1. 변경되지 않은 자체 점검 기준선을 실행합니다.
2. save and export text plus HTML coverage;
2. 텍스트와 HTML 적용 범위를 저장하고 내보냅니다.
3. distinguish an uncovered bin from an unsupported construct;
3. 보호되지 않은 빈과 지원되지 않는 구성을 구별합니다.
4. add legal focused stimulus and an observable check;
4. 법적 초점을 맞춘 자극과 관찰 가능한 점검을 추가합니다.
5. run the focused test and full deterministic regression;
5. 집중 테스트와 전체 결정론적 회귀를 실행합니다.
6. save each passing run under a unique database;
6. 각각의 통과 실행을 고유한 데이터베이스에 저장합니다.
7. merge compatible passing runs;
7. 호환 가능한 통과 실행을 병합합니다.
8. compare targeted bin counts, not only aggregate percentages.
8. 집계 백분율뿐만 아니라 대상 빈 개수도 비교합니다.

Do not add meaningless samples, relax bin definitions, remove illegal bins, or
의미 없는 샘플을 추가하거나, 빈 정의를 완화하거나, 불법 빈을 제거하거나,
edit DUT behavior solely to inflate functional coverage.
기능적 적용 범위를 확대하기 위해서만 DUT 동작을 편집합니다.

## 6.
## Manifests and compatible merges
## 매니페스트 및 호환 가능한 병합

Write one machine-readable or plain-text manifest per run:
실행당 하나의 기계 판독 가능 또는 일반 텍스트 매니페스트를 작성합니다.

```text
run_id:
source_revision_or_hash:
vivado_xsim_version:
simulation_set:
top_and_parameters:
test:
seed:
plusargs:
mode:
runtime_and_watchdog:
expected_verdict:
actual_verdict:
first_failure:
transcript:
waveform:
coverage_database:
```

Add an artifact inventory with exact paths and byte sizes for binary WDB and
바이너리 WDB에 대한 정확한 경로와 바이트 크기가 포함된 아티팩트 인벤토리를 추가하고
coverage files; text-search tools may not enumerate binary formats reliably.
적용 범위 파일; 텍스트 검색 도구는 바이너리 형식을 안정적으로 열거하지 못할 수 있습니다.

Merge only runs with the same source revision, elaborated top/parameters,
병합은 동일한 소스 개정, 정교한 상단/매개변수,
coverage schema, XSim version, and applicable coverage types.
적용 범위 스키마, XSim 버전 및 적용 가능한 적용 범위 유형.
Every merged run
모든 병합 실행
must satisfy its expected functional contract.
기대되는 기능 계약을 충족해야 합니다.
Keep negative tests and failed
부정적인 테스트를 유지하고 실패했습니다.
runs available for diagnosis but outside the passing-regression database.
진단에는 사용할 수 있지만 통과 회귀 데이터베이스 외부에서 실행됩니다.

The final report must include:
최종 보고서에는 다음이 포함되어야 합니다.

```text
Tests and seeds:
UVM errors / fatals:
Assertions passed / failed / disabled / unsupported:
Functional coverage targets before -> after:
Code coverage before -> after:
Merged run IDs:
Excluded failed or incompatible run IDs and reasons:
Replayed failure result:
Artifacts:
```

## Official sources
## 공식 출처

- [UG900 UVM support](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Universal-Verification-Methodology-UVM-Support)
- [UG900 UVM 지원](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Universal-Verification-Methodology-UVM-Support)
- [UG900 SystemVerilog testbench features](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Test-Bench-Feature)
- [UG900 SystemVerilog 테스트벤치 기능](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Test-Bench-Feature)
- [UG900 xsim executable options](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/xsim-Executable-Options)
- [UG900 xsim 실행 가능 옵션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/xsim-Executable-Options)
- [UG900 export_xsim_coverage](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_xsim_coverage)
- [UG900 내보내기_xsim_coverage](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_xsim_coverage)
- [UG900 write_xsim_coverage](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/write_xsim_coverage)
- [UG900 write_xsim_coverage](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/write_xsim_coverage)
