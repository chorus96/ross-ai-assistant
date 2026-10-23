<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Headless Waveform Evidence
# 헤드리스 파형 증거

Produce and compare waveform evidence without a GUI.
GUI 없이 파형 증거를 생성하고 비교합니다.
This keeps diagnosis
이렇게 하면 진단이 유지됩니다.
working on batch hosts with no display, and it keeps a viewer or rendering
디스플레이가 없는 일괄 호스트에서 작업하고 뷰어 또는 렌더링을 유지합니다.
defect from blocking a verdict.
판결을 방해하는 결함.

## Capture
## 캡처

Log a focused object set before advancing simulation, then run the bounded
시뮬레이션을 진행하기 전에 집중된 개체 세트를 기록한 다음 제한된 개체를 실행합니다.
test.
테스트.
`log_wave` controls what enters the database; `add_wave` only controls
`log_wave`는 데이터베이스에 입력되는 내용을 제어합니다. `add_wave`만 제어
display and is unnecessary in batch.
표시되며 일괄적으로 불필요합니다.

For a portable, tool-independent trace, dump VCD instead of or alongside the
이식 가능하고 도구 독립적인 추적의 경우 VCD 대신 또는 함께 VCD를 덤프하십시오.
native database:
기본 데이터베이스:

```tcl
open_vcd <absolute-path>/<target>.vcd
log_vcd [get_objects <focused-signal-set>]
run <bounded-duration>
flush_vcd
close_vcd
```

Verify every artifact exists and is non-empty before citing it.
인용하기 전에 모든 아티팩트가 존재하고 비어 있지 않은지 확인하십시오.
A waveform
파형
path is not evidence until it has been opened or parsed.
path는 열리거나 구문 분석될 때까지 증거가 아닙니다.

## Compare programmatically
## 프로그래밍 방식으로 비교

VCD is plain text, so two runs can be compared directly: baseline versus
VCD는 일반 텍스트이므로 두 실행을 직접 비교할 수 있습니다.
current, one simulator versus another, or one tool release versus another.
현재, 하나의 시뮬레이터와 다른 시뮬레이터 또는 하나의 도구 릴리스와 다른 도구 릴리스.

Compare like for like.
같은 것을 비교하십시오.
The signal set, hierarchy names, timescale, and time
신호 세트, 계층 구조 이름, 시간 척도 및 시간
window must match, and the runs must use the same seed and stimulus.
창이 일치해야 하며 실행은 동일한 시드와 자극을 사용해야 합니다.
Report
신고
the first divergence time and the signal that diverged; later differences are
첫 번째 발산 시간과 발산된 신호; 나중에 차이점은
usually consequences of it.
일반적으로 그 결과.

Expect benign differences that are not defects: unlogged versus logged
결함이 아닌 양성 차이 예상: 기록되지 않은 것과 기록된 것
objects, X-propagation before initialization, and delta-cycle ordering within
객체, 초기화 전 X 전파 및 델타 사이클 순서
the same simulation time.
같은 시뮬레이션 시간.
Compare at settled points rather than mid-delta
중간 델타가 아닌 안정된 지점에서 비교
unless the question is specifically about scheduling.
질문이 구체적으로 일정에 관한 것이 아니라면.

## When the GUI is the problem
## GUI가 문제인 경우

If a waveform viewer, zoom rendering, or ILA import misbehaves, do not accept
파형 뷰어, 확대/축소 렌더링 또는 ILA 가져오기가 오작동하는 경우 허용하지 마십시오.
it as a simulation failure.
시뮬레이션 실패로 간주됩니다.
Re-derive the same fact from the batch transcript
배치 기록에서 동일한 사실을 다시 도출합니다.
and the dumped trace.
그리고 버려진 흔적.
A display defect that does not change the logged values
기록된 값을 변경하지 않는 디스플레이 결함
is cosmetic, and the functional verdict stands on the transcript and terminal
미용적이며 기능적 판단은 성적표와 터미널에 있습니다.
marker.
마커.

Value queries also work without a viewer:
값 쿼리는 뷰어 없이도 작동합니다.

```tcl
get_value -radix hex <object>
report_objects <scope>
```

## Uses
## 용도

- Regression evidence in CI where no display exists.
- 디스플레이가 존재하지 않는 CI의 회귀 증거입니다.
- Golden-versus-current comparison after a tool upgrade, alongside
- 도구 업그레이드 후 황금 대 현재 비교
[version-bisect.md](version-bisect.md).
[버전-bisect.md](버전-bisect.md).
- Cross-simulator value comparison feeding
- 시뮬레이터 간 값 비교 공급
[cross-simulator-arbitration.md](cross-simulator-arbitration.md).
[교차 시뮬레이터-중재.md](교차 시뮬레이터-중재.md).

## Prohibited
## 금지

- Presenting a waveform as proof of correctness.
- 정확성의 증거로 파형을 제시합니다.
- A trace shows behavior; the
- 추적은 동작을 보여줍니다. 는
checker and terminal marker decide PASS.
체커와 터미널 마커가 PASS를 결정합니다.
- Recursively dumping the entire design by default.
- 기본적으로 전체 디자인을 재귀적으로 덤프합니다.
- It is slow, large, and
- 느리고, 크고,
obscures the causal signals.
원인 신호를 모호하게 만듭니다.
- Comparing traces captured with different logging scope and calling the
- 다양한 로깅 범위로 캡처된 추적을 비교하고
differences a regression.
차이점은 회귀입니다.
