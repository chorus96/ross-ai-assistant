<!-- ko-translation: enabled -->
---
name: hls-pipe-decomp
description: 'Decompose a Vitis HLS pipelined region that accesses many streams into a linear chain of smaller sub-pipelines inside a dataflow region, to reduce pipeline-control complexity and improve timing. Keywords: pipeline, dataflow, stream, FIFO, timing, frequency, split pipeline, decompose'
license: MIT
argument-hint: "[<TOP_FUNCTION — function to decompose, or file:LOOP_LABEL>]"
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
# Skill: hls-pipe-decomp
# 스킬: hls-pipe-decomp

Refactor a Vitis HLS pipelined region with many `hls::stream<>` / `m_axi` ports into a linear chain of smaller sub-pipelines under `#pragma HLS dataflow`.
많은 `hls::stream<>` / `m_axi` 포트가 있는 Vitis HLS 파이프라인 영역을 `#pragma HLS dataflow` 아래의 더 작은 하위 파이프라인의 선형 체인으로 리팩터링합니다.
Each sub-pipeline has simpler pipeline-control logic and meets timing more easily.
각 하위 파이프라인은 더 간단한 파이프라인 제어 논리를 가지며 타이밍을 더 쉽게 충족합니다.

Reference: Vitis HLS User Guide (UG1399) — Dataflow Optimization chapter
참조: Vitis HLS 사용자 가이드(UG1399) - 데이터 흐름 최적화 장

## When to use
## 언제 사용하나요?

Trigger on explicit `/hls-pipe-decomp` OR on phrases like "split this pipeline", "lots of streams in one pipelined loop", "control signals on critical path", "FIFO/MAXI status in logic levels".
명시적인 `/hls-pipe-decomp` 또는 "이 파이프라인 분할", "하나의 파이프라인 루프에 많은 스트림", "중요 경로의 제어 신호", "논리 수준의 FIFO/MAXI 상태"와 같은 문구에서 트리거됩니다.
Input from the user: a function name + file, OR a `<file>:<LOOP_LABEL>` pair.
사용자 입력: 함수 이름 + 파일, 또는 `<file>:<LOOP_LABEL>` 쌍.

**Refuse and explain** if the candidate function:
**거절하고 설명** 후보자가 다음과 같은 기능을 하는 경우:
- Is neither a pipelined function nor contains a pipelined inner loop (i.e. no `#pragma HLS PIPELINE` on the function and none on any inner loop).
- 파이프라인 함수도 아니고 파이프라인 내부 루프도 포함하지 않습니다(즉, 함수에 `#pragma HLS PIPELINE`이 없고 내부 루프에도 없음).
- Touches ≤ 4 streaming ports total (decomposition unlikely to help).
- 총 4개의 스트리밍 포트를 터치합니다(분해가 도움이 되지 않을 것임).
- Contains a **feedback FIFO** (a local `hls::stream<>` both read AND written inside the pipelined region — function or loop body).
- **피드백 FIFO**(파이프라인 영역 내에서 읽고 쓴 로컬 `hls::stream<>`(함수 또는 루프 본문))을 포함합니다.
- HLS accepts this pattern, but this skill does not transform it.
- HLS는 이 패턴을 수용하지만 이 스킬은 이를 변형하지 않습니다.

## Invariants (MUST / MUST NOT)
## 불변성(MUST/MUST NOT)

1. **MUST** form a linear chain.
1. **반드시** 선형 체인을 형성해야 합니다.
1. No fork-join.
1. 포크 조인이 없습니다.
1. Exactly one stream between each pair of adjacent stages.
1. 인접한 단계의 각 쌍 사이에는 정확히 하나의 스트림이 있습니다.
2. **MUST** forward the full upstream packet through middle stages even if only part is consumed.
2. **반드시** 일부만 소비되더라도 전체 업스트림 패킷을 중간 단계를 통해 전달해야 합니다.
2. Last stage may drop the forwarding output.
2. 마지막 단계에서는 전달 출력이 삭제될 수 있습니다.
2. Filtering in the middle introduces bubbles.
2. 중간에 필터링하면 거품이 발생합니다.
3. **MUST** preserve the outer function signature exactly.
3. **반드시** 외부 함수 서명을 정확하게 보존해야 합니다.
3. Callers must not need to change.
3. 발신자는 변경할 필요가 없습니다.
4. **MUST NOT** regress II.
4. **해서는 안 됩니다** II로 회귀해서는 안 됩니다.
4. Each sub-pipeline's II ≤ the original.
4. 각 하위 파이프라인의 II ≤ 원본입니다.
4. If you can't meet this, recurse (further-decompose the offending stage) instead of accepting the regression.
4. 이를 충족할 수 없는 경우 회귀를 수용하는 대신 재발(문제가 되는 단계를 추가로 분해)하세요.
5. **MUST NOT** access the same RAM from two sub-stages.
5. **반드시** 두 하위 단계에서 동일한 RAM에 액세스해서는 안 됩니다.
5. If detected, refuse the split and ask the user to choose which stage keeps the RAM.
5. 감지되면 분할을 거부하고 사용자에게 RAM을 유지하는 단계를 선택하도록 요청합니다.
6. Intermediate FIFO depth = **2** (the default RAW latency is 1).
6. 중간 FIFO 깊이 = **2**(기본 RAW 대기 시간은 1입니다).
6. Do not enlarge unless the user explicitly asks.
6. 사용자가 명시적으로 요청하지 않는 한 확대하지 마세요.
7. Use a neutral typedef suffix (e.g.
7. 중립 typedef 접미사를 사용하세요(예:
7. `_pkt`).
7. `_pkt`).
8. New helpers go in the **same file** as the original function (above its definition).
8. 새로운 도우미는 원래 함수(해당 정의 위)와 **동일한 파일**에 들어갑니다.

## Workflow
## 작업 흐름

Confirm with the user at each phase boundary.
각 단계 경계에서 사용자에게 확인하십시오.
Skip per-phase confirmation only if the user says `auto` or `proceed without asking`.
사용자가 '자동' 또는 '묻지 않고 진행'이라고 말한 경우에만 단계별 확인을 건너뜁니다.

1. **Locate.** Read the file(s); find the target function.
1. **찾습니다.** 파일을 읽습니다. 목표 함수를 찾으세요.
2. **Analyze.** Enumerate every `hls::stream<>`, `m_axi`/pointer, RAM access, scalar used in addressing, the existing `#pragma HLS PIPELINE II=?`, and the trip count (static or stream/scalar-controlled).
2. **분석.** 모든 `hls::stream<>`, `m_axi`/포인터, RAM 액세스, 주소 지정에 사용된 스칼라, 기존 `#pragma HLS PIPELINE II=?` 및 이동 횟수(정적 또는 스트림/스칼라 제어)를 열거합니다.
2. Classify each stream as **input** or **output**. **If any local stream is both read AND written inside the pipelined region (function or loop body), stop and refuse** (feedback FIFO; out of scope).
2. 각 스트림을 **입력** 또는 **출력**으로 분류합니다. **로컬 스트림이 파이프라인 영역(함수 또는 루프 본문) 내에서 읽고 쓰여지는 경우 중지하고 거부합니다**(피드백 FIFO, 범위 밖).
2. Report inventory; get sign-off.
2. 재고 보고 승인을 받으세요.
3. **Propose chain.** Default: Read → Process → Write.
3. **체인 제안.** 기본값: 읽기 → 프로세스 → 쓰기.
3. If Read or Write would still touch > ~8 narrow streams or > ~4 wide MAXI ports, propose further-decomposing that part (see *The core transform* below).
3. 읽기 또는 쓰기가 여전히 > ~8개의 좁은 스트림 또는 > ~4개의 넓은 MAXI 포트에 닿는 경우 해당 부분을 추가로 분해할 것을 제안합니다(아래 *핵심 변환* 참조).
3. Render the proposal as `A → (s1) → B → (s2) → C → … → Z` with the stream count each stage will touch.
3. 각 단계가 접하게 될 스트림 수를 사용하여 `A → (s1) → B → (s2) → C → … → Z`로 제안을 렌더링합니다.
3. Get sign-off.
3. 승인을 받으세요.
4. **Design packet.** For each intermediate stream, define a packet type carrying every field that crosses the stage boundary:
4. **패킷을 설계합니다.** 각 중간 스트림에 대해 단계 경계를 넘는 모든 필드를 전달하는 패킷 유형을 정의합니다.
   - Prefer a `struct` with `#pragma HLS aggregate` (or `hls::vector<T, N>` when the payload is N homogeneous elements) — cleaner than manual bit-packing.
   - `#pragma HLS 집계`(또는 페이로드가 N개의 동종 요소인 경우 `hls::Vector<T, N>`)가 있는 `struct`를 선호합니다. 수동 비트 패킹보다 깔끔합니다.
   - Fall back to a packed `ap_uint<>` only when a struct is awkward.
   - 구조체가 어색할 때만 압축된 `ap_uint<>`로 대체하세요.
   - Dynamic trip count seen by downstream → add a 1-bit end-of-stream flag field (e.g. a `bool` in the struct).
   - 다운스트림에서 확인되는 동적 이동 횟수 → 1비트 스트림 끝 플래그 필드를 추가합니다(예: 구조체의 'bool').
Show the packet layout.
패킷 레이아웃을 표시합니다.
Get sign-off.
승인을 받으세요.
5. **Generate helpers.** Insert new helper functions into the same file, above the original.
5. **도우미를 생성합니다.** 동일한 파일의 원본 위에 새 도우미 함수를 삽입합니다.
5. Each helper: takes its intermediate streams by reference; inner loop marked `#pragma HLS PIPELINE II=1` (or matching original II); `#pragma HLS UNROLL` on per-stream inner loops, as the original did; **forwards** the upstream packet (except for the last stage in a chain).
5. 각 도우미는 참조로 중간 스트림을 가져옵니다. `#pragma HLS PIPELINE II=1`(또는 일치하는 원본 II)로 표시된 내부 루프; 원본과 마찬가지로 스트림별 내부 루프의 `#pragma HLS UNROLL`; **전달** 업스트림 패킷(체인의 마지막 단계 제외)
5. Use a template parameter (`<int start, int end>`, `<int bank>`, `<int GROUP_ID>`) when a stage handles a contiguous range/bank/group.
5. 스테이지가 연속 범위/뱅크/그룹을 처리할 때 템플릿 매개변수(`<int start, int end>`, `<intbank>`, `<int GROUP_ID>`)를 사용합니다.
6. **Wire dataflow region.** Replace the original function body with `#pragma HLS dataflow` + intermediate stream declarations + the chain of helper calls.
6. **데이터 흐름 영역 연결.** 원래 함수 본문을 `#pragma HLS 데이터 흐름` + 중간 스트림 선언 + 도우미 호출 체인으로 바꿉니다.
6. Preserve the outer signature exactly.
6. 외부 서명을 정확하게 유지하십시오.
7. **Add stream-depth pragmas.** `#pragma HLS STREAM variable=<name> depth=2` on every intermediate stream.
7. **스트림 깊이 pragma를 추가합니다.** 모든 중간 스트림에 `#pragma HLS STREAM 변수=<name> 깊이=2`.
7. Do not enlarge unless the user asks.
7. 사용자가 요청하지 않는 한 확대하지 마세요.
8. **(Optional) Recurse.** If a sub-stage still has too many streams, apply steps 3-7 to it.
8. **(선택 사항) 반복.** 하위 단계에 여전히 스트림이 너무 많으면 3~7단계를 적용하세요.
8. A Process stage may itself become a `#pragma HLS dataflow` containing one sub-process per resource (e.g. one per DDR channel).
8. 프로세스 단계 자체는 리소스당 하나의 하위 프로세스(예: DDR 채널당 하나)를 포함하는 '#pragma HLS 데이터 흐름'이 될 수 있습니다.
9. **Summary report.** Emit a manual-verification checklist: stages created, intermediate streams + depths, new typedefs, "outer signature preserved? (yes is expected)", "II preserved? (vitis_hls verifies)", "any RAM shared across new stages? (flag if yes)", "next: csim → csynth_design → check II → check timing", and a note that dataflow control logic itself may become the new bottleneck.
9. **요약 보고서.** 수동 확인 체크리스트를 내보냅니다. 생성된 단계, 중간 스트림 + 깊이, 새 형식 정의, "외부 서명 보존 여부(예로 예상됨)", "II 보존 여부(vitis_hls 확인)", "새 단계에서 공유되는 RAM(예인 경우 플래그)", "다음: csim → csynth_design → check II → check timing" 및 데이터 흐름 제어 논리 자체가 새로운 병목 현상이 될 수 있다는 메모입니다.

This skill does NOT run `vitis_hls`, `csim`, `csynth`, or any synthesis tool.
이 스킬은 `vitis_hls`, `csim`, `csynth` 또는 기타 합성 도구를 실행하지 않습니다.
The user verifies II and timing.
사용자는 II와 타이밍을 확인합니다.

## The core transform
## 핵심 변환

Pack the inputs into a packet, then build a linear chain of stages connected by
입력을 패킷으로 묶은 다음 다음과 같이 연결된 선형 단계 체인을 구축합니다.
exactly one stream each.
각각 정확히 하나의 스트림입니다.
Every stage reads a packet, handles only its own subset
모든 단계는 패킷을 읽고 자신의 하위 집합만 처리합니다.
of the work, and forwards the full packet downstream; the last stage does not
작업을 수행하고 전체 패킷을 다운스트림으로 전달합니다. 마지막 단계는 그렇지 않다
forward.
앞으로.
Two independent choices specialize each stage:
두 가지 독립적인 선택이 각 단계를 전문화합니다.

- **Subset selection** — a static range of ports (`for j in [START,END]` unrolled),
- **하위 집합 선택** — 포트의 정적 범위(`for j in [START,END]` 펼쳐짐),
OR a runtime tag carried in the packet (`if (tag's group == THIS_STAGE)`).
또는 패킷에 포함된 런타임 태그(`if (tag's group == THIS_STAGE)`).
- **Termination** — a static loop bound when the trip count is known, OR a 1-bit
- **종료** — 트립 횟수가 알려졌을 때 정적 루프 바인딩 또는 1비트
end-of-stream flag field in the packet when it is dynamic.
동적일 때 패킷의 스트림 끝 플래그 필드입니다.

A sub-stage that still has too many ports is decomposed the same way recursively
여전히 포트가 너무 많은 하위 단계는 동일한 방식으로 재귀적으로 분해됩니다.
(this is how a 2D banks × slots array is handled — range-split applied twice).
(이것은 2D 뱅크 × 슬롯 배열이 처리되는 방식입니다. 범위 분할이 두 번 적용됩니다.)

Example 1 — *static range, fan-in*: each stage reads a range of input streams and packs them.
예 1 — *정적 범위, 팬인*: 각 단계는 다양한 입력 스트림을 읽고 이를 압축합니다.
```cpp
struct AddrPkt { unsigned addr[N]; };   // #pragma HLS aggregate variable=...
template <int START, int END>
void read_range(hls::stream<unsigned> in[N],
                hls::stream<AddrPkt>& prev,   // omit on the first stage
                hls::stream<AddrPkt>& out) {
  for (int i = 0; i < TRIP; i++) {
#pragma HLS pipeline II=1
    AddrPkt p = prev.read();                  // forward upstream packet
    for (int j = START; j <= END; j++) {
#pragma HLS unroll
      p.addr[j] = in[j].read();
    }
    out.write(p);
  }
}
// chain: read_range<0,7>(in, s0); read_range<8,15>(in, s0, s1); ...
```

Example 2 — *runtime tag, fan-out*: each stage reads one packet stream and routes to a subset of the output streams by a runtime id.
예 2 - *런타임 태그, 팬아웃*: 각 단계는 하나의 패킷 스트림을 읽고 런타임 ID를 기준으로 출력 스트림의 하위 집합으로 라우팅합니다.
```cpp
struct Pkt { ap_uint<64> data; ap_uint<6> id; };   // #pragma HLS aggregate variable=...
template <int GROUP>
void route_group(hls::stream<Pkt>& in,
                 hls::stream<Pkt>& out,            // omit on the last stage
                 hls::stream<ap_uint<64> > lane[4]) {
  for (int i = 0; i < TRIP; i++) {
#pragma HLS pipeline II=1
    Pkt p = in.read();                 // single input stream
    if ((p.id >> 2) == GROUP)          // does this packet belong to my group?
      lane[p.id & 3].write(p.data);    // route to one of my 4 output lanes
    out.write(p);                      // forward ALL packets — do not filter (last stage drops this)
  }
}
// chain: pack(ddr, s0); route_group<0>(s0,s1,lane); route_group<1>(s1,s2,lane); ...
```

For a dynamic trip count, drive the loop from the packet's end-of-stream flag
동적 트립 횟수의 경우 패킷의 스트림 끝 플래그에서 루프를 구동합니다.
instead of a fixed bound.
고정된 경계 대신.

## Common failure modes
## 일반적인 실패 모드

| Situation<br>상황 | Response<br>응답 |
|---|---|
| Function not found in file<br>파일에서 함수를 찾을 수 없습니다. | Ask user to confirm name/path; offer fuzzy matches.<br>사용자에게 이름/경로를 확인하도록 요청합니다. 퍼지 일치를 제공합니다. |
| Function already in dataflow<br>이미 데이터 흐름에 있는 함수 | Show current structure; ask if user wants to further-decompose a sub-stage.<br>현재 구조를 표시합니다. 사용자가 하위 단계를 추가로 분해하기를 원하는지 묻습니다. |
| Target is a pipelined loop nested in a larger function<br>대상은 더 큰 함수에 중첩된 파이프라인 루프입니다. | Extract it into its own function first, then proceed.<br>먼저 자체 기능으로 추출한 다음 계속 진행하십시오. |
| Feedback FIFO in the pipelined region (stream read AND written inside the function or loop body)<br>파이프라인 영역의 피드백 FIFO(함수 또는 루프 본문 내부에 스트림 읽기 및 쓰기) | Refuse; out of scope.<br>거절하다; 범위를 벗어났습니다. |
| Stream type uses an opaque custom typedef<br>스트림 유형은 불투명한 사용자 정의 typedef를 사용합니다. | Read the header(s); if still ambiguous, ask user for the underlying width.<br>헤더를 읽으십시오. 여전히 모호한 경우 사용자에게 기본 너비를 물어보세요. |
| Dynamic trip count and downstream needs it<br>동적 이동 횟수 및 다운스트림에 필요 | Use a 1-bit end-of-stream flag in the packet; call it out in the packet design.<br>패킷에 1비트 스트림 끝 플래그를 사용합니다. 패킷 디자인에서 이를 호출합니다. |
| Same RAM accessed from two intended stages<br>두 개의 의도된 단계에서 동일한 RAM에 액세스함 | Refuse the split; ask user which stage keeps the RAM.<br>분할을 거부하십시오. 어느 단계에서 RAM을 유지하는지 사용자에게 묻습니다. |
| Original signature has unusual types (e.g. `T (&arr)[N]`)<br>원래 서명에 특이한 유형이 있습니다(예: `T (&arr)[N]`) | Preserve verbatim. Do not "improve" them.<br>그대로 보존합니다. 이를 "개선"하지 마십시오. |

## Success criteria
## 성공 기준

Done when: the diff was applied, outer signature unchanged, the chain matches what the user signed off on, no fork-join, no RAM shared across stages, and the summary checklist was emitted.
완료 시점: diff가 적용되고, 외부 서명이 변경되지 않고, 체인이 사용자가 승인한 것과 일치하고, 포크 조인이 없고, 여러 단계에서 RAM이 공유되지 않고, 요약 체크리스트가 생성되었습니다.
II/timing verification is the user's responsibility — if they report II regression, deadlock, or mismatch after `csynth_design`, walk back through steps 4-7 with them; the usual bugs are missing forwarding, mis-sized intermediate FIFO, or accidentally-shared RAM.
II/타이밍 확인은 사용자의 책임입니다. `csynth_design` 이후 II 회귀, 교착 상태 또는 불일치를 보고하는 경우 4~7단계를 다시 수행하세요. 일반적인 버그로는 전달 누락, 중간 FIFO 크기 조정 또는 실수로 공유된 RAM 등이 있습니다.

## Machine-Readable Verdict
## 기계가 읽을 수 있는 판결

When you have completed the pipeline decomposition, end your response with:
파이프라인 분해를 완료하면 다음을 사용하여 응답을 종료합니다.
PIPEDECOMP_VERDICT: DECOMPOSED
PIPEDECOMP_VERDICT: 분해됨
