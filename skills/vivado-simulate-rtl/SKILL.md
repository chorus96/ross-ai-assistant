<!-- ko-translation: enabled -->
---
name: vivado-simulate-rtl
description: Runs and diagnoses FPGA RTL simulations through Vivado using XSim, Questa, ModelSim, VCS, Xcelium, Riviera-PRO, or Active-HDL; supports approval-gated interactive repair; creates self-checking testbenches; runs UVM and seeded regressions; closes XSim code and functional coverage; verifies assertions; and captures SAIF or VCD activity. Use when users ask to simulate HDL, interpret or triage existing simulation logs (including Questa, VCS, Xcelium, or other third-party compile, elaboration, or run logs, and simulator crash or tool-defect reports), debug or repair RTL from waveform evidence, run gate-level or timing simulation, measure coverage, run random regression, analyze power activity, or generate portable stimulus.
license: MIT
compatibility: Requires Vivado (validated on 2025.2 and 2026.1) with the Vivado MCP server, and Python 3 for the bundled scripts. Third-party simulators need their own install, license, and version-matched compile_simlib libraries.
metadata:
  author: AMD Skill Developer
  version: "1.0.0"
  stage: beta
allowed-tools: Read Bash Write
---

<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# FPGA RTL Simulation
# FPGA RTL 시뮬레이션

> **Early Access** - this skill may change before general availability.
> **조기 액세스** - 이 스킬은 일반 출시되기 전에 변경될 수 있습니다.

Verify behavior against an explicit contract.
명시적 계약에 따라 동작을 확인합니다.
A clean compile or simulator exit
깔끔한 컴파일 또는 시뮬레이터 종료
is not functional proof.
기능적 증거가 아닙니다.

## Tool policy
## 도구 정책

Use the Vivado MCP for project inspection and Vivado Tcl:
프로젝트 검사 및 Vivado Tcl에는 Vivado MCP를 사용하십시오.

- `Vivado:vivado_start` — start one Tcl session.
- `Vivado:vivado_start` — 하나의 Tcl 세션을 시작합니다.
- `Vivado:vivado_execute` — open/configure the project and launch simulation.
- `Vivado:vivado_execute` — 프로젝트를 열고 구성하고 시뮬레이션을 시작합니다.
- `Vivado:vivado_status` — poll a command that is still running.
- `Vivado:vivado_status` — 아직 실행 중인 명령을 폴링합니다.
- `Vivado:vivado_log_messages` — inspect Vivado errors and warnings.
- `Vivado:vivado_log_messages` — Vivado 오류 및 경고를 검사합니다.
- `Vivado:vivado_doc_search` — verify unfamiliar or release-sensitive commands,
- `Vivado:vivado_doc_search` — 익숙하지 않거나 릴리스에 민감한 명령을 확인합니다.
and find Answer Records, known issues, and documented workarounds for an
답변 기록, 알려진 문제, 문서화된 해결 방법을 찾아보세요.
observed failure signature.
관찰된 실패 서명.

Documentation research is part of diagnosis, not an optional extra.
문서 조사는 선택 사항이 아닌 진단의 일부입니다.
A tool
도구
crash, unsupported construct, or library failure is not final until it has
충돌, 지원되지 않는 구성 또는 라이브러리 오류는
been researched; see
연구되었습니다; 참조
[references/known-issue-research.md](references/known-issue-research.md).
[참조/알려진-문제-연구.md](참조/알려진-문제-연구.md).

Use the exact configured server prefix if it differs from `Vivado`.
'Vivado'와 다른 경우 정확하게 구성된 서버 접두사를 사용하세요.
Never issue
발행하지 않음
two `Vivado:vivado_execute` calls concurrently on one session.
한 세션에서 두 개의 `Vivado:vivado_execute` 호출이 동시에 발생합니다.
Keep one project open
하나의 프로젝트를 열어두세요
while iterating.
반복하는 동안.

Find simulators, licenses and compiled libraries through `PATH`, the
'PATH'를 통해 시뮬레이터, 라이센스 및 컴파일된 라이브러리를 찾으세요.
simulator's environment variables (for example `VCS_HOME` or
시뮬레이터의 환경 변수(예: `VCS_HOME` 또는
`MGLS_LICENSE_FILE`), the project's compiled-library settings, and paths the
`MGLS_LICENSE_FILE`), 프로젝트의 컴파일된 라이브러리 설정 및 경로
user supplied.
사용자가 제공합니다.
Do not scan the file system (`find /`, home directories,
파일 시스템(`find /`, 홈 디렉터리,
network mounts) for them; if they are not found that way, report the
네트워크 마운트)를 위한 것입니다. 그런 식으로 발견되지 않으면 신고하세요.
prerequisite as missing.
전제 조건이 누락되었습니다.

## Read the relevant reference
## 관련 참고자료를 읽어보세요

`references/` and `scripts/` paths are relative to this skill's directory, not
`references/` 및 `scripts/` 경로는 이 스킬의 디렉터리에 상대적입니다.
the workspace.
작업 공간.
Do not search the file system for them.
파일 시스템에서 검색하지 마십시오.

- XSim execution and artifacts: [references/xsim.md](references/xsim.md)
- XSim 실행 및 아티팩트: [references/xsim.md](references/xsim.md)
- Iterative XSim coverage closure:
- 반복적인 XSim 적용 범위 종료:
[references/xsim-coverage-closure.md](references/xsim-coverage-closure.md)
[참조/xsim-coverage-closure.md](참조/xsim-coverage-closure.md)
- XSim UVM, seeded regression, assertions, and functional coverage:
- XSim UVM, 시드 회귀, 어설션 및 기능 적용 범위:
[references/xsim-advanced-verification.md](references/xsim-advanced-verification.md)
[참조/xsim-advanced-verification.md](참조/xsim-advanced-verification.md)
- XSim SAIF, VCD, and portable sub-design stimulus:
- XSim SAIF, VCD 및 휴대용 하위 설계 자극:
[references/xsim-activity-capture.md](references/xsim-activity-capture.md)
[참조/xsim-activity-capture.md](참조/xsim-activity-capture.md)
- Questa setup and execution: [references/questa.md](references/questa.md)
- Questa 설정 및 실행: [references/questa.md](references/questa.md)
- ModelSim: [references/modelsim.md](references/modelsim.md)
- ModelSim: [references/modelsim.md](references/modelsim.md)
- VCS: [references/vcs.md](references/vcs.md)
- VCS: [참조/vcs.md](참조/vcs.md)
- Xcelium: [references/xcelium.md](references/xcelium.md)
- Xcelium: [references/xcelium.md](references/xcelium.md)
- Riviera-PRO: [references/riviera.md](references/riviera.md)
- 리비에라-PRO: [references/riviera.md](references/riviera.md)
- Active-HDL: [references/activehdl.md](references/activehdl.md)
- 액티브-HDL: [references/activehdl.md](references/activehdl.md)
- Release snapshot and shared third-party rules:
- 스냅샷 및 공유된 타사 규칙 출시:
[references/other-vivado-simulators.md](references/other-vivado-simulators.md)
[참조/other-vivado-simulators.md](참조/other-vivado-simulators.md)
- Failure diagnosis: [references/diagnostics.md](references/diagnostics.md)
- 고장 진단: [references/diagnostics.md](references/diagnostics.md)
- XSim error signatures and what they mean:
- XSim 오류 서명과 그 의미:
[references/xsim-error-signatures.md](references/xsim-error-signatures.md)
[참조/xsim-error-signatures.md](참조/xsim-error-signatures.md)
- Answer Records and documented workarounds:
- 답변 기록 및 문서화된 해결 방법:
[references/known-issue-research.md](references/known-issue-research.md)
[참조/알려진-문제-연구.md](참조/알려진-문제-연구.md)
- Deciding design defect versus simulator defect:
- 설계 결함과 시뮬레이터 결함 결정:
[references/cross-simulator-arbitration.md](references/cross-simulator-arbitration.md)
[참조/교차 시뮬레이터-중재.md](참조/교차 시뮬레이터-중재.md)
- Reducing a failure to a reportable case:
- 실패를 보고 가능한 사례로 줄이기:
[references/minimal-reproducer.md](references/minimal-reproducer.md)
[참조/최소 재생산자.md](참조/최소 재생산자.md)
- Host, library, and license preflight:
- 호스트, 라이브러리 및 라이센스 프리플라이트:
[references/environment-preflight.md](references/environment-preflight.md)
[참조/환경-preflight.md](참조/환경-preflight.md)
- Finding the release that introduced a regression:
- 회귀를 도입한 릴리스 찾기:
[references/version-bisect.md](references/version-bisect.md)
[참조/버전-bisect.md](참조/버전-bisect.md)
- Compile, elaborate, and runtime performance:
- 컴파일, 정교화 및 런타임 성능:
[references/simulation-performance.md](references/simulation-performance.md)
[참조/시뮬레이션-성능.md](참조/시뮬레이션-성능.md)
- Waveform evidence without a GUI:
- GUI가 없는 파형 증거:
[references/headless-waveform.md](references/headless-waveform.md)
[참조/headless-waveform.md](참조/headless-waveform.md)
- Approval-gated waveform-first repair:
- 승인 기반 파형 우선 복구:
[references/guided-interactive-repair.md](references/guided-interactive-repair.md)
[참조/guided-interactive-repair.md](참조/guided-interactive-repair.md)
- Adding a backend: [references/adding-a-simulator.md](references/adding-a-simulator.md)
- 백엔드 추가: [references/adding-a-simulator.md](references/adding-a-simulator.md)

Read only the references needed for this request.
이 요청에 필요한 참조만 읽으십시오.

## Required inputs
## 필수 입력

Discover from the workspace and project before asking the user:
사용자에게 질문하기 전에 작업공간과 프로젝트에서 검색하세요.

1. `.xpr` project, or explicit non-project source list
1. `.xpr` 프로젝트 또는 명시적인 비프로젝트 소스 목록
2. simulation set and top module
2. 시뮬레이션 세트 및 상위 모듈
3. backend: default `xsim`; never silently substitute another backend
3. 백엔드: 기본 `xsim`; 다른 백엔드를 자동으로 대체하지 마세요.
4. mode: behavioral, post-synthesis functional, post-implementation
4. 모드: 동작, 합성 후 기능, 구현 후
functional, or post-implementation timing
기능적 또는 구현 후 타이밍
5. test contract and terminal PASS condition
5. 테스트 계약 및 터미널 PASS 조건
6. bounded runtime or testbench-controlled termination
6. 제한된 런타임 또는 테스트벤치 제어 종료
7. selected UVM test, seed list, plusargs, expected assertions, and coverage
7. 선택된 UVM 테스트, 시드 목록, plusargs, 예상 주장 및 적용 범위
targets when advanced verification is requested
사전 검증 요청 시 대상
8. warm-up interval, measurement interval, hierarchy, and object set for SAIF,
8. SAIF에 대한 예열 간격, 측정 간격, 계층 구조 및 개체 집합,
VCD, or portable-stimulus capture
VCD 또는 휴대용 자극 캡처
9. requested artifacts: transcript, waveform, coverage, activity, generated
9. 요청된 아티팩트: 기록, 파형, 적용 범위, 활동, 생성됨
testbench, and scripts
테스트벤치 및 스크립트

Ask only for inputs that cannot be discovered.
검색할 수 없는 입력만 요청하세요.
For third-party simulators,
타사 시뮬레이터의 경우
require the user-provided installation, license, and compiled-library paths.
사용자가 제공한 설치, 라이센스 및 컴파일된 라이브러리 경로가 필요합니다.
Never guess site-specific paths.
사이트별 경로를 추측하지 마세요.

## Preflight
## 비행 전

1. Start or reconnect to a Vivado MCP session.
1. Vivado MCP 세션을 시작하거나 다시 연결하세요.
2. Open the `.xpr` project once, or load the non-project source list.
2. `.xpr` 프로젝트를 한 번 열거나 프로젝트가 아닌 소스 목록을 로드하세요.
3. Confirm part, project mode, simulator, simulation sets,
3. 부품, 프로젝트 모드, 시뮬레이터, 시뮬레이션 세트 확인
top modules, source files, compile order, include directories, defines, and
최상위 모듈, 소스 파일, 컴파일 순서, 디렉토리 포함, 정의 및
language mix.
언어 혼합.
4. Confirm that the selected mode is available.
4. 선택한 모드를 사용할 수 있는지 확인하세요.
4. Post-synthesis and
4. 합성 후 및
post-implementation modes require the corresponding completed design stage.
구현 후 모드에는 해당하는 완성된 설계 단계가 필요합니다.
5. Confirm a finite runtime.
5. 유한한 런타임을 확인하세요.
5. Do not use an unbounded `run all` unless the
5. 다음이 아닌 이상 무제한 '모두 실행'을 사용하지 마세요.
self-checking testbench has a proven watchdog and terminal `$finish`.
자체 검사 테스트벤치에는 입증된 감시 기능과 터미널 '$finish'가 있습니다.
6. For a third-party backend, complete its reference preflight before changing
6. 타사 백엔드의 경우 변경하기 전에 참조 프리플라이트를 완료하세요.
`target_simulator`.
`target_simulator`.
7. When a previous run failed to launch, linked against the wrong libraries, or
7. 이전 실행이 시작되지 않았거나 잘못된 라이브러리에 연결된 경우 또는
the host is unfamiliar, run the environment doctor first.
호스트가 낯설으니 환경닥터를 먼저 실행해보세요.
It is bundled at
에 번들로 제공됩니다.
`scripts/sim_env_doctor.py` in this skill's directory; see
이 스킬 디렉터리의 `scripts/sim_env_doctor.py`; 참조
[references/environment-preflight.md](references/environment-preflight.md).
[참조/환경-preflight.md](참조/환경-preflight.md).

Do not modify design RTL during preflight.
프리플라이트 중에는 디자인 RTL을 수정하지 마십시오.
Adding or editing verification-only
확인 전용 추가 또는 편집
files is allowed when the user requested a testbench or coverage improvement.
사용자가 테스트벤치 또는 적용 범위 개선을 요청한 경우 파일이 허용됩니다.

## Common simulation workflow
## 일반적인 시뮬레이션 작업 흐름

Track this sequence for each target:
각 대상에 대해 다음 순서를 추적하십시오.

```text
- [ ] Discover project, simulation set, top, backend, mode, and contract
- [ ] Complete backend preflight
- [ ] Run a bounded simulation
- [ ] Read the complete transcript and identify the first failure
- [ ] Classify the result
- [ ] Research a documented cause and workaround for a tool or host failure
- [ ] Preserve transcript, waveform, scripts, and coverage artifacts
- [ ] Modify verification code only when requested
- [ ] Re-run the full applicable regression
- [ ] Report evidence, exclusions, and remaining risk
```

### 1.
### Discover
### 발견하다

Use Vivado Tcl to query, not assume:
다음을 가정하지 말고 Vivado Tcl을 사용하여 쿼리하십시오.

```tcl
get_filesets -filter {FILESET_TYPE == SimulationSrcs}
get_property TOP [get_filesets <simset>]
get_files -of_objects [get_filesets <simset>]
get_property target_simulator [current_project]
```

Update compile order before launch.
출시 전에 컴파일 순서를 업데이트하세요.
Do not add duplicate sources already owned
이미 소유한 중복 소스를 추가하지 마세요.
by the project.
프로젝트로.

### 2.
### Run
### 실행

Use the backend reference.
백엔드 참조를 사용하십시오.
Prefer a self-checking testbench and batch execution.
자체 검사 테스트벤치 및 일괄 실행을 선호합니다.
Close a prior active simulation before relaunching.
다시 시작하기 전에 이전 활성 시뮬레이션을 닫습니다.
A long-running
장기 실행
`launch_simulation` is blocking; allow an appropriate tool timeout and poll
`launch_simulation`이 차단 중입니다. 적절한 도구 시간 초과를 허용하고 폴링합니다.
status if the MCP reports it as still running.
MCP가 아직 실행 중이라고 보고하는 경우 상태입니다.

### 3.
### Determine the verdict
### 판결을 결정하다

Read the transcript and require all applicable gates:
성적표를 읽고 해당하는 모든 게이트를 요구하십시오.

- compile and elaboration succeeded;
- 컴파일 및 정교화가 성공했습니다.
- simulator returned normally;
- 시뮬레이터가 정상적으로 반환되었습니다.
- explicit terminal PASS marker or equivalent test-framework result exists;
- 명시적인 터미널 PASS 마커 또는 동등한 테스트 프레임워크 결과가 존재합니다.
- every required test completed;
- 모든 필수 테스트가 완료되었습니다.
- no unexpected error, fatal assertion, or timeout occurred.
- 예기치 않은 오류, 치명적인 어설션 또는 시간 초과가 발생하지 않았습니다.
- UVM summary has zero unexpected errors/fatals and the selected test completed;
- UVM 요약에는 예기치 않은 오류/치명적인 내용이 없으며 선택한 테스트가 완료되었습니다.
- assertion results match the test plan without ignore or severity downgrade;
- 어설션 결과는 무시 또는 심각도 다운그레이드 없이 테스트 계획과 일치합니다.
- randomized failures replay with the exact seed before being called
- 무작위 실패는 호출되기 전에 정확한 시드로 재생됩니다.
reproducible;
재현 가능;
- in timing mode, the simulator reported that SDF annotation succeeded.
- 타이밍 모드에서 시뮬레이터는 SDF 주석이 성공했다고 보고했습니다.
- A
skipped or ignored SDF means a zero-delay run, not a timing PASS.
건너뛰거나 무시된 SDF는 타이밍 PASS가 아니라 지연 없는 실행을 의미합니다.

Classify non-PASS as `design`, `testbench`, `environment`, `timeout`, or
PASS가 아닌 경우 '설계', '테스트벤치', '환경', '타임아웃' 또는 '시간 초과'로 분류합니다.
`tool_defect` (reproducible tool failure on legal input) using
`tool_defect`(적법한 입력 시 재현 가능한 도구 오류) 사용
[references/diagnostics.md](references/diagnostics.md).
[참조/진단.md](참조/진단.md).
Quote the first causal
첫 번째 원인 인용
error and its simulation time.
오류 및 시뮬레이션 시간.
Later cascaded errors are secondary.
나중에 연쇄 오류는 부차적인 오류입니다.

### 4.
### Diagnose with focused evidence
### 집중된 증거로 진단

Select clock/reset, transaction qualifiers, inputs, expected values, observed
시계/재설정, 거래 한정자, 입력, 예상 값, 관찰 선택
outputs, and test status.
출력 및 테스트 상태.
Prefer interface and testbench signals; add internal
인터페이스와 테스트벤치 신호를 선호합니다. 내부 추가
DUT signals only when necessary and available in the elaborated snapshot.
DUT는 필요할 때만 신호를 보내고 정교한 스냅샷에서 사용할 수 있습니다.

Do not edit RTL merely because a simulation failed.
단지 시뮬레이션이 실패했다는 이유만으로 RTL을 편집하지 마십시오.
If the user asked only to
사용자가 요청한 경우에만
diagnose, stop after the evidence-backed diagnosis.
진단하고, 근거가 뒷받침되는 진단 후 중단하세요.
If a DUT change is needed,
DUT 변경이 필요한 경우,
obtain approval and apply the smallest change that satisfies the contract.
승인을 얻고 계약을 충족하는 가장 작은 변경 사항을 적용합니다.

#### Guided interactive repair (optional)
#### 안내식 대화형 복구(선택 사항)

When the user requests step-by-step repair, waveform-first debugging, or an
사용자가 단계별 복구, 파형 우선 디버깅 또는
approval pause before RTL changes, read
RTL 변경 전 승인 일시 중지, 읽기
[references/guided-interactive-repair.md](references/guided-interactive-repair.md).
[참조/guided-interactive-repair.md](참조/guided-interactive-repair.md).
Preserve focused FAIL evidence, stop for an explicit repair/more-evidence/stop
집중된 FAIL 증거를 보존하고 명시적인 수리/추가 증거/중지를 위해 중지합니다.
decision, then—only after approval—make the smallest justified change, rerun
결정을 내린 후 승인 후에만 정당한 최소 변경 사항을 적용하고 다시 실행합니다.
the affected test and applicable regression, and capture comparable PASS
영향을 받은 테스트와 적용 가능한 회귀 분석 및 비교 가능한 PASS 캡처
evidence.
증거.

Do not enable this mode by default.
기본적으로 이 모드를 활성화하지 마십시오.
Its pauses are inappropriate for unattended
일시 중지는 무인 작업에 부적절합니다.
CI, batch regressions, diagnosis-only requests, and pre-runtime failures with no
CI, 일괄 회귀, 진단 전용 요청 및 사전 런타임 실패
waveform.
파형.

### 5.
### Resolve environment and tool failures
### 환경 및 도구 오류 해결

An `environment` classification is a starting point, not an answer.
'환경' 분류는 답이 아닌 출발점이다.
When the
때
simulator crashed, rejected legal source, failed to link, or disagreed with
시뮬레이터 충돌, 법적 소스 거부, 연결 실패 또는 동의하지 않음
another tool, continue until the user has either a verified workaround or a
다른 도구를 사용하는 경우 사용자가 확인된 해결 방법이나
reportable defect:
보고 가능한 결함:

1. Identify the signature and stage using
1. 다음을 사용하여 서명과 단계를 식별합니다.
[references/xsim-error-signatures.md](references/xsim-error-signatures.md).
[참조/xsim-error-signatures.md](참조/xsim-error-signatures.md).
2. Decide design versus simulator with
2. 다음을 사용하여 설계와 시뮬레이터를 결정합니다.
[references/cross-simulator-arbitration.md](references/cross-simulator-arbitration.md)
[참조/교차 시뮬레이터-중재.md](참조/교차 시뮬레이터-중재.md)
when another licensed backend is available.
라이선스가 부여된 다른 백엔드를 사용할 수 있는 경우
3. Search for a documented cause and workaround with
3. 문서화된 원인과 해결 방법을 검색해 보세요.
[references/known-issue-research.md](references/known-issue-research.md),
[참조/알려진-문제-연구.md](참조/알려진-문제-연구.md),
then verify the workaround by rerunning the same bounded test.
그런 다음 동일한 제한된 테스트를 다시 실행하여 해결 방법을 확인하십시오.
4. If the failure is new since an upgrade, locate the boundary with
4. 업그레이드 이후에 오류가 발생한 경우 다음을 사용하여 경계를 찾습니다.
[references/version-bisect.md](references/version-bisect.md).
[참조/버전-bisect.md](참조/버전-bisect.md).
5. If no workaround exists, reduce the case with
5. 해결 방법이 없으면 다음을 사용하여 사례를 줄이세요.
[references/minimal-reproducer.md](references/minimal-reproducer.md) so the
[references/minimal-reproducer.md](references/minimal-reproducer.md) 따라서
defect is reportable.
결함이 보고될 수 있습니다.

Report the Answer Record consulted, whether it applied, and what any
상담한 답변기록, 적용 여부, 내용을 보고합니다.
workaround changes.
해결 방법이 변경됩니다.
Never apply a vendor patch, alter the user's project, or
공급업체 패치를 적용하거나 사용자의 프로젝트를 변경하거나
switch the requested backend without approval.
승인 없이 요청된 백엔드를 전환합니다.
Never claim a workaround works
해결 방법이 효과가 있다고 주장하지 마십시오.
without a rerun.
재방송 없이.

Slow but correct simulation is a performance question, not a failure; use
느리지만 정확한 시뮬레이션은 성능 문제이지 실패가 아닙니다. 사용하다
[references/simulation-performance.md](references/simulation-performance.md).
[참조/시뮬레이션-성능.md](참조/시뮬레이션-성능.md).
A broken viewer or missing display never blocks a verdict; use
깨진 뷰어 또는 누락된 디스플레이가 평결을 방해하지 않습니다. 사용하다
[references/headless-waveform.md](references/headless-waveform.md).
[참조/headless-waveform.md](참조/headless-waveform.md).

### 6.
### Re-run
### 재실행

After any testbench or approved DUT change:
테스트벤치 또는 승인된 DUT 변경 후:

1. close the prior simulation;
1. 이전 시뮬레이션을 닫습니다.
2. rebuild from changed sources;
2. 변경된 소스에서 다시 빌드합니다.
3. run the affected test;
3. 영향을 받은 테스트를 실행하십시오.
4. run the full applicable regression;
4. 적용 가능한 전체 회귀를 실행합니다.
5. compare the same evidence and acceptance gates.
5. 동일한 증거와 수용 게이트를 비교합니다.

Never claim success from a stale snapshot.
오래된 스냅샷에서 성공했다고 주장하지 마세요.

## Creating self-checking testbenches
## 자체 점검 테스트벤치 생성

When the user requests a testbench or coverage improvement:
사용자가 테스트벤치 또는 적용 범위 개선을 요청하는 경우:

- derive legal stimulus and expected behavior from requirements, interfaces,
- 요구사항, 인터페이스로부터 법적 자극과 예상되는 동작을 도출합니다.
existing tests, and protocol documentation;
기존 테스트 및 프로토콜 문서화
- reuse existing clocks, resets, agents, scoreboards, and project conventions;
- 기존 시계, 재설정, 에이전트, 점수판 및 프로젝트 규칙을 재사용합니다.
- include deterministic checks with useful expected/actual messages;
- 유용한 예상/실제 메시지가 포함된 결정론적 검사를 포함합니다.
- emit one unambiguous terminal marker such as `TEST_PASS`;
- `TEST_PASS`와 같은 하나의 명확한 터미널 마커를 내보냅니다.
- include a watchdog that fails with the current phase and simulation time;
- 현재 단계 및 시뮬레이션 시간에 실패한 감시 기능을 포함합니다.
- avoid force/deposit/backdoor access unless the verification plan requires it;
- 검증 계획에서 요구하지 않는 한 강제/입금/백도어 액세스를 피하세요.
- keep tests deterministic by default; record seed and plusargs when randomized;
- 기본적으로 테스트를 결정적으로 유지합니다. 무작위로 추출할 때 시드와 플러스인수를 기록합니다.
- never weaken an assertion, checker, or acceptance gate to obtain PASS.
- PASS를 얻기 위해 주장, 검사기 또는 승인 게이트를 절대로 약화시키지 마십시오.

Generated testbenches are verification artifacts, not proof until they have
생성된 테스트벤치는 검증 아티팩트이며, 테스트가 완료될 때까지는 증거가 아닙니다.
been run successfully.
성공적으로 실행되었습니다.

## XSim coverage closure
## XSim 적용 범위 폐쇄

When coverage is requested, read
보험 적용을 요청하면 다음 내용을 읽어 보십시오.
[references/xsim-coverage-closure.md](references/xsim-coverage-closure.md) and
[references/xsim-coverage-closure.md](references/xsim-coverage-closure.md) 및
iterate:
반복하다:

1. run the unchanged regression and export baseline statement, branch,
1. 변경되지 않은 회귀를 실행하고 기준선 문, 분기를 내보냅니다.
condition, and toggle coverage;
조건 및 토글 적용 범위;
2. identify a reachable uncovered behavior and tie it to a requirement;
2. 도달 가능한 발견되지 않은 행동을 식별하고 이를 요구 사항에 연결합니다.
3. add focused stimulus plus a self-checking observation;
3. 집중된 자극과 자가 점검 관찰을 추가합니다.
4. rerun the targeted test and then the full regression;
4. 대상 테스트를 다시 실행한 다음 전체 회귀를 실행합니다.
5. export a new uniquely named coverage database/report and compare deltas;
5. 고유하게 명명된 새로운 적용 범위 데이터베이스/보고서를 내보내고 델타를 비교합니다.
6. repeat while coverage improves and regression correctness is retained.
6. 적용 범위가 향상되고 회귀 정확성이 유지되는 동안 반복합니다.

Do not game coverage.
게임 취재를 하지 마세요.
Never exclude reachable logic, remove checks, add DUT
도달 가능한 로직을 절대 제외하지 말고, 검사를 제거하고, DUT를 추가하세요.
backdoors, or change RTL solely to increase a metric.
백도어를 사용하거나 측정항목을 늘리기 위해 RTL만 변경하세요.
Separate untested,
테스트되지 않은 별도의
unreachable, and elaborated-out items.
도달할 수 없고 정교한 항목이 포함됩니다.
Ask before adding exclusions or changing
제외를 추가하거나 변경하기 전에 확인하세요.
RTL.
RTL.
Stop at the requested target, a justified plateau, or a blocking
요청된 목표, 정당한 고원 또는 차단 지점에서 정지합니다.
requirement/environment gap.
요구사항/환경 격차.

## XSim advanced verification
## XSim 고급 검증

For UVM, constrained-random, assertion-aware, or functional-coverage requests,
UVM, 제한된 무작위, 어설션 인식 또는 기능 적용 범위 요청의 경우
read [references/xsim-advanced-verification.md](references/xsim-advanced-verification.md).
[references/xsim-advanced-verification.md](references/xsim-advanced-verification.md)를 읽어보세요.

- create one manifest per test and seed;
- 테스트 및 시드당 하나의 매니페스트를 생성합니다.
- select the UVM test and record every plusarg;
- UVM 테스트를 선택하고 모든 plusarg를 기록합니다.
- preserve exact failing seeds and replay before diagnosis;
- 정확한 실패 시드를 보존하고 진단 전에 재생합니다.
- report immediate, concurrent, UVM, and checker results separately;
- 즉시, 동시, UVM 및 검사기 결과를 별도로 보고합니다.
- treat functional bins and crosses as requirement-linked targets;
- 기능적 빈과 교차를 요구사항과 연결된 대상으로 처리합니다.
- merge only compatible passing coverage databases;
- 호환 가능한 통과 범위 데이터베이스만 병합합니다.
- never ignore assertions/coverage or downgrade error/fatal severity to pass.
- 주장/적용 범위를 무시하거나 오류/치명적 심각도를 다운그레이드하여 통과하지 마십시오.

## XSim activity and portable stimulus
## XSim 활동 및 휴대용 자극

For SAIF, VCD, or sub-design replay, read
SAIF, VCD 또는 하위 디자인 재생의 경우 다음을 읽어보세요.
[references/xsim-activity-capture.md](references/xsim-activity-capture.md).
[참조/xsim-activity-capture.md](참조/xsim-activity-capture.md).

- separate reset/warm-up from the bounded measurement interval;
- 제한된 측정 간격과 별도의 재설정/예열;
- log only the intended hierarchy or object set;
- 의도한 계층 구조나 개체 집합만 기록합니다.
- close and verify every activity file;
- 모든 활동 파일을 닫고 확인합니다.
- require the simulation's independent functional PASS;
- 시뮬레이션의 독립적인 기능 PASS가 필요합니다.
- treat `create_testbench` output as scaffolding until it has an oracle,
- 오라클이 생길 때까지 'create_testbench' 출력을 스캐폴딩으로 처리합니다.
watchdog, terminal marker, and successful replay;
워치독, 터미널 마커 및 성공적인 재생;
- do not claim power signoff or cross-simulator portability from artifact
- 아티팩트에서 전원 사인오프 또는 시뮬레이터 간 이식성을 요구하지 마십시오.
generation alone.
세대 혼자.

## Result report
## 결과보고

Report:
보고:

```text
Backend and version:
Project / simulation set / top:
Mode and runtime:
Functional verdict: PASS | FAIL | INCONCLUSIVE
Failure class and first causal message:
Cross-simulator matrix (when arbitrated):
Known issue / Answer Record and whether it applied:
Workaround, verification result, and what it changes:
Tests run / passed / failed:
Coverage before -> after (if requested):
UVM test / seeds / replay:
Assertions passed / failed / disabled / unsupported:
Activity window and hierarchy:
Artifacts: transcript, waveform, coverage, SAIF, VCD, generated testbench/scripts
Changes made:
Not verified / exclusions / blockers:
```

Do not claim timing closure, CDC correctness, formal proof, equivalence,
타이밍 종료, CDC 정확성, 형식적 증명, 동등성을 주장하지 마십시오.
security, safety compliance, or hardware validation from simulation alone.
시뮬레이션만으로 보안, 안전 규정 준수 또는 하드웨어 검증을 수행할 수 있습니다.
