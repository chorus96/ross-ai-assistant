<!-- ko-translation: enabled -->
---
name: vai-peak-bandwidth-analysis
description: From L3 DDR DMA-channel profiling, judge compute- vs memory-bound execution
  and recommend tiling / QoS changes, focused on the channels and operators with the
  most backpressure/starvation stall
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

# Peak L3 DDR bandwidth analysis
# 피크 L3 DDR 대역폭 분석

Analyze the **L3 DDR DMA-channel** profiling that VitisExecutionProvider emits when
VitisExecutionProvider가 다음과 같은 경우에 내보내는 **L3 DDR DMA 채널** 프로파일링을 분석합니다.
`ai_analyzer_enhanced_profiling: ["control_instrumentation"]` is enabled in runner-options config-file for VART flow.
`ai_analyzer_enhanced_profiling: ["control_instrumentation"]`는 VART 흐름에 대한 실행기 옵션 구성 파일에서 활성화됩니다.
Produce: (1) per-direction
생산: (1) 방향별
channel throughput and utilization, (2) a compute-bound vs memory-bound verdict per operator,
채널 처리량 및 활용도, (2) 연산자당 컴퓨팅 바인딩과 메모리 바인딩 비교,
and (3) prioritized improvement recommendations across five families — cut L3 bytes (tiling /
(3) 5개 제품군에 걸쳐 우선순위 개선 권장 사항 - L3 바이트 줄이기(타일링 /
fusion / precision), hide latency (double-buffering / prefetch), balance channels, relieve L2,
융합/정밀), 대기 시간 숨기기(이중 버퍼링/프리페치), 채널 균형, L2 완화,
and DDR QoS — always leading with the operators/channels carrying the most backpressure/
및 DDR QoS — 항상 가장 많은 역압을 전달하는 운영자/채널이 선두에 섭니다.
starvation stall cycles, because that is where the gain is.
기아 실속 사이클, 왜냐하면 그것이 이득이 되는 곳이기 때문입니다.

Run it when the user asks about DDR/L3 bandwidth, channel throughput/utilization, stall
사용자가 DDR/L3 대역폭, 채널 처리량/사용률, 정지에 대해 묻는 경우 실행합니다.
(backpressure/starvation) analysis, whether layers are compute- or memory-bound, or how to
(배압/기아) 분석, 레이어가 컴퓨팅 바인딩인지 메모리 바인딩인지 여부 또는 방법
raise channel utilization.
채널 활용도를 높여보세요.

## Constraint: MCP tools only, no raw disk artifacts
## 제약 조건: MCP 도구만 해당, 원시 디스크 아티팩트 없음

Use ONLY these MCP tools, all of which return computed/summarized results:
계산/요약된 결과를 반환하는 다음 MCP 도구만 사용하세요.
`get_reports`, `get_timelines`, `get_performance_metrics`, `get_single_timing_dataframe`,
`get_reports`, `get_timelines`, `get_performance_metrics`, `get_single_timing_dataframe`,
`get_npu_summary`, `get_memory_map` (for per-buffer ping/pong double-buffer state), and
`get_npu_summary`, `get_memory_map`(버퍼별 ping/pong 이중 버퍼 상태용) 및
(optionally, for exact per-channel/per-operator cycle counts) `get_summary_data`.
(선택적으로 정확한 채널별/연산자별 주기 수를 확인하기 위해) `get_summary_data`.
Do NOT reach
도달하지 마십시오
into raw on-disk files — no `get_vaiml_artifact`, `search_mlir`, `get_make_log`, or
원시 디스크 파일로 — `get_vaiml_artifact`, `search_mlir`, `get_make_log` 또는
`get_spilling_reasons`.
`get_spilling_reasons`.
The spill **magnitude** you need is already in `get_npu_summary`
필요한 유출 **크기**는 이미 `get_npu_summary`에 있습니다.
(`l3_memory_allocation` → the `Spills` entry); you do not need the spilling CSV for this
(`l3_memory_allocation` → `Spills` 항목); 이를 위해 유출 CSV가 필요하지 않습니다
analysis.
분석.

## Background needed to interpret the counters
## 카운터를 해석하는 데 필요한 배경

- **Peak per channel:** each running cycle transfers a fixed 64-bit word.
- **채널당 피크:** 각 실행 주기는 고정된 64비트 워드를 전송합니다.
- On the default
- 기본적으로
**VEK385** embedded platform (1250 MHz) that is **10 GB/s per channel**.
**VEK385** 임베디드 플랫폼(1250MHz), **채널당 10GB/s**.
Other Vitis Versal
기타 Vitis Versal
embedded platforms run at **1000 MHz → 8 GB/s per channel**.
임베디드 플랫폼은 **1000MHz → 채널당 8GB/s**로 실행됩니다.
Read `clock_freq_mhz` from the
'clock_freq_mhz'를 읽어보세요.
performance summary and scale peak = `10 GB/s × clock/1250`.
성능 요약 및 최대 규모 = `10GB/s × 클럭/1250`.
- **Utilization** of a channel = `running / (running + stall)`. **Achieved BW ≈ peak ×
- 채널 **활용** = `실행 중 / (실행 중 + 정지)`. **달성된 BW ≒ 피크 ×
utilization**.
활용**.
Prefer the timeline's own `(MB/s)` column as the authoritative achieved figure;
신뢰할 수 있는 달성 수치로 타임라인 자체 `(MB/s)` 열을 선호합니다.
use peak only as the ceiling.
피크를 천장으로만 사용하십시오.
- **Stall splits into two causes, and their meaning DEPENDS ON DIRECTION.** The general rule is:
- **실속은 두 가지 원인으로 나뉘며 그 의미는 방향에 따라 다릅니다.** 일반적인 규칙은 다음과 같습니다.
**backpressure = the channel's _destination_ is busy / cannot accept; starvation = the
**배압 = 채널의 _destination_이 사용 중이거나 수락할 수 없습니다. 배고픔 =
channel's _source_ cannot supply.** A read channel moves L3 DDR → L2 (source = DDR,
채널의 _source_는 공급할 수 없습니다.** 읽기 채널은 L3 DDR → L2로 이동합니다(소스 = DDR,
destination = L2); a write channel moves L2 → L3 DDR (source = L2, destination = DDR).
목적지 = L2); 쓰기 채널은 L2 → L3 DDR(소스 = L2, 대상 = DDR)로 이동합니다.
So the
그래서
two directions flip which side each stall points at:
각 실속이 어느 쪽을 가리키는지 두 방향으로 뒤집습니다.

| Direction<br>방향         | Backpressure stall (destination busy)<br>배압 실속(대상이 사용 중임)                                                             | Stream starvation stall (source can't supply)<br>스트림 기아 중단(소스가 공급할 수 없음)                                                    |
  | ----------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **Read** (L3→L2)<br>**읽기** (L3→L2)  | **L2** busy — L2 memory bank conflicts, or waiting to acquire the **write** lock on the L2 buffer<br>**L2** 사용 중 — L2 메모리 뱅크가 충돌하거나 L2 버퍼에 대한 **쓰기** 잠금 획득을 기다리는 중입니다. | **L3 DDR** memory controller busy and cannot supply data<br>**L3 DDR** 메모리 컨트롤러가 사용 중이어서 데이터를 공급할 수 없습니다.                                         |
| **Write** (L2→L3)<br>**쓰기** (L2→L3) | **L3 DDR** memory controller busy to accept the data<br>**L3 DDR** 메모리 컨트롤러가 데이터를 수용하는 중입니다.                                              | **L2** busy — L2 memory bank conflicts, or waiting to acquire the **read** lock on the L2 buffer<br>**L2** 사용 중 — L2 메모리 뱅크가 충돌하거나 L2 버퍼에 대한 **읽기** 잠금 획득을 기다리는 중입니다. |

The remediation follows the side, not the label. **DDR-bound** = read **starvation** _or_ write
교정은 라벨이 아닌 측면을 따릅니다. **DDR 바인딩** = 읽기 **기아** _또는_ 쓰기
**backpressure** → DDR-channel QoS + reduce L3 volume. **L2-bound** = read **backpressure** _or_
**배압** → DDR 채널 QoS + L3 볼륨 감소. **L2 경계** = **배압** 읽기 _또는_
write **starvation** → relieve L2 (spill/tiling/L2-chaining, bank-conflict/lock mitigation).
**기아** 작성 → L2 완화(유출/타일링/L2 체인, 은행 충돌/잠금 완화).
Do NOT assume "backpressure = L2" — that only holds for reads; for writes it is the DDR side.
읽기에만 적용되는 "배압 = L2"라고 가정하지 마십시오. 쓰기의 경우 DDR 쪽입니다.

- **Channels:** each NPU column has **2 read + 2 write channels** to DDR, each with QoS
- **채널:** 각 NPU 열에는 DDR에 대한 **2개의 읽기 + 2개의 쓰기 채널**이 있으며 각각 QoS가 있습니다.
parameters set through the Vitis embedded platform.
Vitis 임베디드 플랫폼을 통해 설정된 매개변수입니다.
QoS is the lever for the **DDR-bound** case
QoS는 **DDR 바인딩** 사례의 레버입니다.
(read starvation / write backpressure).
(기아 읽기/배압 쓰기).
- **Platform cap:** the DDR controller has a fixed **bytes/sec ceiling for the whole platform**.
- **플랫폼 한도:** DDR 컨트롤러에는 전체 플랫폼에 대해 고정된 **바이트/초 상한선**이 있습니다.
QoS only redistributes within that cap — it cannot exceed it.
QoS는 해당 한도 내에서만 재분배되며 이를 초과할 수 없습니다.
So the primary lever for a
따라서
bandwidth-bound model is **reducing L3 transfers** (weights + spill), which is what the tiling
대역폭 제한 모델은 **L3 전송을 줄입니다**(가중치 + 유출).
section targets.
섹션 대상.

## Data sources and their shapes
## 데이터 소스 및 그 형태

- **`get_performance_metrics` → `summary`** (one row).
- **`get_performance_metrics` → `summary`**(1행).
- Aggregate (sum over all channels of a
- 집계(모든 채널에 대한 합계
direction) channel counters: `peak_read_total_ch_running_cycles`, `…_stall_cycles`,
방향) 채널 카운터: `peak_read_total_ch_running_cycles`, `…_stall_cycles`,
`…_backpressure_stall_cycles`, `…_starvation_stall_cycles`, and the `peak_write_total_ch_*`
`…_backpress_stall_cycles`, `…_starvation_stall_cycles` 및 `peak_write_total_ch_*`
equivalents; plus `clock_freq_mhz`, `batch_count`, `total_gmacs`, `average_npu_time_usec`,
등가물; 게다가 `clock_freq_mhz`, `batch_count`, `total_gmacs`, `average_npu_time_usec`,
`average_inference_time_usec`.
`average_inference_time_usec`.
These `peak_*_total_ch_*` fields are the **whole-direction
이 `peak_*_total_ch_*` 필드는 **전체 방향입니다.
aggregate**, not a single channel.
단일 채널이 아닌 집합**입니다.
  - **`operators`** table: per op-type `value_usec` (execution time), `count`, `percent`,
  - **`연산자`** 테이블: 작업 유형별 `value_usec`(실행 시간), `count`, `percent`,
`cycles_per_count`, `category` (npu / cpu / host_profiling).
`cycles_per_count`, `category`(npu/cpu/host_profiling).
- **`get_single_timing_dataframe(folder, timeline=<NPU>, inference=0)`** — one row per operator
- **`get_single_timing_dataframe(folder, timeline=<NPU>, inference=0)`** — 연산자당 행 1개
instance.
인스턴스.
Carries `Name`, `Type`, `Execution Time.μs` / `.Cycles`, and the per-op
`Name`, `Type`, `Execution Time.μs` / `.Cycles` 및 작업당 전달
min-throughput channel: `L3 Throughput Min Read.(MB/s)`,
최소 처리량 채널: `L3 처리량 최소 읽기.(MB/s)`,
`L3 Throughput Min Read.Backpressure Stall (%)`, `L3 Throughput Min Read.Starvation Stall (%)`,
`L3 처리량 최소 읽기.배압 스톨(%)`, `L3 처리량 최소 읽기.기아 스톨(%)`,
`L3 Throughput Min Read.Column:Channel` (and the `L3 Throughput Min Write.*` set).
`L3 최소 처리량 읽기.열:채널`(및 `L3 최소 처리량 쓰기.*` 세트).
This ties
이 관계
each operator to its worst channel with per-op stall % — the core of the compute/memory verdict
각 연산자를 작업별 지연 %로 최악의 채널로 이동 — 컴퓨팅/메모리 판정의 핵심
and the per-channel attribution, without needing `get_summary_data`.
`get_summary_data`가 필요 없는 채널별 속성입니다.
Large table: dump with
대형 테이블: 덤프
`format="parquet"`/`csv` + `output_dir` and aggregate by `Type`; report totals and top-N.
`format="parquet"`/`csv` + `output_dir` 및 `Type`별로 집계; 총계 및 상위 N개를 보고합니다.
- **`get_npu_summary`** — `total_l3_allocated`, `total_l3_transfers`, `l3_memory_allocation`
- **`get_npu_summary`** — `total_l3_allocation`, `total_l3_transfers`, `l3_memory_allocation`
(per class: `Spills`, `Weights`, `IFM`, `OFM` with `size_in_bytes`), `gmacs_per_operator`
(클래스별: `Spills`, `Weights`, `IFM`, `OFM`(`size_in_bytes` 포함), `gmacs_per_operator`
(which op-types carry compute), `total_gmacs`.
(연산을 수행하는 작업 유형), `total_gmacs`.
Large payload — read only those fields.
대형 페이로드 - 해당 필드만 읽습니다.
- **`get_memory_map(folder, level="L2")`** — one row per buffer.
- **`get_memory_map(folder, level="L2")`** — 버퍼당 한 행.
- Two uses here:
- 여기서는 두 가지 용도로 사용됩니다.
  - **Double-buffering:** `double_buffer` = `ping` / `pong` (DMA overlaps compute) vs `single`
  - **이중 버퍼링:** `double_buffer` = `ping` / `pong`(DMA 오버랩 계산) 대 `single`
(no overlap → compute waits on the transfer).
(겹침 없음 → 컴퓨팅이 전송을 기다림)
`force_single_buffer` true = pinned single,
`force_single_buffer` true = 고정된 단일,
cannot ping/pong as-is.
있는 그대로는 핑/퐁을 할 수 없습니다.
  - **L2 provisioning:** aggregate `size` per `be_layer_name` (and by `buf_type`) to get each
  - **L2 프로비저닝:** `be_layer_name`별로(및 `buf_type`별로) `size`를 집계하여 각각을 가져옵니다.
operator's **total L2 footprint** and its **buffer granularity** (few coarse buffers vs many
운영자의 **총 L2 공간** 및 **버퍼 세분성**(거친 버퍼가 적음 대 다수)
fine ones).
괜찮아요).
This is the MCP-visible proxy for how much L2 / how many stamps an operator got.
이는 L2 양/운영자가 얻은 스탬프 수에 대한 MCP 표시 프록시입니다.
  - Other fields: `be_layer_name` (join key to the operator/layer), `buf_type`
  - 기타 필드: `be_layer_name`(연산자/레이어에 대한 조인 키), `buf_type`
(IFM / OFM / WTS / HEAP_STACK), `offset`, `is_l2_no_spill`.
(IFM / OFM / WTS / HEAP_STACK), '오프셋', 'is_l2_no_spill'.
`partition_memory_size` is often
`partition_memory_size`는 종종
`0` — the tool does NOT give L2 capacity, so treat capacity as external context (see the stamp
`0` — 도구는 L2 용량을 제공하지 않으므로 용량을 외부 컨텍스트로 처리합니다(스탬프 참조).
note in family D).
가족 D)에 메모하십시오.
  - Ping/pong and these buffers live at **L2** (memtile), so query `level="L2"`; L3 rows do not
  - 핑/퐁과 이러한 버퍼는 **L2**(memtile)에 있으므로 `level="L2"`를 쿼리하세요. L3 행은 그렇지 않습니다.
carry meaningful double-buffer state.
의미 있는 이중 버퍼 상태를 전달합니다.
Large table — dump with `format="parquet"`/`csv` +
큰 테이블 — `format="parquet"`/`csv`로 덤프 +
`output_dir` and aggregate/filter to the layers of interest (never dump raw rows).
`output_dir` 및 관심 레이어로 집계/필터링합니다(원시 행을 덤프하지 않음).
- **`get_summary_data` (optional, large)** — `data[].perModelCharts[].peakReadBwSummary` /
- **`get_summary_data`(선택 사항, 대형)** — `data[].perModelCharts[].peakReadBwSummary` /
`peakWriteBwSummary`.
'peakWriteBwSummary'.
Exact per-channel and per-operator cycle counts:
정확한 채널별 및 작업자별 주기 수:
`chRunningCycles[]`, `chBackpressureStallCycles[]`, `chStreamStarvationStallCycles[]` (each a
`chRunningCycles[]`, `chBackpressionStallCycles[]`, `chStreamStarvationStallCycles[]`(각각
`{name:"Column N Channel M", cycles}` list); the `totalCh{Running,Stall,Backpressure,Starvation}Cycles`
`{이름:"열 N 채널 M", 주기}` 목록); `totalCh{Running,Stall,BackPressure,Starvation}주기`
scalars (== the perf-metrics aggregates); and `operator{Running,Backpressure,StreamStarvation}StallCycles[]`
스칼라(== 성능 지표 집계); 및 `연산자{Running,BackPressure,StreamStarvation}StallCycles[]`
(per op-type `{name, cycles, elements}`).
(작업 유형 `{이름, 주기, 요소}`당).
Use only when the user wants exact per-channel or
사용자가 정확한 채널별 또는
per-operator cycle counts; otherwise the timeline's per-op stall % is enough.
작업자별 주기 수; 그렇지 않으면 타임라인의 작업별 지연 비율로 충분합니다.

## Single-direction captures
## 단일 방향 캡처

An `ai_analyzer_enhanced_profiling` capture usually instruments **one direction**.
`ai_analyzer_enhanced_profiling` 캡처는 일반적으로 **한 방향**을 계측합니다.
Detect it: if
감지: 만약
`peak_write_total_ch_backpressure_stall_cycles == -1` (or the running/stall totals are 0), the
`peak_write_total_ch_backpression_stall_cycles == -1`(또는 실행/지속 총계는 0),
write channels were not captured in this folder — and vice versa for reads.
쓰기 채널은 이 폴더에서 캡처되지 않았으며 읽기의 경우에도 마찬가지입니다.
Say so plainly and,
그렇게 솔직하게 말하고,
if the user wants both directions, ask for the sibling capture (e.g. a `read_*` folder alongside
사용자가 양방향을 원하는 경우 형제 캡처(예: 'read_*' 폴더와 함께)를 요청하세요.
a `write_*` folder).
`write_*` 폴더).
Never present the empty direction as "no stalls".
빈 방향을 "No Stalls"로 제시하지 마십시오.

## Method
## 방법

1. **Locate the NPU timeline.** `get_reports` → `get_timelines(folder, report_id)` on the
1. **NPU 타임라인을 찾으세요.** `get_reports` → `get_timelines(folder, report_id)`
HARDWARE/SIMULATED timing report; the NPU timeline is the non-`CPU` one (e.g.
하드웨어/시뮬레이트된 타이밍 보고서; NPU 타임라인은 'CPU'가 아닌 타임라인입니다(예:
`vaiml_par_0/0`).
`vaiml_par_0/0`).
2. **Channel-level throughput (from `get_performance_metrics.summary`).** For each captured
2. **채널 수준 처리량(`get_performance_metrics.summary`에서).** 캡처된 각 항목에 대해
direction compute utilization = `running/(running+stall)` and achieved BW = `peak ×
방향 계산 활용도 = `running/(running+stall)` 및 달성된 BW = `peak ×
utilization`.
활용`.
Report which cause dominates the stall: compare `…_backpressure_stall_cycles`
어떤 원인이 정지를 지배하는지 보고: `…_backpress_stall_cycles` 비교
vs `…_starvation_stall_cycles`, then translate to a **side** using the direction table above:
vs `…_starvation_stall_cycles`, 그런 다음 위의 방향 표를 사용하여 **side**로 변환합니다.
read starvation → DDR, read backpressure → L2; write backpressure → DDR, write starvation →
기아 → DDR 읽기, 배압 → L2 읽기; 배압 쓰기 → DDR, 기아 쓰기 →
L2.
State the dominant side (DDR-bound vs L2-bound), not just the raw label.
원시 레이블뿐만 아니라 주요 측면(DDR 바인딩과 L2 바인딩)을 명시합니다.
3. **Per-operator compute vs memory bound (from `get_single_timing_dataframe`).** Aggregate by
3. **연산자별 계산과 메모리 바운드(`get_single_timing_dataframe`에서).** 집계 기준
`Type`: sum `Execution Time.μs`, and take the mean read/write `Backpressure %` and
`Type`: `Execution Time.μs`를 합산하고 `BackPressure %` 읽기/쓰기 평균을 취합니다.
`Starvation %` and `(MB/s)`.
`굶주림 %` 및 `(MB/s)`.
Join `gmacs_per_operator` from `get_npu_summary` to know which
`get_npu_summary`에서 `gmacs_per_operator`에 가입하여 어느 것이 무엇인지 알아보세요.
types carry MACs.
유형에는 MAC이 있습니다.
Classify each op-type:
각 작업 유형을 분류합니다.
   - **Compute-bound:** low total stall %, and the op carries real GMACs.
   - **컴퓨팅 바운드:** 총 지연 비율이 낮으며 작업은 실제 GMAC를 전달합니다.
   - Its time is arithmetic.
   - 그 시간은 산술입니다.
   - **Memory-bound, DDR side:** high read **starvation %** _or_ high write **backpressure %** →
   - **메모리 바운드, DDR 측:** 높은 읽기 **기아 %** _또는_ 높은 쓰기 **배압 %** →
the L3 DDR controller is the limiter.
L3 DDR 컨트롤러는 리미터입니다.
QoS candidate (+ reduce L3 volume).
QoS 후보(+ L3 볼륨 감소).
   - **Memory-bound, L2 side:** high read **backpressure %** _or_ high write **starvation %** →
   - **메모리 바운드, L2 측:** 높은 읽기 **배압 %** _또는_ 높은 쓰기 **기아 %** →
L2 bank conflicts / lock contention is the limiter.
L2 뱅크 충돌/잠금 경합이 제한됩니다.
Spill/tiling/L2-chaining and
유출/타일링/L2 체인 및
bank-conflict candidate — **not** QoS.
은행 충돌 후보 - **QoS 아님**.
   - **Movement op:** zero GMAC (Transpose/Reshape/Slice/Concat/BufferUnpad…) → memory-bound by
   - **이동 작업:** GMAC 0(Transpose/Reshape/Slice/Concat/BufferUnpad…) → 메모리 바인딩
nature; only worth attention if its execution-time share is non-trivial.
자연; 실행 시간 점유율이 사소하지 않은 경우에만 주목할 가치가 있습니다.
Note the important nuance: an op can hold most of the model's GMACs (high arithmetic
중요한 뉘앙스에 유의하세요. op는 모델의 GMAC 대부분을 보유할 수 있습니다(높은 산술 연산).
intensity, _should_ be compute-bound) yet run **memory-bound in practice** because it is
강도는 _컴퓨팅 제한적이어야 함) 그러나 **실제로는 메모리 제한**을 실행합니다.
read-starved — low achieved MB/s + high starvation % = **DDR latency/QoS-limited, not
읽기 부족 — 낮은 MB/s 달성 + 높은 기아 상태 % = **DDR 대기 시간/QoS 제한, 아님
bandwidth-saturated**.
대역폭 포화**.
Call that out; it is the highest-value finding.
그것을 불러내십시오. 그것은 가장 높은 가치를 지닌 발견이다.
4. **L3 read-reduction potential (from `get_npu_summary`).** Report the re-fetch ratio
4. **L3 읽기 감소 가능성(`get_npu_summary`에서).** 다시 가져오기 비율 보고
`total_l3_transfers / total_l3_allocated` (>1 means bytes cross DDR repeatedly — tiling can
`total_l3_transfers / total_l3_allocation`(>1은 바이트가 DDR을 반복적으로 교차함을 의미합니다. 타일링은 가능합니다.
reclaim it) and the `l3_memory_allocation` split (Spills / Weights / IFM / OFM).
회수) 및 `l3_memory_allocation` 분할(Spills / Weights / IFM / OFM).
This tells
이것은 말한다
you which read category dominates and therefore which tiling lever matters most.
카테고리를 읽는 것이 지배적이므로 타일링 레버가 가장 중요합니다.
5. **Ping/pong (double-buffer) check for the high-stall operators.** Only for the operators that
5. **스톨이 높은 운영자를 위한 핑/퐁(이중 버퍼) 확인.**
step 3 flags with large stall — do NOT dump the whole buffer table for every layer.
큰 지연이 있는 3단계 플래그 - 모든 레이어에 대해 전체 버퍼 테이블을 덤프하지 마십시오.
Call
전화
`get_memory_map(folder, level="L2")`, then for each high-stall operator match its layer via
`get_memory_map(folder, level="L2")`, 각 높은 정지 연산자에 대해 해당 레이어를 다음을 통해 일치시킵니다.
`be_layer_name` and inspect the buffers on the stalling **direction**: the **IFM/WTS** buffers
`be_layer_name`을 실행하고 정지 **방향**에서 버퍼를 검사합니다: **IFM/WTS** 버퍼
for any read-side stall (read starvation _or_ read backpressure), the **OFM** buffer for any
모든 읽기 측 지연(읽기 고갈 _또는_ 읽기 역압)에 대해 **OFM** 버퍼
write-side stall (write backpressure _or_ write starvation).
쓰기 측 지연(쓰기 역압 _또는_ 쓰기 기아).
A buffer with
버퍼
`double_buffer == "single"` on the stalling side is a **prime cause of that stall** — the
지연 측의 `double_buffer == "single"`이 **해당 지연의 주요 원인**입니다.
compute cannot overlap with its transfer.
컴퓨팅은 전송과 겹칠 수 없습니다.
Converting it to ping/pong hides the transfer behind
핑/퐁으로 변환하면 전송 내용이 숨겨집니다.
compute and removes stall cycles _without_ reducing bytes.
바이트를 줄이지 않고 정지 주기를 계산하고 제거합니다.
Flag each such buffer with its op,
각 버퍼에 해당 op로 플래그를 지정합니다.
`buf_type`, and `size`.
'buf_type' 및 'size'.
If `force_single_buffer == true`, note it is pinned single (something —
`force_single_buffer == true`인 경우 단일로 고정됩니다(뭔가 —
often an L2-capacity or spill constraint — is preventing double-buffering, so freeing L2 is a
종종 L2 용량 또는 유출 제약으로 인해 이중 버퍼링이 방지되므로 L2를 해제하는 것은
prerequisite).
전제 조건).
Rank these by the stalling op's absolute stall cycles.
지연 작업의 절대 지연 주기를 기준으로 순위를 매깁니다.
6. **L2 provisioning check for the L2-bound operators.** For operators whose stall is L2-side
6. **L2 바인딩 운영자를 위한 L2 프로비저닝 확인.** L2 측 스톨이 있는 운영자의 경우
(read backpressure / write starvation) and whose buffers are _already_ ping/pong (so B is not
(배압 읽기/기아 상태 쓰기) 버퍼가 _이미_ 핑/퐁 상태(따라서 B는 그렇지 않음)
the fix), aggregate `size` per `be_layer_name` from the same `get_memory_map(level="L2")` call
수정 사항), 동일한 `get_memory_map(level="L2")` 호출에서 `be_layer_name`당 `size`를 집계합니다.
to get each operator's **total L2 footprint** and **buffer granularity** (count × size).
각 연산자의 **총 L2 공간** 및 **버퍼 세분성**(개수 × 크기)을 가져옵니다.
Compare
비교
the high-stall op against a low-stall neighbor: a markedly **smaller total L2 / coarser (fewer,
낮은 실속 이웃에 대한 높은 실속 작업: 현저하게 **더 작은 총 L2/거칠게(더 적음,
larger) buffers** signals it is **under-stamped** (too few L2 write ports to absorb the stream)
더 큰) 버퍼**는 **스탬프가 부족하다는 신호**(스트림을 흡수하기에는 L2 쓰기 포트가 너무 적음)
— the family-D "more stamps / finer buffers" lever.
— 제품군 D의 "더 많은 스탬프/더 미세한 버퍼" 레버.
Do NOT read stamp count from the
우표 개수를 읽지 마세요.
mapped-graph `Tiling` entry count; it is unreliable (see family D).
매핑된 그래프 `타일링` 항목 수; 신뢰할 수 없습니다(가족 D 참조).
7. **Rank by gain and recommend.** Order operators and channels by **absolute stall cycles**
7. **이득에 따라 순위를 매기고 추천합니다.** **절대적인 스톨 주기**를 기준으로 운영자와 채널을 주문합니다.
(not %), so a large op with moderate % outranks a tiny op at 90%.
(%가 아님), 따라서 중간 %의 대규모 작전이 90%의 작은 작전보다 순위가 높습니다.
Lead every recommendation
모든 추천을 주도하세요
with the top stall contributors.
최고의 스톨 기여자들과 함께.

## Recommendations to produce
## 생산 권장 사항

Give concrete, prioritized suggestions, each tied to a number from the tools above.
구체적이고 우선순위가 높은 제안을 제시하세요. 각 제안은 위 도구의 숫자와 연결되어 있습니다.
First pick
첫 번째 선택
the right _family_ from the stall signal, then the specific lever:
실속 신호에서 오른쪽 _family_, 그다음 특정 레버:

First translate each stall to a **side** with the direction table (read starvation / write
먼저 각 칸을 방향 테이블을 사용하여 **side**로 변환합니다(기아 읽기/쓰기).
backpressure = **DDR**; read backpressure / write starvation = **L2**), then:
배압 = **DDR**; 배압 읽기 / 기아 쓰기 = **L2**), 그런 다음:

| Signal (from the tools)<br>신호(도구에서)                                                    | Side<br>측면                      | Lever family<br>레버 패밀리                                             |
| -------------------------------------------------------------------------- | ------------------------- | -------------------------------------------------------- |
| Read **starvation** or write **backpressure** high, **and** MB/s near peak<br>**기아**를 읽거나 **배압** 높음, **및** 최고치에 가까운 MB/s 쓰기 | DDR, volume-saturated<br>DDR, 볼륨 포화     | **A** cut L3 bytes + **E** QoS<br>**A** L3 바이트 줄임 + **E** QoS                           |
| Read **starvation** or write **backpressure** high, **but** MB/s low<br>**기아**를 읽거나 **배압**을 높게 쓰지만 **MB/s는 낮습니다.       | DDR, latency-limited<br>DDR, 지연 시간 제한      | **B** hide latency + **C** balance + **E** QoS<br>**B** 대기 시간 숨기기 + **C** 밸런스 + **E** QoS           |
| Read **backpressure** or write **starvation** high<br>**배압** 읽기 또는 **기아** 높음 쓰기                         | L2 (bank conflict / lock)<br>L2(은행 충돌/잠금) | **A** cut spill/L2-chaining + **D** relieve L2 (+ **B**)<br>**A** 컷 유출/L2 체인 + **D** L2 완화(+ **B**) |
| Some channels idle while others saturate<br>일부 채널은 유휴 상태이고 다른 채널은 포화 상태입니다.                                   | channel imbalance<br>채널 불균형         | **C** spread load<br>**C** 확산하중                                        |
| Low total stall + carries GMACs<br>낮은 총 실속 + GMAC 운반                                            | genuinely compute-bound<br>진정한 컴퓨팅 기반   | none — leave it<br>없음 — 그대로 놔두세요                                          |

### A.
### Reduce L3 byte volume
### L3 바이트 볼륨 줄이기

When near the platform bytes/sec cap (re-fetch ratio ≫ 1, MB/s near peak).
플랫폼 바이트/초 한도에 가까울 때(다시 가져오기 비율 ≫ 1, 최고에 가까운 MB/초)
Map the
지도
`l3_memory_allocation` split to the lever, and always state the tradeoff:
`l3_memory_allocation`을 레버로 분할하고 항상 절충 사항을 명시합니다.

- **Tiling — weights dominate** → larger output/spatial tiles (output-stationary) so each weight
- **타일링 — 가중치가 지배적** → 더 큰 출력/공간 타일(출력 고정)이므로 각 가중치
fetch is amortized over more work → fewer weight re-streams.
가져오기는 더 많은 작업 → 더 적은 가중치 재스트리밍으로 분할됩니다.
- **Tiling — spills dominate** (common; spills often exceed weights) → biggest lever.
- **타일링 — 유출이 지배적임** (일반적이며 유출이 종종 무게를 초과함) → 가장 큰 레버.
- Retile so
- 그래서 Retile
producer→consumer stays resident in **L2 (L2 chaining)**, removing both the spill write and its
생산자→소비자는 **L2(L2 체인)**에 상주하며 유출된 쓰기와 해당 유출을 모두 제거합니다.
read-back.
다시 읽기.
This relieves the **L2-bound** stalls (read backpressure / write starvation) and
이는 **L2 바운드** 지연(읽기 역압/쓰기 기아 상태)을 완화하고
cuts total L3 volume.
총 L3 볼륨을 줄입니다.
- **Tiling — IFM re-reads** → fewer/larger tiles reduce conv halo overlap (usually a small pool).
- **타일링 — IFM 다시 읽기** → 타일 수가 적거나 많을수록 변환 후광 중복이 줄어듭니다(일반적으로 작은 풀).
- **Operator fusion / L2 chaining** (distinct from tiling) → fuse producer→consumer so the
- **Operator fusion / L2 chaining** (타일링과 다름) → 퓨즈 생산자 → 소비자이므로
intermediate never spills to L3.
중간은 결코 L3으로 유출되지 않습니다.
Attacks the spill footprint at its source; the `vai-custom-op-candidates`
유출 지점의 유출 발자국을 공격합니다. 'vai-custom-op-candidates'
skill finds these fusion chains.
스킬은 이러한 융합 사슬을 찾습니다.
- **Concat / pseudo-op elimination** → spills are often _forced_ by a successor that is a
- **Concat / pseudo-op 제거** → 유출은 종종 후임자에 의해 _강제_됩니다.
pseudo-op / L3-buffer node / templated graph.
의사 연산/L3 버퍼 노드/템플릿 그래프.
Removing or rewriting that boundary removes the
해당 경계를 제거하거나 다시 작성하면
forced spill (write + read).
강제 유출(쓰기 + 읽기).
- **Batching for weight reuse** → a larger batch amortizes each weight fetch over more samples
- **무게 재사용을 위한 일괄 처리** → 더 큰 배치는 더 많은 샘플에 대해 각 가중치 가져오기를 상각합니다.
(fewer weight reads/sample), at the cost of larger activation residency — trades against spill.
(더 적은 가중치 읽기/샘플), 더 큰 활성화 상주 비용으로 — 유출에 대비한 거래입니다.
- **Precision / weight compression / sparsity** → lower bit-width, weight packing, or pruning
- **정밀도/가중치 압축/희소성** → 낮은 비트 폭, 가중치 패킹 또는 가지치기
cut bytes across IFM/OFM/WTS/spill. **Accuracy-affecting — flag as a model-level decision, not
IFM/OFM/WTS/spill 전체에서 바이트를 줄입니다. **정확도에 영향을 미침 — 플래그가 아닌 모델 수준 결정으로 표시
a pure perf knob.**
순수한 성능 노브.**
- **Tiling tradeoff (state it):** bigger tiles raise L2/L1 pressure and can _cause more spilling_.
- **타일링 트레이드오프(설명):** 타일이 클수록 L2/L1 압력이 높아져 _더 많은 유출이 발생할 수 있습니다_.
When the **L2-bound** side is already hot (read backpressure / write starvation), prefer tiling
**L2-바운드** 측면이 이미 뜨거운 경우(배압 읽기/기아 상태 쓰기) 타일링을 선호합니다.
that improves L2 chaining over tiling that merely maximizes weight reuse.
이는 단지 가중치 재사용을 최대화하는 타일링보다 L2 체인을 개선합니다.
Larger contiguous L3
더 큰 연속 L3
bursts also improve DDR row-buffer locality, easing read starvation even where volume is fixed.
또한 버스트는 DDR 행 버퍼 지역성을 개선하여 볼륨이 고정된 경우에도 읽기 부족 현상을 완화합니다.

### B.
### Hide latency / raise utilization (same bytes, fewer stall cycles)
### 대기 시간 숨기기/사용률 높이기(동일한 바이트, 더 적은 정지 주기)

When starvation is high but MB/s is well under peak (latency-limited, not saturated).
기아 상태는 높지만 MB/s가 최대치보다 훨씬 낮은 경우(지연 시간 제한, 포화 상태 아님)

- **Double-buffering (ping/pong)** — from the step-5 check, convert `single`-buffered buffers on
- **이중 버퍼링(핑/퐁)** — 5단계 확인에서 '단일' 버퍼 버퍼를 다음으로 변환합니다.
the stalling side to ping/pong so the DMA overlaps compute.
DMA가 계산과 겹치도록 정지 측면을 ping/pong으로 지정합니다.
Often the fastest win for a
종종 가장 빠른 승리를 거두는 경우가 많습니다.
latency-limited op (low MB/s), where cutting bytes would not help.
대기 시간이 제한된 작업(낮은 MB/s), 바이트를 줄이는 것은 도움이 되지 않습니다.
List each candidate:
각 후보자를 나열하십시오.
operator, `buf_type` (IFM/WTS for a read-side stall, OFM for a write-side stall), `size`,
연산자, `buf_type`(읽기 측 지연의 경우 IFM/WTS, 쓰기 측 지연의 경우 OFM), `size`,
current `double_buffer` state. **Tradeoff:** ping/pong doubles that buffer's L2 footprint, so it
현재 `double_buffer` 상태. **절충점:** 핑/퐁은 해당 버퍼의 L2 공간을 두 배로 늘립니다.
competes with spill — when L2 is saturated, pair it with L2-chaining/tiling that frees L2, or
유출과 경쟁합니다. L2가 포화되면 L2를 해제하는 L2 체인/타일링과 쌍을 이루거나
apply it on the read side.
읽는 쪽에 적용하세요.
If `force_single_buffer == true`, the buffer is pinned single and
`force_single_buffer == true`인 경우 버퍼는 단일로 고정되고
the underlying L2/spill constraint must be lifted first.
기본 L2/유출 제약 조건을 먼저 해제해야 합니다.
- **Prefetch / schedule overlap** → issue weight/IFM loads ahead of the consuming compute so the
- **프리페치/일정 중복** → 컴퓨팅을 소비하기 전에 가중치/IFM 로드를 발행하여
read latency is hidden behind prior work; a compiler-scheduling lever.
읽기 대기 시간은 이전 작업 뒤에 숨겨져 있습니다. 컴파일러 스케줄링 레버.

### C.
### Spread load across channels (fix imbalance, not arbitration)
### 채널 전체에 로드 분산(중재가 아닌 불균형 수정)

When the per-channel arrays (`get_summary_data`) or per-op `Column:Channel` show some channels
채널별 배열(`get_summary_data`) 또는 작업별 `열:채널`이 일부 채널을 표시하는 경우
saturated while others sit idle.
다른 사람들이 유휴 상태에 있는 동안 포화 상태입니다.

- **Channel / DMA load balancing** → distribute buffers/transfers across all 2R+2W channels per
- **채널/DMA 로드 밸런싱** → 모든 2R+2W 채널에 버퍼/전송 분산
column so aggregate utilization rises without changing QoS priorities.
열을 사용하므로 QoS 우선순위를 변경하지 않고도 집계 활용도가 높아집니다.
Complementary to QoS,
QoS를 보완하며,
which only re-prioritizes _among contending_ channels.
이는 _경합하는_ 채널 중에서만 우선순위를 다시 지정합니다.
- **Partitioning / placement** → how the graph is partitioned across NPU columns decides which
- **분할/배치** → 그래프가 NPU 열에 걸쳐 어떻게 분할되는지에 따라 어떤 열이 결정됩니다.
producer→consumer pairs cross L3 vs stay on-column in L2.
생산자 → 소비자 쌍은 L3을 통과하고 L2의 열에 유지됩니다.
Keeping hot pairs on the same column
동일한 열에 핫 쌍 유지
cuts L3 round-trips.
L3 왕복을 줄입니다.

### D.
### Relieve L2 contention (the L2-bound case)
### L2 경합 완화(L2 바인딩 사례)

When the L2 side is the limiter — **read backpressure** (L2 can't accept, write-lock wait) or
L2 측이 제한자인 경우 — **읽기 역압**(L2는 허용할 수 없음, 쓰기 잠금 대기) 또는
**write starvation** (L2 can't supply, bank conflict / read-lock wait).
**쓰기 기아**(L2가 공급할 수 없음, 뱅크 충돌/읽기 잠금 대기).

- **Bank-conflict / lock mitigation** → change buffer placement/stride/alignment so concurrent
- **뱅크 충돌/잠금 완화** → 버퍼 배치/스트라이드/정렬을 동시에 변경
L2 accesses hit different banks and reduce lock contention.
L2 액세스는 다른 뱅크에 도달하고 잠금 경합을 줄입니다.
Targeted L2-bound fix, separate from
별도의 대상 L2 바인딩 수정
tiling.
타일링.
- **More stamps for under-provisioned L2-bound operators** → a **stamp** = 4×4 = 16 cores across
- **과소 프로비저닝된 L2 바운드 운영자를 위한 추가 스탬프** → **스탬프** = 4×4 = 전체 코어 16개
**4 columns**, and each column has its own L2 memtile, so a stamp gives an operator its own set
**4개 열**, 각 열에는 자체 L2 memtile이 있으므로 스탬프는 운영자에게 자체 세트를 제공합니다.
of L2 IFM/OFM/WTS buffers and parallel DMA write ports.
L2 IFM/OFM/WTS 버퍼 및 병렬 DMA 쓰기 포트.
An L2-bound operator running on **few
**몇 개에서 실행되는 L2 바인딩 연산자
stamps** has few L2 write ports to absorb the DDR stream → backpressure.
스탬프**에는 DDR 스트림 → 역압을 흡수하는 L2 쓰기 포트가 거의 없습니다.
Adding stamps (up to 9)
스탬프 추가(최대 9개)
adds L2 buffers and parallel ports, and finer per-buffer tiling reduces per-buffer bank/lock
L2 버퍼와 병렬 포트를 추가하고, 버퍼당 타일링을 더욱 세밀하게 조정하여 버퍼 뱅크당/잠금을 줄입니다.
contention.
논쟁.
This is usually the **biggest** L2-bound lever.
이는 일반적으로 **가장 큰** L2 바인딩 레버입니다.
  - **How to read it (MCP-only):** stamp count is NOT reliably exposed by the graph tools — do
  - **읽는 방법(MCP에만 해당):** 스탬프 수는 그래프 도구에 의해 안정적으로 노출되지 않습니다.
**not** infer it from the mapped-graph `Tiling` entry count (that is per-tile shapes, not
**그렇지** 매핑된 그래프 '타일링' 항목 수에서 추론하지 마세요(즉, 타일별 모양이지
stamps, and misreads badly).
우표를 찍고 잘못 읽습니다).
Instead use `get_memory_map(level="L2")`: aggregate `size` per
대신 `get_memory_map(level="L2")`을 사용하세요.
`be_layer_name` for the high-stall operator and compare its **total L2 footprint and buffer
높은 정지 연산자에 대한 `be_layer_name`을 사용하고 해당 **총 L2 공간과 버퍼를 비교합니다.
granularity** against a well-behaved (low-stall) neighbor.
선량하게 행동하는(로우 스톨) 이웃에 대한 세분성**.
An L2-bound op with markedly
현저하게 L2 바운드 작업
**smaller total L2 / coarser (fewer, larger) buffers** than its peers is under-stamped.
**동종 제품보다 더 작은 총 L2 / 더 거친(더 적고 더 큰) 버퍼**는 언더스탬프됩니다.
  - **Utilization check:** if columns-per-stamp is known (a stamp spans 4 columns; a Versal
  - **활용도 확인:** 스탬프당 열 수를 알고 있는 경우(스탬프는 4개 열에 걸쳐 있으며 Versal은
AIE-ML memtile is ~512 KB/column → ~2 MB L2 per stamp), compare the op's total L2 against
AIE-ML memtile은 ~512KB/열 → 스탬프당 ~2MB L2입니다. 작업의 총 L2를 다음과 비교합니다.
stamps × ~2 MB.
스탬프 × ~2MB.
Low utilization with high backpressure means the stall is the single stamp's
높은 배압과 낮은 활용도는 스톨이 단일 스탬프임을 의미합니다.
write-port / bank contention, not L2 capacity — so add stamps and/or finer buffers rather than
L2 용량이 아닌 쓰기 포트/뱅크 경합 — 따라서 L2 용량 대신 스탬프 및/또는 더 미세한 버퍼를 추가하십시오.
just enlarging buffers.
버퍼만 늘리면 됩니다.
State the memtile size as external context: the tool returns
외부 컨텍스트로 memtile 크기를 명시합니다. 도구가 반환됩니다.
`partition_memory_size = 0`, so capacity is not measured.
`partition_memory_size = 0`이므로 용량이 측정되지 않습니다.
- **L2 capacity reallocation** → give more memtile L2 to the spilling / highest-stall L2 buffers
- **L2 용량 재할당** → 유출/가장 높은 스톨 L2 버퍼에 더 많은 감성 L2 제공
so they stay resident (and to make room for ping/pong from B).
그래서 그들은 상주 상태를 유지합니다(그리고 B에서 탁구를 위한 공간을 만들기 위해).

### E.
### QoS changes for low-performing channels
### 성능이 낮은 채널에 대한 QoS 변경

For the **DDR-bound** case only — **read starvation** _or_ **write backpressure** (both = the L3
**DDR 바인딩** 경우에만 — **기아 상태 읽기** _또는_ **배압 쓰기**(둘 다 = L3
DDR controller is the limiter).
DDR 컨트롤러는 리미터입니다).

- Identify the low-utilization DDR-bound channels (timeline `Column:Channel` per op, or
- 활용도가 낮은 DDR 바인딩 채널을 식별합니다(작업당 타임라인 '열:채널' 또는
`get_summary_data` per-channel arrays): read channels high in starvation, write channels high in
`get_summary_data` 채널별 배열): 기아 상태가 높은 채널을 읽고, 기아 상태가 높은 채널을 씁니다.
backpressure.
배압.
- Raise the QoS priority/arbitration weight for those NPU-column channels in the Vitis embedded
- Vitis 임베디드의 NPU 열 채널에 대한 QoS 우선순위/중재 가중치를 높입니다.
platform so the DDR controller serves them sooner (supplies reads / accepts writes faster).
플랫폼이므로 DDR 컨트롤러가 더 빨리 서비스를 제공합니다(읽기 제공/쓰기 허용 속도가 더 빨라짐).
- The **L2-bound** cases — read backpressure and write starvation — are **not** QoS candidates;
- **L2 바인딩** 사례(읽기 역압 및 쓰기 기아 상태)는 QoS 후보가 **아닙니다**.
their fix is L2/tiling/bank-conflict (A/D).
해결책은 L2/타일링/뱅크 충돌(A/D)입니다.
Do not aim QoS at them.
QoS를 목표로 삼지 마십시오.
- Respect the platform bytes/sec cap: QoS reallocates bandwidth between channels, so call out
- 플랫폼 바이트/초 한도를 준수합니다. QoS는 채널 간에 대역폭을 재할당하므로
which channels give it up.
어떤 채널이 포기하는지.
If most DDR-bound channels are limited together, QoS alone can't add
대부분의 DDR 바인딩 채널이 함께 제한되는 경우 QoS만으로는 추가할 수 없습니다.
total bandwidth — pair it with L3 volume reduction (A).
총 대역폭 - L3 볼륨 감소(A)와 쌍을 이룹니다.

## Output
## 출력

Lead with a one-line verdict (compute- vs memory-bound overall, and the dominant stall cause per
한 줄로 결론을 내립니다(전체적으로 컴퓨팅 및 메모리 바인딩, 각 작업의 주요 지연 원인).
direction).
방향).
Then: a channel utilization/stall table per captured direction; a per-op-type table
그런 다음: 캡처된 방향별 채널 활용/지속 테이블. 작업 유형별 테이블
(exec µs, GMACs?, read starv %, write bp %, MB/s, verdict) sorted by execution time; for the
(exec µs, GMACs?, read starv %, write bp %, MB/s, verdict) 실행 시간별로 정렬됨 에 대한
top-stall operators, a ping/pong column or note (single vs ping/pong on the stalling-side
상단 실속 운영자, 핑/퐁 열 또는 메모(스톨 측의 단일 대 핑/퐁
buffer); then a prioritized recommendation list drawn from families A–E (pick each via the
버퍼); 그런 다음 제품군 A~E에서 추출한 우선 순위 권장 사항 목록(각각 선택
signal→lever table) led by the highest absolute-stall operators/channels.
신호→레버 테이블)은 가장 높은 절대 실속 연산자/채널이 주도합니다.
Quote concrete numbers
구체적인 숫자를 인용하세요
from the tools; use tables for per-op and per-channel rows.
도구에서; 작업별 및 채널별 행에 대한 테이블을 사용합니다.

## Caveats to always state
## 항상 언급해야 할 주의 사항

- **Backpressure and starvation point at opposite sides depending on direction** — backpressure =
- **방향에 따라 반대쪽에 있는 배압 및 기아 지점** — 배압 =
destination busy, starvation = source can't supply.
대상이 사용 중이고 기아 상태 = 소스를 공급할 수 없습니다.
Read: backpressure = L2, starvation = DDR.
읽기: 배압 = L2, 기아 = DDR.
Write: backpressure = DDR, starvation = L2.
쓰기: 배압 = DDR, 기아 = L2.
Never route a write-backpressure stall to an L2 fix
쓰기 역압 지연을 L2 수정으로 라우팅하지 마십시오.
or a write-starvation stall to QoS (see the direction table in Background).
또는 QoS에 대한 쓰기 부족 지연(백그라운드의 방향 테이블 참조)
- `peak_*_total_ch_*` are **aggregates over all channels** of a direction, not one peak channel.
- `peak_*_total_ch_*`는 하나의 피크 채널이 아닌 특정 방향의 **모든 채널**에 대한 집계입니다.
- Achieved BW from utilization assumes the stated per-channel peak; cross-check against the
- 활용을 통해 달성된 BW는 명시된 채널별 피크를 가정합니다. 에 대해 교차 확인
timeline `(MB/s)` column and flag any large disagreement.
타임라인 `(MB/s)` 열을 선택하고 큰 불일치가 있는 경우 플래그를 지정합니다.
- A single capture is usually one direction — never report the uncaptured direction as
- 단일 캡처는 일반적으로 한 방향입니다. 캡처되지 않은 방향을 다음과 같이 보고하지 마십시오.
stall-free (see "Single-direction captures").
실속이 없습니다("단일 방향 캡처" 참조).
- Stall **percentages** flag intensity; rank real gain by **absolute stall cycles**.
- 스톨 **백분율** 플래그 강도; **절대적 스톨 사이클**을 기준으로 실제 이득의 순위를 매깁니다.
- These are DMA-channel counters.
- 이는 DMA 채널 카운터입니다.
- A DDR-bound stall (read starvation / write backpressure) with
- DDR 바운드 지연(읽기 기아/쓰기 역압)
low MB/s is a DDR **latency/QoS** limit, not necessarily a bandwidth-volume limit — do not
낮은 MB/s는 DDR **지연 시간/QoS** 제한이며 반드시 대역폭-볼륨 제한은 아닙니다.
recommend cutting bytes when the fix is QoS, and vice versa.
수정 사항이 QoS인 경우 바이트를 자르는 것이 좋습니다. 반대의 경우도 마찬가지입니다.
