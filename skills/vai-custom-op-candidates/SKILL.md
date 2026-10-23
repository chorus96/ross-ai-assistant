<!-- ko-translation: enabled -->
---
name: vai-custom-op-candidates
description: Find customOperator optimization chains and their ONNX node boundaries
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
---
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
-->

# Finding customOperator optimization opportunities
# customOperator 최적화 기회 찾기

Goal: find every place in a compiled model where one custom AIE kernel could replace a group of operators, and report them ranked, each with the ONNX start and end node that bound it.
목표: 하나의 사용자 정의 AIE 커널이 연산자 그룹을 대체할 수 있는 컴파일된 모델의 모든 위치를 찾고 각 위치를 바인딩한 ONNX 시작 및 끝 노드와 함께 순위를 보고합니다.

Work through sections 1–6 in order, format with section 7, then run the checklist in section 8 before replying.
섹션 1~6을 순서대로 진행하고 섹션 7로 형식을 지정한 다음 응답하기 전에 섹션 8의 체크리스트를 실행하세요.
Each rule is stated once, in its own section; where another section needs it, it is referenced by number rather than repeated.
각 규칙은 해당 섹션에 한 번만 명시됩니다. 다른 섹션에서 필요한 경우 반복되지 않고 번호로 참조됩니다.

---

## 0.
## Output contract
## 출력 계약

The entire reply is these things, in this order, and nothing else:
전체 답변은 순서대로 다음과 같습니다.

1. the summary table (§7.1)
1. 요약표(§7.1)
2. the context block (§7.4)
2. 컨텍스트 블록(§7.4)
3. **the dominant-operator line** — the gather call's `headline` string, printed verbatim when it is non-empty, after the context block (§7.4)
3. **주요 연산자 라인** — 수집 호출의 `헤드라인` 문자열, 비어 있지 않은 경우 컨텍스트 블록 뒤에 축어적으로 인쇄됨(§7.4)
4. the topology drawing (§7.5)
4. 토폴로지 도면(§7.5)

The order is fixed.
순서는 정해져 있습니다.
The question asked is where a custom op would help, so the table answers it first; the dominating operator is context for that answer rather than a preface to it.
질문은 사용자 정의 작업이 어디에 도움이 되는지에 관한 것이므로 테이블이 먼저 이에 대한 답변을 제공합니다. 지배적인 연산자는 해당 답변의 서문이 아니라 해당 답변의 컨텍스트입니다.
It still has to appear, and in full: when one operator takes more of the model's time than every fusion in the table combined, a reader who never reaches it has been misled.
그것은 여전히 ​​나타나야 하며, 전체적으로: 한 운영자가 테이블의 모든 융합을 합친 것보다 모델 시간의 더 많은 시간을 소비한다면, 그것에 도달하지 못한 독자는 잘못된 길로 인도된 것입니다.
Print it after the context block, never inside it as a bullet, and never as a substitute for a row.
컨텍스트 블록 뒤에 인쇄하고 글머리 기호로 내부에 인쇄하거나 행을 대체하지 마십시오.
The sentence itself comes back from the tool, so printing it is a copy, not a composition — five wordings of a placement rule did not hold before that.
문장 자체는 도구에서 돌아오므로 이를 인쇄하는 것은 구성이 아닌 사본입니다. 이전에는 배치 규칙의 다섯 가지 문구가 유지되지 않았습니다.

**Write for someone who has never seen this skill or its tools.** No tool names, no field names, no section numbers.
**이 기술이나 해당 도구를 본 적이 없는 사람을 위해 작성하십시오.** 도구 이름, 필드 이름, 섹션 번호가 없습니다.
`zero_gain_count`, `no_span_count`, `nodes_without_timing`, `get_fusion_candidates` and "this is the §6.4 case" all appeared in shipped replies; every one of them asks the reader to know how the answer was produced.
`zero_gain_count`, `no_span_count`, `nodes_without_timing`, `get_fusion_candidates` 및 "이것은 §6.4 사례입니다"는 모두 배송된 응답에 나타났습니다. 그들 모두는 독자에게 대답이 어떻게 산출되었는지 알고 싶어합니다.
Say the thing in English: "4 regions were found and dropped for having nothing absorbable", "3 of its 7 operators have no timing row of their own".
영어로 말해보세요: "4개 지역은 흡수할 수 있는 것이 없기 때문에 발견되어 삭제되었습니다.", "7명의 운영자 중 3명은 자체 타이밍 행이 없습니다."
Node names, operator types and shapes are the model's vocabulary, not ours — those stay.
노드 이름, 연산자 유형 및 모양은 모델의 어휘이지 우리의 어휘는 아닙니다.

**Say nothing while working.** No "I'll start by gathering…", no "Let me fetch…", no "Good, 100% offload confirmed", no announcing the next step.
**작업하는 동안 아무 말도 하지 마세요.** 아니요 "모음부터 시작하겠습니다…", 아니요 "...를 가져오겠습니다", 아니요 "좋습니다. 100% 오프로드 확인됨", 다음 단계를 알리지 않습니다.
Text emitted between tool calls is part of the reply and counts against this.
도구 호출 사이에 생성된 텍스트는 응답의 일부이며 이에 포함됩니다.
Run the tools silently; the table is the first thing you write.
도구를 자동으로 실행합니다. 테이블은 당신이 쓰는 첫 번째 것입니다.

**No prose dumps.** Extracted timing rows, per-node cost lists, chain-walk findings, span-tracing steps, commentary on what a tool returned — working material, not output.
**산문 덤프가 없습니다.** 추출된 타이밍 행, 노드당 비용 목록, 체인 워크 결과, 범위 추적 단계, 도구 반환 내용에 대한 설명(출력이 아닌 작업 자료).

**But facts still get reported.** Several rules require disclosing something (breaker types, truncated timing, unresolved shapes, match coverage).
**그러나 사실은 여전히 ​​보고됩니다.** 여러 규칙에서는 무언가(브레이커 유형, 잘린 타이밍, 해결되지 않은 모양, 일치 범위)를 공개해야 합니다.
Those are results: they go in the context block or a table cell, one clipped line each.
결과는 다음과 같습니다. 컨텍스트 블록이나 테이블 셀에 각각 한 줄씩 잘립니다.
This is a ban on narration, not on disclosure.
이는 공개가 아닌 내레이션 금지입니다.
Where the base rules require saying you did not read a whole paginated table, the context block is where that goes.
기본 규칙에서 페이지가 매겨진 전체 테이블을 읽지 않았다고 요구하는 경우 컨텍스트 블록이 해당 위치에 있습니다.

**Stop after the topology.** No "Key observation", no summary of the table, no restating a caveat, no "the real lever here is…".
**토폴로지 이후에 중지합니다.** "주요 관찰" 없음, 표 요약 없음, 주의 사항 다시 설명 없음, "여기서 실제 레버는…" 없음.
If the analysis found nothing worth a row, that is still a table — one row reading `*(none reportable)*` — plus the context block and the topology, not a paragraph of apology.
분석 결과 행에 해당하는 값이 발견되지 않은 경우에도 사과 문단이 아닌 테이블('*(none Reportable)*'을 읽는 행 1개)과 컨텍스트 블록 및 토폴로지가 추가됩니다.

**Compose the answer once, when the data is in.** Everything above is written after the last tool call, in one pass.
**데이터가 입력되면 답변을 한 번 작성하세요.** 위의 모든 내용은 마지막 도구 호출 후 한 번에 작성됩니다.
Do not begin the table and revise it: two replies shipped with a half-written row ending `... wait |  |  |  |`, a line reading "Let me correct:", and a second table underneath.
테이블을 시작하고 수정하지 마십시오. `... wait |  |  |  |`, "수정하겠습니다:"라는 줄과 그 아래에 두 번째 테이블이 있습니다.
Both tables render.
두 테이블 모두 렌더링됩니다.
A reader sees an abandoned draft and cannot tell which one is the answer.
독자는 버려진 초안을 보고 어느 것이 답인지 알 수 없습니다.
Working out that a grouping was wrong is fine — doing it in the reply is not.
그룹화가 잘못되었다는 것을 알아내는 것은 괜찮습니다. 응답에서 그것을 하는 것은 그렇지 않습니다.

---

## 1.
## Gather (one call)
## 집결(한 번의 통화)

```
get_fusion_candidates(folder)
```

It returns everything the reply needs: every fusible region in the model already costed, spanned, linked and ranked, plus `npu_time_us` (the denominator for every `% of NPU`) and `reason_counts`. **The regions you report come from here and nowhere else, and so do their figures.**
이는 응답에 필요한 모든 것을 반환합니다. 모델의 모든 가용성 영역은 이미 비용이 지정되고, 확장되고, 연결되고 순위가 매겨져 있으며, 'npu_time_us'(모든 'NPU %'의 분모) 및 'reason_counts'도 포함됩니다. **귀하가 보고한 지역은 여기에서 나온 것이며 다른 곳에서는 나오지 않으며 해당 수치도 마찬가지입니다.**

A candidate's `total_us` and `gain_us` are summed across its `occurrences`.
후보자의 'total_us'와 'gain_us'는 '발생 횟수'에 걸쳐 합산됩니다.
`get_region_cost` costs **one instance** of the nodes you hand it.
`get_region_cost`는 귀하가 전달하는 노드의 **인스턴스** 하나에 대한 비용을 지불합니다.
The two are therefore not comparable, and calling `get_region_cost` on a candidate to "confirm" it does not confirm anything — on `ssd_resnet34` it replaced a correct `169585.48 µs` row with `575.67 µs`, a figure that also covered only 2 of the region's 3 nodes. **Never overwrite a candidate's `total_us` or `gain_us` with a `get_region_cost` result.** That tool is for a region the user named that is not in the candidate list.
따라서 둘은 비교할 수 없으며 후보에 대해 `get_region_cost`를 호출하여 "확인"해도 아무 것도 확인되지 않습니다. `ssd_resnet34`에서는 올바른 `169585.48 µs` 행이 `575.67 µs`로 대체되었습니다. 이 수치는 지역의 3개 노드 중 2개만 포함하는 수치이기도 합니다. **후보자의 `total_us` 또는 `gain_us`를 `get_region_cost` 결과로 덮어쓰지 마십시오.** 해당 도구는 사용자가 명명한 지역 중 후보 목록에 없는 지역을 위한 것입니다.

Rows go in `rank` order, `rank: 1` first, and `rank: 1` is the row you bold.
행은 '순위' 순서로 표시됩니다. '순위: 1'이 먼저 표시되고 '순위: 1'이 굵은 행으로 표시됩니다.
Do not re-order on a judgement about which region is more interesting.
어느 지역이 더 흥미로운지에 대한 판단을 다시 주문하지 마십시오.

Make it your first call and do not follow it with `get_subgraph_reasons` or `get_npu_time_summary` for figures it has already given you — each extra round-trip costs about six seconds of the reader's time, measured across the P0 set, and buys nothing.
첫 번째 호출로 만들고 이미 제공한 수치에 대해 `get_subgraph_reasons` 또는 `get_npu_time_summary`를 사용하지 마세요. 각각의 추가 왕복에는 P0 세트 전체에서 측정된 독자 시간의 약 6초가 소요되며 아무것도 구매하지 않습니다.

**1.1 is the analysis.** It removes the heavy-compute operators from the operator graph and takes the connected runs that remain, so the region set is exhaustive by construction and identical on every call.
**1.1은 분석입니다.** 연산자 그래프에서 계산량이 많은 연산자를 제거하고 남아 있는 연결된 실행을 가져오므로 지역 집합은 구성에 따라 철저하고 모든 호출에서 동일합니다.
Use its `candidates` in the order given.
주어진 순서대로 '후보'를 사용하세요.

This replaces walking the graph.
이는 그래프를 걷는 것을 대체합니다.
Do not call `get_operator_edges` to look for regions it missed, do not assemble a region by hand, and do not report a region that is not in its list.
누락된 영역을 찾기 위해 `get_operator_edges`를 호출하지 말고, 영역을 직접 조립하지 말고, 목록에 없는 영역을 보고하지 마세요.
Enumerating by hand is what made this analysis unrepeatable: on `ssd_resnet34` three runs of this same question put the top opportunity at **4715, 433 and 8739 µs**, and on `yolo12m` one run said nothing was reportable while the next found a 15.5% win.
직접 열거하는 것이 이 분석을 반복할 수 없게 만든 이유입니다. `ssd_resnet34`에서 이 동일한 질문을 세 번 실행하면 **4715, 433 및 8739 µs**에 최고의 기회가 있었고, `yolo12m`에서는 한 번 실행하면 보고할 수 있는 것이 없다고 말했고 다음 실행에서는 15.5%의 승리를 발견했습니다.
Neither run was careless; the search simply started somewhere different.
어느 쪽도 부주의하지 않았습니다. 검색은 단순히 다른 곳에서 시작되었습니다.

A candidate whose span cannot be cut is already gone from the list (counted in `no_span_count`), so every entry you are given can be reported as-is.
범위를 잘라낼 수 없는 후보는 이미 목록에서 사라졌으므로(`no_span_count`로 계산됨) 제공된 모든 항목은 있는 그대로 보고될 수 있습니다.

Also read from 1.1:
1.1에서도 읽어보세요:

- `dominant_operators` — single operators above 5% of NPU time. **§7.4 requires reporting these.**
- `dominant_operators` — NPU 시간의 5%를 초과하는 단일 연산자. **§7.4에서는 이를 보고해야 합니다.**
- `cpu_operators` — anything not on the NPU (§2.4).
- `cpu_operators` — NPU에 없는 모든 것(§2.4).
- `zero_gain_count` — regions found and dropped for having nothing absorbable (§2.3).
- `zero_gain_count` — 흡수할 수 있는 것이 없어 발견되어 삭제된 영역입니다(§2.3).
- Report the number in English, never the field name.
- 숫자는 영어로 보고하고 필드 이름은 보고하지 마세요.
- `zero_gain_regions` — the costliest of those, named. **Report each one in the context block**: what the block is, its span, its cost, and its `ceiling_us` described as a ceiling in words.
- `zero_gain_regions` — 가장 비용이 많이 드는 지역입니다. **컨텍스트 블록에서 각각 보고하세요**: 블록이 무엇인지, 범위, 비용, 단어로 천장으로 설명되는 'ceiling_us'.
- On `Anduril_yolox_l` the SPP block — a Sigmoid, three MaxPools and a Concat — costs 628.74 µs and scored zero, and a bare count let a reader conclude the model had nothing in it worth looking at.
- 'Anduril_yolox_l'에서 SPP 블록(Sigmoid, MaxPool 3개, Concat 1개)은 비용이 628.74μs이고 점수는 0이며, 독자는 이 모델에 살펴볼 가치가 전혀 없다고 결론을 내릴 수 있습니다.
A ceiling is not a gain.
천장은 이익이 아닙니다.
Write "a fused kernel could not save more than 272 µs of it", never "expected gain 272 µs", and never put it in the table: a fused SPP still pools three times, and what fusion actually removes is the traffic between those pools, which sits inside the pools' own measured time where it cannot be separated out.
"융합된 커널은 272μs 이상을 저장할 수 없습니다"라고 쓰고 "272μs의 이득을 기대하지 않습니다"라고 쓰지 말고 테이블에 넣지 마십시오. 융합된 SPP는 여전히 세 번 풀을 수행하며 실제로 제거하는 것은 분리할 수 없는 풀 자체 측정 시간 내에 있는 해당 풀 사이의 트래픽입니다.

Do not call `get_single_timing_dataframe`: it is paginated, mixes leaf operators with containers and `PM Load` rows, and carries a duplicate of every operator row on the CPU timeline, so a hand lookup understates some regions and double-counts others while looking entirely plausible.
`get_single_timing_dataframe`을 호출하지 마세요. 페이지가 매겨져 있고, 리프 연산자를 컨테이너 및 `PM Load` 행과 혼합하고, CPU 타임라인에 모든 연산자 행의 복사본을 전달하므로 수동 조회에서는 일부 영역을 과소평가하고 다른 영역은 두 배로 계산하면서 완전히 그럴듯하게 보입니다.

**Divide every `% of NPU` by `npu_time_us`.** Call `get_subgraph_reasons` only if you need the per-subgraph list rather than the counts, and `get_npu_time_summary` only for `operators_overlap` or the end-to-end figure.
**모든 `NPU의 %`를 `npu_time_us`로 나눕니다.** 개수가 아닌 하위 그래프별 목록이 필요한 경우에만 `get_subgraph_reasons`를 호출하고 `operators_overlap` 또는 종단 간 수치에 대해서만 `get_npu_time_summary`를 호출하세요.

That value is the NPU time the **Performance Summary tab displays**, so the analysis and the UI agree about the same model.
해당 값은 **성능 요약 탭에 표시되는** NPU 시간이므로 분석과 UI는 동일한 모델에 대해 일치합니다.
Do not compute a denominator yourself: the timing table mixes leaf operators, program-memory loads, host phases, nested containers and a duplicate of every operator row on the CPU timeline, and a subset sum still looks plausible when it is wrong.
분모를 직접 계산하지 마십시오. 타이밍 테이블은 리프 연산자, 프로그램 메모리 로드, 호스트 단계, 중첩된 컨테이너 및 CPU 타임라인의 모든 연산자 행의 복제본을 혼합하며 하위 집합 합계는 틀렸을 때 여전히 타당해 보입니다.

The tool also returns `operator_sum_us` — the per-operator npu-category total — and `operators_overlap`.
이 도구는 또한 연산자별 npu 카테고리 합계인 `operator_sum_us`와 `operators_overlap`을 반환합니다.
When operators overlap, that sum exceeds `npu_time_us`, because overlapping work is counted once per operator but only once in the busy span.
연산자가 겹치는 경우 해당 합계는 'npu_time_us'를 초과합니다. 왜냐하면 겹치는 작업은 연산자당 한 번 계산되지만 사용 중인 범위에서는 한 번만 계산되기 때문입니다.
Region costs are built from per-operator times, so in that case the region percentages can add up to more than 100%.
지역 비용은 연산자별 시간을 기준으로 계산되므로 이 경우 지역 비율의 합이 100%를 넘을 수 있습니다.
Still divide by `npu_time_us`; if `operators_overlap` is true, say so in one clause in the context block so the reader is not surprised by the arithmetic.
여전히 `npu_time_us`로 나눕니다. `operators_overlap`이 true인 경우 컨텍스트 블록의 한 절에서 그렇게 말하여 독자가 산술에 놀라지 않도록 하세요.

`inference_time_us` is end-to-end including host time.
`inference_time_us`는 호스트 시간을 포함하여 엔드투엔드입니다.
It is context only — never the denominator.
이는 문맥일 뿐이지 분모는 아닙니다.

State the `npu_time_us` you used, in µs, in the context block.
컨텍스트 블록에 사용한 `npu_time_us`를 µs 단위로 명시하세요.

---

## 2.
## What is a candidate
## 후보자 란 무엇입니까?

### 2.1 Size
### 2.1 크기

At least two operators, **not** counting those whose reason is `PseudoOp`.
최소 두 명의 연산자('PseudoOp'인 연산자는 포함하지 **않음**).
When presenting a candidate, still show all its elements including the `PseudoOp` ones.
후보자를 제시할 때 'PseudoOp' 항목을 포함한 모든 요소를 ​​계속 표시합니다.

**A single operator is not a fusion candidate** — there is nothing to fuse — so exclude it when it already runs on the NPU, whatever it costs.
**단일 연산자는 융합 후보가 아닙니다** - 융합할 것이 없으므로 비용이 얼마든지 이미 NPU에서 실행 중인 경우 이를 제외하세요.
The one exception is §2.4: a lone operator that fell back to the **CPU** is a genuine opportunity, because moving it onto the NPU is the win.
한 가지 예외는 §2.4입니다. **CPU**로 대체된 단독 운영자는 NPU로 이동하는 것이 승리이기 때문에 진정한 기회입니다.
Excluded single NPU operators are not opportunities and do not appear in the table.
제외된 단일 NPU 사업자는 기회가 아니며 표에 표시되지 않습니다.

**This rule applies to a whole candidate, never to a part of one.** It asks "is this entire candidate one operator?" — not "does this candidate contain a one-operator branch?".
**이 규칙은 전체 후보에 적용되며 결코 하나의 일부에 적용되지 않습니다.** "이 전체 후보가 하나의 연산자입니까?"라고 묻습니다. — "이 후보에 단일 연산자 분기가 포함되어 있습니까?"가 아닙니다.
A split-and-rejoin or fan-in region (§3) is a multi-operator candidate no matter how short its individual branches are: three parallel MaxPools feeding a Concat is a four-operator region, not four single operators, and excluding it is wrong.
분할 및 재결합 또는 팬인 영역(§3)은 개별 분기가 아무리 짧더라도 다중 연산자 후보입니다. Concat에 공급하는 3개의 병렬 MaxPool은 4개의 단일 연산자가 아닌 4개의 연산자 영역이므로 이를 제외하는 것은 잘못된 것입니다.
Count the operators in the merged region and apply the rule to that number alone.
병합된 영역의 연산자 수를 계산하고 해당 숫자에만 규칙을 적용합니다.

### 2.2 Boundaries — break at any heavy compute operator
### 2.2 경계 — 무거운 계산 연산자에서 중단

Heavy compute means the ops doing the model's real arithmetic, as opposed to the data-movement operators a custom op absorbs.
대규모 컴퓨팅은 사용자 지정 작업이 흡수하는 데이터 이동 연산자와 달리 모델의 실제 산술을 수행하는 작업을 의미합니다.
Match case-insensitively and include at least `conv`, `gemm`, `matmul`, `lstm`, `gru`, `rnn`, `attention`, `einsum`.
대소문자를 구분하지 않고 일치시키고 최소한 `conv`, `gemm`, `matmul`, `lstm`, `gru`, `rnn`, `attention`, `einsum`을 포함합니다.
Elementwise and shape ops (`Mul`, `Add`, `Reshape`, `Slice`, `Concat`, `Transpose`, `Pad`, …) are **not** breakers.
요소별 및 모양 작업(`Mul`, `Add`, `Reshape`, `Slice`, `Concat`, `Transpose`, `Pad`, …)은 차단기가 **아닙니다**.

The list is **open**: if a model's arithmetic sits in a type not listed, that type is a breaker too.
목록은 **공개**되어 있습니다. 모델의 연산이 목록에 없는 유형에 있는 경우 해당 유형도 차단기입니다.
Sanity-check before starting — if breakers are under roughly a tenth of the operators, the set is too narrow for this model and the whole graph will come back as one candidate.
시작하기 전에 온전성 검사 - 차단기가 연산자의 약 10분의 1 미만인 경우 이 모델에 대한 집합이 너무 좁으므로 전체 그래프가 하나의 후보로 다시 나타납니다.
Widen it once, then proceed; do not loop.
한 번 넓힌 다음 계속 진행하십시오. 반복하지 마십시오.
Name the types you used in the context block.
컨텍스트 블록에서 사용한 유형의 이름을 지정하십시오.

Elsewhere in this skill "Conv/Gemm boundary" means a boundary at any heavy compute operator.
이 기술의 다른 부분에서 "Conv/Gemm 경계"는 무거운 계산 연산자의 경계를 의미합니다.

**Arithmetic that is not a boundary.** Some operators do real arithmetic but are not chain breakers, because fusing the region around them is still worthwhile — pooling is the common case, and an SPP block's parallel MaxPools feeding a Concat is the common case.
**경계가 아닌 산술.** 일부 연산자는 실제 산술을 수행하지만 체인 차단기가 아닙니다. 왜냐하면 주변 영역을 융합하는 것이 여전히 가치가 있기 때문입니다. 풀링이 일반적인 경우이고 SPP 블록의 병렬 MaxPools가 Concat에 공급하는 것이 일반적인 경우입니다.
Those regions are valid candidates, but the arithmetic inside them is **not** absorbable: a custom op replacing three MaxPools and a Concat still has to pool.
해당 지역은 유효한 후보이지만 그 내부의 산술은 흡수될 수 **없습니다**. 3개의 MaxPool을 대체하는 맞춤형 작업과 Concat은 여전히 ​​풀링을 해야 합니다.
Treat `pool`, `resize`, `interpolate`, `softmax`, `layernorm` and similar as arithmetic for §6's gain exclusion while leaving them as non-breakers here.
`pool`, `resize`, `interpolate`, `softmax`, `layernorm` 등을 §6의 이득 제외에 대한 산술과 유사하게 처리하고 여기서는 중단되지 않는 항목으로 남겨둡니다.
Two different questions — where a region ends, and what inside it can disappear — and they do not have the same answer.
영역이 끝나는 곳과 그 안에서 무엇이 사라질 수 있는지에 대한 두 가지 다른 질문에 대한 답은 동일하지 않습니다.

### 2.3 Other restrictions
### 2.3 기타 제한 사항

- Break a chain if it has multiple outputs (the candidate list already keeps these together).
- 여러 출력이 있는 경우 체인을 끊습니다(후보 목록은 이미 이러한 출력을 함께 유지합니다).
- The candidate's measured **cost** must be greater than 0.
- 후보자의 측정된 **비용**은 0보다 커야 합니다.
- A region whose cost could not be fetched has _unknown_ cost, not zero (§6.1).
- 비용을 가져올 수 없는 지역의 비용은 0이 아닌 _알 수 없음_입니다(§6.1).
- **The Expected gain cell is always a number.** `see note`, `n/a`, a dash or a sentence is not a gain, and a row a reader cannot rank is not a suggestion.
- **예상 이득 셀은 항상 숫자입니다.** `참고 참조`, `해당 없음`, 대시 또는 문장은 이득이 아니며 독자가 순위를 매길 수 없는 행은 제안이 아닙니다.
- If there is no number to write, the region belongs in the context block, not the table.
- 쓸 숫자가 없으면 해당 영역은 테이블이 아닌 컨텍스트 블록에 속합니다.
- **`reportable: false` from `get_region_cost` settles it.** The gain is zero, so the row offers a reader nothing to act on.
- **`get_region_cost`의 `reportable: false`로 해결됩니다.** 이득은 0이므로 행은 독자에게 조치를 취할 내용을 제공하지 않습니다.
- Do not restyle it into the table — a `0.00`, a dash, or a live gain cell with the caveat parked in another column are all the same row.
- 테이블에 스타일을 다시 지정하지 마세요. '0.00', 대시 또는 다른 열에 경고가 있는 라이브 게인 셀은 모두 동일한 행입니다.
- **A region whose gain is `0.00` is not a suggestion.** The table is a list of things worth building; a row offering nothing to gain is noise in it, however interesting the region.
- **게인이 '0.00'인 지역은 권장 사항이 아닙니다.** 표는 구축할 가치가 있는 항목의 목록입니다. 얻을 것이 없는 행은 그 안에 있는 소음이지만 지역이 아무리 흥미롭습니다.
- Keep the finding, move it out of the table: one line in the context block naming the region and why nothing is absorbable (§7.4).
- 결과를 유지하고 테이블 밖으로 이동합니다. 영역 이름을 지정하는 컨텍스트 블록의 한 줄과 흡수 가능한 항목이 없는 이유(§7.4).
- This is not the same as dropping it silently — "the SPP block looks fusible but its three MaxPools are the work" is worth a reader's time, just not a row.
- 이는 조용히 삭제하는 것과는 다릅니다. "SPP 블록은 가용성으로 보이지만 3개의 MaxPool이 작업입니다."는 독자의 시간을 들일 가치가 있지만 한 줄도 아닙니다.

### 2.4 CPU fallback
### 2.4 CPU 대체

If offload is below 100%, every operator running on CPU is an opportunity in its own right, single or not.
오프로드가 100% 미만인 경우 CPU에서 실행되는 모든 운영자는 단일 여부에 관계없이 그 자체로 기회입니다.
List these first, ranked by their cost, before the fusion candidates.
융합 후보보다 먼저 비용순으로 순위를 매기십시오.
They use the same table (§7.1); their "region" is the CPU operator or the run of adjacent CPU operators.
그들은 동일한 테이블(§7.1)을 사용합니다. 해당 "영역"은 CPU 운영자 또는 인접한 CPU 운영자의 실행입니다.

---

## 3.
## Reading the candidate list
## 후보자 목록 읽기

`get_fusion_candidates` has already done the parts that used to go wrong here: a region is a maximal connected run, so a split-and-rejoin, a fan-in on a Concat and a plain chain all arrive as **one** candidate with every leg included, and no two candidates share a node.
`get_fusion_candidates`는 여기서 잘못되었던 부분을 이미 수행했습니다. 영역은 최대 연결 실행이므로 분할 및 재결합, Concat의 팬인 및 일반 체인은 모두 모든 레그가 포함된 **하나** 후보로 도착하며 두 후보는 노드를 공유하지 않습니다.
Nothing needs merging, extending or de-duplicating.
병합, 확장 또는 중복 제거가 필요하지 않습니다.

What is left for you:
당신에게 남은 것은 무엇입니까?

### 3.1 A region can still be too large to be one custom op
### 3.1 지역은 여전히 하나의 사용자 정의 작업이 되기에는 너무 클 수 있습니다.

A candidate covering more than roughly a quarter of the model's operators is not a kernel — it says the model is poorly partitioned.
모델 운영자의 대략 4분의 1 이상을 포함하는 후보는 커널이 아닙니다. 즉, 모델이 제대로 분할되지 않았다고 말합니다.
Report that once in the context block and do not give it a row.
컨텍스트 블록에 한 번만 보고하고 행을 제공하지 마세요.
Check §5.1 first: N copies of a small repeating unit is a motif, not an oversized region.
§5.1을 먼저 확인하세요. 작은 반복 단위의 N개 복사본은 큰 영역이 아니라 모티프입니다.

### 3.2 Occurrences are already grouped
### 3.2 발생 항목이 이미 그룹화되어 있습니다.

Repeats of one region arrive as a single candidate with its `occurrences` count and its cost and gain summed across them — one kernel serves them all, including repeats that differ only in shape, which arrive with a `boundary_shapes` list.
한 지역의 반복은 '발생 횟수'와 그 비용 및 이득이 합산된 단일 후보로 도착합니다. 하나의 커널은 'boundary_shapes' 목록과 함께 도착하는 모양만 다른 반복을 포함하여 모든 지역에 서비스를 제공합니다.
Do not group or split them yourself: doing it by hand was the least repeatable part of the answer, with one model giving seven grouped rows in one run and six ungrouped rows plus a `+7 more` footnote in the next, from identical figures.
직접 그룹화하거나 분할하지 마십시오. 손으로 수행하는 것이 대답에서 가장 반복하기 어려운 부분이었습니다. 한 모델은 한 번에 7개의 그룹화된 행을 제공하고 그룹화되지 않은 6개의 행과 다음 번에는 동일한 수치에서 '+7 more' 각주를 제공했습니다.

### 3.3 The list is ranked by gain, not by interest
### 3.3 목록은 관심도가 아닌 이익을 기준으로 순위가 매겨집니다.

Take the candidates in the order given.
주어진 순서대로 후보자를 선택하세요.
If you reorder them, you are substituting a judgement for a measurement, and the reader is looking at row one.
순서를 바꾸면 측정값을 판단으로 대체하고 독자는 행 1을 보고 있습니다.

### 3.4 Name each region for someone who did not write the model
### 3.4 모델을 작성하지 않은 사람의 이름을 각 지역에 지정하세요.

The tool gives node names and operator types; it cannot tell you that a `Split → Slice → Neg → Concat → Mul → Add` run is a rotary embedding, or that a `Mul → Add` after every convolution is a learnable-affine block.
이 도구는 노드 이름과 연산자 유형을 제공합니다. 'Split → Slice → Neg → Concat → Mul → Add' 실행이 회전식 임베딩이거나 모든 컨볼루션 후의 'Mul → Add'가 학습 가능한 아핀 블록임을 알 수 없습니다.
That recognition is the most useful thing in the reply — an engineer who has never seen this model should be able to tell from the Region cell what code the kernel would replace.
이러한 인식은 응답에서 가장 유용한 것입니다. 이 모델을 본 적이 없는 엔지니어는 커널이 어떤 코드를 대체할지 영역 셀에서 알 수 있어야 합니다.
Use `get_onnx_model_topology` names, which carry the original module paths, and say what the block _is_, not just which operators it contains.
원래 모듈 경로를 전달하는 `get_onnx_model_topology` 이름을 사용하고 블록에 포함된 연산자뿐만 아니라 블록이 무엇인지를 말하세요.

---

## 4.
## ONNX span — ask the backend, do not compose it
## ONNX 범위 — 백엔드에 요청하고 구성하지 마세요.

For each merged region, call:
병합된 각 지역에 대해 다음을 호출하세요.

```
get_onnx_span(folder, region_nodes=[<the region's operator-DAG node names>])
```

Report its `start` and `end` **verbatim**.
'시작'과 '끝'을 **말 그대로** 보고하세요.
They are real node names, verified against this model.
이는 이 모델에 대해 검증된 실제 노드 이름입니다.

It does, in code, everything this step used to ask you to do by hand: maps `_Duplicated#N` and compiler-generated names back to ONNX, walks **every** path between the region's nodes so a branching region keeps all its legs, pulls in the glue that the compiled graph fused away, widens the boundary across adjacent Quantize/Dequantize pairs to a fixed point, and checks the span is a legal cut.
이 단계에서 수동으로 수행해야 하는 모든 작업을 코드에서 수행합니다. `_Duplicated#N` 및 컴파일러 생성 이름을 ONNX에 다시 매핑하고, 분기 영역이 모든 다리를 유지하도록 영역 노드 사이의 **모든** 경로를 이동하고, 컴파일된 그래프가 융합된 접착제를 끌어당기고, 인접한 Quantize/DeQuantize 쌍의 경계를 고정점으로 넓히고, 범위가 합법적인 절단인지 확인합니다.

Use what it returns:
반환되는 내용을 사용하세요.

| Field<br>필드                                                             | Use<br>사용                                                                 |
| ----------------------------------------------------------------- | ------------------------------------------------------------------- |
| `start`, `end`<br>'시작', '끝'                                                    | the Region cell's `start → end` line, verbatim<br>지역 셀의 '시작 → 끝' 줄, 그대로                      |
| `nodes`<br>'노드'                                                           | the region's ONNX node list — use it for the `original` graph link<br>지역의 ONNX 노드 목록 - `원본` 그래프 링크에 사용  |
| `node_count`<br>`노드_개수`                                                      | the **Nodes** column<br>**노드** 열                                                |
| `is_fusion_candidate`, `heavy_compute_inside`, `rejection_reason`<br>`is_fusion_candidate`, `heavy_compute_inside`, `rejection_reason` | **if false, this is not a row.** See below<br>**false인 경우 행이 아닙니다.** 아래를 참조하세요.                          |
| `cuttable`, `cut_error`<br>`컷 가능`, `컷_오류`                                           | if false, say so in the context block; the span is not usable as-is<br>거짓인 경우 컨텍스트 블록에 그렇게 명시합니다. 스팬은 그대로 사용할 수 없습니다. |
| `unresolved`<br>'해결되지 않음'                                                      | region names with no ONNX counterpart; if non-empty, say so<br>ONNX 대응물이 없는 지역 이름; 비어 있지 않으면 그렇게 말하세요         |
| `extended_over_quant`<br>`확장_오버_퀀트`                                             | true when the boundary moved to enclose a quantize pair<br>양자화 쌍을 둘러싸기 위해 경계가 이동한 경우 참             |

**`is_fusion_candidate: false` is a rejection, not a warning.** It means a heavy compute operator sits _inside_ the region rather than bounding it (§2.2), so the region is at least two regions that were joined through a Conv/Gemm/MatMul.
**`is_fusion_candidate: false`는 경고가 아니라 거부입니다.** 이는 무거운 계산 연산자가 영역을 제한하지 않고 영역 _내부_에 위치하므로(§2.2) 해당 영역은 Conv/Gemm/MatMul을 통해 결합된 최소 두 개의 영역임을 의미합니다.
Do not cost it, do not rank it, do not report it.
비용을 지불하지 말고, 순위를 매기지 말고, 보고하지 마십시오.
Split at each name in `heavy_compute_inside`, call the tool again on each piece, and report the pieces that come back clean.
`heavy_compute_inside`의 각 이름에서 분할하고, 각 조각에 대해 도구를 다시 호출하고, 깨끗하게 돌아온 조각을 보고합니다.
A region of a dozen Convs and Resizes is the model's arithmetic — reporting it as a fusion candidate claims a kernel can absorb the convolutions, and then the gain honestly computes to `0.00`, which reads as "no opportunity here" when the truth is "the region was wrong".
12개의 Convs 및 Resizes 영역은 모델의 산술입니다. 이를 융합 후보로 보고하면 커널이 컨볼루션을 흡수할 수 있다고 주장하고 이득은 정직하게 '0.00'으로 계산됩니다. 이는 진실이 "영역이 잘못되었습니다"일 때 "여기에는 기회가 없습니다"라고 읽습니다.

If every candidate in a model is rejected this way, the answer is that the model has no fusion opportunity — say that in one line.
모델의 모든 후보가 이런 방식으로 거부되면 대답은 모델에 융합 기회가 없다는 것입니다. 한 줄로 말하면 됩니다.
It is a real finding, and far better than a row that had to be zeroed to be true.
이는 실제 결과이며, 사실이 되기 위해 0으로 조정해야 했던 행보다 훨씬 낫습니다.

**Never write a span the tool did not give you.** Measured across 179 hand-composed spans, about half were unusable: abbreviated (`attn/MatMul`), a placeholder for a repeated block (`blocks.N`), a tensor name rather than a node (`..._output_0`), or absent.
**도구에서 제공하지 않은 범위는 절대 작성하지 마세요.** 직접 구성한 179개 범위에서 측정한 결과 약 절반은 사용할 수 없었습니다. 축약형(`attn/MatMul`), 반복 블록에 대한 자리 표시자(`blocks.N`), 노드가 아닌 텐서 이름(`..._output_0`) 또는 없음.
Those are all failure modes of composing the answer; none of them survive reporting what the tool returns.
이것들은 모두 답을 구성하는 실패 모드입니다. 그들 중 누구도 도구가 무엇을 반환하는지 보고하고 살아남지 못합니다.

If the call fails for a region, that region has no reportable span — say so in the context block rather than inventing one.
특정 지역에 대한 호출이 실패하면 해당 지역에는 보고 가능한 범위가 없습니다. 즉, 컨텍스트 블록을 만드는 대신 컨텍스트 블록에서 그렇게 하십시오.

## 5.
## Motifs — one row per pattern, not per occurrence
## 모티브 — 발생당이 아닌 패턴당 한 행

The same structural region usually appears many times.
동일한 구조적 영역은 일반적으로 여러 번 나타납니다.
One kernel serves every occurrence, so reporting each separately floods the table and buries the winner.
하나의 커널이 모든 발생을 처리하므로 각각을 개별적으로 보고하면 테이블이 넘쳐 승자를 묻게 됩니다.

### 5.1 Decompose a repeated region into its repeating unit
### 5.1 반복 영역을 반복 단위로 분해

Before anything else here: if a merged region is **N copies of the same short sub-pattern chained together**, it is not one big region.
다른 것보다 먼저: 병합된 영역이 **함께 연결된 동일한 짧은 하위 패턴의 N 복사본**인 경우 이는 하나의 큰 영역이 아닙니다.
It is that sub-pattern with `Occurrences = N`.
'발생 횟수 = N'인 하위 패턴입니다.
The compiler produces these routinely by unrolling — one source operator becomes a cascade of `_Duplicated#N` steps, each step identical in structure.
컴파일러는 언롤링을 통해 이를 일상적으로 생성합니다. 하나의 소스 연산자는 `_Duplicated#N` 단계의 계단식이 되며, 각 단계는 구조가 동일합니다.

Report the **unit**, not the cascade: `Nodes` is the unit's operator count, `Occurrences` is the number of steps, and cost and gain sum across them.
캐스케이드가 아닌 **유닛**을 보고합니다. '노드'는 유닛의 연산자 수이고, '발생'은 단계 수이며, 이들에 대한 비용 및 이득 합계입니다.
Getting this wrong cascades into three other problems — the over-merge guard (§3.1) fires on a region that is not really oversized, the graph links (§7.3) become an unusable list of every node in the cascade, and the binding check cannot be satisfied.
잘못된 캐스케이드가 발생하면 세 가지 다른 문제가 발생합니다. 즉, 초과 병합 가드(§3.1)가 실제로 크기가 크지 않은 영역에서 실행되고, 그래프 링크(§7.3)가 캐스케이드에 있는 모든 노드의 사용할 수 없는 목록이 되며, 바인딩 검사가 충족될 수 없습니다.
If you find yourself writing a node count in the dozens with `Occurrences 1`, and the region visibly repeats, decompose it first.
'Occurrences 1'을 사용하여 수십 개의 노드 수를 작성하고 영역이 눈에 띄게 반복되는 경우 먼저 분해하세요.

A region that genuinely is one large unique block stays as it is — this rule is about repetition, not size.
실제로 하나의 큰 고유 블록인 영역은 그대로 유지됩니다. 이 규칙은 크기가 아니라 반복에 관한 것입니다.

**Key** = (ONNX `op_type` sequence, boundary shapes, boundary dtypes).
**키** = (ONNX 'op_type' 시퀀스, 경계 모양, 경계 dtypes).
Get shapes and dtypes from `get_onnx_model_topology` with `include_shapes=true` (`output_shape` / `output_dtype`, null when unresolved).
`include_shapes=true`(`output_shape` / `output_dtype`, 해결되지 않은 경우 null)를 사용하여 `get_onnx_model_topology`에서 모양 및 dtype을 가져옵니다.
Use only **boundary** shapes — the input of the span's first node and the output of its last; intermediates are implied by the op sequence and splitting on them separates identical regions for no reason.
범위의 첫 번째 노드의 입력과 마지막 노드의 출력인 **경계** 모양만 사용하세요. 중간체는 op 시퀀스에 의해 암시되며 중간체에 대한 분할은 아무 이유 없이 동일한 영역을 분리합니다.

Shape and dtype are in the key deliberately: a kernel written for one activation shape or dtype does not serve another, so the same op sequence at two shapes is **two rows**, each with its own occurrences and gain.
Shape 및 dtype은 의도적으로 키에 있습니다. 하나의 활성화 모양 또는 dtype에 대해 작성된 커널은 다른 활성화 모양을 제공하지 않으므로 두 모양의 동일한 연산 시퀀스는 **두 행**이며 각각 자체 발생 및 이득이 있습니다.
Never merge across shapes or dtypes.
모양이나 dtype을 병합하지 마세요.

If shapes are unresolved, group on op-type sequence + dtype and note it in the context block.
모양이 해결되지 않은 경우 op-type 시퀀스 + dtype으로 그룹화하고 컨텍스트 블록에 기록해 둡니다.

Rank on **aggregate** gain across occurrences: the implementation cost is paid once, so a modest motif appearing 12 times can beat a single expensive one-off.
발생 전반에 걸쳐 **총체적** 이득 순위: 구현 비용은 한 번만 지불되므로 12번 나타나는 적당한 모티프가 값비싼 일회성 비용을 능가할 수 있습니다.

---

## 6.
## Expected gain — absorbing data movement
## 예상 이득 - 데이터 이동 흡수

### 6.1 Cost and gain — ask the backend, do not add it up
### 6.1 비용과 이득 - 백엔드에 물어보고 합산하지 마세요.

For each merged region, call:
병합된 각 지역에 대해 다음을 호출하세요.

```
get_region_cost(folder, region_nodes=[<the same names given to get_onnx_span>])
```

Report what it returns.
반환된 내용을 보고합니다.
Do not recompute any of it:
다음 중 어떤 것도 다시 계산하지 마세요.

| Field<br>필드                                 | Use<br>사용                                                                                          |
| ------------------------------------- | -------------------------------------------------------------------------------------------- |
| `total_us`<br>`total_us`                            | the **Current cost** cell, verbatim<br>**현재 비용** 셀, 축어적으로                                                          |
| `gain_us`<br>'gain_us'                             | the **Expected gain** for one occurrence, verbatim — absorbable time already held to the cap<br>한 번의 발생에 대한 **기대 이득**을 문자 그대로 — 흡수 가능한 시간이 이미 한도에 도달했습니다. |
| `cap_us`<br>`cap_us`                              | the ceiling that produced it (§6.3); quote it if you explain the gain<br>그것을 생산한 천장(§6.3); 이득을 설명한다면 인용해 보세요                        |
| `largest_node`, `largest_us`<br>`가장 큰_노드`, `가장 큰_우리`          | the operation the cap keeps; name it if you explain the cap<br>캡이 유지하는 작업; 모자에 대해 설명한다면 이름을 지어주세요                                  |
| `reportable`, `not_reportable_reason`<br>`신고 가능`, `신고 가능_이유 없음` | **false means this is not a table row** — §2.3, and check §6.4 first if the region repeats<br>**false는 테이블 행이 아님을 의미합니다** — §2.3, 영역이 반복되는 경우 먼저 §6.4를 확인하세요.   |
| `missing_siblings`<br>'missing_siblings'                    | **non-empty means the region is incomplete — see below**<br>**비어 있지 않으면 지역이 불완전함을 의미합니다. 아래를 참조하세요**                                     |
| `nodes_without_timing`<br>`타이밍 없는 노드`                | region nodes with **no timing row at all**<br>**타이밍 행이 전혀 없음**이 있는 지역 노드                                                   |
| `nodes`<br>'노드'                               | per-node `name`, `type`, `us` and `absorbable`, most expensive first<br>노드별 `name`, `type`, `us` 및 `absorbable`, 가장 비싼 것부터                         |
| `matched` / `requested`<br>`일치` / `요청`               | the coverage figure<br>적용 범위 수치                                                                          |

**`missing_siblings` is a stop, not a note.** The compiler unrolls one source operator into a run of `X_Duplicated#0..#N`, and they are one fusible unit.
**`missing_siblings`는 메모가 아닌 중지입니다.** 컴파일러는 하나의 소스 연산자를 `X_Duplicated#0..#N` 실행으로 풀며 이는 하나의 가용성 단위입니다.
If the list is non-empty you named some and dropped others, so every figure for this region is understated.
목록이 비어 있지 않은 경우 일부 이름을 지정하고 일부는 삭제했으므로 이 지역의 모든 수치는 과소평가됩니다.
Add them to `region_nodes`, call `get_onnx_span` and `get_region_cost` again, and report the second answer.
이를 `region_nodes`에 추가하고 `get_onnx_span` 및 `get_region_cost`를 다시 호출한 후 두 번째 답변을 보고합니다.
On petrv2-bevseg the same tail region came out at 434 µs, 649 µs and 1079 µs on three runs; only the run that named all five siblings was right.
petrv2-bevseg에서 동일한 꼬리 영역은 3회 실행에서 434μs, 649μs 및 1079μs로 나타났습니다. 다섯 형제 모두의 이름을 지은 달리기만이 옳았습니다.

**A node in `nodes_without_timing` costs nothing here.** It was fused into a neighbour and its time already sits in that neighbour's row, so `total_us` is right without it.
**`nodes_without_timing`에 있는 노드는 여기서 비용이 전혀 들지 않습니다.** 이웃에 융합되었고 그 시간은 이미 해당 이웃의 행에 있으므로 `total_us`는 노드가 없어도 옳습니다.
Never give it a nearby node's value to fill the gap — that inflated one region by 20% and the wrong figure was indistinguishable from a measured one.
간격을 메우기 위해 근처 노드의 값을 제공하지 마십시오. 그러면 한 영역이 20% 증가하고 잘못된 수치는 측정된 수치와 구별할 수 없게 됩니다.
If the list is non-empty, say so in the context block as `N of M nodes have no timing row (fused)`.
목록이 비어 있지 않은 경우 컨텍스트 블록에서 'M개 노드 중 N개에 타이밍 행이 없습니다(융합)'라고 명시하세요.

If the call fails for a region, that region's cost is unknown: its gain cell is `unmeasured`, which sorts below every measured gain.
지역에 대한 호출이 실패하면 해당 지역의 비용은 알 수 없습니다. 해당 이득 셀은 측정되지 않은 모든 이득 아래로 정렬되는 '측정되지 않음'입니다.
Never print `0.00` for a cost you could not fetch.
가져올 수 없는 비용에 대해 '0.00'을 인쇄하지 마세요.

### 6.2 Why `gain_us` is the gain
### 6.2 왜 'gain_us'가 이득인가?

You do not compute this — §6.1 returns it — but the row has to be defensible, so:
이를 계산하지 않고(§6.1에서 반환) 행은 방어 가능해야 하므로 다음과 같습니다.

- A fusion absorbs **data movement**.
- 융합은 **데이터 이동**을 흡수합니다.
- It does not absorb arithmetic: the kernel still has to convolve, pool, resize and normalise.
- 산술 연산을 흡수하지 않습니다. 커널은 여전히 ​​컨볼루션, 풀링, 크기 조정 및 정규화를 수행해야 합니다.
- `get_region_cost` marks each node `absorbable` from its measured operator type, sums those, and holds the result to `cap_us`.
- 'get_region_cost'는 측정된 연산자 유형에서 각 노드를 '흡수 가능'으로 표시하고 이를 합산한 후 결과를 'cap_us'에 보관합니다.
- The classification is done there rather than from `get_onnx_span`'s `data_movement_ops` because the two lists are in different name spaces.
- 두 목록이 서로 다른 이름 공간에 있기 때문에 `get_onnx_span`의 `data_movement_ops`가 아닌 여기에서 분류가 수행됩니다.
- Costs are keyed by operator-DAG names (`Concat_3399_Duplicated#4`, type `BufferUnpadInnermostAdf`, 360 µs); the ONNX span lists `Concat_3399`.
- 비용은 운영자 DAG 이름(`Concat_3399_Duplicated#4`, 유형 `BufferUnpadInnermostAdf`, 360 µs)에 따라 결정됩니다. ONNX 범위에는 `Concat_3399`가 나열됩니다.
- Intersecting them sums whichever names happen to appear in both, which is not a quantity with a meaning.
- 그것들을 교차시키는 것은 두 가지 모두에 나타나는 이름의 합계이며, 이는 의미가 있는 양이 아닙니다.
- Use `get_onnx_span`'s classification to _understand_ a region and `gain_us` to _report_ it.
- 지역을 _이해_하려면 `get_onnx_span` 분류를 사용하고 이를 _보고_하려면 `gain_us`를 사용하세요.
- A motif's gain is `gain_us` times its occurrences.
- 모티프의 이득은 'gain_us'에 해당 발생 횟수를 곱한 것입니다.

> **Arithmetic is never gain, whatever its subgraph reason says.** A subgraph reason describes how something was compiled, not whether a fused kernel absorbs it — on a recurrent model the LSTM cells are themselves `TemplatedGraph`, and counting them claims the network computes itself for free.
> **하위 그래프 이유가 무엇이든 산술은 결코 이득이 아닙니다.** 하위 그래프 이유는 융합 커널이 그것을 흡수하는지 여부가 아니라 무언가가 어떻게 컴파일되었는지 설명합니다. 순환 모델에서 LSTM 셀은 그 자체로 'TemplateGraph'이며, 이를 계산하면 네트워크가 무료로 자체적으로 계산한다고 주장합니다.
> The `absorbable` flag is the authority, and the arithmetic it excludes covers both region boundaries (conv, gemm, matmul, lstm, …) and arithmetic that sits _inside_ a region without ending it (pooling, resize, softmax, normalisation).
> 'absorbable' 플래그는 권위이며, 여기서 제외되는 산술은 지역 경계(conv, gemm, matmul, lstm, …)와 지역을 종료하지 않고 지역 _내부_에 위치하는 산술(풀링, 크기 조정, 소프트맥스, 정규화)을 모두 포함합니다.
> An SPP block of three MaxPools feeding a Concat is a legitimate candidate whose pooling is not absorbable.
> Concat에 공급되는 3개의 MaxPool로 구성된 SPP 블록은 풀링이 흡수되지 않는 합법적인 후보입니다.

### 6.3 Why there is a cap
### 6.3 한도가 있는 이유

`cap_us` is `total_us` minus the single most expensive node in the region.
`cap_us`는 `total_us`에서 해당 지역에서 가장 비싼 단일 노드를 뺀 값입니다.
Whatever that operation does — arithmetic or moving data — the fused kernel still has to do it, so the best a fusion can achieve is eliminating everything _except_ that one operation.
산술이든 데이터 이동이든 그 작업이 무엇이든 융합 커널은 여전히 ​​이를 수행해야 하므로 융합이 달성할 수 있는 최선의 방법은 해당 작업을 _제외_한 모든 것을 제거하는 것입니다.

It binds hardest exactly where this analysis looks hardest.
이 분석이 가장 어려워 보이는 바로 그 부분에 가장 강하게 결합됩니다.
A region of pure data movement has every node absorbable, so the raw sum equals the region cost _by construction_, and without the cap the table would claim the region becomes free.
순수 데이터 이동 영역에는 흡수 가능한 모든 노드가 있으므로 원시 합계는 _구성별_ 지역 비용과 동일하며, 상한선이 없으면 테이블에서는 해당 지역이 무료가 된다고 주장합니다.
`gain_us` already has the cap applied; a gain equal to region cost means it came from somewhere else.
`gain_us`에는 이미 한도가 적용되어 있습니다. 지역 비용과 동일한 이득은 그것이 다른 곳에서 왔다는 것을 의미합니다.

**The figures must agree.** `total_us`, `cap_us` and `gain_us` come from one call.
**수치는 일치해야 합니다.** `total_us`, `cap_us` 및 `gain_us`는 하나의 호출에서 나옵니다.
If a sentence anywhere in the reply quotes a cost, cap or gain that differs from the cell beside it, it was derived rather than reported — an earlier reply managed three mutually inconsistent caps for one region this way.
답변의 어느 문장에서 옆에 있는 셀과 다른 비용, 한도 또는 이득을 인용하는 경우 이는 보고된 것이 아니라 파생된 것입니다. 이전 답변에서는 한 지역에 대해 서로 일치하지 않는 세 가지 한도를 이런 방식으로 관리했습니다.
Quote each once, from the tool.
도구에서 각각 한 번씩 인용해 보세요.

### 6.4 Regions priced against a kernel the model already has
### 6.4 모델이 이미 보유하고 있는 커널에 대해 가격이 책정되는 지역

`get_fusion_candidates` does this for you: a candidate carrying `gain_source` was priced against a fused kernel this model already emits elsewhere, and its `occurrences` covers every place the motif appears.
`get_fusion_candidates`가 이 작업을 수행합니다. `gain_source`를 포함하는 후보는 이 모델이 이미 다른 곳에서 방출하는 융합 커널에 비해 가격이 책정되었으며 해당 `발생`은 모티프가 나타나는 모든 위치를 포괄합니다.
Report those figures and say in the context block that the target came from an existing kernel, naming it.
해당 수치를 보고하고 컨텍스트 블록에서 대상이 기존 커널에서 왔다고 이름을 지정하여 말하십시오.
The rest of this section is why that number is trustworthy.
이 섹션의 나머지 부분에서는 해당 번호를 신뢰할 수 있는 이유를 설명합니다.

#### Why such a region scores zero without it
#### 그러한 지역이 없으면 왜 0점을 얻나요?

`cap_us` is `total_us` minus the largest node, so a region with exactly **one** timed node has a cap of zero and scores `0.00` however expensive it is.
`cap_us`는 `total_us`에서 가장 큰 노드를 뺀 값이므로 정확히 **1** 시간 제한 노드가 있는 지역의 한도는 0이고 점수는 '0.00'입니다.
That is not always a real absence of opportunity.
그렇다고 해서 항상 기회가 없는 것은 아닙니다.
It is what a region looks like when the compiler folded the rest of it into a neighbour: the surviving node carries the cost, and the region cannot be scored from the inside.
이는 컴파일러가 나머지 부분을 이웃 노드로 접었을 때 영역의 모습입니다. 살아남은 노드가 비용을 부담하며 영역은 내부에서 점수를 매길 수 없습니다.

Before writing such a region off, ask whether this model already fuses the sequence somewhere else:
그러한 영역을 쓰기 전에 이 모델이 이미 다른 곳에서 시퀀스를 융합하고 있는지 물어보십시오.

```
get_fusion_precedent(folder, op_types=[<the span's ONNX op types, in order>])
```

If `found` is true, the model contains a kernel that already does this fusion, and `fused_avg_us` is what one occurrence costs when it happens — measured, on this hardware, in this model.
'found'가 true인 경우 모델에는 이미 이러한 융합을 수행하는 커널이 포함되어 있으며 'fused_avg_us'는 이 융합이 발생할 때 발생하는 비용입니다(이 하드웨어, 이 모델에서 측정됨).
The gain is then `total_us - fused_avg_us` per occurrence, floored at zero, and it replaces the cap-derived `0.00`.
그런 다음 이득은 발생당 `total_us - fused_avg_us`이며 0으로 바닥이 표시되며 상한선 파생 `0.00`을 대체합니다.
Say in the context block that the target came from an existing kernel and name it.
컨텍스트 블록에서 대상이 기존 커널에서 왔다고 말하고 이름을 지정합니다.

For the motif as a whole the tool has already done the arithmetic: report `motif_cost_us` as Current cost, `motif_gain_us` as Expected gain and `motif_occurrences` as Occurrences, all verbatim.
전체적으로 모티프에 대해 도구는 이미 산술을 수행했습니다. 'motif_cost_us'를 현재 비용으로, 'motif_gain_us'를 예상 이득으로, 'motif_occurrences'를 발생으로 보고합니다.
Do not subtract one kernel's total from another's — they cover different numbers of occurrences, and doing it that way overstated dfine_m's affine motif by 15%.
한 커널의 총계를 다른 커널의 총계에서 빼지 마십시오. 서로 다른 발생 횟수를 포괄하므로 그렇게 하면 dfine_m의 아핀 모티프가 15% 과장됩니다.

This is worth the call whenever a `0.00` region repeats.
이는 `0.00` 영역이 반복될 때마다 호출할 가치가 있습니다.
D-FINE's learnable-affine `Mul → Add` is the case that motivated it: 25 occurrences, 3325 µs, 10% of NPU time, every one scored `0.00` because the Mul had been folded into the preceding convolution — while the same model emitted the fused `Mul`+`Add` kernel 15 times elsewhere at a quarter of the cost.
D-FINE의 학습 가능 아핀 'Mul → Add'는 동기를 부여한 사례입니다. 25회 발생, 3325 µs, NPU 시간의 10%, Mul이 이전 컨볼루션으로 접혀졌기 때문에 모두 '0.00' 점수를 얻었습니다. 반면 동일한 모델은 비용의 4분의 1로 다른 곳에서 융합된 'Mul'+'Add' 커널을 15번 방출했습니다.

If `found` is false there is no measured target, and `0.00` stands.
'found'가 거짓이면 측정된 목표가 없으며 '0.00'이 됩니다.

**Two different things can make a gain cell look empty.
**두 가지 다른 요인으로 인해 게인 셀이 비어 있는 것처럼 보일 수 있습니다.
Use the right one — they are not interchangeable.**
올바른 것을 사용하세요. 서로 바꿔 사용할 수 없습니다.**

| Cell<br>셀         | Means<br>수단                                                                                                      | When<br>언제                                                                                                                                                      |
| ------------ | ---------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `unmeasured`<br>'측정되지 않음' | Cost unknown: `get_region_cost` failed for this region<br>비용 알 수 없음: 이 지역에 대한 'get_region_cost' 실패                                                     | §6.1 only. Sorts below every measured gain.<br>§6.1에만 해당. 측정된 모든 게인 아래에 정렬합니다.                                                                                                               |
| `0.00`       | Measured, and nothing in the span is absorbable — every layer in it is heavy compute, or the span is empty<br>측정되었으며 범위의 어떤 것도 흡수될 수 없습니다. 모든 레이어가 계산량이 많거나 범위가 비어 있습니다. | Never a table row (§2.3): report it in the context block. If a _data-movement_ region scores `0.00`, suspect the span or check §6.4 before writing it off<br>테이블 행이 아님(§2.3): 컨텍스트 블록에서 보고합니다. _data-movement_ 영역의 점수가 '0.00'인 경우 범위를 의심하거나 기록하기 전에 §6.4를 확인하세요. |

Do not conflate the two.
두 가지를 혼동하지 마십시오.
`unmeasured` hides a fetch failure; `0.00` is a finding.
'unmeasured'는 가져오기 실패를 숨깁니다. '0.00'은 결과입니다.

- **Count each node once.** Gains are never added across rows that share nodes; if two rows share one, the merge was not applied.
- **각 노드를 한 번씩 계산합니다.** 노드를 공유하는 행 전체에는 이득이 추가되지 않습니다. 두 행이 하나를 공유하는 경우 병합이 적용되지 않았습니다.

**What this number covers** (context block, one line, not a paragraph): the data movement a fused kernel absorbs.
**이 숫자가 다루는 내용**(컨텍스트 블록, 단락이 아닌 한 줄): 융합 커널이 흡수하는 데이터 이동입니다.
It does **not** include per-subgraph launch overhead or program-memory reload, both of which a fusion also removes, so the figure is conservative.
하위 그래프별 실행 오버헤드나 프로그램 메모리 재로드는 포함되지 **않으며**, 두 가지 모두 융합으로 제거되므로 수치는 보수적입니다.
Report `reason_counts` and the absorbable share of NPU time — `PseudoOp` + `TemplatedGraph` excluding heavy compute — and let the reader conclude.
'reason_counts'와 NPU 시간의 흡수 가능한 비율(과중한 컴퓨팅을 제외한 'PseudoOp' + 'TemplateGraph')을 보고하고 독자가 결론을 내리도록 하세요.

---

## 7.
## Output
## 출력

### 7.1 Table
### 7.1 표

Write the header row exactly as the first column below — `Occurrences`, not `Occ.`; `Expected gain`, not `Gain`.
헤더 행을 아래 첫 번째 열과 정확히 동일하게 작성하세요. 'Occ'가 아니라 'Occurrences'입니다. '이익'이 아닌 '기대 이익'.
A reader meeting the table for the first time should not have to ask what a column is, and there is nowhere in a static reply to expand an abbreviation.
처음으로 테이블을 만나는 독자는 칼럼이 무엇인지 물어볼 필요가 없으며 정적 응답에는 약어를 확장하는 곳이 없습니다.

| Column<br>칼럼            | Contents<br>목차                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Region<br>지역            | Short name and operator types, then the ONNX span `start → end` on a second line in the same cell. **The two lines are not interchangeable.** A label such as `Concat → Add → Sigmoid → ScatterND` names operator _types_; the span names the two ONNX _nodes_ the cut runs between (`Concat_5315 → ScatterND_5991`). A row that carries only the type chain cannot be cut from, and one shipped that way. Every row has both.<br>짧은 이름과 연산자 유형, ONNX 범위는 동일한 셀의 두 번째 줄에 `start → end`로 표시됩니다. **두 줄은 서로 바꿔서 사용할 수 없습니다.** `Concat → Add → Sigmoid → ScatterND`와 같은 레이블은 연산자 _types_를 지정합니다. 스팬은 컷이 사이에서 실행되는 두 ONNX _nodes_의 이름을 지정합니다(`Concat_5315 → ScatterND_5991`). 유형 체인만 전달하는 행은 잘라낼 수 없으며 해당 행은 그런 식으로 배송됩니다. 모든 행에는 둘 다 있습니다. |
| Shape / dtype<br>모양 / dtype     | The candidate's `boundary_shape` and `boundary_dtype`, verbatim. When a motif spans several shapes the candidate carries `boundary_shapes` — list them, since one kernel source is instantiated at each. A dtype on its own (`int8`, `bf16`) is not an answer — a reader sizing a kernel needs the shape, and it is in the candidate<br>후보자의 'boundary_shape' 및 'boundary_dtype'을 축어적으로 표현합니다. 모티프가 여러 모양에 걸쳐 있는 경우 후보는 'boundary_shapes'를 전달합니다. 각각 하나의 커널 소스가 인스턴스화되므로 이를 나열합니다. dtype 자체(`int8`, `bf16`)는 답이 아닙니다. 커널 크기를 조정하는 독자에게는 모양이 필요하며 이는 후보에 있습니다.                                                                                           |
| Nodes<br>노드             | The candidate's `node_count` — its operators in the compiled graph, including pseudo-ops. **Not** the ONNX span's node count, which counts different things and differs: one reply printed 31 here and "9 of 27 region nodes" three lines later<br>후보의 `node_count` - 의사 작업을 포함하여 컴파일된 그래프의 연산자입니다. **아닙니다** ONNX 스팬의 노드 수는 서로 다른 항목을 세고 다릅니다. 한 응답은 여기에 31을 인쇄하고 "9/27 지역 노드"는 세 줄 뒤에 인쇄됩니다.                                                                                                                                                                                |
| Occurrences<br>발생       | Instances of this motif<br>이 모티브의 사례                                                                                                                                                                                                                                                                                                                                                                                                        |
| Current cost<br>현재 비용      | Measured cost of the region, µs<br>지역의 측정된 비용, µs                                                                                                                                                                                                                                                                                                                                                                                                |
| **Expected gain**<br>**기대수익** | **µs, and % of NPU time**<br>**μs 및 NPU 시간의 %**                                                                                                                                                                                                                                                                                                                                                                                                      |
| Graphs<br>그래프            | `original · optimized` — two links, one cell (§7.3)<br>`원본 · 최적화` — 링크 2개, 셀 1개(§7.3)                                                                                                                                                                                                                                                                                                                                                                            |

### 7.2 Table rules
### 7.2 테이블 규칙

- One row per motif of merged regions.
- 병합된 영역의 모티프당 한 행입니다.
- Never per occurrence, never per pre-merge fragment.
- 발생 시에는 절대 안 되고, 병합 전 조각별로도 안 됩니다.
- Sort by Expected gain descending, ties by Current cost.
- 예상 이익 내림차순으로 정렬하고 현재 비용으로 동점을 지정합니다.
- Show the **top 5–10**; if more qualify, add `(+N more, together X µs)` as a line of text **below the table**.
- **상위 5~10**을 표시합니다. 더 많은 한정이 있는 경우 `(+N more, together X µs)`를 **표 아래** 텍스트 줄로 추가하세요.
- It is not a row: written as one it becomes a single cell in a seven-column table and renders as a broken row.
- 행이 아닙니다. 하나로 작성하면 7열 테이블의 단일 셀이 되고 깨진 행으로 렌더링됩니다.
- **Bold the highest-gain row.**
- **가장 높은 이득 행을 굵게 표시합니다.**
- Percentages are **% of NPU time only**.
- 백분율은 **NPU 시간의 %**입니다.
- Do not add an end-to-end column or an e2e figure.
- 엔드투엔드 열이나 e2e 수치를 추가하지 마세요.
- No prompt / "create a customOp with…" column — the Region cell's `start → end` already carries the boundary nodes.
- 프롬프트 없음/"…을 사용하여 customOp 생성" 열 - 지역 셀의 `시작 → 끝`에 이미 경계 노드가 포함되어 있습니다.
- No separate ONNX span column.
- 별도의 ONNX 범위 열이 없습니다.
- Exclude anything failing §2, any subchain of a longer chain, and any pre-merge fragment.
- §2에 실패한 모든 것, 더 긴 체인의 하위 체인 및 사전 병합 조각을 제외합니다.

### 7.3 Graph links
### 7.3 그래프 링크

`get_onnx_span` returns them built.
`get_onnx_span`은 빌드된 항목을 반환합니다.
Paste `links.markdown` into the Graphs cell, verbatim:
`links.markdown`을 그래프 셀에 그대로 붙여넣습니다.

```
[original](…&graphType=original) · [optimized](…&graphType=optimized)
```

Two links, one cell — **the Graphs cell, and nowhere else**.
링크 2개, 셀 1개 — **그래프 셀만 있고 다른 곳은 없습니다**.
The Region cell is plain text — **both of its lines**.
지역 셀은 **두 줄 모두** 일반 텍스트입니다.
Naming the block as a link there (`[Mul, Add + Reshape glue](aianalyzer://op?names=…)`) puts a second, differently-scoped link in the row and the reader cannot tell which one the Graphs column duplicates; writing the span as two links (`[/…/lab/Mul](aianalyzer://op?name=…) → [/…/lab/Add](…)`) is the same mistake, and it makes the span unreadable as text.
블록의 이름을 링크로 지정하면(`[Mul, Add + Reshape Glue](aianalyzer://op?names=…)`) 행에 다른 범위의 두 번째 링크가 추가되고 독자는 그래프 열이 어느 링크인지 알 수 없습니다. 범위를 두 개의 링크(`[/…/lab/Mul](aianalyzer://op?name=…) → [/…/lab/Add](…)`)로 작성하는 것은 동일한 실수이며 범위를 텍스트로 읽을 수 없게 만듭니다.
Write it as `` `/…/lab/Mul → /…/lab/Add` ``.
`` `/…/lab/Mul → /…/lab/Add```로 작성합니다.
Both shipped, on the same model, in separate sweeps.
둘 다 동일한 모델로 별도의 스윕으로 배송되었습니다.
Do not compose a link yourself, do not edit the ones you are given, and do not add a third: `original` shows what you would cut to build the kernel and `optimized` shows the region as the compiler sees it, which is what the decision needs.
링크를 직접 작성하지 말고, 제공된 링크를 편집하지 말고, 세 번째 링크를 추가하지 마십시오. `original`은 커널을 빌드하기 위해 잘라낼 내용을 표시하고, `optimized`는 컴파일러가 보는 대로 영역을 표시합니다. 이는 결정에 필요한 것입니다.
A `mapped` link answers a placement question at the cost of a third name space to get wrong.
'매핑된' 링크는 세 번째 이름 공간이 잘못되는 대가를 치르면서 배치 질문에 답합니다.

They are built rather than described because every part of them was got wrong while it was described.
설명하는 동안 모든 부분이 잘못되었기 때문에 설명이 아닌 작성되었습니다.
Across 21 replies: `name=` — which highlights one node of a multi-node region and renders identically to a correct link — appeared in 8; one reply used the same URL for both graphs, so "optimized" showed the reader exactly what "original" did; `#` was left unencoded, which truncates the URL; and node lists were cut short with no sign of it.
21개의 답변에 걸쳐: 다중 노드 영역의 한 노드를 강조 표시하고 올바른 링크와 동일하게 렌더링하는 `name=`이 8개에 나타났습니다. 한 답변은 두 그래프에 대해 동일한 URL을 사용했기 때문에 "최적화"는 독자에게 "원본"이 수행한 작업을 정확하게 보여주었습니다. '#'은 인코딩되지 않은 상태로 남아 있어 URL이 잘립니다. 그리고 노드 목록은 아무런 표시도 없이 짧게 잘렸습니다.
None of that is visible in the rendered table.
렌더링된 테이블에는 그 어떤 것도 표시되지 않습니다.
The link is blue either way.
링크는 어느 쪽이든 파란색입니다.

The two graphs do **not** share a name space — `original` takes ONNX names, `optimized` takes operator-DAG names with their `_Duplicated#N` suffixes — which is the trap the built links exist to remove.
두 그래프는 이름 공간을 공유하지 **않습니다**. `original`은 ONNX 이름을 사용하고, `optimized`는 `_Duplicated#N` 접미사가 포함된 Operator-DAG 이름을 사용합니다. 이는 제거하기 위해 빌드된 링크가 존재하는 트랩입니다.

If `links.truncated` is non-zero the region was too large to link whole; label the cell `optimized (30 of 87)` so the reader can tell a partial highlight from a small region.
`links.truncated`가 0이 아니면 영역이 너무 커서 전체를 연결할 수 없습니다. 독자가 작은 영역의 부분적인 강조 표시를 알 수 있도록 셀에 '최적화(30/87)'라는 라벨을 붙입니다.

### 7.4 Context block
### 7.4 컨텍스트 블록

A few short lines under the table, one fact each, no paragraphs.
표 아래에 몇 줄의 짧은 줄이 있고 각각 하나의 사실이 있으며 단락은 없습니다.

**The first line is fixed.** Write it exactly in this shape, so a reader can check any percentage in the table against it at a glance:
**첫 번째 줄은 고정되어 있습니다.** 독자가 표의 백분율을 한눈에 확인할 수 있도록 다음과 같은 모양으로 정확하게 작성하십시오.

```
NPU time: <npu_time_us> µs/inference (Performance Summary)
```

Append `; operator sum <operator_sum_us> µs — operators overlap, so region percentages can exceed 100%` when `operators_overlap` is true, and nothing extra when it is false.
`; 추가 연산자 합계 <operator_sum_us> µs — 연산자가 겹치므로 'operators_overlap'이 true인 경우 영역 비율이 100%를 초과할 수 있고, false인 경우에는 추가 항목이 없습니다.
Use the figures from `get_npu_time_summary` verbatim; do not round them differently or restate them elsewhere in the reply.
`get_npu_time_summary`의 수치를 그대로 사용하세요. 다르게 반올림하거나 응답의 다른 곳에 다시 언급하지 마십시오.

**Link a single operator with `name=` and no `graphType`.** The dominating-operator line and any operator named in prose may link to that one node, but adding `graphType` to such a link tells the viewer to open that graph immediately — and the reply then opens the optimized graph before the reader has looked at the table.
**`graphType` 없이 `name=`으로 단일 연산자를 연결합니다.** 지배 연산자 행과 산문으로 명명된 모든 연산자는 해당 노드에 연결할 수 있지만 이러한 링크에 `graphType`을 추가하면 보는 사람에게 해당 그래프를 즉시 열도록 지시합니다. 그러면 응답은 독자가 테이블을 보기 전에 최적화된 그래프를 엽니다.
The Graphs cell is where a reader chooses which graph to open.
그래프 셀은 독자가 열려는 그래프를 선택하는 곳입니다.

**A dominating operator goes after this block, on its own.** The gather call returns `headline`: when it is a non-empty string, print that sentence **verbatim** immediately after the context block and before the topology.
**주요 연산자는 자체적으로 이 블록 뒤에 옵니다.** 수집 호출은 `헤드라인`을 반환합니다. 비어 있지 않은 문자열인 경우 컨텍스트 블록 바로 뒤와 토폴로지 앞에 **그대로** 해당 문장을 인쇄합니다.
Do not rewrite it, shorten it, merge it into a context bullet, or fold it into a table row.
다시 작성하거나, 줄이거나, 컨텍스트 글머리 기호로 병합하거나, 테이블 행으로 접지 마십시오.
It is empty when no operator dominates, and then the context block is followed straight by the topology.
연산자가 지배하지 않는 경우 비어 있으며, 컨텍스트 블록 바로 뒤에 토폴로지가 이어집니다.
You may add a link on the operator it names (`name=` only, no `graphType` — see above).
이름이 지정된 연산자에 링크를 추가할 수 있습니다(`name=`만 가능, `graphType` 없음 - 위 참조).

**When the best row is under about 1% of NPU time, say so in the first context line.** A table led by a 0.07 µs-per-cent row answers "where would a custom op help?" with "nowhere, technically here", and the reader deserves that in words rather than having to read the percentage and infer it.
**가장 좋은 행이 NPU 시간의 약 1% 미만인 경우 첫 번째 컨텍스트 줄에서 그렇게 명시하십시오.** 0.07 µs-per-cent 행으로 이어지는 테이블은 "사용자 지정 작업이 어디에 도움이 될까요?"라고 대답합니다. "아무데도, 기술적으로는 여기"라고 말해야 하며, 독자는 백분율을 읽고 추론할 필요 없이 말로 설명할 가치가 있습니다.

Then the remaining lines.
그런 다음 나머지 줄.
Always: the subgraph reason counts; the absorbable share of NPU time.
항상: 하위 그래프 이유가 중요합니다. NPU 시간의 흡수 가능한 점유율.
Add only when applicable: nodes with no timing row (§6.1); heavy-compute types used (§2.2); a region whose cost could not be fetched (§6.1); shapes unresolved (§5); a region capped or mis-scoped (§6.3); a region too large to be a custom op (§3.1).
적용 가능한 경우에만 추가: 타이밍 행이 없는 노드(§6.1); 사용된 대용량 컴퓨팅 유형(§2.2) 비용을 가져올 수 없는 지역(§6.1) 해결되지 않은 모양(§5); 제한되거나 범위가 잘못 지정된 지역(§6.3) 사용자 정의 작업이 되기에는 영역이 너무 큽니다(§3.1).
Past about six lines you are explaining — cut back.
설명하고 있는 6줄 정도를 지나서 — 줄이세요.

### 7.5 Topology
### 7.5 토폴로지

A drawing of the full optimized graph with the selected regions marked, **below** the table.
표 **아래**에 선택된 영역이 표시된 전체 최적화 그래프 그림.
It shows where each opportunity sits and is checkable evidence you walked the whole graph, so cover it end to end.
이는 각 기회가 어디에 있는지 보여주고 전체 그래프를 살펴보았으므로 확인할 수 있는 증거이므로 끝까지 다루십시오.

Keep it a map, not a listing: show structure and mark regions; collapse repetition (`×N`, not N copies); no per-node costs, subtypes or reasons — those are the table's job.
목록이 아닌 지도로 유지: 구조를 표시하고 영역을 표시합니다. 축소 반복(`×N`, N 복사본이 아님); 노드당 비용, 하위 유형 또는 이유가 없습니다. 이것이 테이블의 작업입니다.
If it does not fit roughly a screen, collapse more repetition rather than truncating the graph.
화면에 대충 맞지 않으면 그래프를 자르는 것보다 반복을 더 많이 축소하세요.

---

## 8.
## Checklist before replying
## 답장 전 체크리스트

- [ ] Reply is table → context block → topology, with no narration anywhere, nothing after the topology, and no
- [ ] 응답은 테이블 → 컨텍스트 블록 → 토폴로지이며 어디에도 설명이 없고 토폴로지 뒤에도 없고 없음
abandoned or restarted table (§0).
중단되거나 다시 시작된 테이블(§0).
- [ ] Every region's figures came from `get_region_cost`, quoted not recomputed; nodes with no timing row counted as
- [ ] 모든 지역의 수치는 'get_region_cost'에서 나온 것이며, 인용된 값은 다시 계산되지 않았습니다. 타이밍 행이 없는 노드는 다음으로 계산됩니다.
zero and disclosed, never given a neighbour's value (§6.1).
0이고 공개되며 이웃의 가치가 제공되지 않습니다(§6.1).
- [ ] Every reported region came back `is_fusion_candidate: true`; rejected ones were split at `heavy_compute_inside` and
- [ ] 보고된 모든 지역이 'is_fusion_candidate: true'로 돌아왔습니다. 거부된 항목은 `heavy_compute_inside`에서 분할되었으며
re-resolved, not reported and not silently dropped (§4).
다시 해결되고 보고되지 않으며 자동으로 삭제되지 않습니다(§4).
- [ ] Single-operator exclusion applied to whole candidates only, never to branches inside one; CPU operators listed as opportunities (§2.1, §2.4).
- [ ] 단일 연산자 제외는 전체 후보에만 적용되며, 하나의 내부 분기에는 적용되지 않습니다. 기회로 나열된 CPU 운영자(§2.1, §2.4).
- [ ] Every reported region came from `get_fusion_candidates`; none was assembled by hand (§1.1).
- [ ] 보고된 모든 지역은 'get_fusion_candidates'에서 나왔습니다. 손으로 조립한 것은 하나도 없습니다(§1.1).
- [ ] Rows are in the order the candidate list gave them; no region exceeds ~¼ of the operators (§3.1, §3.3).
- [ ] 행은 후보 목록에서 제공한 순서대로 되어 있습니다. 어떤 지역도 연산자의 ~¼을 초과하지 않습니다(§3.1, §3.3).
- [ ] Each Region cell names what the block _is_, not just its operator types (§3.4).
- [ ] 각 지역 셀은 연산자 유형뿐만 아니라 블록의 이름을 지정합니다(§3.4).
- [ ] No Region cell contains a link — every graph link is in the Graphs cell (§7.3).
- [ ] 링크가 포함된 지역 셀이 없습니다. 모든 그래프 링크는 그래프 셀(§7.3)에 있습니다.
- [ ] Every row's cost and gain is the candidate's own `total_us` / `gain_us`, not a `get_region_cost` result (§1).
- [ ] 모든 행의 비용과 이득은 `get_region_cost` 결과(§1)가 아니라 후보 자신의 `total_us` / `gain_us`입니다.
- [ ] Rows are in `rank` order and `rank: 1` is the bolded row (§1).
- [ ] 행은 '순위' 순서로 되어 있으며 '순위: 1'은 굵게 표시된 행입니다(§1).
- [ ] Every Region cell carries the node-name span, not just the operator-type chain (§7.1).
- [ ] 모든 지역 셀은 연산자 유형 체인(§7.1)뿐만 아니라 노드 이름 범위를 전달합니다.
- [ ] Every span came from `get_onnx_span` and is reported verbatim — none composed, abbreviated or invented (§4).
- [ ] 모든 스팬은 `get_onnx_span`에서 왔으며 그대로 보고됩니다. 구성되거나 축약되거나 고안된 것은 없습니다(§4).
- [ ] Motifs split by shape and dtype (§5).
- [ ] 모양과 dtype으로 분할된 모티프(§5).
- [ ] No heavy-compute operator sits inside a region, and none of its time is counted as gain (§2.2, §6).
- [ ] 과도한 계산을 수행하는 운영자는 지역 내에 있지 않으며 해당 시간은 이득으로 계산되지 않습니다(§2.2, §6).
- [ ] Repeated cascades decomposed into their repeating unit with Occurrences (§5.1).
- [ ] 반복 계단식은 발생(§5.1)을 통해 반복 단위로 분해됩니다.
- [ ] Every row's cost and gain are `total_us` and `gain_us` as returned, and every figure quoted in prose matches the
- [ ] 모든 행의 비용과 이득은 반환된 `total_us` 및 `gain_us`이며, 산문에 인용된 모든 수치는
cell beside it (§6.1, §6.3).
옆에 있는 셀(§6.1, §6.3).
- [ ] No region came back with a non-empty `missing_siblings` — each was re-resolved with the siblings added (§6.1).
- [ ] 비어 있지 않은 `missing_siblings`가 있는 지역이 반환되지 않았습니다. 각 지역은 형제를 추가하여 다시 해결되었습니다(§6.1).
- [ ] `% of NPU` divides by `npu_time_us` from `get_npu_time_summary`, stated verbatim in the first context line (§1.4, §7.4).
- [ ] `NPU의 %`는 첫 번째 컨텍스트 줄(§1.4, §7.4)에 그대로 명시된 `get_npu_time_summary`에서 `npu_time_us`로 나눕니다.
- [ ] No region's cost or gain includes a `PM Load` row — `PM Load` is not a graph node and never belongs in `region_nodes` (§6.1).
- [ ] 지역의 비용이나 이득에는 'PM Load' 행이 포함되지 않습니다. 'PM Load'는 그래프 노드가 아니며 'region_nodes'(§6.1)에 속하지 않습니다.
- [ ] `unmeasured` used only where `get_region_cost` failed, never for a measured region that simply has nothing absorbable (§6.1).
- [ ] 'unmeasured'는 'get_region_cost'가 실패한 경우에만 사용되며 단순히 흡수할 수 있는 것이 없는 측정된 영역에는 절대 사용되지 않습니다(§6.1).
- [ ] Every Graphs cell is `links.markdown` from `get_onnx_span`, pasted verbatim — no hand-written link anywhere in the
- [ ] 모든 그래프 셀은 `get_onnx_span`의 `links.markdown`이며 그대로 붙여넣었습니다.
reply, and no `name=` (§7.3).
응답하고 `name=`은 없습니다(§7.3).
- [ ] No arithmetic counted as gain, including non-breaker arithmetic such as pooling (§2.2, §6).
- [ ] 풀링(§2.2, §6)과 같은 비차단기 산술을 포함하여 산술이 게인으로 계산되지 않습니다.
- [ ] Rows are in descending Expected gain order — check the column top to bottom before sending; the largest gain
- [ ] 행이 내림차순입니다. 예상 이득 순서 - 보내기 전에 열을 위에서 아래로 확인하세요. 가장 큰 이득
must be row one, or the reader's eye lands on the wrong opportunity (§7.2).
그렇지 않으면 독자의 눈이 잘못된 기회에 도달하게 됩니다(§7.2).
- [ ] Percentages are % of NPU only (§7.2).
- [ ] 백분율은 NPU의 %입니다(§7.2).
- [ ] When `headline` is non-empty, it appears **verbatim and unedited after the context block**, before the topology (§0, §7.4).
- [ ] `헤드라인`이 비어 있지 않으면 **컨텍스트 블록 뒤**, 토폴로지(§0, §7.4) 앞에 그대로 및 편집되지 않은 상태로 표시됩니다.
- [ ] No single-operator link carries `graphType` — that opens a graph unasked (§7.4).
- [ ] 요청하지 않은 그래프를 여는 `graphType`을 전달하는 단일 연산자 링크는 없습니다(§7.4).
- [ ] No tool name, field name or section number anywhere in the reply (§0).
- [ ] 응답 어디에도 도구 이름, 필드 이름 또는 섹션 번호가 없습니다(§0).
- [ ] The Nodes column is the candidate's operator count, and any "N of M" in the context uses the same M (§7.1).
- [ ] 노드 열은 후보의 연산자 수이며 컨텍스트의 모든 "N of M"은 동일한 M(§7.1)을 사용합니다.
- [ ] The `(+N more …)` tail is a line below the table, not a row in it (§7.2).
- [ ] `(+N more …)` 꼬리는 테이블의 행이 아니라 테이블 아래의 줄입니다(§7.2).
- [ ] Every row came back `reportable: true`; regions with a zero gain are named in the context block (§2.3, §6.1).
- [ ] 모든 행이 `reportable: true`로 돌아왔습니다. 게인이 0인 영역은 컨텍스트 블록(§2.3, §6.1)에서 이름이 지정됩니다.
- [ ] Every zero-gain region the tool named is reported in the context block, with its cost and its ceiling described as a ceiling (§1.1).
- [ ] 도구가 명명한 모든 제로 이득 영역은 컨텍스트 블록에 보고되며 해당 비용과 상한선은 상한선(§1.1)으로 설명됩니다.
- [ ] Every Expected gain cell is a number, and none of them is `0.00` — zero-gain and unquantified regions are named
- [ ] 모든 예상 이득 셀은 숫자이며 그 중 어느 것도 '0.00'이 아닙니다. - 이득이 0이고 정량화되지 않은 영역의 이름이 지정됩니다.
in the context block instead (§2.3, §7.4).
대신 컨텍스트 블록에 있습니다(§2.3, §7.4).
- [ ] Any repeated region scoring `0.00` was checked with `get_fusion_precedent` before being written off (§6.4).
- [ ] '0.00' 점수를 받은 반복된 지역은 삭제되기 전에 'get_fusion_precedent'로 확인되었습니다(§6.4).
- [ ] Column headers are spelled out, not abbreviated (§7.1).
- [ ] 열 헤더는 축약되지 않고 철자로 표시됩니다(§7.1).
- [ ] Every operator name written anywhere in the reply came from a tool result — no name assembled from an
- [ ] 답글에 적힌 모든 운영자 이름은 도구 결과에서 나온 것입니다.
operator type and a remembered index (§7.3).
연산자 유형 및 기억된 인덱스(§7.3).
