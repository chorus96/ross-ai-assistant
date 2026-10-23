<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# XSim Coverage Closure
# XSim 적용 범위 폐쇄

Use this workflow to improve code and functional coverage without weakening
이 워크플로를 사용하여 코드를 약화시키지 않고 코드 및 기능 적용 범위를 개선하세요.
verification.
확인.
XSim supports statement, branch, condition, and toggle code
XSim은 명령문, 분기, 조건 및 토글 코드를 지원합니다.
coverage plus SystemVerilog covergroups, coverpoints, bins, transitions, and
커버리지와 SystemVerilog 커버그룹, 커버포인트, 빈, 전환 및
crosses.
십자가.
Read [xsim-advanced-verification.md](xsim-advanced-verification.md)
[xsim-advanced-verification.md](xsim-advanced-verification.md) 읽기
for UVM, seeded regressions, and assertion-aware verdicts.
UVM, 시드 회귀 및 어설션 인식 판정의 경우.

## Contents
## 목차

- Define the goal
- 목표를 정의하세요
- Capture a baseline
- 기준선 캡처
- Read and classify gaps
- 격차를 읽고 분류하세요.
- Write focused tests
- 집중 테스트 작성
- Iterate and merge
- 반복 및 병합
- Close functional coverage
- 기능 적용 범위 닫기
- Stop and report
- 중지하고 신고하세요.
- Integrity rules
- 무결성 규칙

## 1.
## Define the goal
## 목표를 정의하세요

Record before running:
실행 전 기록:

- DUT hierarchy and source scope
- DUT 계층 구조 및 소스 범위
- required code-coverage types
- 필수 코드 적용 범위 유형
- functional covergroups, points, crosses, and target bins
- 기능적 커버그룹, 포인트, 크로스, 타겟 빈
- target metric or closure criterion for each coverage model
- 각 적용 범위 모델에 대한 목표 지표 또는 마감 기준
- applicable tests and required PASS markers
- 적용 가능한 테스트 및 필수 PASS 마커
- permitted testbench files
- 허용된 테스트벤치 파일
- exclusions already approved by the verification plan
- 검증 계획에 의해 이미 승인된 제외

Coverage percentage is not the contract.
보장 비율은 계약이 아닙니다.
Functional coverage must trace to the
기능적 적용 범위는 다음을 추적해야 합니다.
test plan; code coverage traces to exercised implementation structure.
테스트 계획; 코드 적용 범위는 실행된 구현 구조를 추적합니다.
Tie
넥타이
every new test to reachable behavior and an observable expected result.
도달 가능한 동작과 관찰 가능한 예상 결과에 대한 모든 새로운 테스트.

## 2.
## Capture a baseline
## 기준선 캡처

Preserve the existing regression and baseline testbench files unchanged.
기존 회귀 및 기준 테스트벤치 파일을 변경하지 않고 유지합니다.
Assign unique names so each run is auditable:
각 실행을 감사할 수 있도록 고유한 이름을 할당합니다.

```tcl
set simset [get_filesets <simset>]
set_property target_simulator XSim [current_project]
set_property xsim.elaborate.coverage.name baseline $simset
set_property xsim.elaborate.coverage.dir <absolute-artifact-dir>/baseline $simset
set_property xsim.elaborate.coverage.type sbct $simset
set xopts [get_property xsim.simulate.xsim.more_options $simset]
regsub -all {(^|[[:space:]])-onfinish[[:space:]]+(quit|stop)} $xopts {} xopts
set new_xopts [string trim "$xopts -onfinish stop"]
set_property -name XSIM.SIMULATE.XSIM.MORE_OPTIONS -value $new_xopts -objects $simset
catch {close_sim -force}
launch_simulation -simset $simset -mode behavioral
```

Before closing the live simulation, persist the in-memory coverage:
실시간 시뮬레이션을 닫기 전에 인메모리 적용 범위를 유지하십시오.

```tcl
file mkdir <absolute-artifact-dir>/baseline-report
write_xsim_coverage \
  -cov_db_name baseline \
  -cov_db_dir <absolute-artifact-dir>/baseline

export_xsim_coverage \
  -cov_db_name baseline \
  -cov_db_dir <absolute-artifact-dir>/baseline \
  -output_dir <absolute-artifact-dir>/baseline-report \
  -report_format all \
  -open_html false
```

Use `sbct` for statement, branch, condition, and toggle coverage.
명령문, 분기, 조건 및 토글 적용 범위에는 'sbct'를 사용하세요.
If the
만약
project accepts descriptive values, `all` is equivalent.
프로젝트는 설명 값을 허용하며 `all`은 동일합니다.
Query the installed
설치된 쿼리
release before setting it:
설정하기 전에 릴리스하십시오.

```tcl
list_property_value XSIM.ELABORATE.COVERAGE.TYPE $simset
```

Some releases return an empty list for this string property.
일부 릴리스에서는 이 문자열 속성에 대해 빈 목록을 반환합니다.
In that case use
그런 경우에는 사용
the documented `sbct`, set it, and verify the readback with
문서화된 `sbct`를 설정하고 다음을 사용하여 리드백을 확인합니다.
`get_property XSIM.ELABORATE.COVERAGE.TYPE $simset`.
`get_property XSIM.ELABORATE.COVERAGE.TYPE $simset`.

`write_xsim_coverage` must run while the simulation session is active.
시뮬레이션 세션이 활성화된 동안 `write_xsim_coverage`를 실행해야 합니다.
Closing
폐회
first discards unsaved in-memory data.
먼저 저장되지 않은 메모리 내 데이터를 삭제합니다.
In Vivado's integrated XSim flow, a
Vivado의 통합 XSim 흐름에서는
testbench `$finish` ends runtime but leaves the simulation snapshot loaded, so
테스트벤치 `$finish`는 런타임을 종료하지만 시뮬레이션 스냅샷은 로드된 상태로 둡니다.
write coverage immediately after `launch_simulation` returns and before
`launch_simulation` 반환 직후와 그 이전에 적용 범위를 작성하세요.
`close_sim`.
'close_sim'.
XSim documents `-onfinish stop` as the explicit mechanism; preserve
XSim은 `-onfinish stop`을 명시적인 메커니즘으로 문서화합니다. 보존하다
any existing `xsim.simulate.xsim.more_options` when adding it rather than
기존 `xsim.simulate.xsim.more_options`를 추가하는 대신
blindly replacing project options.
맹목적으로 프로젝트 옵션을 교체합니다.

Create every coverage database and report parent directory before invoking
호출하기 전에 모든 적용 범위 데이터베이스를 생성하고 상위 디렉터리를 보고합니다.
XSim/export.
XSim/내보내기.
`export_xsim_coverage` can print final scores even after failing to
`export_xsim_coverage`는 실패한 후에도 최종 점수를 인쇄할 수 있습니다.
create a missing output directory, so require the report files to exist.
누락된 출력 디렉터리를 생성하므로 보고서 파일이 있어야 합니다.

Require the baseline regression to pass.
통과하려면 기준 회귀가 필요합니다.
Do not optimize coverage from a
다음에서 적용 범위를 최적화하지 마십시오.
failing baseline until the failure is classified and resolved.
실패가 분류되고 해결될 때까지 실패 기준선.

## 3.
## Read and classify gaps
## 격차를 읽고 분류하세요.

Read `<output-dir>/codeCoverageReport/dashboard.txt` for aggregate code
집계 코드는 `<output-dir>/codeCoverageReport/dashboard.txt`를 읽어보세요.
metrics and the functional-coverage text/HTML sections for covergroup,
커버 그룹에 대한 지표 및 기능 범위 텍스트/HTML 섹션,
coverpoint, cross, and bin counts.
커버포인트, 크로스, 빈 카운트.
Use per-file reports for source-level code
소스 수준 코드에 대한 파일별 보고서 사용
inspection.
검사.
The code aggregate includes testbench and support modules unless
코드 집계에는 테스트벤치 및 지원 모듈이 포함됩니다.
scope is constrained; report DUT-scoped metrics rather than improving a score
범위가 제한되어 있습니다. 점수를 향상시키는 대신 DUT 범위의 측정항목을 보고합니다.
with testbench-only activity.
테스트벤치 전용 활동으로.
For each material uncovered item, record:
발견되지 않은 각 항목에 대해 다음을 기록하십시오.

```text
Source / hierarchy:
Coverage type:
Uncovered line, branch, condition outcome, or toggle:
Reachability: reachable | unreachable | elaborated-out | unknown
Requirement:
Stimulus needed:
Observable check:
```

For functional coverage, also record:
기능적 적용 범위에 대해서는 다음 사항도 기록하십시오.

```text
Covergroup instance:
Coverpoint or cross:
Target bin:
Current hit count:
Test-plan requirement:
```

Prioritize:
우선순위:

1. reachable safety/error handling and protocol corner cases;
1. 접근 가능한 안전/오류 처리 및 프로토콜 코너 케이스
2. reachable branches and condition outcomes tied to requirements;
2. 요구 사항에 연결된 도달 가능한 분기 및 조건 결과
3. reset, enable, backpressure, overflow/underflow, saturation, and recovery;
3. 재설정, 활성화, 배압, 오버플로/언더플로, 포화 및 복구;
4. meaningful control/status toggles;
4. 의미 있는 제어/상태 전환;
5. incidental datapath toggles.
5. 부수적인 데이터 경로 토글.

Do not infer reachability solely from a zero count.
0개 개수만으로 연결 가능성을 추론하지 마세요.
Check parameters,
매개변수를 확인하고,
generates, constants, protocol legality, and design requirements.
생성, 상수, 프로토콜 합법성 및 설계 요구 사항.

## 4.
## Write focused self-checking tests
## 집중적인 자체 점검 테스트 작성

Reuse the existing verification infrastructure, but do not overwrite the
기존 검증 인프라를 재사용하되 덮어쓰지 마십시오.
baseline testbench.
베이스라인 테스트벤치.
Add a new test case, sequence, or testbench file to the
새 테스트 케이스, 시퀀스 또는 테스트벤치 파일을
simulation set so baseline behavior remains reproducible.
기본 동작이 재현 가능하도록 시뮬레이션 세트.

When independent testbench tops are required, give each one a simulation set:
독립적인 테스트벤치 상단이 필요한 경우 각각에 시뮬레이션 세트를 제공하십시오.

```tcl
create_fileset -simset sim_cov_01
add_files -fileset sim_cov_01 <absolute-new-testbench-path>
set_property top <new-testbench-top> [get_filesets sim_cov_01]
update_compile_order -fileset sim_cov_01
```

Run each applicable simulation set separately, persist its uniquely named
적용 가능한 각 시뮬레이션 세트를 별도로 실행하고 고유한 이름을 유지합니다.
coverage database, require functional PASS, then merge the compatible
적용 범위 데이터베이스, 기능적 PASS가 필요하고 호환 가능한 데이터베이스를 병합합니다.
databases for the regression report.
회귀 보고서용 데이터베이스.

Every added scenario must:
추가된 모든 시나리오는 다음을 충족해야 합니다.

- drive legal stimulus to the uncovered behavior;
- 밝혀지지 않은 행동에 대한 법적 자극을 유도합니다.
- assert the expected externally observable result;
- 외부에서 관찰 가능한 예상 결과를 주장합니다.
- preserve existing assertions and scoreboards;
- 기존 주장과 점수판을 보존합니다.
- include a deterministic terminal result and watchdog;
- 결정론적 최종 결과 및 감시 기능을 포함합니다.
- document random seed and plusargs if randomized;
- 무작위 시드와 plusargs를 무작위로 기록합니다.
- avoid hierarchy force/deposit unless explicitly required by the test plan.
- 테스트 계획에서 명시적으로 요구하지 않는 한 계층 구조 강제/보증을 피하세요.

Do not edit synthesizable RTL solely for coverage.
적용 범위만을 위해 합성 가능한 RTL을 편집하지 마십시오.
If a coverage gap exposes a
보장 공백으로 인해
design defect or genuinely dead code, present the evidence and ask before an
설계 결함 또는 완전히 죽은 코드인 경우 증거를 제시하고 담당 직원에게 문의하십시오.
RTL change.
RTL 변경.

## 5.
## Iterate
## 반복

For iteration `N`:
'N' 반복의 경우:

1. create a unique database and report directory such as `iter-01`;
1. 'iter-01'과 같은 고유한 데이터베이스 및 보고서 디렉터리를 생성합니다.
2. run the focused test;
2. 집중 테스트를 실행하세요.
3. run every applicable existing regression test;
3. 적용 가능한 모든 기존 회귀 테스트를 실행합니다.
4. require all tests to pass;
4. 모든 테스트를 통과해야 합니다.
5. save coverage before `close_sim`;
5. `close_sim` 이전에 적용 범위를 저장합니다.
6. export text and HTML;
6. 텍스트 및 HTML 내보내기;
7. compare each metric and the specific targeted item to the previous run.
7. 각 지표와 특정 대상 항목을 이전 실행과 비교합니다.

Example persistence:
지속성 예시:

```tcl
write_xsim_coverage \
  -cov_db_name iter-01 \
  -cov_db_dir <absolute-artifact-dir>/iter-01

export_xsim_coverage \
  -cov_db_name iter-01 \
  -cov_db_dir <absolute-artifact-dir>/iter-01 \
  -output_dir <absolute-artifact-dir>/iter-01-report \
  -report_format all \
  -open_html false
```

For a regression with separate coverage databases, merge during export:
별도의 적용 범위 데이터베이스를 사용한 회귀의 경우 내보내기 중에 병합합니다.

```tcl
export_xsim_coverage \
  -cov_db_name test-reset -cov_db_dir <artifact-dir>/run-reset \
  -cov_db_name test-saturate -cov_db_dir <artifact-dir>/run-saturate \
  -merge_dir <artifact-dir>/merged \
  -merge_db_name regression-01 \
  -output_dir <artifact-dir>/regression-01-report \
  -report_format all \
  -open_html false
```

Use exact paths and names returned by the run.
실행에서 반환된 정확한 경로와 이름을 사용하세요.
Verify that database and report
해당 데이터베이스와 보고서를 확인하세요.
files were created; a Tcl command returning without an exception is not enough.
파일이 생성되었습니다. 예외 없이 반환되는 Tcl 명령만으로는 충분하지 않습니다.
The exported report can contain both code and functional coverage from the
내보낸 보고서에는 다음의 코드와 기능 적용 범위가 모두 포함될 수 있습니다.
same XSim database.
동일한 XSim 데이터베이스.

## 6.
## Functional coverage closure
## 기능적 보장 폐쇄

Close named requirements, not only an aggregate score:
종합 점수뿐만 아니라 명명된 요구 사항을 닫습니다.

1. capture the baseline hit count for each target bin or cross;
1. 각 타겟 빈 또는 크로스에 대한 기준 적중 횟수를 캡처합니다.
2. confirm that the construct is supported by the installed XSim release;
2. 설치된 XSim 릴리스에서 구성이 지원되는지 확인하십시오.
3. derive legal stimulus from the requirement;
3. 요구사항으로부터 법적 자극을 얻습니다.
4. add an assertion, scoreboard check, or reference-model observation;
4. 주장, 점수판 확인 또는 참조 모델 관찰을 추가합니다.
5. rerun the focused test and the full applicable regression;
5. 집중 테스트와 적용 가능한 전체 회귀를 다시 실행합니다.
6. require PASS before admitting the database to the merge;
6. 데이터베이스를 병합하기 전에 PASS가 필요합니다.
7. verify that the specific target count increased;
7. 특정 목표 수가 증가했는지 확인합니다.
8. preserve bins that remain unreachable or blocked with evidence.
8. 접근할 수 없거나 증거로 막힌 쓰레기통을 보존하세요.

Do not:
하지 말아야 할 것:

- sample a covergroup artificially outside the behavior being measured;
- 측정 중인 동작 외부에서 인위적으로 커버 그룹을 샘플링합니다.
- broaden a bin merely to count an easier event;
- 더 쉬운 이벤트를 계산하기 위해 저장소를 확장합니다.
- delete or ignore an uncovered legal bin;
- 발견되지 않은 법적 상자를 삭제하거나 무시합니다.
- merge failed negative tests into passing-regression coverage;
- 실패한 음성 테스트를 통과 회귀 적용 범위에 병합합니다.
- merge databases with different source revisions, elaborated parameters,
- 다양한 소스 개정, 정교한 매개변수가 있는 데이터베이스를 병합합니다.
covergroup schemas, XSim versions, or coverage settings.
커버그룹 스키마, XSim 버전 또는 커버리지 설정.

For randomized closure, record each test and `-sv_seed`.
무작위 종료의 경우 각 테스트와 `-sv_seed`를 기록합니다.
Merge only compatible
병합만 호환 가능
passing seeds and replay every failure with the exact manifest before
시드를 전달하고 이전에 정확한 매니페스트를 사용하여 모든 실패를 재생합니다.
classifying it as reproducible.
재현 가능한 것으로 분류합니다.

## 7.
## Plateau and stop conditions
## 고원 및 정지 조건

Continue only when the preceding iteration:
이전 반복이 수행되는 경우에만 계속합니다.

- retains full regression correctness;
- 완전한 회귀 정확성을 유지합니다.
- closes its targeted item or produces a useful diagnosis;
- 대상 항목을 닫거나 유용한 진단을 생성합니다.
- makes measurable progress toward the agreed goal.
- 합의된 목표를 향해 측정 가능한 진전을 이루었습니다.

Stop when:
다음과 같은 경우에 중지하세요.

- the requested target is met;
- 요청한 목표가 충족되었습니다.
- remaining items are proven unreachable or elaborated out;
- 나머지 항목은 도달할 수 없거나 자세히 설명되어 있습니다.
- the same metric and uncovered set do not improve for two focused iterations;
- 동일한 측정 항목과 발견되지 않은 세트는 두 번의 집중 반복에 대해 개선되지 않습니다.
- requirements, stimulus models, licenses, resources, or observability block
- 요구 사항, 자극 모델, 라이선스, 리소스 또는 관찰 가능성 블록
further progress;
추가 진전;
- the next step would require RTL changes or exclusions without approval.
- 다음 단계에서는 승인 없이 RTL 변경 또는 제외가 필요합니다.

## 8.
## Exclusions
## 제외

Do not create a coverage-exclusion file by default.
기본적으로 적용 범위 제외 파일을 생성하지 마십시오.
An exclusion requires:
제외에는 다음이 필요합니다.

1. evidence that the item is unreachable, generated out, third-party code, or
1. 항목이 도달할 수 없거나, 생성되었거나, 제3자 코드라는 증거 또는
outside the verification scope;
검증 범위를 벗어나는 경우
2. a written rationale tied to configuration or requirements;
2. 구성 또는 요구 사항과 관련된 서면 근거
3. explicit user approval;
3. 명시적인 사용자 승인
4. separate reporting of raw and exclusion-adjusted metrics.
4. 원시 측정항목과 제외 조정 측정항목을 별도로 보고합니다.

Apply an approved file only during report generation:
보고서 생성 중에만 승인된 파일을 적용합니다.

```tcl
export_xsim_coverage \
  -cov_db_name <name> \
  -cov_db_dir <dir> \
  -ccExclusionFile <approved-file> \
  -output_dir <report-dir> \
  -report_format all \
  -open_html false
```

Never exclude reachable code merely to meet a target.
단순히 목표를 달성하기 위해 도달 가능한 코드를 제외하지 마십시오.

## Final report
## 최종 보고서

```text
Coverage scope:
Regression result:
Metric       Baseline   Final   Delta
Statement:
Branch:
Condition:
Toggle:
Functional covergroup:
Target coverpoints / crosses / bins:
New tests and requirements exercised:
Remaining reachable gaps:
Unreachable/elaborated-out items:
Approved exclusions (raw and adjusted metrics):
Coverage databases and reports:
```

## Official sources
## 공식 출처

- [UG900 code coverage](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Code-Coverage-Support)
- [UG900 코드 적용 범위](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Code-Coverage-Support)
- [UG900 XSim executable options](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/xsim-Executable-Options)
- [UG900 XSim 실행 가능 옵션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/xsim-Executable-Options)
- [UG900 write_xsim_coverage](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/write_xsim_coverage)
- [UG900 write_xsim_coverage](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/write_xsim_coverage)
- [UG900 export_xsim_coverage](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_xsim_coverage)
- [UG900 내보내기_xsim_coverage](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_xsim_coverage)
