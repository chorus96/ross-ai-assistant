<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# Sub-agent prompt template for a single optimization attempt
# 단일 최적화 시도를 위한 하위 에이전트 프롬프트 템플릿

Use this template when launching a sub-agent for one optimization attempt.
한 번의 최적화 시도를 위해 하위 에이전트를 시작할 때 이 템플릿을 사용하십시오.
Fill in all `[bracketed]` fields before sending.
보내기 전에 '[괄호 안의]' 필드를 모두 입력하세요.

---

Task: Apply [specific technique] to `src/*.cpp`.
작업: `src/*.cpp`에 [특정 기술]을 적용합니다.

Goal: [state the optimization criteria, e.g. "minimize true latency in seconds"].
목표: [최적화 기준을 명시하세요. "실제 대기 시간을 초 단위로 최소화"].

Baseline:
기준:
- cosim latency: [X] cycles
- cosim 대기 시간: [X] 주기
- post-route clock: [Y] ns
- 경로 후 시계: [Y] ns
- true latency: [Z] ms  (= X × Y)
- 실제 지연 시간: [Z]ms(= X × Y)
- LUT: [A], FF: [B], DSP: [C], BRAM: [D]
- LUT: [A], FF: [B], DSP: [C], BRAM: [D]

Hypothesis:
가설:
- What you will change: [describe exactly what pragma or algorithmic change to apply and where]
- 변경할 사항: [적용할 pragma 또는 알고리즘 변경 사항과 적용 위치를 정확히 설명]
- What you expect to happen: [e.g. "II on the k-loop should drop from 5 to 2 because we now have 2 independent memory ports for C"]
- 일어날 것으로 예상되는 일: [예: "k-루프의 II는 이제 C에 대한 2개의 독립 메모리 포트를 가지므로 5에서 2로 낮아져야 합니다."]
- What would confirm the hypothesis: [e.g. "csynth loop table shows the hot loop's II reduced from N to M"]
- 가설을 확증하는 것은 무엇입니까? [예: "csynth 루프 테이블은 N에서 M으로 감소된 핫 루프의 II를 보여줍니다."]
- What would falsify it: [e.g. "II unchanged, meaning the bottleneck is a recurrence rather than a memory dependency"]
- 무엇을 위조할 것인가: [예: "II는 변경되지 않았습니다. 즉, 병목 현상은 메모리 의존성이 아니라 재발이라는 의미입니다."]

Steps:
단계:

### 1.
### Read the current kernel
### 현재 커널 읽기

Read `src/*.cpp`.
`src/*.cpp`를 읽어보세요.
Understand the current implementation before touching anything.
무엇이든 만지기 전에 현재 구현을 이해하세요.
Also run:
또한 다음을 실행합니다:
```bash
python3 .claude/skills/synth/scripts/report_summary.py $TOP_FUNCTION/hls/syn/report/$TOP_FUNCTION_csynth.rpt
```
to confirm you understand the baseline loop structure and bottlenecks.
기본 루프 구조와 병목 현상을 이해하고 있는지 확인합니다.

### 2.
### Apply the change
### 변경사항 적용

Make **only** the change described above.
위에 설명된 대로 **만** 변경하세요.
Do not fix, refactor, or optimize anything else
다른 어떤 것도 수정, 리팩토링, 최적화하지 마세요.
while you are in here — even if you notice an opportunity.
당신이 여기에 있는 동안 — 기회를 발견하더라도 말이죠.
One change per attempt.
시도당 하나의 변경이 가능합니다.

### 3.
### Run csim
### csim 실행

```bash
vitis-run --mode hls --config $CONFIG --work_dir $TOP_FUNCTION --csim
```

Read `$TOP_FUNCTION/hls/csim/report/$TOP_FUNCTION_csim.log`.
`$TOP_FUNCTION/hls/csim/report/$TOP_FUNCTION_csim.log`를 읽어보세요.
If it ends with `FAIL` or a non-zero error count,
`FAIL` 또는 0이 아닌 오류 개수로 끝나는 경우
fix the logic error (if trivial) or abort and report why.
논리 오류(사소한 경우)를 수정하거나 중단하고 이유를 보고합니다.
Do not proceed to csynth after a
다음 이후에는 csynth로 진행하지 마십시오.
failing csim.
csim이 실패했습니다.

### 4.
### Run csynth
### csynth 실행

```bash
v++ --mode hls --config $CONFIG --work_dir $TOP_FUNCTION --compiler
```

Then investigate **thoroughly**:
그런 다음 **철저히** 조사해 보세요.

#### 4a.
#### Quick summary
#### 빠른 요약
```bash
python3 .claude/skills/synth/scripts/report_summary.py $TOP_FUNCTION/hls/syn/report/$TOP_FUNCTION_csynth.rpt
```

Check:
확인:
- Did II change on the loop you targeted?
- 당신이 목표로 삼은 루프가 바뀌었나요?
- Did II change on any other loop (side effect)?
- 다른 루프에서 II가 변경되었나요(부작용)?
- Did total latency change?
- 총 지연 시간이 변경되었나요?
- By the expected factor?
- 예상되는 요인으로?
- Did resource usage change?
- 리소스 사용량이 변경되었나요?
- By how much?
- 얼마만큼?

#### 4b.
#### Violation Report
#### 위반 신고
Open `$TOP_FUNCTION/hls/syn/report/$TOP_FUNCTION_csynth.rpt` and read the `Violation Report` section completely.
`$TOP_FUNCTION/hls/syn/report/$TOP_FUNCTION_csynth.rpt`를 열고 `위반 보고서` 섹션을 완전히 읽어보세요.
Does the remaining violation match your hypothesis about what is limiting II?
나머지 위반 사항은 II를 제한하는 것에 대한 가설과 일치합니까?
If the violation is different from what you expected, note it — the tool is telling you
위반 사항이 예상한 것과 다른 경우 이를 기록해 두십시오. 도구에서 이를 알려줍니다.
the actual bottleneck.
실제 병목 현상.

#### 4c.
#### Inferred directives
#### 추론된 지시어
Read `$TOP_FUNCTION/hls/syn/inferred_directives.ini`.
`$TOP_FUNCTION/hls/syn/inferred_directives.ini`를 읽어보세요.
Did HLS auto-apply any directive that interacts
HLS가 상호 작용하는 지시어를 자동 적용했습니까?
with your change (e.g. auto-partition, auto-flatten, auto-pipeline)?
변경 사항(예: 자동 파티션, 자동 평면화, 자동 파이프라인)이 적용됩니까?

#### 4d.
#### Warnings in the synthesis log
#### 합성 로그의 경고
```bash
grep -i "warning\|violation" $TOP_FUNCTION/hls/syn/report/$TOP_FUNCTION_csynth.rpt | head -40
```

#### 4e.
#### IR inspection (when verifying compiler behavior)
#### IR 검사(컴파일러 동작 검증 시)
When you want to confirm the compiler actually applied the transformation you intended
컴파일러가 의도한 변환을 실제로 적용했는지 확인하려는 경우
(e.g. that an array was partitioned, or a loop was unrolled in the IR):
(예: 배열이 분할되었거나 IR에서 루프가 풀린 경우):

```bash
VITIS_BASE=$(dirname $(dirname $(which vitis-run)))
export LD_LIBRARY_PATH="$VITIS_BASE/lib/lnx64.o:$VITIS_BASE/lnx64/lib/lnx64.o${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
"$VITIS_BASE/lnx64/tools/clang-3.9-csynth/bin/llvm-dis" $TOP_FUNCTION/hls/.autopilot/db/a.g.bc -o - 2>/dev/null | grep -A30 "<pattern of interest>"
```

Things to look for:
찾아야 할 사항:
- **After `ARRAY_PARTITION`**: multiple GEP chains or multiple `alloca`s for the partitioned array.
- **`ARRAY_PARTITION` 이후**: 분할된 배열에 대한 여러 GEP 체인 또는 여러 `alloca`.
- **After `UNROLL factor=N`**: N repetitions of the loop body in the IR.
- **`UNROLL 요소=N` 이후**: IR에서 루프 본문이 N 반복됩니다.
- **After `LOOP_FLATTEN`**: single induction variable instead of nested loop counters.
- **`LOOP_FLATTEN` 이후**: 중첩된 루프 카운터 대신 단일 유도 변수.

#### 4f.
#### Log the QoR
#### QoR 기록
```bash
python3 .claude/skills/synth/scripts/qor_log.py --label "[technique-applied]" --csv qor_history.csv
```

### 5.
### Assess vs. hypothesis
### 평가 대 가설

Write down explicitly:
명시적으로 적어보세요:
- Did the result match the hypothesis?
- 결과가 가설과 일치했나요?
- If yes: what does this confirm about the design?
- 그렇다면: 이는 디자인에 대해 무엇을 확인시켜 줍니까?
- If no: what does the actual result reveal?
- 그렇지 않다면 실제 결과는 무엇을 드러내나요?
- Is there a secondary bottleneck?
- 2차 병목 현상이 있나요?

This assessment determines the next step.
이 평가를 통해 다음 단계가 결정됩니다.

### 6.
### Decide whether to escalate
### 에스컬레이션할지 결정

| csynth result<br>신디사이저 결과 | Action<br>액션 |
|---|---|
| Latency improved ≥ 10% OR II reduced on dominant loop<br>지연 시간 개선 ≥ 10% 또는 우성 루프에서 II 감소 | Run `run cosim`<br>`run cosim`을 실행하세요. |
| Latency unchanged, but hypothesis confirmed (e.g. "acknowledge II" change)<br>지연 시간은 변경되지 않았지만 가설은 확인되었습니다(예: '인정 II' 변경). | Commit; move to next idea<br>커밋; 다음 아이디어로 이동 |
| Latency got worse<br>지연 시간이 더 심해졌습니다. | Investigate. Do not immediately revert — understand why first. Then decide.<br>조사하다. 즉시 되돌리지 마십시오. 먼저 이유를 이해하십시오. 그런 다음 결정하십시오. |
| csim FAIL<br>csim 실패 | Fix or revert.<br>수정하거나 되돌리세요. |

#### If running cosim:
#### cosim을 실행하는 경우:
```bash
vitis-run --mode hls --config $CONFIG --work_dir $TOP_FUNCTION --cosim
```
Read `$TOP_FUNCTION/hls/sim/report/$TOP_FUNCTION_cosim.rpt`.
`$TOP_FUNCTION/hls/sim/report/$TOP_FUNCTION_cosim.rpt`를 읽어보세요.
Compare cosim cycles to csynth estimate.
cosim 주기를 csynth 추정치와 비교합니다.
Note any discrepancy (cosim includes stall cycles; csynth does not).
불일치가 있는지 확인하십시오(cosim에는 정지 주기가 포함되어 있지만 csynth에는 포함되지 않음).

#### If running impl:
#### impl을 실행하는 경우:
```bash
vitis-run --mode hls --config $CONFIG --work_dir $TOP_FUNCTION --impl
```
Read `$TOP_FUNCTION/hls/reports/hls_impl_pnr.rpt` for the post-route clock period.
경로 후 클럭 기간에 대해서는 `$TOP_FUNCTION/hls/reports/hls_impl_pnr.rpt`를 읽어보세요.
Compute: `true latency = cosim_cycles × period_ns`.
계산: `실제 대기 시간 = cosim_cycles × period_ns`.

### 7.
### Report back
### 다시 신고하세요

Provide a concise summary:
간결한 요약을 제공하세요.

```
Technique applied: [exact pragma or code change]
File changed: src/*.cpp lines [N-M]

Hypothesis: [what you expected]
Confirmed: [yes/no/partial]
Reason: [why the result matched or didn't match the hypothesis]

csynth result:
  Latency: [baseline] → [new] cycles (csynth estimate)
  II (targeted loop): [baseline] → [new]
  II (other loops): [any unexpected changes]
  Resources: LUT [A]→[A'], FF [B]→[B'], DSP [C]→[C'], BRAM [D]→[D']
  Timing: [estimated] ns (target: [X] ns)
  Violations: [list any new or resolved violations]
  Inferred directives: [any surprises?]
  IR observation: [what you saw in a.g.bc, if inspected]

cosim result (if run):
  Latency: [X'] cycles (vs csynth estimate [Y'] — delta: [Z%])

impl result (if run):
  Post-route clock: [Y'] ns
  True latency: [Z'] ms (vs baseline [Z] ms)

Recommendation: [keep / revert / investigate further]
```
