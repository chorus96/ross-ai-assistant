<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Adding a Simulator Backend
# 시뮬레이터 백엔드 추가

Add a backend only after the installed Vivado release officially supports it.
설치된 Vivado 릴리스가 공식적으로 지원한 후에만 백엔드를 추가하세요.
Do not infer support from a compatible HDL command line.
호환되는 HDL 명령줄에서 지원을 추론하지 마세요.

## 1.
## Verify support
## 지원 확인

Search current AMD documentation with `Vivado:vivado_doc_search`:
`Vivado:vivado_doc_search`로 현재 AMD 문서를 검색하세요.

- UG900 Supported Simulators
- UG900 지원 시뮬레이터
- UG900 `export_simulation`
- UG900 `수출_시뮬레이션`
- UG900 third-party simulator setup
- UG900 타사 시뮬레이터 설정
- UG973 Compatible Third-Party Tools
- UG973 호환 타사 도구

Record:
기록:

```text
Vivado release:
Simulator product and executable-reported version:
Canonical export_simulation identifier:
Supported host platforms:
Integrated launch support:
Behavioral / post-synthesis / post-implementation support:
Source URLs:
```

Stop if the backend is absent or the version/platform is unsupported.
백엔드가 없거나 버전/플랫폼이 지원되지 않으면 중지하세요.
Do not
하지 마십시오
describe experimental command compatibility as supported integration.
실험적인 명령 호환성을 지원되는 통합으로 설명합니다.

## 2.
## Implement the backend contract
## 백엔드 계약 구현

Create a direct reference linked from `SKILL.md` with these sections:
다음 섹션을 사용하여 `SKILL.md`에서 연결된 직접 참조를 만듭니다.

### File organization
### 파일 구성

Give a fully supported backend its own `references/<simulator>.md` when it has
완전히 지원되는 백엔드에 자체 `references/<simulator>.md`를 제공하세요.
distinct setup, launch, artifact, coverage, or diagnostic behavior.
고유한 설정, 실행, 아티팩트, 적용 범위 또는 진단 동작.
Keep
유지
closely related, lightly supported backends in
밀접하게 관련되어 있고 약간 지원되는 백엔드
`references/other-vivado-simulators.md` only while each remains concise and
`references/other-vivado-simulators.md`는 각각 간결하고
unambiguous.
모호하지 않다.
Every reference must be linked directly from `SKILL.md`; do not
모든 참조는 `SKILL.md`에서 직접 연결되어야 합니다. 하지 마십시오
make one reference the only route to another.
하나의 참조를 다른 참조에 대한 유일한 경로로 만듭니다.

### Discovery triggers
### 발견 트리거

Product names, aliases, executable names, and explicit user phrases that select
선택하는 제품 이름, 별칭, 실행 파일 이름 및 명시적인 사용자 문구
the backend.
백엔드.
Never select it merely because another backend is unavailable.
단순히 다른 백엔드를 사용할 수 없다는 이유만으로 선택하지 마세요.

### Prerequisites
### 전제조건

- installation and executable version check
- 설치 및 실행 버전 확인
- license environment
- 라이센스 환경
- supported host
- 지원되는 호스트
- version-matched `compile_simlib` output
- 버전이 일치하는 `compile_simlib` 출력
- required compiler/runtime environment
- 필수 컴파일러/런타임 환경
- supported HDL, encrypted IP, and mixed-language constraints
- 지원되는 HDL, 암호화된 IP 및 혼합 언어 제약 조건

Site-specific paths must come from the user or existing project configuration.
사이트별 경로는 사용자 또는 기존 프로젝트 구성에서 가져와야 합니다.

### Vivado integration
### 비바도 통합

- exact `target_simulator` value
- 정확한 `target_simulator` 값
- canonical `export_simulation -simulator` identifier
- 표준 `export_simulation -simulator` 식별자
- project and simulation-fileset properties
- 프로젝트 및 시뮬레이션 파일 세트 속성
- `compile_simlib` simulator identifier
- `compile_simlib` 시뮬레이터 식별자
- supported launch/export modes
- 지원되는 실행/내보내기 모드

Verify each name against command help or official docs for the target release.
대상 릴리스의 명령 도움말이나 공식 문서를 통해 각 이름을 확인하세요.

### Execution
### 실행

- integrated launch behavior
- 통합 실행 동작
- generated-script directory and entry points
- 생성된 스크립트 디렉터리 및 진입점
- compile, elaborate/optimize, and simulate order
- 순서를 컴파일하고, 정교하게/최적화하고, 시뮬레이션합니다.
- bounded runtime and safe termination
- 제한된 런타임 및 안전한 종료
- environment that must be active in the execution shell
- 실행 셸에서 활성화되어야 하는 환경
- timeout and cancellation behavior
- 시간 초과 및 취소 동작

Prefer Vivado-generated scripts over handwritten compiler commands.
손으로 작성한 컴파일러 명령보다 Vivado가 생성한 스크립트를 선호합니다.

### Artifacts
### 유물

- per-stage logs
- 단계별 로그
- waveform/database format
- 파형/데이터베이스 형식
- generated scripts and library map
- 생성된 스크립트 및 라이브러리 맵
- coverage support and format, if any
- 적용 범위 지원 및 형식(있는 경우)
- seed, plusargs, generics, parameters, and defines
- 시드, plusargs, 제네릭, 매개변수 및 정의

### Verdict mapping
### 평결 매핑

Document exact evidence for:
다음에 대한 정확한 증거를 문서화하십시오.

- analysis failure
- 분석 실패
- elaboration failure
- 정교화 실패
- runtime assertion/error/fatal
- 런타임 어설션/오류/치명적
- timeout
- 시간 초과
- normal simulator exit
- 일반 시뮬레이터 종료
- explicit functional PASS
- 명시적 기능 통과

Normal exit alone must not map to functional PASS.
일반 종료만으로는 기능적 PASS에 매핑되어서는 안 됩니다.

### Diagnostics
### 진단

Include backend-specific license, library, encrypted-IP, mixed-language,
백엔드별 라이선스, 라이브러리, 암호화된 IP, 혼합 언어,
optimization, SDF, waveform, and exit-code failure signatures.
최적화, SDF, 파형 및 종료 코드 오류 서명.

## 3.
## Add evaluation coverage
## 평가 범위 추가

Add scenarios to the skill's evaluation suite for:
다음에 대한 기술 평가 모음에 시나리오를 추가합니다.

1. missing installation or license;
1. 설치 또는 라이센스가 누락되었습니다.
2. missing/stale compiled AMD libraries;
2. 누락되거나 오래된 컴파일된 AMD 라이브러리;
3. behavioral PASS with transcript and waveform;
3. 성적표와 파형이 포함된 행동 PASS;
4. assertion failure classification;
4. 주장 실패 분류;
5. one supported netlist/timing mode, if applicable;
5. 해당되는 경우 하나의 지원되는 넷리스트/타이밍 모드;
6. unsupported platform/version rejection.
6. 지원되지 않는 플랫폼/버전 거부.

Fixtures must be deterministic and self-checking.
설비는 결정적이고 자체 점검이 가능해야 합니다.
Record expected artifacts and
예상되는 아티팩트를 기록하고
forbidden fallback behavior.
금지된 대체 동작.

## 4.
## Validate
## 검증

Before claiming support:
지원을 요청하기 전에:

- validate frontmatter, links, and canonical identifiers;
- 머리말, 링크, 정식 식별자를 검증합니다.
- run generated-script smoke tests on every claimed host/backend combination
- 청구된 모든 호스트/백엔드 조합에 대해 생성된 스크립트 스모크 테스트 실행
available to the team;
팀에서 사용할 수 있습니다.
- test at least one AMD primitive or IP requiring compiled libraries;
- 컴파일된 라이브러리가 필요한 최소한 하나의 AMD 프리미티브 또는 IP를 테스트합니다.
- verify timeout handling;
- 시간 초과 처리를 확인합니다.
- verify an assertion produces FAIL and a terminal marker produces PASS;
- 어설션이 FAIL을 생성하고 터미널 마커가 PASS를 생성하는지 확인합니다.
- confirm artifacts exist at reported paths;
- 보고된 경로에 아티팩트가 존재하는지 확인합니다.
- test skill discovery on the intended model classes.
- 의도한 모델 클래스에 대한 기술 발견을 테스트합니다.

Mark combinations without an installed/licensed simulator as `UNVERIFIED`.
설치/라이센스가 부여된 시뮬레이터가 없는 조합은 '확인되지 않음'으로 표시합니다.

## 5.
## Keep release data out of the core
## 릴리스 데이터를 코어 외부로 유지

Do not add version numbers, host matrices, environment variables, or
버전 번호, 호스트 매트릭스, 환경 변수 또는
backend-specific commands to the common workflow unless they are universal.
범용적인 것이 아닌 이상 백엔드별 명령을 일반 워크플로에 적용합니다.
Keep them in the backend reference so a release update changes one file and its
릴리스 업데이트가 하나의 파일과 해당 파일을 변경하도록 백엔드 참조에 보관하십시오.
evaluations.
평가.

When AMD changes support:
AMD가 지원을 변경하는 경우:

1. update the release snapshot and source links;
1. 릴리스 스냅샷 및 소스 링크를 업데이트합니다.
2. rerun backend evaluations;
2. 백엔드 평가를 다시 실행합니다.
3. preserve obsolete guidance only in a clearly labeled legacy section when
3. 명확하게 라벨이 붙은 레거시 섹션에서만 쓸모없는 지침을 보존하십시오.
users still need it;
사용자에게는 여전히 필요합니다.
4. remove the support claim if validation no longer passes.
4. 검증이 더 이상 통과되지 않으면 지원 요청을 제거하십시오.
