<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Tool Version Bisect
# 도구 버전 이등분

Identify the release where a working simulation started failing.
작동 중인 시뮬레이션이 실패하기 시작한 릴리스를 식별합니다.
A regression
회귀
with a named first-bad version is far more actionable than "it broke", and the
이름이 지정된 첫 번째 불량 버전을 사용하는 것이 "깨졌음"보다 훨씬 실행 가능성이 높습니다.
last-good release is an immediate fallback for the user.
마지막으로 좋은 릴리스는 사용자를 위한 즉각적인 대체입니다.

## When to bisect
## 양분할 때

- The user says it worked in an earlier Vivado release.
- 사용자는 이전 Vivado 릴리스에서 작동했다고 말합니다.
- The failure signature appeared after an upgrade.
- 업그레이드 후에 실패 서명이 나타났습니다.
- An Answer Record claims a fix in a release where the failure still occurs.
- 응답 기록에서는 오류가 계속 발생하는 릴리스의 수정 사항을 요구합니다.

Bisect needs at least two installed releases.
Bisect에는 설치된 릴리스가 두 개 이상 필요합니다.
If only one is installed, say
하나만 설치된 경우 다음과 같이 말합니다.
so; do not ask the user to install tools as a precondition without approval.
그래서; 사용자에게 승인 없이 전제 조건으로 도구를 설치하도록 요청하지 마십시오.

## Controlled variables
## 통제변수

Hold everything except the tool version constant:
도구 버전을 제외한 모든 항목을 일정하게 유지합니다.

- the same source set, defines, generics, include paths, and top;
- 동일한 소스 세트, 정의, 제네릭, 포함 경로 및 상단;
- the same bounded runtime, seed, and terminal marker;
- 동일한 제한된 런타임, 시드 및 터미널 마커
- the same mode, behavioral or post-implementation;
- 동일한 모드, 동작 또는 구현 후;
- freshly compiled third-party libraries per release, because
- 릴리스별로 새로 컴파일된 타사 라이브러리
`compile_simlib` output is version-specific;
`compile_simlib` 출력은 버전별로 다릅니다.
- a separate working directory per release, so stale `xsim.dir` artifacts
- 릴리스마다 별도의 작업 디렉토리가 있으므로 오래된 `xsim.dir` 아티팩트
cannot leak between runs.
실행 사이에 누출될 수 없습니다.

## Procedure
## 절차

1. Confirm the signature in the current release.
1. 현재 릴리스의 서명을 확인하십시오.
2. Find one release where the test passes.
2. 테스트를 통과한 릴리스 하나를 찾으세요.
2. Check the oldest available install
2. 사용 가능한 가장 오래된 설치를 확인하세요.
first; if it also fails, the regression is older than the installed set and
먼저; 이것도 실패하면 회귀는 설치된 세트보다 오래되었으며
the bisect cannot conclude.
이분법은 결론을 내릴 수 없습니다.
3. Order the installed releases and bisect between last-good and first-bad.
3. 설치된 릴리스를 주문하고 마지막 좋은 버전과 첫 번째 나쁜 버전을 이등분합니다.
4. Rerun the identical bounded test at each step and compare the signature,
4. 각 단계에서 동일한 제한된 테스트를 다시 실행하고 서명을 비교합니다.
not just pass or fail.
단지 합격이나 불합격이 아닙니다.
5. Report the first bad release, the last good release, and the exact delta in
5. 첫 번째 잘못된 릴리스, 마지막으로 좋은 릴리스 및 정확한 델타를 보고합니다.
behavior.
행동.

## Confounders
## 혼란스러운 사람들

Project upgrade prompts, regenerated IP output products, and migrated
프로젝트 업그레이드 프롬프트, IP 출력 제품 재생성 및 마이그레이션
constraints can change the design between releases.
제약 조건은 릴리스 간에 디자인을 변경할 수 있습니다.
Prefer a non-project
프로젝트가 아닌 것을 선호합니다
source list or generated scripts so the sources are identical.
소스 목록 또는 생성된 스크립트를 사용하여 소스가 동일하도록 합니다.
When a project
프로젝트를 할 때
upgrade was unavoidable, state it; the result is then a release-plus-upgrade
업그레이드가 불가피했다고 명시해 주세요. 그 결과는 릴리스 + 업그레이드입니다.
comparison, not a pure tool bisect.
비교는 순수한 도구 이분법이 아닙니다.

An IP version change between releases is a different root cause from a
릴리스 간 IP 버전 변경은 다른 근본 원인입니다.
simulator change.
시뮬레이터 변경.
Record the IP versions used in each run.
각 실행에 사용된 IP 버전을 기록합니다.

## After the bisect
## 이분법 이후

Feed the first-bad release into
첫 번째 불량 릴리스를 다음에 공급합니다.
[known-issue-research.md](known-issue-research.md).
[알려진-문제-연구.md](알려진-문제-연구.md).
A regression with a known
알려진 회귀
release boundary often matches an existing Answer Record or a fixed-in note,
릴리스 경계는 기존 답변 기록이나 고정 메모와 일치하는 경우가 많습니다.
and it strengthens a new defect report considerably.
새로운 결함 보고서를 상당히 강화합니다.

## Report
## 신고

```text
Signature under test:
Releases evaluated (in order):
Last good / first bad:
Delta observed at the boundary:
Sources held constant: yes | no (explain)
IP or project upgrade involved:
Fallback recommendation:
Research result for the first bad release:
```
