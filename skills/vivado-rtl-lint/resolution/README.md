<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# RTL Lint Resolution Guides
# RTL 린트 해결 가이드

Per-violation fix references for `synth_design -lint`.
`synth_design -lint`에 대한 위반별 수정 참조입니다.
Each file provides the Vivado error pattern, root cause, fix options with code examples, and validation TCL.
각 파일은 Vivado 오류 패턴, 근본 원인, 코드 예제가 포함된 수정 옵션 및 검증 TCL을 제공합니다.

## Available Guides
## 사용 가능한 가이드

| Rule<br>규칙 | Severity<br>심각도 | Description<br>설명 | Fix<br>수정 |
|------|----------|-------------|-----|
| [ASSIGN-1](ASSIGN-1.md)<br>[ASSIGN-1](ASSIGN-1.md) | WARNING<br>경고 | Arithmetic overflow / truncation<br>산술 오버플로/잘림 | Widen output or saturate<br>출력을 넓히거나 포화시키세요 |
| [ASSIGN-2](ASSIGN-2.md)<br>[ASSIGN-2](ASSIGN-2.md) | CRITICAL<br>심각 | Mixed signed/unsigned arithmetic<br>부호 있는/부호 없는 혼합 산술 | `$signed()` casting<br>`$signed()` 캐스팅 |
| [ASSIGN-3](ASSIGN-3.md)<br>[ASSIGN-3](ASSIGN-3.md) | CRITICAL<br>심각 | Shift amount exceeds width<br>이동량이 너비를 초과함 | Correct parameter or add validation<br>매개변수 수정 또는 유효성 검사 추가 |
| [ASSIGN-5](ASSIGN-5.md)<br>[ASSIGN-5](ASSIGN-5.md) | WARNING<br>경고 | Signal used but not assigned<br>사용되었지만 할당되지 않은 신호 | Remove or add driver<br>드라이버 제거 또는 추가 |
| [ASSIGN-6](ASSIGN-6.md)<br>[ASSIGN-6](ASSIGN-6.md) | WARNING<br>경고 | Signal assigned but not read<br>신호가 할당되었으나 읽히지 않음 | Remove dead code<br>데드 코드 제거 |
| [ASSIGN-7](ASSIGN-7.md)<br>[ASSIGN-7](ASSIGN-7.md) | CRITICAL<br>심각 | Multiple drivers on same bits<br>동일한 비트의 여러 드라이버 | Non-overlapping bit ranges<br>겹치지 않는 비트 범위 |
| [ASSIGN-8](ASSIGN-8.md)<br>[ASSIGN-8](ASSIGN-8.md) | CRITICAL<br>심각 | Array dimension mismatch in comparison<br>비교 시 배열 차원 불일치 | Match operand dimensions<br>피연산자 차원 일치 |
| [ASSIGN-9](ASSIGN-9.md)<br>[ASSIGN-9](ASSIGN-9.md) | WARNING<br>경고 | Unassigned output port bits<br>할당되지 않은 출력 포트 비트 | Remove port or complete assignment<br>포트 제거 또는 할당 완료 |
| [ASSIGN-10](ASSIGN-10.md)<br>[ASSIGN-10](ASSIGN-10.md) | WARNING<br>경고 | Unused I/O port bits<br>사용되지 않은 I/O 포트 비트 | Remove port or reduce width<br>포트를 제거하거나 너비를 줄이세요 |
| [ASSIGN-12](ASSIGN-12.md)<br>[ASSIGN-12](ASSIGN-12.md) | CRITICAL<br>심각 | Port connected to tri-state (1'bz)<br>3상태(1'bz)에 연결된 포트 | Tie to constant<br>상수에 연결 |
| [ASSIGN-14](ASSIGN-14.md)<br>[ASSIGN-14](ASSIGN-14.md) | CRITICAL<br>심각 | Duplicate case branches<br>중복된 사례 분기 | Remove duplicates or correct values<br>중복된 값을 제거하거나 값을 수정하세요. |
| [INFER-1](INFER-1.md)<br>[INFER-1](INFER-1.md) | CRITICAL<br>심각 | Latch inference<br>래치 추론 | Add else/default or convert to sequential<br>else/default를 추가하거나 순차로 변환하세요. |
| [INFER-2](INFER-2.md)<br>[INFER-2](INFER-2.md) | CRITICAL<br>심각 | Incomplete case statement<br>불완전한 사례 진술 | Add default clause<br>기본 조항 추가 |
| [INFER-3](INFER-3.md)<br>[INFER-3](INFER-3.md) | INFO<br>정보 | `===` converted to `==` for synthesis<br>합성을 위해 `===`를 `==`로 변환 | Replace with `==` or guard with translate_off<br>`==`로 바꾸거나 Translate_off로 보호 |
| [CLOCK-1](CLOCK-1.md)<br>[시계-1](시계-1.md) | CRITICAL<br>심각 | Mixed clock edges (posedge/negedge)<br>혼합 클록 에지(posedge/negedge) | Unify to single edge<br>단일 엣지로 통합 |
| [QOR-1](QOR-1.md)<br>[QOR-1](QOR-1.md) | WARNING<br>경고 | Arithmetic precision loss (operator merging)<br>산술 정밀도 손실(연산자 병합) | Preserve full width or merge operators<br>전체 너비 유지 또는 병합 연산자 |
| [RESET-1](RESET-1.md)<br>[리셋-1](리셋-1.md) | CRITICAL<br>심각 | Multiple asynchronous resets<br>다중 비동기 재설정 | Single async reset<br>단일 비동기 재설정 |
| [RESET-2](RESET-2.md)<br>[리셋-2](리셋-2.md) | WARNING<br>경고 | Incomplete async reset coverage<br>불완전한 비동기 재설정 적용 범위 | Reset all registers in block<br>블록의 모든 레지스터를 재설정합니다. |
| [RESET-3](RESET-3.md)<br>[리셋-3](리셋-3.md) | CRITICAL<br>심각 | Incomplete sync reset / enable driven by sync reset<br>불완전한 동기화 재설정/동기화 재설정으로 인한 활성화 | Reset all registers in block<br>블록의 모든 레지스터를 재설정합니다. |

## Adding New Guides
## 새 가이드 추가

Create `<RULE-ID>.md` with: Vivado message pattern, root cause, fix options (before/after code), validation TCL, and references.
Vivado 메시지 패턴, 근본 원인, 수정 옵션(코드 전/후), 유효성 검사 TCL 및 참조를 포함하여 `<RULE-ID>.md`를 생성합니다.
See any existing guide as a template.
기존 가이드를 템플릿으로 확인하세요.
