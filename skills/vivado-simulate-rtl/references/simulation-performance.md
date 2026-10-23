<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Simulation Performance
# 시뮬레이션 성능

Answer "where does the time go" with measurements, then reduce the dominant
측정을 통해 "시간은 어디로 가는가"에 답한 다음 우세한 시간을 줄입니다.
stage.
무대.
Do not speculate about simulator speed.
시뮬레이터 속도에 대해 추측하지 마십시오.

## Measure first
## 먼저 측정하세요

Record wall time separately for each stage:
각 단계마다 벽 시간을 별도로 기록합니다.

- analysis: `xvlog` and `xvhdl`;
- 분석: `xvlog` 및 `xvhdl`;
- elaboration: `xelab`;
- 정교화: `xelab`;
- runtime: `xsim` up to the terminal marker.
- 런타임: `xsim`을 터미널 마커까지.

Report the dominant stage and its share.
지배적인 단계와 그 점유율을 보고합니다.
An elaboration problem and a runtime
정교화 문제와 런타임
problem have no remedies in common, and most "simulation is slow" reports are
문제에는 공통적인 해결 방법이 없으며 대부분의 "시뮬레이션이 느립니다" 보고서는
one or the other, not both.
둘 중 하나가 아니라 둘 중 하나입니다.

Also record simulated time reached, wall time per simulated microsecond, peak
또한 도달한 시뮬레이션 시간, 시뮬레이션된 마이크로초당 벽 시간, 피크를 기록합니다.
memory, host CPU count, and whether the filesystem is local or networked.
메모리, 호스트 CPU 수, 파일 시스템이 로컬인지 네트워크인지 여부.
A network-mounted working directory can dominate everything else.
네트워크에 마운트된 작업 디렉토리는 다른 모든 것을 지배할 수 있습니다.

## Common causes by stage
## 단계별 일반적인 원인

Analysis: recompiling unchanged sources every run, very large generated files,
분석: 매 실행마다 변경되지 않은 소스 재컴파일, 매우 큰 생성 파일,
or IP being regenerated.
또는 IP가 재생성되고 있습니다.

Elaboration: heavy debug level, deep parameterized hierarchy, large
정교함: 무거운 디버그 수준, 깊은 매개변수화된 계층 구조, 대규모
concatenations and aggregates, and full-design optimization being redone each
연결 및 집계, 전체 설계 최적화가 각각 다시 수행됩니다.
time.
시간.
Incremental elaboration helps only when the snapshot inputs are stable.
증분 정교화는 스냅샷 입력이 안정적인 경우에만 도움이 됩니다.

Runtime: logging every signal into the waveform database, coverage enabled on
런타임: 모든 신호를 파형 데이터베이스에 기록하고 적용 범위는 다음과 같습니다.
the whole design, an unnecessarily long runtime bound, zero-delay or
전체 디자인, 불필요하게 긴 런타임 제한, 지연 없음 또는
delta-cycle churn, verbose transcript output, and DPI or SystemC boundary
델타 주기 변동, 자세한 기록 출력 및 DPI 또는 SystemC 경계
crossings in the inner loop.
내부 루프의 교차점.

## Levers, with their cost
## 레버(비용 포함)

- Reduce debug visibility.
- 디버그 가시성을 줄입니다.
- Faster, but less observable; do not do this on a
- 더 빠르지만 관찰하기가 어렵습니다. 이 일을 하지 마세요
run whose purpose is diagnosis.
진단이 목적인 달리기.
- Log a focused signal set instead of the whole design.
- 전체 설계 대신 집중된 신호 세트를 기록합니다.
- Faster and usually
- 더 빠르고 일반적으로
better evidence.
더 나은 증거.
- Disable coverage for pure performance runs, then re-enable for the coverage
- 순수 성능 실행에 대해서는 적용 범위를 비활성화한 다음 적용 범위에 대해 다시 활성화합니다.
run.
달리다.
Never report coverage from a run that had it disabled.
비활성화된 실행에서 적용 범위를 보고하지 마십시오.
- Use a runtime bound justified by the test, not a round number.
- 어림수가 아닌 테스트에 의해 정당화된 런타임 범위를 사용하십시오.
- Exploit multi-threading where the release supports it.
- 릴리스에서 지원하는 경우 멀티스레딩을 활용하세요.

Every lever changes what the run proves.
모든 레버는 실행이 증명하는 것을 변경합니다.
State which ones were active when
언제 활성 상태였는지 명시
reporting a functional verdict.
기능적 판정을 보고합니다.

## Comparing simulators
## 시뮬레이터 비교

A cross-simulator timing comparison is only meaningful with identical debug
시뮬레이터 간 타이밍 비교는 동일한 디버그에서만 의미가 있습니다.
level, logging scope, coverage settings, optimization, source set, and host.
수준, 로깅 범위, 적용 범위 설정, 최적화, 소스 세트 및 호스트.
Otherwise it measures configuration, not the tool.
그렇지 않으면 도구가 아닌 구성을 측정합니다.
Run on the same machine,
동일한 머신에서 실행하고,
repeat at least twice, and report the spread.
최소한 두 번 반복하고 확산을 보고합니다.

If XSim is genuinely slower for a specific construct, reduce to a small case
XSim이 특정 구성에 대해 실제로 느린 경우 작은 사례로 줄이십시오.
and research it with
그리고 그걸로 연구해봐
[known-issue-research.md](known-issue-research.md) before reporting it as a
[known-issue-research.md](known-issue-research.md)로 신고하기 전에
product gap.
제품 격차.

## Prohibited
## 금지

- Claiming a speedup from a single unrepeated run.
- 반복되지 않는 단일 실행으로 속도 향상을 요구합니다.
- Comparing a debug-enabled run against a debug-disabled run.
- 디버그가 활성화된 실행과 디버그가 비활성화된 실행을 비교합니다.
- Presenting a faster configuration as equivalent when it captures less.
- 더 적은 양을 캡처할 때 더 빠른 구성을 동등한 것으로 제시합니다.

## Report
## 신고

```text
Host / CPUs / filesystem:
Analysis / elaborate / runtime wall time:
Dominant stage and share:
Simulated time reached, wall time per simulated unit:
Peak memory:
Settings in effect (debug, logging, coverage, threads, runtime bound):
Change applied and measured effect (repeat count and spread):
What the faster configuration no longer proves:
```
