<!-- ko-translation: enabled -->
---
name: hls-optimize
description: Iteratively optimize the HLS kernel against a given criteria (e.g. minimize latency, reduce DSP usage, maximize throughput) by applying and measuring pragma and algorithmic changes.
license: MIT
argument-hint: <optimization-criteria>
metadata:
  author: "Alex Li, Sai Akhil Ayyagari"
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

# Skill: hls-optimize
# 기술: hls-최적화

Iteratively modify the kernel to improve performance against an optimization criteria, using the other skills to measure progress and committing each attempt.
다른 기술을 사용하여 진행 상황을 측정하고 각 시도를 커밋하여 최적화 기준에 대한 성능을 향상시키기 위해 커널을 반복적으로 수정합니다.

**Optimization criteria**: $ARGUMENTS
**최적화 기준**: $ARGUMENTS

Examples:
예:
- `/hls-optimize minimize latency`
- `/hls-최적화 대기 시간 최소화`
- `/hls-optimize minimize DSP usage while meeting timing`
- `/hls-optimize 타이밍을 맞추면서 DSP 사용량 최소화`
- `/hls-optimize maximize throughput (minimize II)`
- `/hls-optimize 처리량 최대화(최소화 II)`
- `/hls-optimize reduce LUT count below 5000`
- `/hls-optimize LUT 수를 5000 미만으로 줄입니다`

## Init component and git repository
## 구성 요소 및 Git 저장소 초기화

**MANDATORY**: Before any optimization work, you MUST read and execute the `./reference/init-component.md` file.
**필수**: 최적화 작업을 수행하기 전에 `./reference/init-comComponent.md` 파일을 읽고 실행해야 합니다.

**DO NOT** write your own git init commands. **DO NOT** use simplified git commands.
**하지 마세요** git init 명령을 직접 작성하지 마세요. **간단한 git 명령을 사용하지 마세요**.
You MUST execute the `./reference/init-component.md` file, which includes the `vitis-comp.json` check, specific git configuration, and .gitignore setup.
`vitis-comp.json` 검사, 특정 git 구성 및 .gitignore 설정이 포함된 `./reference/init-comComponent.md` 파일을 실행해야 합니다.

This ensures the HLS component is properly set up, with consistent git configuration, proper .gitignore setup, and correct baseline commit format across all optimization sessions.
이렇게 하면 모든 최적화 세션에서 일관된 git 구성, 적절한 .gitignore 설정 및 올바른 기본 커밋 형식을 사용하여 HLS 구성 요소가 올바르게 설정됩니다.

## Philosophy
## 철학

**One idea at a time.** Each attempt changes exactly one thing — one pragma, one algorithmic transformation, one loop restructuring.
**한 번에 하나의 아이디어.** 시도할 때마다 정확히 한 가지, 즉 하나의 프라그마, 하나의 알고리즘 변환, 하나의 루프 재구성이 변경됩니다.
Never combine independent changes in a single attempt.
한 번의 시도로 독립적인 변경 사항을 결합하지 마십시오.
You cannot attribute cause if multiple things changed simultaneously.
여러 가지가 동시에 변경된 경우 원인을 파악할 수 없습니다.

**Investigate before moving on.** After each synthesis, read the reports thoroughly.
**계속 진행하기 전에 조사하십시오.** 각 합성 후에 보고서를 철저하게 읽으십시오.
Do not just check whether latency went up or down — understand *why*.
지연 시간이 증가했는지 감소했는지만 확인하지 말고 *이유*를 이해하세요.
Read the loop detail table.
루프 세부정보 테이블을 읽어보세요.
Check II and achieved vs. target.
II와 달성 대 목표를 확인하세요.
Look for warnings in the log.
로그에서 경고를 찾아보세요.
If something unexpected happened, dig into it before proceeding.
예상치 못한 일이 발생했다면 진행하기 전에 자세히 살펴보세요.

**Use all available tools.** The synth skill gives access to BC file inspection, QoR JSON, inferred directives, and schedule details.
**사용 가능한 모든 도구를 사용합니다.** 합성 기술을 사용하면 BC 파일 검사, QoR JSON, 추론된 지시문 및 일정 세부 정보에 액세스할 수 있습니다.
Use `hls-component-basic-info` skill and the `hls-synth-report` skill for a quick overview.
빠른 개요를 보려면 `hls-comComponent-basic-info` 스킬과 `hls-synth-report` 스킬을 사용하세요.
In synthesis log to inspect how the pragma used in the code.
합성 로그에서 코드에서 pragma가 어떻게 사용되었는지 검사합니다.
Use `./hls-scripts/qor_log.py` to track the history of changes.
변경 내역을 추적하려면 `./hls-scripts/qor_log.py`를 사용하세요.
Verify that the change had the expected effect — not just in the report, but in the IR and schedule.
보고서뿐만 아니라 IR 및 일정에서도 변경 사항이 예상한 효과를 가져왔는지 확인합니다.

**Experimental changes are valid.** Sometimes the right move is to make a change that you don't expect to improve the metric, but that tests a hypothesis about the tool's behavior.
**실험적 변경 사항은 유효합니다.** 측정항목이 개선될 것으로 기대하지는 않지만 도구 동작에 대한 가설을 테스트하는 변경을 수행하는 것이 올바른 조치인 경우도 있습니다.
Document what you expected and what actually happened.
예상했던 것과 실제로 일어난 일을 문서로 기록하세요.
Understanding the compiler's behavior is itself valuable.
컴파일러의 동작을 이해하는 것 자체가 중요합니다.

**Do not rush to run cosim or impl.** Use csim+csynth as the inner loop.
**cosim 또는 impl을 서두르지 마십시오.** csim+csynth를 내부 루프로 사용하십시오.
Only escalate to cosim when csynth shows meaningful improvement (≥ 10% in cycles or a structural change like II reduction).
csynth가 의미 있는 개선(주기에서 10% 이상 또는 II 감소와 같은 구조적 변화)을 보이는 경우에만 cosim으로 에스컬레이션하세요.
Only run impl when cosim confirms improvement.
cosim이 개선을 확인한 경우에만 impl을 실행하세요.
The expensive steps are a gate, not a rubber stamp.
값비싼 계단은 고무도장이 아니라 문이다.

**Verify Vitis environment.**
**Vitis 환경을 확인합니다.**

```bash
which vitis-run
```

If not found, check if the environment variable XILINX_VITIS is set.
찾을 수 없으면 환경 변수 XILINX_VITIS가 설정되어 있는지 확인하십시오.
If not, ask the user for the value of XILINX_VITIS.
그렇지 않은 경우 사용자에게 XILINX_VITIS 값을 물어보세요.
Then source the settings file:
그런 다음 설정 파일을 소싱합니다.

```bash
which vitis-run  # must be on PATH — set XILINX_VITIS to your Vitis install if not
```

Do not proceed until `vitis-run` is on PATH.
'vitis-run'이 PATH에 있을 때까지 진행하지 마세요.

## Step 0: Establish baseline and get HLS design data
## 0단계: 기준선 설정 및 HLS 설계 데이터 가져오기

### 0a.
### Establish a baseline
### 기준선 설정

Before making any change, ensure you have a measured baseline to compare against.
변경하기 전에 비교할 측정된 기준선이 있는지 확인하십시오.
The baseline consists of:
기준선은 다음으로 구성됩니다.

1. **cosim latency** (cycles) from cosimulation report
1. 공동 시뮬레이션 보고서의 **cosim 대기 시간**(주기)
2. **post-route clock period** (ns) from the `hls-impl-report` skill's `postRoute` value (implementation report)
2. **라우트 후 클록 기간**(ns) `hls-impl-report` 스킬의 `postRoute` 값(구현 보고서)
3. **true latency** = cycles × period
3. **실제 대기 시간** = 주기 × 기간
4. **resource utilization** (LUT, FF, DSP, BRAM, URAM) from the `hls-impl-report` skill's `placeRouteResourceSummary` value (implementation report)
4. **리소스 활용**(LUT, FF, DSP, BRAM, URAM) `hls-impl-report` 스킬의 `placeRouteResourceSummary` 값(구현 보고서)

When you establish the baseline, you must make sure the csimulation, synthesis, cosimulation and implementation flows have all been run successfully.
기준선을 설정할 때 csimulation, 종합, cosimulation 및 구현 흐름이 모두 성공적으로 실행되었는지 확인해야 합니다.
If any of the reports is missing or stale, use skill of `hls-run-flow` to run the corresponding flow before recording the baseline.
누락되거나 오래된 보고서가 있는 경우 'hls-run-flow' 기술을 사용하여 기준선을 기록하기 전에 해당 흐름을 실행하세요.

### 0b.
### Get HLS design reports and data
### HLS 설계 보고서 및 데이터 받기

**MANDATORY**: You MUST call the following functions to get ALL report-related data. **DO NOT** read `.rpt` files directly using file read tools — always use these functions.
**필수**: 모든 보고서 관련 데이터를 가져오려면 다음 함수를 호출해야 합니다. **DO NOT** 파일 읽기 도구를 사용하여 `.rpt` 파일을 직접 읽지 마십시오. 항상 이 기능을 사용하십시오.

- `hls-synth-report` skill: get the synthesis report and log content.
- `hls-synth-report` 스킬: 합성 보고서 및 로그 내용을 가져옵니다.
  - Returns `synthesisReportContent`, `synthesisPragmaReportContent`, and `synthesisLogContent`
  - `synesisReportContent`, `synesisPragmaReportContent` 및 `synesisLogContent`를 반환합니다.
- `hls-cosim-report` skill: get the cosimulation report content and dataflow process/channel content.
- `hls-cosim-report` 기술: 공동 시뮬레이션 보고서 콘텐츠와 데이터 흐름 프로세스/채널 콘텐츠를 가져옵니다.
  - Returns `coSimulationReportContent` and `dataflowProcessAndChannelContent`
  - `coSimulationReportContent` 및 `dataflowProcessAndChannelContent`를 반환합니다.
- `hls-impl-report` skill： get the implementation report content.
- `hls-impl-report` 스킬: 구현 보고서 내용을 가져옵니다.
  - Returns `postRoute`, `placeRouteResourceSummary` and `placeRouteFailFast`
  - `postRoute`, `placeRouteResourceSummary` 및 `placeRouteFailFast`를 반환합니다.

### 0c.
### Run HLS flows if reports are missing or stale
### 보고서가 누락되었거나 오래된 경우 HLS 흐름 실행

If these reports do not exist or are stale, you must use the `hls-run-flow` skill to run the active component HLS flow.
이러한 보고서가 존재하지 않거나 오래된 경우 'hls-run-flow' 기술을 사용하여 활성 구성 요소 HLS 흐름을 실행해야 합니다.
Before running csim flow, you must check 'csim.profile_tripcount=1' in .cfg file.
csim flow를 실행하기 전에 .cfg 파일에서 'csim.profile_tripcount=1'을 확인해야 합니다.
If it is not set, update the trip count.
설정되지 않은 경우 운행 횟수를 업데이트하세요.
When you find it changes the trip count, you must run all flows (csim, csynth, cosim, impl) to get the updated reports before recording the baseline.
이동 횟수가 변경된 것을 발견하면 기준을 기록하기 전에 모든 흐름(csim, csynth, cosim, impl)을 실행하여 업데이트된 보고서를 가져와야 합니다.

### 0d.
### Log the baseline QoR data for future comparison
### 향후 비교를 위해 기준 QoR 데이터 기록
On the unmodified kernel first.
수정되지 않은 커널에서 먼저.

Call `getSkillResourcePath` function with skillName and relativePath `./hls-scripts/qor_log.py` to get the full path to the script.
SkillName 및 상대 경로 `./hls-scripts/qor_log.py`를 사용하여 `getSkillResourcePath` 함수를 호출하여 스크립트의 전체 경로를 가져옵니다.

Then run `python3 "$qorLogScriptFullPath" --report "$synthesisReportContent"` to get a concise summary of the baseline.
그런 다음 `python3 "$qorLogScriptFullPath" --report "$synesisReportContent"`를 실행하여 기준선의 간결한 요약을 가져옵니다.
The `report` argument is required; the `label` and `csv` arguments are optional.
'report' 인수가 필요합니다. 'label' 및 'csv' 인수는 선택사항입니다.
Record the baseline before starting optimization iterations.
최적화 반복을 시작하기 전에 기준선을 기록하십시오.

## Step 1: Analyze the kernel
## 1단계: 커널 분석

You must call `hls-component-basic-info` skill to get the basic component info, including `component_name`, `source_files`, `testbench_files`, `include_paths`, `top_function`.
`comComponent_name`, `source_files`, `testbench_files`, `include_paths`, `top_function`을 포함한 기본 구성 요소 정보를 가져오려면 `hls-comComponent-basic-info` 스킬을 호출해야 합니다.
Read this information along with the `synthesisReportContent` and `synthesisLogContent` to analyze the kernel and identify bottlenecks.
커널을 분석하고 병목 현상을 식별하려면 `synesisReportContent` 및 `synesisLogContent`와 함께 이 정보를 읽어보세요.

- **High II on inner loops** → memory port bottleneck or long recurrence chain.
- **내부 루프의 High II** → 메모리 포트 병목 현상 또는 긴 반복 체인.
- Check loop table for `II achieved` vs `II target`.
- `II 달성`과 `II 목표`에 대한 루프 테이블을 확인하세요.
- Read the warning messages in `synthesisLogContent` for the root cause.
- 근본 원인은 `synesisLogContent`의 경고 메시지를 읽어보세요.
- **Large iteration latency on unpipelined loops** → missing pipeline pragma, or loop not eligible for pipelining (e.g. variable trip count, non-trivial control flow, contains sub-loop).
- **파이프라인되지 않은 루프의 큰 반복 대기 시간** → 파이프라인 프라그마 누락 또는 파이프라이닝에 적합하지 않은 루프(예: 가변 트립 수, 중요하지 않은 제어 흐름, 하위 루프 포함)
- **Low resource utilization** → opportunity to trade area for performance (unrolling, partitioning).
- **낮은 리소스 활용도** → 성능을 위해 영역을 교환할 수 있는 기회(언롤링, 파티셔닝)
- DSP at 2% means 98% of compute is still sequential.
- DSP가 2%라는 것은 컴퓨팅의 98%가 여전히 순차적임을 의미합니다.
- **High resource utilization** → risk of fit; verify after every change.
- **높은 자원 활용도** → 적합성 위험; 모든 변경 후에 확인하십시오.
- **Sequential outer loop containing pipelined inner loop** → inner pipeline restarts on every iteration, dominant latency term.
- **파이프라인된 내부 루프를 포함하는 순차 외부 루프** → 내부 파이프라인은 모든 반복에서 다시 시작되며, 지배적인 대기 시간 기간입니다.

Also read the Violation Report in the `synthesisLogContent` carefully — it identifies exactly which dependency causes each II violation, including the variable name and memory port.
또한 `synesisLogContent`에 있는 위반 보고서를 주의 깊게 읽으십시오. 이는 변수 이름 및 메모리 포트를 포함하여 각 II 위반을 유발하는 종속성을 정확하게 식별합니다.

## Step 2: Form a hypothesis
## 2단계: 가설 세우기

Before touching any code, write down:
코드를 터치하기 전에 다음 사항을 적어 두십시오.

1. **What you will change** (exact pragma or code transformation)
1. **변경할 내용**(정확한 pragma 또는 코드 변환)
2. **What you expect to happen** in the `synthesisReportContent` (loop II, latency, resources)
2. **`synesisReportContent`에서 예상되는 상황**(루프 II, 대기 시간, 리소스)
3. **What would confirm the hypothesis** (e.g. "II drops from 5 to 2 because we have 2 memory ports now")
3. **가설을 확증하는 것은 무엇입니까**(예: "현재 메모리 포트가 2개이므로 II가 5에서 2로 떨어집니다.")
4. **What would falsify it** (e.g. "II stays at 5 despite partition → the bottleneck is the FMA recurrence, not memory")
4. **위조하는 방법**(예: "II는 분할에도 불구하고 5로 유지 → 병목 현상은 메모리가 아닌 FMA 재발입니다.")

This forces you to understand the compiler before running it, and makes failed experiments informative.
이를 통해 컴파일러를 실행하기 전에 컴파일러를 이해하고 실패한 실험에 대한 정보를 얻을 수 있습니다.

## Step 3: Choose one optimization technique
## 3단계: 하나의 최적화 기술 선택

Pick one technique per attempt.
시도당 하나의 기술을 선택하십시오.
See the full technique catalogue below.
아래의 전체 기술 카탈로그를 참조하십시오.

### Pragma-only changes (lowest risk, try first)
### Pragma 전용 변경(위험이 가장 낮음, 먼저 시도)

| Technique<br>기술 | Pragma<br>프라그마 | Effect<br>효과 | When to use<br>언제 사용하나요? |
|---|---|---|---|
| Acknowledge II<br>인정 II | `#pragma HLS PIPELINE II=<N>`<br>`#pragma HLS 파이프라인 II=<N>` | Prioritize II over satisfying timing pressure<br>타이밍 압박을 만족시키는 것보다 II를 우선시하십시오. | II violation in csynth report (to acknowledge the real achievable II)<br>csynth 보고서의 II 위반(실제로 달성 가능한 II를 인정하기 위해) |
| Loop unroll (partial)<br>루프 풀기(일부) | `#pragma HLS UNROLL factor=<N>`<br>`#pragma HLS UNROLL 요소=<N>` | Replicates loop body N times, reduces trip count by N<br>루프 본문을 N번 복제하고 트립 횟수를 N만큼 줄입니다. | Inner loop with known trip count divisible by N<br>N으로 나눌 수 있는 알려진 이동 횟수가 있는 내부 루프 |
| Loop unroll (full)<br>루프 풀기(전체) | `#pragma HLS UNROLL`<br>`#pragma HLS UNROLL` | Fully unrolls loop; all iterations in parallel<br>루프를 완전히 펼칩니다. 모든 반복을 병렬로 수행 | Short loops (≤ 16 iterations), sufficient resources, constant loop trip count<br>짧은 루프(반복 16회 이하), 충분한 리소스, 일정한 루프 트립 횟수 |
| Array partition (cyclic)<br>어레이 파티션(순환) | `#pragma HLS ARRAY_PARTITION variable=<x> cyclic factor=<N> dim=<D>`<br>`#pragma HLS ARRAY_PARTITION 변수=<x> 순환 인자=<N> 희미=<D>` | Creates N memory banks; N parallel accesses per cycle<br>N개의 메모리 뱅크를 생성합니다. 주기당 N 병렬 액세스 | Memory port II bottleneck<br>메모리 포트 II 병목 현상 |
| Array partition (complete)<br>어레이 파티션(완전) | `#pragma HLS ARRAY_PARTITION variable=<x> complete dim=<D>`<br>`#pragma HLS ARRAY_PARTITION 변수=<x> 전체 희미함=<D>` | Fully registers the array (no BRAM)<br>어레이를 완전히 등록합니다(BRAM 없음). | Small arrays only<br>작은 배열만 |
| Array partition (block)<br>어레이 파티션(블록) | `#pragma HLS ARRAY_PARTITION variable=<x> block factor=<N> dim=<D>`<br>`#pragma HLS ARRAY_PARTITION 변수=<x> 블록 요소=<N> 희미함=<D>` | Splits into N contiguous blocks<br>N개의 연속된 블록으로 분할됩니다. | When access pattern is block-sequential<br>액세스 패턴이 블록 순차인 경우 |
| Loop flatten<br>루프 편평화 | `#pragma HLS LOOP_FLATTEN`<br>`#pragma HLS LOOP_FLATTEN` | Merges nested loops into a single loop counter<br>중첩된 루프를 단일 루프 카운터로 병합합니다. | Nested loops with loop-invariant bounds<br>루프 불변 범위가 있는 중첩 루프 |
| Dataflow<br>데이터 흐름 | `#pragma HLS DATAFLOW`<br>`#pragma HLS DATAFLOW` | Task-level pipelining between functions or loops<br>함수 또는 루프 간의 작업 수준 파이프라인 | Sequential stages that can overlap, non-innerloop pipelining<br>내부 루프가 아닌 파이프라이닝과 겹칠 수 있는 순차적 단계 |
| Expression balance<br>표현의 균형 | `#pragma HLS EXPRESSION_BALANCE`<br>`#pragma HLS EXPRESSION_BALANCE` | Allow expression tree balancing on float<br>부동 소수점에서 식 트리 균형 조정 허용 | Floating point reductions in critical path<br>중요 경로의 부동 소수점 감소 |

### Algorithmic changes (higher risk, higher reward)
### 알고리즘 변경(더 높은 위험, 더 높은 보상)

| Technique<br>기술 | Effect<br>효과 | When to use<br>언제 사용하나요? |
|---|---|---|
| Loop reordering<br>루프 재정렬 | Changes memory access pattern; may improve spatial locality or expose better reuse<br>메모리 액세스 패턴을 변경합니다. 공간적 지역성을 향상시키거나 더 나은 재사용을 제공할 수 있습니다. | Poor memory reuse in inner loop; can turn random access into sequential<br>내부 루프의 메모리 재사용이 좋지 않습니다. 랜덤 액세스를 순차 액세스로 전환할 수 있습니다. |
| Tiling / blocking<br>타일링/블로킹 | Reuses on-chip data, reduces off-chip bandwidth, make buffer size constant<br>온칩 데이터 재사용, 오프칩 대역폭 감소, 버퍼 크기 일정화 | Memory bandwidth bound, variable size data to chunk into fixed size buffer<br>메모리 대역폭 제한, 가변 크기 데이터를 고정 크기 버퍼로 청크 |
| Local buffer<br>로컬 버퍼 | Stage data into local arrays to decouple memory access from compute<br>데이터를 로컬 배열로 스테이징하여 컴퓨팅에서 메모리 액세스를 분리합니다. | Repeated accesses to same memory region<br>동일한 메모리 영역에 대한 반복 액세스 |
| Scalarization<br>스칼라화 | Store data into individual scalars to allow arbitrary accesses and additional simplifications<br>임의 액세스 및 추가 단순화를 허용하기 위해 데이터를 개별 스칼라에 저장합니다. | Repeated accesses to the same memory element; can allow infinite access per cycle<br>동일한 메모리 요소에 대한 반복적인 액세스 주기당 무한한 액세스를 허용할 수 있습니다. |
| Loop fusion<br>루프 융합 | Merge two loop body into the same loop to reduce storage and allow for pipelining<br>두 개의 루프 본문을 동일한 루프로 병합하여 저장 공간을 줄이고 파이프라이닝을 허용합니다. | Two consecutive loops with identical loop trip count working on the same buffers<br>동일한 버퍼에서 작동하는 동일한 루프 트립 횟수를 갖는 두 개의 연속 루프 |
| Loop fission<br>루프 핵분열 | Split the loop body into two separate loop to create independent processes in dataflow context<br>루프 본문을 두 개의 개별 루프로 분할하여 데이터 흐름 컨텍스트에서 독립적인 프로세스를 만듭니다. | Loop body working on independent arrays/streams causing stall or complex control<br>독립적인 배열/스트림에서 작동하는 루프 본체로 인해 정지 또는 복잡한 제어가 발생함 |
| Replace `float` with `ap_fixed<W,I>`<br>`float`를 `ap_fixed<W,I>`로 교체하세요. | Shorter pipeline, fewer DSPs, faster clock<br>더 짧은 파이프라인, 더 적은 수의 DSP, 더 빠른 클럭 | Fixed-point arithmetic acceptable for the application<br>애플리케이션에 허용되는 고정 소수점 연산 |
| Replace division with reciprocal multiply<br>나눗셈을 역곱셈으로 대체 | Eliminates expensive divider<br>값비싼 분배기를 제거합니다. | Division inside a loop by constant divisor<br>루프 내에서 상수 제수로 나누기 |

### HLS library usage
### HLS 라이브러리 사용

```cpp
#include <ap_int.h>              // ap_int<N>, ap_uint<N> — arbitrary-width integers
#include <ap_fixed.h>            // ap_fixed<W,I> — fixed-point arithmetic
#include <ap_float.h>            // ap_float<W,E> — floating-point arithmetic with custom precision
#include <hls_math.h>            // hls::sqrt, hls::exp, etc. — HLS-optimized math
#include <hls_vector.h>          // hls::vector<T,N> — SIMD-style vectorized types
#include <hls_array_partition.h> // hls::scatter<T,K>, hls::gather<T,K> — SIMD-style scatter/gather on local arrays
#include <hls_stream.h>          // hls::stream<T> — FIFO communication between tasks
#include <hls_burst_maxi.h>      // hls::burst_maxi — high-performance M-AXI burst
```

## Step 4: Implement the change
## 4단계: 변경 사항 구현

Use the template to launch a sub-agent for each attempt.
템플릿을 사용하여 각 시도에 대해 하위 에이전트를 시작합니다.
Fill in all bracketed fields before launching.
실행하기 전에 괄호 안의 필드를 모두 입력하세요.

Key rules for the sub-agent:
하위 에이전트의 주요 규칙:
- Make **only** the described change.
- 설명된 변경 **만** 수행합니다.
- Do not opportunistically fix other things.
- 기회주의적으로 다른 것을 고치지 마십시오.
- After each tool run, **read the output carefully** before deciding to proceed.
- 각 도구 실행 후 진행을 결정하기 전에 **출력을 주의 깊게 읽으십시오**.
- If csynth warnings or loop table do not match the hypothesis, investigate before escalating.
- csynth 경고 또는 루프 테이블이 가설과 일치하지 않는 경우 에스컬레이션하기 전에 조사하세요.

## Step 5: Investigate the result thoroughly
## 5단계: 결과를 철저히 조사합니다.

For `run csynth`, do not just check the summary.
`run csynth`의 경우 요약만 확인하지 마세요.
Investigate systematically:
체계적으로 조사:

### 5a.
### Check for ignored-pragma warnings in the HLS log
### HLS 로그에서 무시된 pragma 경고를 확인하세요.

Do this first — before reading any report.
보고서를 읽기 전에 먼저 이 작업을 수행하십시오.
HLS frequently silently ignores pragmas that cannot be applied and emits a warning explaining why.
HLS는 적용할 수 없는 pragma를 자동으로 무시하고 이유를 설명하는 경고를 표시하는 경우가 많습니다.
If a pragma was ignored, the change you intended was **not applied** and the csynth result reflects the unmodified schedule.
pragma가 무시된 경우 의도한 변경 사항은 **적용되지** 않았으며 csynth 결과는 수정되지 않은 일정을 반영합니다.
Reading the reports without knowing this leads to wrong conclusions.
이를 모르고 보고서를 읽으면 잘못된 결론을 얻게 됩니다.

check `pragma\|ignored\|cannot\|not applied\|warning` in `$synthesisLogContent` for any ignored pragma warnings.
무시된 pragma 경고에 대해서는 `$synesisLogContent`에서 `pragma\|ignored\|cannot\|not Apply\|warning`을 확인하세요.

Common ignored-pragma patterns to look for:
찾아야 할 일반적인 무시된 pragma 패턴은 다음과 같습니다.

| Warning text (approximate)<br>경고 텍스트(대략) | Meaning<br>의미 |
|---|---|
| `pragma HLS PIPELINE ignored`<br>`pragma HLS 파이프라인이 무시됨` | Loop is not eligible for pipelining (e.g. contains a sub-loop, variable trip count, or non-trivial control flow).<br>루프는 파이프라이닝에 적합하지 않습니다(예: 하위 루프, 가변 트립 횟수 또는 중요하지 않은 제어 흐름 포함). |
| `pragma HLS UNROLL factor=N ignored`<br>`pragma HLS UNROLL 요소=N 무시됨` | Loop body cannot be fully unrolled (e.g. trip count not statically known, or factor does not divide trip count).<br>루프 본문을 완전히 펼칠 수 없습니다(예: 이동 횟수를 정적으로 알 수 없거나 요소가 이동 횟수를 나누지 않음). |
| `pragma HLS ARRAY_PARTITION … ignored`<br>`pragma HLS ARRAY_PARTITION … 무시됨` | Variable is not a local array, or partition dim exceeds array rank.<br>변수가 로컬 배열이 아니거나 파티션 치수가 배열 순위를 초과합니다. |
| `pragma HLS LOOP_FLATTEN ignored`<br>`pragma HLS LOOP_FLATTEN이 무시됨` | Loop bounds are not loop-invariant — flattening would change semantics.<br>루프 경계는 루프 불변이 아닙니다. 평면화하면 의미가 변경됩니다. |
| `pipeline II target … cannot be achieved`<br>'파이프라인 2 목표…달성 불가능' | The target II was set but the scheduler could only achieve a higher value. Check the Violation Report for root cause.<br>목표 II가 설정되었지만 스케줄러는 더 높은 값만 달성할 수 있었습니다. 근본 원인은 위반 보고서를 확인하세요. |

If a pragma was ignored, fix the pragma or restructure the code and re-run before drawing any conclusions from the reports.
pragma가 무시된 경우 보고서에서 결론을 도출하기 전에 pragma를 수정하거나 코드를 재구성하고 다시 실행하십시오.

### 5b.
### Read the synthesis report
### 종합 보고서 읽기

Use the `synthesisReportContent` to check the loop table for the loops of interest.
관심 있는 루프에 대한 루프 테이블을 확인하려면 `synesisReportContent`를 사용하세요.
Look at II achieved vs. target, iteration latency, and resource usage.
II 달성 대 목표, 반복 대기 시간 및 리소스 사용량을 살펴보세요.

Check:
확인:
- Did II change on the expected loop?
- 예상 루프에서 II가 변경되었나요?
- Did II change on any other loop (unexpected side effect)?
- 다른 루프에서 II가 변경되었나요(예기치 않은 부작용)?
- Did total latency change by the expected factor?
- 예상 요인에 따라 총 지연 시간이 변경되었나요?
- Did resource usage change?
- 리소스 사용량이 변경되었나요?
- By how much?
- 얼마만큼?

Also check for general synthesis warnings:
또한 일반적인 합성 경고도 확인하세요.

check `warning\|critical\|fail\|error` in `synthesisReportContent` and `synthesisLogContent` for any synthesis warnings.
합성 경고가 있는지 `synesisReportContent`에서 `warning\|critical\|fail\|error`를 확인하고 `synesisLogContent`를 확인하세요.

### 5c.
### Read the Violation
### 위반 사항 읽기

Check violation in `synthesisLogContent` for any II or Timing violations.
II 또는 타이밍 위반이 있는지 `synesisLogContent`에서 위반을 확인하세요.

### 5d.
### Read the inferred directives
### 추론된 지시어 읽기

First, find the inferred directives file within the component directory:
먼저 구성 요소 디렉터리에서 추론된 지시문 파일을 찾습니다.

```bash
find $component -name "inferred_directives.ini" 2>/dev/null
```

Then read the file to check whether HLS auto-applied any directives you didn't expect.
그런 다음 파일을 읽어 HLS가 예상하지 못한 지시문을 자동 적용했는지 확인하세요.
Array partitions, loop flattening, pipeline pragmas can be inferred automatically and can interact with your pragma.
배열 파티션, 루프 평면화, 파이프라인 pragma는 자동으로 추론될 수 있으며 pragma와 상호 작용할 수 있습니다.

### 5e.
### Verify transformation in synthesis log (when hypothesis is about compiler behavior)
### 합성 로그에서 변환 확인(가설이 컴파일러 동작에 관한 경우)

When you want to verify that HLS actually applied the transformation you intended (e.g., that a partition was respected, or that a loop was actually unrolled), search `synthesisLogContent` for key indicators.
HLS가 의도한 변환을 실제로 적용했는지 확인하려면(예: 파티션이 존중되었는지 또는 루프가 실제로 풀렸는지) 핵심 지표에 대해 'synesisLogContent'를 검색하세요.

**Search patterns by pragma type:**
**pragma 유형별로 패턴 검색:**

| Pragma<br>프라그마 | Search pattern in `synthesisLogContent`<br>`synesisLogContent`의 검색 패턴 | Confirmation<br>확인 |
|--------|----------------------------------------|--------------|
| `ARRAY_PARTITION`<br>`ARRAY_PARTITION` | `partitioned\<br>`파티션됨\|partition.*factor\<br>파티션.*인자\|partition.*complete`<br>파티션.*완료` | Look for "partitioned into N banks" or similar messages<br>"N개 뱅크로 분할됨" 또는 유사한 메시지를 찾으세요. |
| `UNROLL factor=N`<br>`언롤 요소=N` | `unroll\<br>`풀기\|unrolled\<br>펼쳐진\|trip count`<br>여행 횟수` | Look for "loop completely unrolled" or "loop unrolled by factor N"<br>"완전히 풀린 루프" 또는 "인자 N에 의해 풀린 루프"를 찾습니다. |
| `LOOP_FLATTEN`<br>`LOOP_FLATTEN` | `flatten\<br>`평탄화\|flattened\<br>평탄화\|merged`<br>병합` | Look for "loop flattened" or "loops merged" messages<br>"루프 병합" 또는 "루프 병합" 메시지를 찾습니다. |
| `PIPELINE`<br>`파이프라인` | `pipeline\<br>`파이프라인\|pipelined\<br>파이프라인\|II=` | Look for "loop pipelined with II=N"<br>"II=N으로 파이프라인된 루프"를 찾으세요. |
| `DATAFLOW`<br>'데이터플로우' | `dataflow\<br>`데이터 흐름\|FIFO\<br>FIFO\|channel`<br>채널` | Look for dataflow region and channel creation messages<br>데이터 흐름 지역 및 채널 생성 메시지를 찾습니다. |

**Example grep commands:**
**grep 명령 예:**

```bash
# Check if partition was applied
echo "$synthesisLogContent" | grep -i "partition"

# Check if unroll was applied
echo "$synthesisLogContent" | grep -i "unroll"

# Check if pipeline achieved target II
echo "$synthesisLogContent" | grep -i "pipeline.*II"

# General transformation summary
echo "$synthesisLogContent" | grep -iE "applied|transformed|optimized|inferred"
```

**Key things to look for:**
**찾아야 할 주요 사항:**
- After `ARRAY_PARTITION`: messages confirming partition into N banks, or warnings if partition was not applied
- `ARRAY_PARTITION` 이후: N개 뱅크로 분할을 확인하는 메시지 또는 분할이 적용되지 않은 경우 경고
- After `UNROLL factor=N`: messages showing "unrolled by factor N" or "completely unrolled"
- `UNROLL 요소=N` 이후: "요소 N에 의해 언롤링됨" 또는 "완전히 언롤링됨"을 표시하는 메시지
- After `LOOP_FLATTEN`: messages showing nested loops were merged into a single loop
- `LOOP_FLATTEN` 이후: 중첩 루프를 표시하는 메시지가 단일 루프로 병합되었습니다.
- After `PIPELINE`: messages showing achieved II value
- `PIPELINE` 이후: 달성된 II 값을 보여주는 메시지

**If the expected transformation message is not found**, the pragma may have been ignored — cross-reference with Step 5a for ignored-pragma warnings.
**예상되는 변환 메시지를 찾을 수 없는 경우** pragma가 무시되었을 수 있습니다. 무시된 pragma 경고에 대해서는 5a단계를 상호 참조하세요.

### 5f.
### Log the result
### 결과 기록

```bash
python3 "$qorLogScriptFullPath" --report "$synthesisReportContent" --label "<technique-applied>" --csv .memory/qor_history.csv
```

This appends a row to `.memory/qor_history.csv` for tracking across attempts.
이는 여러 시도를 추적하기 위해 `.memory/qor_history.csv`에 행을 추가합니다.

## Step 6: Record the result — MANDATORY after every attempt
## 6단계: 결과 기록 — 모든 시도 후에 필수

**Do this before deciding whether to escalate or start the next attempt.
**에스컬레이션할지 아니면 다음 시도를 시작할지 결정하기 전에 이 작업을 수행하세요.
Never skip this step.**
이 단계를 절대 건너뛰지 마세요.**

### 6a.
### Update `.memory/optimize_outcomes.md`
### `.memory/optimize_outcomes.md` 업데이트

Append a row to the outcomes table immediately after reading the csynth report.
csynth 보고서를 읽은 후 즉시 결과 테이블에 행을 추가합니다.
Use the present attempt's data — do not wait until the end of the session.
현재 시도의 데이터를 사용합니다. 세션이 끝날 때까지 기다리지 마십시오.
Include:
포함:
- Technique name
- 기술명
- Observed effect (cycles before → after, II change)
- 관찰된 효과(주기 전 → 후, II 변경)
- Whether the hypothesis was confirmed
- 가설이 확인되었는지 여부
- Any unexpected behavior, side effects, or lessons
- 예상치 못한 행동, 부작용 또는 교훈

This file is read at the start of every session.
이 파일은 모든 세션이 시작될 때 읽혀집니다.
If it is not updated after each attempt, the next session will repeat experiments that have already been run.
시도할 때마다 업데이트되지 않으면 다음 세션에서는 이미 실행된 실험을 반복합니다.

### 6b.
### Commit the state — MANDATORY, including for failed attempts
### 상태 커밋 — MANDATORY(실패한 시도 포함)

**Every attempt gets its own commit before anything else happens — including attempts that did not improve the metric.** The commit must happen *before* any `git checkout`, `git restore`, file edit, or next-attempt work.
**측정항목을 개선하지 못한 시도를 포함하여 모든 시도는 다른 일이 발생하기 전에 자체 커밋을 받습니다.** 커밋은 `git checkout`, `git Restore`, 파일 편집 또는 다음 시도 작업 *전에* 이루어져야 합니다.
Skipping the commit on failure loses the experiment from history and is the most common mistake in this workflow.
실패 시 커밋을 건너뛰는 것은 기록에서 실험을 잃게 되며 이 워크플로에서 가장 흔한 실수입니다.

Order of operations after every attempt (no exceptions):
모든 시도 후 작업 순서(예외 없음):
1. Update `.memory/optimize_outcomes.md` (Step 6a).
1. `.memory/optimize_outcomes.md`를 업데이트합니다(6a단계).
2. `git add` + `git commit` (this step) — even if the build broke or the result was worse.
2. `git add` + `git commit`(이 단계) — 빌드가 중단되었거나 결과가 더 나빴더라도 마찬가지입니다.
3. *Only then* may you revert, adapt, or move on.
3. *그때에만* 되돌리거나, 적응하거나, 계속 나아갈 수 있습니다.

**If the attempt improved the target metric or confirmed a useful hypothesis (keep the code):**
**시도를 통해 목표 측정항목이 개선되었거나 유용한 가설이 확인된 경우(코드 유지):**

```bash
git add <source_file> .memory/optimize_outcomes.md
git commit -F - <<'EOF'
<technique>: <what changed> → <observed effect>

Hypothesis: <what you expected>
Observed: <what actually happened in csynth/cosim/impl>
Baseline: cosim=<X> cycles, post-route=<Y> ns, true latency=<Z> ms
After:    cosim=<X'> cycles, post-route=<Y'> ns, true latency=<Z'> ms
Resources: LUT <A>→<A'>, DSP <B>→<B'>
EOF
```

**If the attempt did not help, latency got worse or not improved — commit first, then revert:**
**시도가 도움이 되지 않으면 대기 시간이 악화되거나 개선되지 않은 것입니다. 먼저 커밋한 다음 되돌리세요.**

```bash
# 1. Commit the failed state as-is (code + memory update) so the exact change is preserved
git add <source_file> .memory/optimize_outcomes.md
git commit --allow-empty -F - <<'EOF'
experiment(<technique>): <what changed> — did not help

Hypothesis: <what you expected>
Observed: <what actually happened — why it failed or made QoR worse / etc.>
EOF

# 2. Verify the commit actually landed before reverting. If `git status` is not clean
#    after the commit, STOP and investigate — do not run the checkout below.
git log -1 --oneline
git status --short

# 3. Only after the commit is confirmed, revert the kernel to the previous working state.
git checkout HEAD~1 -- <source_file>
```

`--allow-empty` ensures the commit still lands even if the only thing that changed was a file that has since been auto-regenerated to its prior content (e.g., a report).
`--allow-empty`는 이전 내용(예: 보고서)으로 자동 재생성된 파일만 변경된 경우에도 커밋이 계속 실행되도록 합니다.
The verification step (`git log -1` + `git status --short`) is what catches the case where the commit silently did not happen — if you see your commit in `git log` and `git status` is clean, the failed attempt is safely preserved and you may revert.
확인 단계(`git log -1` + `git status --short`)는 커밋이 자동으로 발생하지 않은 경우를 포착하는 것입니다. `git log`에서 커밋이 확인되고 `git status`가 깨끗하면 실패한 시도가 안전하게 보존되고 되돌릴 수 있습니다.

Committing before reverting preserves the exact failing code in history, making it possible to inspect, diff, or re-apply it later.
되돌리기 전에 커밋하면 실패한 코드를 정확하게 기록에 보존하여 나중에 검사하고, 비교하고, 다시 적용할 수 있습니다.
A failed experiment that teaches you something is not wasted work.
무언가를 가르쳐 주는 실패한 실험은 낭비되는 작업이 아닙니다.

## Step 7: Decide whether to escalate
## 7단계: 에스컬레이션할지 결정

| csynth result<br>신디사이저 결과 | Action<br>액션 |
|---|---|
| Latency improved ≥ 10% OR II reduced on a dominant loop<br>지연 시간 개선 ≥ 10% 또는 지배적 루프에서 II 감소 | Escalate: run cosimulation flow<br>에스컬레이션: 통합 시뮬레이션 흐름 실행 |
| Latency unchanged, but hypothesis was confirmed (e.g. II matched expected value)<br>지연 시간은 변하지 않았지만 가설은 확인되었습니다(예: II가 예상 값과 일치함). | Valid data. Move on to the next hypothesis.<br>유효한 데이터. 다음 가설로 넘어갑니다. |
| Latency got worse<br>지연 시간이 더 심해졌습니다. | Investigate why, then **commit (Step 6b) before reverting**. The result may reveal a real constraint.<br>이유를 조사한 다음 **되돌리기 전에 커밋(6b단계)**하세요. 결과는 실제 제약 조건을 드러낼 수 있습니다. |
| csim FAIL<br>csim 실패 | Fix logic error or revert immediately.<br>논리 오류를 수정하거나 즉시 되돌리세요. |

For `run cosim`: call `hls-cosim-report` skill to get `coSimulationReportContent` and `dataflowProcessAndChannelContent`.
`cosim 실행`의 경우 `hls-cosim-report` 스킬을 호출하여 `coSimulationReportContent` 및 `dataflowProcessAndChannelContent`를 가져옵니다.
Compare cosim cycles to csynth estimate.
cosim 주기를 csynth 추정치와 비교합니다.
For designs with memory backpressure, cosim can be significantly worse than csynth estimates; for compute-bound kernels, they should be close.
메모리 배압이 있는 설계의 경우 cosim은 csynth 추정치보다 훨씬 나쁠 수 있습니다. 컴퓨팅 바인딩된 커널의 경우 가까워야 합니다.

For `run impl`: only run when cosim confirms latency improvement.
`run impl`의 경우: cosim이 대기 시간 개선을 확인한 경우에만 실행됩니다.
Call `hls-impl-report` skill and use `postRoute` for the actual post-route clock period and compute true latency = cosim_cycles × period_ns.
'hls-impl-report' 스킬을 호출하고 실제 경로 후 클록 기간에 대해 'postRoute'를 사용하고 실제 대기 시간 = cosim_cycles × period_ns를 계산합니다.

**DO NOT** read `.rpt` files directly using file read tools to extract information.
**하지 마세요** 정보 추출을 위해 파일 읽기 도구를 사용하여 `.rpt` 파일을 직접 읽지 마세요.
Always use the provided functions to ensure consistency and correctness.
일관성과 정확성을 보장하기 위해 항상 제공된 기능을 사용하십시오.

## Step 8: Update skills with lessons learned
## 8단계: 학습한 내용으로 기술 업데이트

After each attempt — successful or not — update the relevant `SKILL.md` with findings that are general enough to apply to future kernels:
성공 여부에 관계없이 각 시도 후에 관련 `SKILL.md`를 향후 커널에 적용할 수 있을 만큼 일반적인 결과로 업데이트합니다.

- If a technique had an unexpected interaction with HLS (e.g., auto-inferred flatten conflicted with your pipeline pragma), note it.
- 기술에 HLS와 예상치 못한 상호 작용이 있는 경우(예: 자동 추론 평면화가 파이프라인 프라그마와 충돌함) 이를 기록해 두세요.
- If the Violation Report pointed to a root cause that wasn't obvious from the loop table, note the pattern.
- 위반 보고서가 루프 테이블에서 명확하지 않은 근본 원인을 지적한 경우 패턴을 기록해 두십시오.
- If the IR inspection revealed something unexpected (e.g., the partition was not reflected in the IR), note it.
- IR 검사 결과 예상치 못한 내용이 발견된 경우(예: 파티션이 IR에 반영되지 않음) 이를 기록해 두십시오.

## Cost model for the inner loop
## 내부 루프의 비용 모델

| Step<br>단계 | Typical runtime<br>일반적인 런타임 | When to run<br>실행 시기 |
|---|---|---|
| `run csim`<br>`csim 실행` | 5–30 s | Always — correctness gate<br>항상 - 정확성 게이트 |
| `run csynth`<br>`신신스 실행` | 10–60 s | Always — the main investigation tool<br>항상 — 주요 조사 도구 |
| `run cosim`<br>'코심 실행' | 30 s – 5 min<br>30초 – 5분 | When csynth shows ≥ 10% improvement in latency cycles, or a structural improvement (II reduction on dominant loop)<br>csynth가 레이턴시 주기가 10% 이상 개선되거나 구조적 개선을 보이는 경우(우성 루프의 II 감소) |
| `run impl`<br>`impl 실행` | 1 – 30 min<br>1~30분 | When cosim confirms improvement<br>cosim이 개선을 확인한 경우 |

## Stopping criteria
## 중지 기준

Stop iterating when any of the following is true:
다음 중 하나라도 해당되면 반복을 중지합니다.
- The optimization criteria is met.
- 최적화 기준이 충족되었습니다.
- Three consecutive attempts produced no improvement, and no new hypothesis remains to test.
- 세 번 연속 시도했지만 개선되지 않았으며 테스트할 새로운 가설도 남아 있지 않습니다.
- Resource utilization is approaching device limits (> 60% of any constrained resource) and further improvement requires more resources.
- 리소스 활용도가 장치 한도(제한된 리소스의 60% 초과)에 가까워지고 있으며 추가 개선을 위해서는 더 많은 리소스가 필요합니다.
- The improvement per attempt has diminished to < 5% of baseline.
- 시도당 개선이 기준선의 5% 미만으로 감소했습니다.

When stopping criteria are met, summarize the lessons learned.
중지 기준이 충족되면 배운 내용을 요약합니다.

**In interactive mode**: ask the user if they want to update the relevant `SKILL.md` with the new insights.
**대화형 모드**: 사용자에게 관련 `SKILL.md`를 새로운 통찰력으로 업데이트할지 물어보세요.
The update role is described in step 8 above.
업데이트 역할은 위의 8단계에 설명되어 있습니다.

**In non-interactive mode** (invoked via `claude --print`): skip the prompt — do NOT update any `SKILL.md`.
**비대화형 모드**(`claude --print`를 통해 호출됨): 프롬프트를 건너뛰고 `SKILL.md`를 업데이트하지 마세요.
Print the summary and exit.
요약을 인쇄하고 종료합니다.

## Known outcomes
## 알려진 결과

Outcomes are recorded in `.memory/optimize_outcomes.md`.
결과는 `.memory/optimize_outcomes.md`에 기록됩니다.
Read this file at the start of each session to avoid repeating experiments.
실험을 반복하지 않으려면 각 세션을 시작할 때 이 파일을 읽으세요.
