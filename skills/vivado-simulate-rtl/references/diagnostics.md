<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Simulation Diagnostics
# 시뮬레이션 진단

Classify before editing.
편집하기 전에 분류하세요.
A simulator failure is not automatically a DUT defect.
시뮬레이터 오류는 자동으로 DUT 결함이 아닙니다.

## Triage order
## 분류 순서

1. Find the first causal error in the complete stage log.
1. 전체 단계 로그에서 첫 번째 원인 오류를 찾습니다.
2. Identify the stage: analysis, elaboration, runtime, or artifact export.
2. 분석, 정교화, 런타임 또는 아티팩트 내보내기 등 단계를 식별합니다.
3. Record simulation time, test/phase, seed, plusargs, mode, and backend.
3. 시뮬레이션 시간, 테스트/단계, 시드, plusargs, 모드 및 백엔드를 기록합니다.
4. Classify as design, testbench, environment, timeout, or tool defect.
4. 설계, 테스트벤치, 환경, 시간 초과 또는 도구 결함으로 분류합니다.
5. Preserve evidence before rerunning.
5. 재실행하기 전에 증거를 보존하세요.

Later errors often cascade from the first one.
이후 오류는 첫 번째 오류에서 연속적으로 발생하는 경우가 많습니다.

## Failure classes
## 실패 등급

### Environment
### 환경

Typical evidence:
전형적인 증거:

- executable or license unavailable;
- 실행 파일 또는 라이센스를 사용할 수 없습니다.
- compiled simulation library missing, stale, or mapped incorrectly;
- 컴파일된 시뮬레이션 라이브러리가 누락되었거나 오래되었거나 잘못 매핑되었습니다.
- source/include file missing;
- 소스/포함 파일이 누락되었습니다.
- unsupported simulator, host, language, or encrypted IP;
- 지원되지 않는 시뮬레이터, 호스트, 언어 또는 암호화된 IP
- wrong simulation top or compile order;
- 잘못된 시뮬레이션 상단 또는 컴파일 순서;
- netlist/SDF mismatch;
- 넷리스트/SDF 불일치;
- generated script cannot find its tools.
- 생성된 스크립트에서 해당 도구를 찾을 수 없습니다.

Action: report the exact prerequisite or mismatch.
조치: 정확한 전제조건 또는 불일치를 보고하십시오.
Do not edit RTL or silently
RTL을 편집하거나 자동으로 편집하지 마십시오.
switch simulators.
스위치 시뮬레이터.

### Testbench
### 테스트벤치

Typical evidence:
전형적인 증거:

- stimulus violates the interface protocol;
- 자극이 인터페이스 프로토콜을 위반합니다.
- checker/reference model is inconsistent with requirements;
- 검사기/참조 모델이 요구 사항과 일치하지 않습니다.
- reset polarity or timing is wrong;
- 재설정 극성 또는 타이밍이 잘못되었습니다.
- testbench races the DUT at a clock edge;
- 테스트벤치는 클록 에지에서 DUT를 경주합니다.
- watchdog is too short for legal latency;
- 워치독은 법적 대기 시간에 비해 너무 짧습니다.
- test never observes the response it drove;
- 테스트는 자신이 유도한 응답을 결코 관찰하지 않습니다.
- test declares PASS without checking outputs.
- 테스트는 출력을 확인하지 않고 PASS를 선언합니다.

Action: tie the defect to the verification contract, make the smallest
조치: 결함을 검증 계약에 연결하고 가장 작은 것으로 만듭니다.
verification-only correction when requested, and rerun all affected tests.
요청 시 검증만 수정하고 영향을 받은 모든 테스트를 다시 실행합니다.

### Design
### 디자인

Required evidence:
필수 증거:

- legal stimulus reached the DUT;
- 법적 자극이 DUT에 도달했습니다.
- the expected result follows from a stated requirement/reference model;
- 예상되는 결과는 명시된 요구 사항/참조 모델을 따릅니다.
- observed output diverges at a specific event/time;
- 관찰된 출력은 특정 이벤트/시간에 분기됩니다.
- checker and environment are credible;
- 검사기와 환경은 신뢰할 수 있습니다.
- focused waveforms support the same conclusion.
- 집중된 파형은 동일한 결론을 뒷받침합니다.

Action: if the request is diagnosis-only, stop.
조치: 요청이 진단 전용인 경우 중지하십시오.
Otherwise ask before modifying
그렇지 않으면 수정하기 전에 문의하세요.
RTL and apply the smallest requirement-driven fix.
RTL을 수행하고 가장 작은 요구 사항 기반 수정 사항을 적용합니다.

### Timeout
### 시간 초과

Typical evidence:
전형적인 증거:

- no terminal PASS/FAIL condition;
- 터미널 PASS/FAIL 조건 없음;
- simulation remains active beyond the justified bound;
- 시뮬레이션은 정당화된 범위를 넘어 활성 상태로 유지됩니다.
- handshake or state machine makes no progress;
- 핸드셰이크 또는 상태 시스템이 진행되지 않습니다.
- testbench omitted `$finish` or a watchdog;
- 테스트벤치에서 `$finish` 또는 워치독이 생략되었습니다.
- an external simulator waits in GUI/interactive mode.
- 외부 시뮬레이터는 GUI/대화형 모드에서 대기합니다.

Action: preserve the transcript and waveform, terminate safely, identify the
조치: 기록과 파형을 보존하고, 안전하게 종료하고,
last completed phase/event, and classify the likely design/testbench/environment
마지막으로 완료된 단계/이벤트 및 예상되는 설계/테스트벤치/환경 분류
cause only when evidence supports it.
증거가 뒷받침하는 경우에만 원인이 됩니다.

### Tool defect
### 도구 결함

Required evidence:
필수 증거:

- the input is legal and the environment is correct (supported versions,
- 입력이 적법하고 환경이 올바릅니다(지원되는 버전,
matching libraries, fresh netlist/SDF);
일치하는 라이브러리, 새로운 넷리스트/SDF);
- the tool crashes, rejects, or mis-writes that input reproducibly, for
- 재현 가능한 입력을 도구가 충돌하거나 거부하거나 잘못 기록합니다.
example a simulator internal error or a Vivado-generated file the simulator
예: 시뮬레이터 내부 오류 또는 Vivado 생성 파일 시뮬레이터
cannot parse;
구문 분석할 수 없습니다.
- a rerun of the same step from the same input shows the same result.
- 동일한 입력에서 동일한 단계를 다시 실행하면 동일한 결과가 표시됩니다.

Action: report the tool, version, step and reproduction command.
조치: 도구, 버전, 단계 및 재생 명령을 보고합니다.
Do not edit
편집하지 마세요
RTL or Vivado-generated files to force a PASS; a workaround may be tried only
PASS를 강제하기 위한 RTL 또는 Vivado 생성 파일; 해결 방법은 시도해 볼 수 있습니다.
in a separate scratch directory, labeled as diagnostic, and never produces the
진단이라는 레이블이 붙은 별도의 스크래치 디렉토리에 있으며 절대 생성하지 않습니다.
reported verdict.
판결을 보고했습니다.

## Stage-specific checks
## 단계별 점검

### Analysis / compile
### 분석/컴파일

Check:
확인:

- language and file type assignment;
- 언어 및 파일 유형 할당;
- source/library compile order;
- 소스/라이브러리 컴파일 순서;
- include directories and macro definitions;
- 디렉토리와 매크로 정의를 포함합니다.
- VHDL standard and SystemVerilog enablement;
- VHDL 표준 및 SystemVerilog 활성화;
- duplicate modules/packages;
- 중복 모듈/패키지;
- syntax error's first location.
- 구문 오류의 첫 번째 위치입니다.

### Elaboration / optimization
### 정교화/최적화

Check:
확인:

- top module/entity and architecture;
- 최상위 모듈/엔티티 및 아키텍처;
- missing design unit or library mapping;
- 디자인 단위 또는 라이브러리 매핑이 누락되었습니다.
- parameter/generic and port mismatch;
- 매개변수/일반 및 포트 불일치;
- unresolved `glbl` or primitive models;
- 해결되지 않은 'glbl' 또는 기본 모델;
- encrypted IP support;
- 암호화된 IP 지원;
- optimized-away debug signals;
- 최적화된 디버그 신호;
- snapshot name and mode.
- 스냅샷 이름 및 모드.

### Runtime
### 런타임

Check:
확인:

- first assertion severity and message;
- 첫 번째 주장 심각도 및 메시지
- expected versus observed values;
- 기대값 대 관찰값;
- clock/reset and valid/ready qualifiers;
- 시계/재설정 및 유효/준비 한정자;
- unknown/high-impedance propagation;
- 알려지지 않은/고임피던스 전파;
- seed and randomization failure;
- 시드 및 무작위화 실패;
- watchdog and terminal marker;
- 워치독 및 터미널 마커;
- simulator error/fatal counts.
- 시뮬레이터 오류/치명적 수.

### Post-implementation timing
### 구현 후 시기

Check:
확인:

- implementation and generated netlist freshness;
- 구현 및 생성된 넷리스트 신선도;
- SDF and netlist originate from the same run;
- SDF와 넷리스트는 동일한 실행에서 시작됩니다.
- correct annotation scope;
- 올바른 주석 범위
- timing checks and pulse rejection messages;
- 타이밍 확인 및 펄스 거부 메시지;
- reset/recovery behavior under delays.
- 지연 시 재설정/복구 동작.

A timing verdict needs the simulator's own annotation-success message: XSim
타이밍 판정에는 시뮬레이터 자체의 주석 성공 메시지가 필요합니다: XSim
`[XSIM 43-3452] SDF backannotation was successful`, Questa/ModelSim
`[XSIM 43-3452] SDF 역주석이 성공했습니다.`, Questa/ModelSim
`(vsim-3587) SDF Backannotation Successfully Completed`, VCS
`(vsim-3587) SDF 역주석이 성공적으로 완료되었습니다`, VCS
`SDF annotation begin` followed by `SDF annotation end` (not
`SDF 주석 시작` 다음에 `SDF 주석 끝`이 옵니다(안됨).
`SDF annotation aborted`), Xcelium `Annotation completed successfully`.
`SDF 주석이 중단되었습니다.`), Xcelium `주석이 성공적으로 완료되었습니다.`
Some
일부
simulators give up on an SDF with only warnings and still run the netlist
시뮬레이터는 경고만 보고 SDF를 포기하고 여전히 넷리스트를 실행합니다.
with zero delay: Xcelium `*W,SDFCNC ... skipping annotation`, and VCS aborts
지연 시간 없음: Xcelium `*W,SDFCNC ... 주석 건너뛰기` 및 VCS 중단
annotation after repeated `SDFCOM_NNTC` negative setup/hold errors unless
반복된 `SDFCOM_NNTC` 부정 설정/보류 오류 후 주석
elaborated with `+neg_tchk`.
`+neg_tchk`로 자세히 설명했습니다.
Such a run is not a timing result:
이러한 실행은 타이밍 결과가 아닙니다.
report the annotation failure, never a timing PASS, even when `TEST_PASS`
'TEST_PASS'인 경우에도 타이밍 PASS가 아닌 주석 실패를 보고합니다.
printed.
인쇄되었습니다.

An SDF parse error at the end of the file (Questa `syntax error, unexpected
파일 끝에 SDF 구문 분석 오류가 있습니다(Questa`구문 오류, 예상치 못한
$end`, Xcelium `*E,SDFIRE` on the last line) usually means the SDF Vivado
$end`, 마지막 줄의 Xcelium `*E,SDFIRE`)는 일반적으로 SDF Vivado를 의미합니다.
wrote is structurally incomplete.
쓴 내용은 구조적으로 불완전합니다.
Compare its opening and closing parenthesis
여는 괄호와 닫는 괄호를 비교하세요.
counts.
중요합니다.
If they differ, re-run `write_sdf -mode timesim` from the routed
서로 다른 경우 라우팅된에서 `write_sdf -mode timesim`을 다시 실행하세요.
checkpoint to show it is reproducible, and classify it as `tool_defect` in
재현 가능함을 보여주는 체크포인트를 만들고, 'tool_defect'로 분류합니다.
Vivado's SDF writer, not a design or environment failure.
Vivado의 SDF Writer는 설계나 환경의 실패가 아닙니다.
Handle any patched
모든 패치 처리
SDF as described under Tool defect.
도구 결함에 설명된 SDF입니다.

Do not treat timing simulation as static timing closure or equivalence proof.
타이밍 시뮬레이션을 정적 타이밍 종료 또는 등가 증명으로 취급하지 마십시오.

### Coverage export
### 커버리지 내보내기

Check:
확인:

- coverage was enabled before elaboration;
- 상세화 이전에 적용 범위가 활성화되었습니다.
- `write_xsim_coverage` ran before closing XSim;
- XSim을 닫기 전에 `write_xsim_coverage`가 실행되었습니다.
- database and report paths exist;
- 데이터베이스 및 보고서 경로가 존재합니다.
- expected DUT hierarchy is included;
- 예상되는 DUT 계층 구조가 포함됩니다.
- raw and exclusion-adjusted reports are not confused;
- 원시 보고서와 제외 조정 보고서는 혼동되지 않습니다.
- merged runs use compatible design/elaboration configurations.
- 병합된 실행은 호환 가능한 설계/정교 구성을 사용합니다.

## Focused waveform recipe
## 집중 파형 레시피

Include:
포함:

1. clock and reset;
1. 시계 및 재설정;
2. transaction qualifiers and control;
2. 거래 한정자 및 통제;
3. stimulus data;
3. 자극 데이터;
4. reference/expected value;
4. 참조/예상 값;
5. DUT result and status;
5. DUT 결과 및 상태;
6. checker state and terminal status;
6. 체커 상태 및 터미널 상태;
7. one or two internal signals only if boundary evidence is insufficient.
7. 경계 증거가 불충분한 경우에만 하나 또는 두 개의 내부 신호.

Avoid recursive capture by default.
기본적으로 재귀 캡처를 방지합니다.

## Diagnosis report
## 진단 보고서

```text
Backend / mode:
Stage:
First causal message:
Simulation time / test phase:
Failure class:
Requirement:
Evidence:
Likely root cause:
Recommended next action:
Artifacts:
```

State uncertainty directly.
불확실성을 직접적으로 설명합니다.
Use `INCONCLUSIVE` when the contract, stimulus,
계약, 자극,
logs, or observability are insufficient.
로그 또는 관찰 가능성이 충분하지 않습니다.
