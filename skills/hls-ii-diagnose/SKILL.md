<!-- ko-translation: enabled -->
---
name: hls-ii-diagnose
description: 'Diagnose the root cause of an II (initiation interval) violation in a synthesized Vitis HLS component, classify the bottleneck, and cite the supporting evidence — without modifying source or running flows.'
license: MIT
argument-hint: <component_location> [loop-or-function-label]
metadata:
  author: "Zayn He"
  version: "1.0.0"
---
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# hls-ii-diagnose
# hls-ii-진단

Diagnose **why** a pipelined loop or function did not reach its target II, classify the bottleneck into a known category, and cite the evidence.
파이프라인 루프 또는 함수가 목표 II에 도달하지 못한 **이유**를 진단하고, 병목 현상을 알려진 카테고리로 분류하고, 증거를 인용합니다.
This skill produces a *diagnosis only* — the calling agent (e.g.
이 기술은 *진단만* — 호출 에이전트(예:
`hls-optimize`) decides whether and how to fix it.
`hls-optimize`)는 수정 여부와 수정 방법을 결정합니다.

## Why this skill exists
## 이 스킬이 존재하는 이유

An II violation can stem from several distinct root causes that need completely different fixes, and the csynth summary table does not distinguish them — its violation labels are coarse (e.g. "Memory Dependency" is reported for recurrences that have nothing to do with memory ports), so agents reading only the summary often misdiagnose.
II 위반은 완전히 다른 수정이 필요한 몇 가지 고유한 근본 원인에서 비롯될 수 있으며 csynth 요약 테이블은 이를 구별하지 않습니다. 위반 레이블은 대략적입니다(예: 메모리 포트와 관련 없는 재발에 대해 "메모리 종속성"이 보고됨). 따라서 요약만 읽는 에이전트는 흔히 오진합니다.
Sweeping `#pragma HLS PIPELINE II=<N>` is also not a substitute for diagnosis: it works only when the fix happens to be an II target, can hurt timing, and never finds a structural fix (inline a callee, disable a shared mux).
`#pragma HLS PIPELINE II=<N>` 스위핑도 진단을 대체할 수 없습니다. 수정 사항이 II 대상인 경우에만 작동하고 타이밍을 손상시킬 수 있으며 구조적 수정 사항(호출 수신자 인라인, 공유 mux 비활성화)을 찾지 못합니다.
This skill forces a real diagnosis first.
이 기술은 먼저 실제 진단을 강제합니다.

## When to use
## 언제 사용하나요?

Use after C synthesis (`csynth`) has completed, when a pipelined loop or function has an achieved II greater than 1 — either `II achieved` > `II target` when a target is set, or `II achieved` > 1 with no target set.
C 합성(`csynth`)이 완료된 후 파이프라인 루프 또는 함수의 달성된 II가 1보다 큰 경우 사용합니다. 목표가 설정된 경우 `II 달성` > `II 목표` 또는 목표가 설정되지 않은 `II 달성` > 1입니다.

**Out of scope:** choosing/applying the fix (that is the caller's decision), running any flow, and designs whose II already meets target.
**범위 외:** 수정 사항 선택/적용(즉, 호출자의 결정), 흐름 실행, II가 이미 목표를 충족하는 디자인.

## Inputs
## 입력

- `component_location` (string, required) — absolute path to the Vitis HLS component directory (the folder containing `vitis-comp.json`).
- `comComponent_location`(문자열, 필수) — Vitis HLS 구성 요소 디렉터리(`vitis-comp.json`이 포함된 폴더)의 절대 경로입니다.
  - If the user does not specify it and you are inside the Vitis Unified IDE, call `getActiveComponentLocation`.
  - 사용자가 이를 지정하지 않고 Vitis Unified IDE 내부에 있는 경우 `getActiveComponentLocation`을 호출하세요.
  - Otherwise ask the user.
  - 그렇지 않으면 사용자에게 물어보세요.
- `target_label` (string, optional) — a specific loop or function to focus on.
- `target_label`(문자열, 선택 사항) — 집중할 특정 루프 또는 함수.
- If omitted, diagnose the loop/function with the worst `II achieved / II target` ratio.
- 생략된 경우 최악의 'II 달성 / II 목표' 비율로 루프/기능을 진단합니다.

## Prerequisite
## 전제조건

The component must already be synthesized.
구성 요소가 이미 합성되어 있어야 합니다.
This skill does **not** run flows.
이 기술은 흐름을 실행하지 **않습니다**.
If the synthesis reports/log are missing or stale, ask the caller to run `hls-run-flow` first.
종합 보고서/로그가 누락되었거나 오래된 경우 호출자에게 먼저 'hls-run-flow'를 실행하도록 요청하세요.

## Workflow
## 작업 흐름

### Step 1 — Get the synthesis report and log
### 1단계 - 종합 보고서 및 로그 가져오기

Invoke the `hls-synth-report` skill on `component_location` to obtain `synthesisReportContent`, `synthesisPragmaReportContent`, and `synthesisLogContent`.
`comComponent_location`에서 `hls-synth-report` 기술을 호출하여 `synesisReportContent`, `synesisPragmaReportContent` 및 `synesisLogContent`를 가져옵니다.
Use the report to find the loop/function with the worst II ratio if no `target_label` was given.
'target_label'이 지정되지 않은 경우 보고서를 사용하여 II 비율이 최악인 루프/함수를 찾으세요.

### Step 2 — Extract the pipelining block(s)
### 2단계 - 파이프라이닝 블록 추출

**Do not rely on the summary tables.** The real evidence is in the per-loop/function block delimited by `Pipelining loop ...` / `Pipelining function ...` and `Pipelining result ...` in the synthesis log.
**요약 표에 의존하지 마세요.** 실제 증거는 합성 로그의 '파이프라이닝 루프 ...' / '파이프라이닝 기능 ...' 및 '파이프라이닝 결과 ...'로 구분된 루프별/함수 블록에 있습니다.
These blocks contain the carried-dependence warning (`HLS 200-880`), scheduler notes, and resource-limit messages.
이러한 블록에는 종속성 경고(`HLS 200-880`), 스케줄러 메모 및 리소스 제한 메시지가 포함되어 있습니다.

**Use the raw `logs/hls_compile.log`, not the filtered log.** The `Pipelining loop ...` / `Pipelining result ...` markers are `INFO:` lines, and the `hls-synth-report` skill filters `INFO:` lines out of `synthesisLogContent` — so they are missing from the filtered content.
**필터링된 로그가 아닌 원시 `logs/hls_compile.log`를 사용합니다.** `Pipelining loop ...` / `Pipelining result ...` 마커는 `INFO:` 라인이고 `hls-synth-report` 기술은 `synesisLogContent`에서 `INFO:` 라인을 필터링하므로 필터링된 콘텐츠에서 누락됩니다.
Read the component's raw log at `<component_location>/logs/hls_compile.log`.
`<comComponent_location>/logs/hls_compile.log`에서 구성 요소의 원시 로그를 읽습니다.

Pull the block for each loop/function with `awk` over the markers:
마커 위에 `awk`를 사용하여 각 루프/함수에 대한 블록을 당깁니다.

```bash
LOG=<component_location>/logs/hls_compile.log
# all pipelining blocks (each ends with its "Pipelining result" line showing Target/Final II):
awk '/Pipelining (loop|function)/{f=1} f; /Pipelining result/{f=0}' "$LOG"
# jump straight to the result lines to spot violations (Final II > Target II, or > 1 with no target):
grep -E "Pipelining result" "$LOG"
# then read the full block for the violating target by name:
awk "/Pipelining (loop|function) '<target_label>'/{f=1} f; /Pipelining result/{f=0}" "$LOG"
```

Read the full block — not just the result line.
결과 라인뿐만 아니라 전체 블록을 읽으십시오.
Note the carried variable name and type (scalar vs array element), the distance/offset, the producer/consumer line numbers (`HLS 200-880`), and any timing (`HLS 200-887`) or resource-limit messages.
전달된 변수 이름 및 유형(스칼라 대 배열 요소), 거리/오프셋, 생산자/소비자 라인 번호(`HLS 200-880`), 타이밍(`HLS 200-887`) 또는 리소스 제한 메시지를 기록해 두십시오.

### Step 2b — Confirm the long-latency element in the schedule report
### 2b단계 - 일정 보고서에서 지연 시간이 긴 요소 확인

When the violation is a carried dependence (a recurrence across iterations), the log's `HLS 200-880` line names the carried *variable* and the producer/consumer ops, but it does **not** always reveal *which* operation on the path carries the latency (e.g. it may name only a `store`/`load` of the carried scalar, while the real cost is a non-inlined call between them).
위반이 전달된 종속성(반복에 걸쳐 반복)인 경우 로그의 `HLS 200-880` 줄은 전달된 *변수*와 생산자/소비자 작업의 이름을 지정하지만 경로의 어떤* 작업이 지연 시간을 전달하는지 항상 드러내지는 **않습니다**(예를 들어 전달된 스칼라의 `저장`/`로드` 이름만 지정할 수 있지만 실제 비용은 둘 사이의 인라인되지 않은 호출입니다).
Do not infer this from source alone — confirm it in the **authoritative schedule report**:
소스에서만 이를 추론하지 말고 **권한 있는 일정 보고서**에서 확인하세요.

```
<component_location>/hls/.autopilot/db/<module>.verbose.sched.rpt
```

where `<module>` is the violating loop/function's module name from the `HLS 200-880` message (e.g.
여기서 `<module>`은 `HLS 200-880` 메시지에 있는 위반 루프/함수의 모듈 이름입니다(예:
`test_Pipeline_encryption_cfb8_loop`).
`test_Pipeline_encryption_cfb8_loop`).
Each line shows an operation, the schedule stages it spans (`[20/20]…[1/20]` = a 20-stage operation), and its delay, e.g.:
각 줄에는 작업, 해당 작업에 걸쳐 있는 일정 단계(`[20/20]…[1/20]` = 20단계 작업) 및 지연이 표시됩니다. 예:

```
ST_3 : Operation 72 [20/20] (1.18ns) ---> "%output_block = call i8 @process, ..." [cfb.hpp:442] ---> 'call' ...
```

Trace the recurrence on the carried variable through the stages: the operation that spans the most stages (or whose delay dominates) between the carried store and its next-iteration load **is** the long-latency element — and tells you directly whether it is a `call` (→ inline), a memory `load`/`store` (→ lower memory latency), or a long arithmetic chain (→ restructure).
단계를 통해 전달된 변수의 반복을 추적합니다. 전달된 저장소와 다음 반복 로드 사이의 대부분의 단계(또는 지연이 지배적)에 걸쳐 있는 작업은 **긴 대기 시간 요소입니다** — 그리고 그것이 `호출`(→ 인라인)인지, 메모리 `로드`/`저장`(→ 낮은 메모리 대기 시간) 또는 긴 산술 체인(→ 재구성)인지 직접 알려줍니다.
Cite the specific `ST_n … Operation … (delay)` lines as evidence.
구체적인 `ST_n … Operation … (delay)` 라인을 증거로 인용하세요.

### Step 3 — Classify the bottleneck
### 3단계 - 병목 현상 분류

First decide whether the limiter is a **carried dependence** (a recurrence across iterations) or a **resource shortage** (a single-iteration access that exceeds available resources).
먼저 제한이 **전달된 종속성**(반복에 걸쳐 반복)인지 **리소스 부족**(사용 가능한 리소스를 초과하는 단일 반복 액세스)인지 결정합니다.
Then pick the specific category:
그런 다음 특정 카테고리를 선택하세요.

**Pre-check**
**사전 확인**

- **(a) No II target** — no `II=<N>` was specified.
- **(a) II 대상 없음** - `II=<N>`이 지정되지 않았습니다.
- Without a target, timing constraints take priority over most II-related constraints, so HLS may relax II to whatever satisfies timing.
- 대상이 없으면 타이밍 제약 조건이 대부분의 II 관련 제약 조건보다 우선하므로 HLS는 타이밍을 만족하는 모든 항목으로 II를 완화할 수 있습니다.
- You are encouraged to verify this is actually the cause — e.g. some operations on the recurrence path were given longer latency to satisfy timing — before classifying as (a).
- 이것이 실제로 원인인지 확인하는 것이 좋습니다. (a)로 분류되기 전에 반복 경로의 일부 작업에는 타이밍을 충족하기 위해 더 긴 대기 시간이 주어졌습니다.
- The natural check is whether re-synthesizing with an explicit II target would close the gap (but that is the caller's action, not this skill's).
- 자연스러운 확인은 명시적인 II 대상을 사용한 재합성이 격차를 줄일 수 있는지 여부입니다(그러나 이는 발신자의 행동이지 이 스킬의 행동이 아닙니다).

**Carried-dependence bottlenecks**
**운반 의존성 병목 현상**

- **(b) Long latency on the recurrence path** — the recurrence carries a dependence whose round-trip latency the scheduler cannot fit into the target II, so II grows to cover it.
- **(b) 재발 경로의 긴 대기 시간** - 반복은 스케줄러가 대상 II에 맞지 않는 왕복 대기 시간의 종속성을 전달하므로 II가 이를 처리하기 위해 커집니다.
- The general task is to find *which element on the path is long and why the scheduler cannot shorten it*.
- 일반적인 작업은 *경로의 어떤 요소가 길고 스케줄러가 이를 단축할 수 없는 이유*를 찾는 것입니다.
- Common sources (not exhaustive):
- 일반적인 소스(완전하지는 않음):
  - a non-inlined function call, treated as a black-box latency the scheduler cannot retime across;
  - 인라인되지 않은 함수 호출은 스케줄러가 시간을 재조정할 수 없는 블랙박스 대기 시간으로 처리됩니다.
  - a memory load/store whose access latency sits on the path;
  - 액세스 대기 시간이 경로에 있는 메모리 로드/저장;
  - a registered shared-resource mux inserted by resource sharing;
  - 자원 공유에 의해 삽입된 등록된 공유 자원 다중화기;
  - a long combinational chain that the scheduler lengthened to satisfy timing (overlaps with (a) when no II target was set).
  - 타이밍을 만족시키기 위해 스케줄러가 길어지는 긴 조합 체인(II 목표가 설정되지 않은 경우 (a)와 겹침).

Identify the specific source using the schedule report (Step 2b) — the carried variable, the operations on the path (`load`/`store`/`call`/`add`/…), and their per-operation latencies — because the fix direction depends on it (inline the callee, lower the memory latency, disable sharing, or restructure to shorten the chain).
일정 보고서(2b단계)를 사용하여 특정 소스를 식별합니다. 즉, 전달된 변수, 경로의 작업(`load`/`store`/`call`/`add`/…) 및 작업별 대기 시간 - 수정 방향이 이에 따라 달라지기 때문입니다(호출 수신자 인라인, 메모리 대기 시간 낮추기, 공유 비활성화 또는 체인 단축을 위한 재구성).

**Resource-shortage bottlenecks**
**리소스 부족 병목 현상**

- **(c) Memory port shortage** — within a single iteration, the unrolled accesses to a memory exceed its available ports.
- **(c) 메모리 포트 부족** - 단일 반복 내에서 메모리에 대한 확장된 액세스가 사용 가능한 포트를 초과합니다.
- This is *not* a cross-iteration dependence; it surfaces as a different class of violation in the log.
- 이는 교차 반복 의존성이 *아닙니다*. 이는 로그에 다른 위반 클래스로 표시됩니다.

**Categories are not mutually exclusive — report every cause the evidence supports.** Two situations call for reporting more than one:
**범주들은 상호 배타적이지 않습니다. 증거가 뒷받침하는 모든 원인을 보고하십시오.** 두 가지 상황에서는 둘 이상의 보고가 필요합니다.

- **Co-occurring causes.** A single violation can have more than one real cause at once.
- **공동 발생 원인.** 단일 위반에는 동시에 둘 이상의 실제 원인이 있을 수 있습니다.
- The most common pair is (a) + (b): no II target was set *and* the recurrence carries a long-latency element.
- 가장 일반적인 쌍은 (a) + (b)입니다. II 목표가 설정되지 않았고* 반복에 긴 대기 시간 요소가 포함됩니다.
- Here both are true and both matter — without a target the scheduler never tries to shorten the path, and even with a target the path latency still caps the achievable II.
- 여기서 둘 다 사실이고 둘 다 중요합니다. 대상이 없으면 스케줄러는 경로를 단축하려고 시도하지 않으며 대상이 있어도 경로 대기 시간은 여전히 ​​달성 가능한 II를 제한합니다.
- Report both, and make the relationship explicit (e.g. "(b) is the physical limiter; (a) is why the scheduler accepted the relaxed II").
- 둘 다 보고하고 관계를 명시적으로 만듭니다(예: "(b)는 물리적 제한자입니다. (a)는 스케줄러가 완화된 II를 수락한 이유입니다.")
- The `next_step` should then cover both levers (set an II target *and* shorten the path).
- 그런 다음 `next_step`은 두 레버를 모두 포함해야 합니다(II 목표를 설정하고 *경로를 단축함).
- **Genuine ambiguity.** If the evidence cannot distinguish between two plausible categories, report both with their supporting evidence rather than guessing.
- **진짜 모호성.** 증거가 두 가지 그럴듯한 범주를 구별할 수 없는 경우 추측보다는 뒷받침하는 증거와 함께 두 가지를 모두 보고합니다.

### Step 4 — Locate the source
### 4단계 - 소스 찾기

Identify the source location of the bottleneck: the carried variable, the callee, or the memory access, depending on the category.
병목 현상의 소스 위치(범주에 따라 전달된 변수, 피호출자 또는 메모리 액세스)를 식별합니다.
Use the line numbers from the `HLS 200-880` message and the source files reported by `hls-component-basic-info` if needed.
필요한 경우 `HLS 200-880` 메시지의 줄 번호와 `hls-comComponent-basic-info`에서 보고된 소스 파일을 사용하세요.

### Step 5 — Cite the evidence
### 5단계 - 증거 인용

For every claim in the diagnosis, point at the supporting artifact: synthesis-log line(s) (the `HLS 200-880`/`HLS 200-887` text and the `Pipelining result` line), the `verbose.sched.rpt` `ST_n … Operation … (delay)` line(s) that identify the long-latency element (Step 2b), and source line(s).
진단의 모든 주장에 대해 지원 아티팩트인 합성 로그 라인(`HLS 200-880`/`HLS 200-887` 텍스트 및 `파이프라이닝 결과` 라인), 지연 시간이 긴 요소(2b단계)를 식별하는 `verbose.sched.rpt` `ST_n … Operation … (delay)` 라인 및 소스 라인을 가리킵니다.
The caller must be able to verify the diagnosis independently.
호출자는 진단을 독립적으로 확인할 수 있어야 합니다.
For a category (b) classification, the schedule-report citation is what distinguishes a grounded diagnosis from a guess — prefer it over inferring the long-latency element from source.
카테고리 (b) 분류의 경우 일정 보고서 인용은 근거 있는 진단을 추측과 구별하는 요소입니다. 소스에서 긴 대기 시간 요소를 추론하는 것보다 선호합니다.

## Output
## 출력

Emit a structured diagnosis:
구조화된 진단을 내보냅니다.

- `target_label` — the loop or function being diagnosed
- `target_label` — 진단 중인 루프 또는 함수
- `achieved_ii` / `target_ii`
- `달성_ii` / `목표_ii`
- `category` — one or more of (a)–(c) above (report all that the evidence supports; see Step 3); for (b), name the specific long-latency source on the path
- '범주' - 위의 (a)~(c) 중 하나 이상(증거가 뒷받침하는 모든 내용 보고, 3단계 참조) (b)의 경우 경로에서 지연 시간이 긴 특정 소스의 이름을 지정하세요.
- `evidence` — log line numbers, source line numbers, schedule excerpts
- `evidence` — 로그 줄 번호, 소스 줄 번호, 일정 발췌
- `next_step` — high-level recommendation (e.g. "specify II target", "inline callee", "lower memory latency", "increase memory ports", "restructure to break recurrence").
- `next_step` — 높은 수준의 권장 사항(예: "II 대상 지정", "인라인 호출 수신자", "메모리 대기 시간 단축", "메모리 포트 늘리기", "반복을 막기 위한 재구성")
- Pragma or directive names (e.g.
- 프라그마 또는 지시문 이름(예:
- `inline`, `bind_storage`) may appear **only when paired with the mechanism they address**.
- `inline`, `bind_storage`)는 **해당 메커니즘과 쌍을 이루는 경우에만** 나타날 수 있습니다.
- The caller makes the final decision on whether and how to apply them.
- 적용 여부와 방법에 대한 최종 결정은 호출자가 내립니다.

## Constraints
## 제약

This skill MUST NOT:
이 기술은 다음을 해서는 안 됩니다:

- Modify source files.
- 소스 파일을 수정합니다.
- Run csim / csynth / cosim / impl flows. (Use `hls-run-flow` for that.)
- csim / csynth / cosim / impl 흐름을 실행합니다. (이를 위해서는 `hls-run-flow`를 사용하세요.)
- Recommend a fix without naming the underlying mechanism.
- 기본 메커니즘의 이름을 지정하지 않고 수정을 권장합니다.
