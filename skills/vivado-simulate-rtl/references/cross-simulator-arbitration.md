<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Cross-Simulator Arbitration
# 교차 시뮬레이터 중재

Decide whether a failure belongs to the design or to the simulator by running
다음을 실행하여 오류가 설계에 속하는지 아니면 시뮬레이터에 속하는지 결정합니다.
the same test on more than one backend.
둘 이상의 백엔드에서 동일한 테스트를 수행합니다.
This is an additional experiment, not
이는 추가 실험이 아닌
a backend substitution.
백엔드 대체.

## When to arbitrate
## 중재해야 할 때

- XSim crashes, hangs, or reports a fatal on source that looks legal.
- XSim이 충돌하거나 중단되거나 합법적인 소스에서 치명적인 오류를 보고합니다.
- A result disagrees with synthesis, hardware, or the user's expectation.
- 결과가 합성, 하드웨어 또는 사용자의 기대와 일치하지 않습니다.
- A construct is rejected as unsupported.
- 구성이 지원되지 않아 거부됩니다.
- An IP or example design fails to simulate.
- IP 또는 예시 설계가 시뮬레이션에 실패했습니다.

Do not arbitrate merely because a self-checking test failed honestly.
단순히 자체 점검 테스트가 정직하게 실패했다는 이유만으로 중재하지 마십시오.
A
reproducible functional mismatch with a credible checker is a design
신뢰할 수 있는 검사기와 재현 가능한 기능적 불일치가 디자인입니다.
investigation first.
먼저 조사.

## Preconditions
## 전제 조건

Arbitration needs a second simulator that the site actually provides:
중재에는 사이트에서 실제로 제공하는 두 번째 시뮬레이터가 필요합니다.

- user-supplied installation path and license configuration;
- 사용자 제공 설치 경로 및 라이센스 구성
- version-matched libraries from `compile_simlib` for that simulator;
- 해당 시뮬레이터에 대한 `compile_simlib`의 버전 일치 라이브러리;
- identical sources, include paths, defines, generics, top, timescale, and
- 동일한 소스, 경로 포함, 정의, 일반, 최상위, 시간 척도 및
seed;
종자;
- the same bounded runtime and the same terminal marker.
- 동일한 제한된 런타임과 동일한 터미널 마커.

Read the backend reference before running anything.
무엇이든 실행하기 전에 백엔드 참조를 읽어보세요.
Any backend that is
모든 백엔드
unavailable, unlicensed, or unsupported on this host is an `UNVERIFIED` cell,
이 호스트에서 사용할 수 없거나 라이센스가 없거나 지원되지 않는 셀은 'UNVERIFIED' 셀입니다.
never a pass and never a silent omission.
결코 통과하지도 않고 조용히 누락하지도 않습니다.

## Procedure
## 절차

1. Export scripts per simulator with the canonical identifier:
1. 표준 식별자를 사용하여 시뮬레이터별로 스크립트를 내보냅니다.

```tcl
export_simulation -simulator <xsim|questa|modelsim|vcs|xcelium|riviera|activehdl> \
  -directory <absolute-output-dir> \
  -lib_map_path <absolute-compiled-library-dir>
```

2. Run each generated compile, elaborate, and simulate flow unchanged.
2. 생성된 각 컴파일을 실행하고, 정교하게 만들고, 흐름을 시뮬레이션합니다.
3. Record, per backend: version, verdict, first causal message and stage, and
3. 백엔드별 기록: 버전, 판정, 첫 번째 원인 메시지 및 단계
the transcript path.
성적표 경로.
4. Build the matrix before interpreting any single cell.
4. 단일 셀을 해석하기 전에 행렬을 작성하십시오.

## Interpretation
## 해석

| Pattern<br>패턴 | Reading<br>독서 |
|---|---|
| Every backend fails the same way<br>모든 백엔드가 같은 방식으로 실패합니다. | Design or testbench defect. Diagnose normally.<br>설계 또는 테스트벤치 결함. 정상적으로 진단합니다. |
| Only XSim fails<br>XSim만 실패함 | Likely simulator defect. Reduce, then research.<br>시뮬레이터 결함일 가능성이 있습니다. 줄인 다음 연구하세요. |
| Only one third-party backend fails<br>하나의 타사 백엔드만 실패합니다. | That backend's setup or defect; check libraries and version support first.<br>해당 백엔드의 설정 또는 결함 먼저 라이브러리와 버전 지원을 확인하세요. |
| Verdicts agree, values differ<br>평결은 일치하지만 가치는 다릅니다 | Suspect timescale/resolution, X-propagation, optimization, uninitialized state, or a race.<br>의심스러운 시간/해상도, X-전파, 최적화, 초기화되지 않은 상태 또는 경합. |

A value difference caused by a race or by reliance on unspecified scheduling
경쟁이나 불특정 일정에 따른 가치 차이
is a testbench defect, even though the simulators disagree.
시뮬레이터가 동의하지 않더라도 테스트벤치 결함입니다.
Fix the test, do
테스트를 수정하고
not file a tool bug.
도구 버그를 신고하지 마세요.

Majority vote is evidence, not proof of LRM correctness.
다수결은 LRM 정확성에 대한 증거가 아니라 증거입니다.
When you claim one
하나를 청구할 때
simulator is wrong, cite the LRM, a user guide, or an Answer Record found via
시뮬레이터가 잘못되었습니다. LRM, 사용자 가이드 또는 다음을 통해 찾은 답변 기록을 인용하세요.
[known-issue-research.md](known-issue-research.md).
[알려진-문제-연구.md](알려진-문제-연구.md).

## Prohibited
## 금지

- Switching the user's requested backend because another one passes.
- 다른 백엔드가 통과했기 때문에 사용자가 요청한 백엔드를 전환합니다.
- Reporting a backend as passing when its license or host blocked the run.
- 라이센스나 호스트가 실행을 차단한 경우 백엔드를 통과한 것으로 보고합니다.
- Changing sources, defines, or runtime between backends and still calling the
- 백엔드 간 소스, 정의 또는 런타임을 변경하고 계속 호출
comparison a controlled experiment.
통제된 실험을 비교합니다.
- Concealing that a "workaround" backend produces different coverage,
- "해결 방법" 백엔드가 다른 적용 범위를 생성한다는 사실을 숨기고,
waveform, or performance characteristics.
파형 또는 성능 특성.

## Report
## 신고

```text
Design / simset / top / mode:
Contract and terminal marker:
Runtime bound / seed / defines:

| Simulator | Version | Verdict | First causal message | Log |
|---|---|---|---|---|

Reading:
Cited source for the correctness claim:
Recommended next step:
Unverified cells and why:
```

## Official sources
## 공식 출처

- [UG900 supported simulators](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 지원 시뮬레이터](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 export_simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG900 내보내기_시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG973 compatible third-party tools](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
- [UG973 호환 타사 도구](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
