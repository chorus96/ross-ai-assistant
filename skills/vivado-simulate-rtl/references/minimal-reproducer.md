<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Minimal Reproducer and Defect Package
# 최소 재생산 및 결함 패키지

A tool defect is only actionable with a small, self-contained case and an
도구 결함은 작고 독립적인 케이스와
exact command.
정확한 명령.
Building that case is usually the expensive part, and it is
그 케이스를 만드는 것은 일반적으로 비용이 많이 드는 부분입니다.
work this skill can do.
이 기술로 할 수 있는 일.

## Invariant
## 불변

The failure signature is the thing being preserved: the same message
실패 서명은 보존되는 것입니다: 동일한 메시지
identifier, at the same stage, from the same tool.
동일한 도구의 동일한 단계에 있는 식별자입니다.
Record it before reducing
줄이기 전에 녹음하세요
anything.
무엇이든.

If the signature changes during reduction, you have moved into a different
축소 중에 서명이 변경되면 다른 위치로 이동한 것입니다.
problem.
문제.
Undo the last step.
마지막 단계를 실행 취소하세요.

## Rules
## 규칙

Work on a copy.
사본을 작업하십시오.
Never reduce the user's golden project in place.
사용자의 황금 프로젝트를 제자리에서 축소하지 마십시오.
Reduction
감소
deletes design content by intent, so it is destructive by nature.
의도적으로 디자인 콘텐츠를 삭제하므로 본질적으로 파괴적입니다.

Keep a written ablation record: what was removed, and whether the signature
서면 절제 기록 보관: 제거된 내용 및 서명 여부
survived.
살아 남았습니다.
A reproducer without that record cannot be called minimal.
그 기록이 없는 재생산은 미니멀이라 할 수 없다.

## Reduction ladder
## 감소사다리

Work coarse to fine and rerun after each step:
각 단계 후에 거칠게 작업하고 다시 실행하세요.

1. Remove unrelated simulation sets, tests, and top-level stimulus.
1. 관련되지 않은 시뮬레이션 세트, 테스트 및 최상위 자극을 제거합니다.
2. Cut the hierarchy to the smallest subtree that still fails.
2. 여전히 실패하는 가장 작은 하위 트리로 계층 구조를 잘라냅니다.
3. Replace IP with a stub only if the signature survives.
3. 서명이 남아 있는 경우에만 IP를 스텁으로 바꾸십시오.
3. For SecureIP,
3. SecureIP의 경우,
GT, or encrypted models the IP is often essential to the bug; say so
GT 또는 암호화된 모델의 경우 IP가 버그에 필수적인 경우가 많습니다. 그렇게 말해
instead of stubbing it away.
그것을 뭉개는 대신.
4. Drop files not referenced by the surviving subtree.
4. 살아남은 하위 트리에서 참조하지 않는 파일을 삭제합니다.
5. Shrink parameters, generics, widths, and array bounds.
5. 매개변수, 제네릭, 너비 및 배열 경계를 축소합니다.
6. Inline or trim packages down to the declarations actually used.
6. 실제로 사용된 선언까지 패키지를 인라인하거나 다듬습니다.
7. For an elaboration crash, remove the testbench body entirely.
7. 정교하게 충돌이 발생하는 경우 테스트벤치 본체를 완전히 제거하십시오.
7. Elaboration
7. 정교함
failures need no oracle.
실패에는 오라클이 필요하지 않습니다.

Stop when removing any remaining element makes the signature disappear.
나머지 요소를 제거하면 중지하여 서명이 사라집니다.

## Oracle requirement
## 오라클 요구 사항

An elaboration or compile crash reproduces on the command alone.
정교화 또는 컴파일 충돌은 명령에서만 재현됩니다.

A wrong-results case is different: the package must contain a self-checking
잘못된 결과의 경우는 다릅니다. 패키지에는 자체 점검 항목이 포함되어 있어야 합니다.
oracle that states the expected value and prints an unambiguous terminal
예상 값을 명시하고 명확한 터미널을 인쇄하는 oracle
marker.
마커.
Without it, the reader cannot tell a tool defect from a disagreement
그것이 없으면 독자는 도구 결함과 불일치를 구분할 수 없습니다.
about intent.
의도에 대해.

## Package contents
## 패키지 내용

- reduced sources and any required data files;
- 축소된 소스 및 필요한 데이터 파일
- the exact `xvlog` / `xvhdl` / `xelab` / `xsim` commands, or the generated
- 정확한 `xvlog` / `xvhdl` / `xelab` / `xsim` 명령 또는 생성된
scripts;
스크립트;
- tool version banner as printed by the tool, not from memory;
- 메모리가 아닌 도구에 의해 인쇄된 도구 버전 배너.
- host OS and distribution, and whether the host is supported;
- 호스트 OS 및 배포판, 호스트 지원 여부
- library mapping and `-lib_map_path` used;
- 라이브러리 매핑 및 `-lib_map_path`가 사용되었습니다.
- complete stage logs, not tails;
- 꼬리가 아닌 완전한 단계 로그;
- the cross-simulator matrix from
- 크로스 시뮬레이터 매트릭스
[cross-simulator-arbitration.md](cross-simulator-arbitration.md);
[교차 시뮬레이터-중재.md](교차 시뮬레이터-중재.md);
- research findings from
- 연구 결과
[known-issue-research.md](known-issue-research.md), including the Answer
[known-issue-research.md](known-issue-research.md), 답변 포함
Record checked and whether it applied;
기록 확인 및 적용 여부
- workaround status and what it changes;
- 해결 방법 상태 및 변경 사항
- a single runnable script that reproduces the failure from a clean directory.
- 깨끗한 디렉터리에서 오류를 재현하는 실행 가능한 단일 스크립트.

Verify the package by running that script in a fresh directory before handing
처리하기 전에 새 디렉터리에서 해당 스크립트를 실행하여 패키지를 확인하세요.
it over.
끝났어.
An unverified package wastes a triage cycle.
확인되지 않은 패키지는 분류 주기를 낭비합니다.

## Confidentiality
## 기밀성

Reduce proprietary RTL to the minimum that still reproduces the signature, and
서명을 재현하는 최소한의 독점 RTL을 줄이고,
tell the user what remains in the package.
사용자에게 패키지에 무엇이 남아 있는지 알려줍니다.
Do not upload, attach, or transmit
업로드, 첨부, 전송하지 마세요.
customer sources anywhere.
어디서나 고객 소스.
Preparing a package is not permission to file it.
패키지를 준비하는 것은 파일을 제출할 수 있는 권한이 아닙니다.

## Report
## 신고

```text
Signature preserved:
Original size -> reduced size (files, lines, hierarchy depth):
Ablation record:
Reproduce command:
Tool version / host:
Cross-simulator matrix:
Known-issue research result:
Workaround and its cost:
Package path and clean-directory verification:
```
