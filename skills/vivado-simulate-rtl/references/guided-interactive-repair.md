<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Guided Interactive Repair
# 안내식 대화형 수리

Use this optional mode when the user requests waveform-first debugging,
사용자가 파형 우선 디버깅을 요청할 때 이 옵션 모드를 사용하십시오.
step-by-step repair, approval before RTL changes, or a teaching workflow.
단계별 수리, RTL 변경 전 승인 또는 교육 워크플로.
Do not enable it merely because a simulation fails.
단지 시뮬레이션이 실패했다는 이유만으로 활성화하지 마십시오.

## Applicability
## 적용 가능성

Use guided repair when:
다음과 같은 경우 안내식 수리를 사용하십시오.

- a runtime failure has a live snapshot or usable waveform;
- 런타임 오류에는 실시간 스냅샷 또는 사용 가능한 파형이 있습니다.
- the user wants diagnosis followed by a possible DUT repair;
- 사용자는 진단 후 가능한 DUT 수리를 원합니다.
- stopping for evidence review will not disrupt unattended automation.
- 증거 검토를 위해 중지해도 무인 자동화가 중단되지 않습니다.

Do not require this mode for analysis, elaboration, license, or environment
분석, 정교화, 라이센스 또는 환경을 위해 이 모드가 필요하지 않습니다.
failures that have no runtime waveform.
런타임 파형이 없는 오류.
Do not use its pauses for unattended CI,
무인 CI에 일시 중지를 사용하지 마십시오.
batch regressions, or a diagnosis-only request.
일괄 회귀 또는 진단 전용 요청.

Workspace-specific restrictions—protected directories, target order, forbidden
작업 공간별 제한 사항 - 보호된 디렉터리, 대상 순서, 금지됨
source inspection, or display requirements—remain workspace policy.
소스 검사 또는 디스플레이 요구 사항 - 작업 공간 정책을 유지합니다.
They are
그들은
not universal simulation rules.
보편적인 시뮬레이션 규칙이 아닙니다.

## Session discipline
## 세션 규율

Keep one project open and issue one Vivado Tcl command at a time.
하나의 프로젝트를 열어두고 한 번에 하나의 Vivado Tcl 명령을 실행하십시오.
Before every
매 전에
relaunch:
다시 시작:

```tcl
catch {close_sim -force}
```

If an earlier snapshot remains open, select and close it explicitly:
이전 스냅샷이 열려 있는 경우 명시적으로 선택하고 닫습니다.

```tcl
current_sim <simulation-name>
close_sim -force
```

Confirm resulting state instead of assuming a repeated command executed.
반복된 명령이 실행되었다고 가정하는 대신 결과 상태를 확인하십시오.

## Workflow
## 작업 흐름

### 1.
### Establish the contract
### 계약 체결

Record the target, mode, test, expected terminal result, runtime bound, and
대상, 모드, 테스트, 예상되는 최종 결과, 런타임 바인딩 및
signals implied by the requirement.
요구사항이 암시하는 신호.
Run the unchanged test and read the
변경되지 않은 테스트를 실행하고
complete transcript.
완전한 성적표.

Classify the first causal failure as design, testbench, environment, or
첫 번째 원인 실패를 설계, 테스트벤치, 환경 또는
timeout.
시간 초과.
Do not expose or modify RTL merely because a simulator reported an
시뮬레이터가 다음을 보고했다는 이유만으로 RTL을 노출하거나 수정하지 마십시오.
error.
오류.

### 2.
### Build FAIL evidence
### FAIL 증거 구축

For a runtime failure, select:
런타임 실패의 경우 다음을 선택하십시오.

1. clock and reset;
1. 시계 및 재설정;
2. transaction qualifiers;
2. 거래 한정자;
3. stimulus and reference value;
3. 자극 및 기준 값;
4. observed DUT result;
4. 관찰된 DUT 결과;
5. checker and terminal status;
5. 체커 및 터미널 상태;
6. a minimal internal signal only when boundary evidence is insufficient.
6. 경계 증거가 불충분한 경우에만 최소한의 내부 신호.

Discover actual paths from the live snapshot.
라이브 스냅샷에서 실제 경로를 찾아보세요.
Do not use recursive waveform
재귀 파형을 사용하지 마십시오
capture by default.
기본적으로 캡처합니다.
Create a uniquely named FAIL wave configuration when the
다음과 같은 경우 고유한 이름의 FAIL 웨이브 구성을 생성합니다.
backend supports it, and preserve a signal manifest even when the run is
백엔드는 이를 지원하며 실행이 실행되는 경우에도 신호 매니페스트를 유지합니다.
headless.
머리가 없는.

Record:
기록:

```text
Tests completed before failure:
First causal message and time:
Failure classification:
Requirement implicated:
FAIL waveform or database:
Signal manifest:
What the evidence proves:
```

### 3.
### Pause for the user
### 사용자를 위해 일시중지

Stop before reading or changing DUT RTL.
DUT RTL을 읽거나 변경하기 전에 중지하십시오.
Offer these decisions through the
이러한 결정은 다음을 통해 제공됩니다.
product's structured question mechanism when available:
가능한 경우 제품의 구조화된 질문 메커니즘:

- **Repair** — inspect the implicated RTL and apply the smallest justified fix.
- **수리** — 관련 RTL을 검사하고 가장 작은 정당한 수정 사항을 적용합니다.
- **More evidence** — add only signals or transcript context needed to resolve
- **추가 증거** — 해결에 필요한 신호 또는 녹취 내용만 추가하세요.
uncertainty.
불확실성.
- **Stop** — preserve evidence and make no source change.
- **중지** — 증거를 보존하고 소스를 변경하지 않습니다.

Do not treat silence or an unrelated reply as repair approval.
침묵이나 관련 없는 답변을 수리 승인으로 간주하지 마십시오.
If the user has
사용자가
already given the decision for this target, or for every target, act on it
이 목표 또는 모든 목표에 대해 이미 결정이 내려졌으므로 이에 따라 조치를 취하세요.
without asking again; still stop at any gate they have not answered.
다시 묻지도 않고; 응답하지 않는 게이트에는 여전히 정차합니다.

### 4.
### Repair only after approval
### 승인 후에만 수리

Inspect only sources needed to test the evidence-backed hypothesis.
증거 기반 가설을 테스트하는 데 필요한 소스만 검사합니다.
A
testbench may be inspected when the failure classification or user request
장애 분류 또는 사용자 요청 시 테스트벤치를 점검할 수 있습니다.
requires it; do not adopt anti-spoiler restrictions unless the workspace
그것을 요구합니다; 작업 공간이 아닌 한 스포일러 방지 제한 사항을 채택하지 마십시오.
defines them.
그들을 정의합니다.

Make the smallest requirement-driven change.
요구 사항에 따라 가장 작은 변경을 수행합니다.
Do not refactor, weaken a checker,
리팩토링하지 말고 체커를 약화시키십시오.
change backend, lower severity, or alter expected behavior to obtain PASS.
백엔드를 변경하거나 심각도를 낮추거나 예상 동작을 변경하여 PASS를 획득하세요.

### 5.
### Re-run and build PASS evidence
### 재실행 및 PASS 증거 구축

Close the prior simulation and rebuild from changed sources.
이전 시뮬레이션을 닫고 변경된 소스에서 다시 빌드합니다.
Run the affected
영향을 받은 실행
test, then the applicable regression.
테스트한 다음 적용 가능한 회귀를 수행합니다.
Require the normal functional-verdict
정상적인 기능 판정이 필요합니다.
gates from `SKILL.md`.
`SKILL.md`의 게이트.

Create PASS evidence using the same boundary signals and ordering as the FAIL
FAIL과 동일한 경계 신호 및 순서를 사용하여 PASS 증거를 생성합니다.
evidence so the repaired event is directly comparable.
증거가 있으므로 수리된 사건이 직접적으로 비교 가능합니다.
If exact in-memory
정확한 메모리 내인 경우
side-by-side views are unavailable, preserve separate FAIL and PASS artifacts.
나란히 보기를 사용할 수 없으며 별도의 FAIL 및 PASS 아티팩트를 유지합니다.

### 6.
### Report and stop
### 신고하고 중지하세요

Report:
보고:

```text
Target and requirement:
Failure classification:
FAIL evidence:
Approved repair:
Files changed:
Affected test result:
Regression result:
PASS evidence:
Artifacts:
Not verified:
```

Ask before moving to another target.
다른 대상으로 이동하기 전에 물어보세요.
Do not turn approval for one repair into
한 번의 수리에 대한 승인을 다음으로 전환하지 마십시오.
approval for unrelated changes.
관련 없는 변경에 대한 승인.

Before writing a final report that covers several targets, re-read each saved
여러 대상을 포괄하는 최종 보고서를 작성하기 전에 저장된 각 대상을 다시 읽어보세요.
FAIL log and quote its first failing check verbatim and its count of passed
FAIL은 첫 번째 실패한 검사를 그대로 기록하고 통과한 횟수를 인용합니다.
tests before that failure.
실패하기 전에 테스트합니다.
Do not reconstruct them from memory.
메모리에서 재구성하지 마십시오.
