<!-- ko-translation: enabled -->
---
name: vai-perf-analysis
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
description: Analyze performance of VAIML compiled models. Uses the AI Analyzer SDK
  to extract detailed timing, operator metrics, and performance summaries, then answers
  any question about model performance -- layer timing, custom op cost, bottlenecks,
  partition breakdown, comparisons between runs, and more. Requires a Ryzen AI / Vitis
  AI Python environment with `dlanalyzer` available -- either by sourcing a venv activate
  script (`--t`) or by running inside a pre-provisioned environment such as the official
  Vitis AI Docker container. Use when the user wants to analyze model performance,
  find slow layers, compare compilation runs, understand custom op overhead, or generate
  a performance report.
---
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->

# Performance Analysis Skill
# 성과 분석 기술

## Access command
## 액세스 명령

This skill must always be accessed through the worker, never in a different way.
이 기술은 항상 작업자를 통해 접근해야 하며 다른 방식으로는 접근할 수 없습니다.
Allowed workers: `vai-custom-op-worker`, or the `vai-flag-configuration` skill,
허용되는 작업자: `vai-custom-op-worker` 또는 `vai-flag-configuration` 스킬,
which is its own entry point.
이는 자체 진입점입니다.

## Description
## 설명

Analyze performance of VAIML compiled models using the AI Analyzer SDK
AI 분석기 SDK를 사용하여 VAIML 컴파일 모델의 성능 분석
(`dlanalyzer`) shipped with the Ryzen AI environment.
(`dlanalyzer`)는 Ryzen AI 환경과 함께 제공됩니다.
Can answer any natural
자연스러운 대답을 할 수 있다
language question about model performance -- layer timing, custom op cost,
모델 성능에 관한 언어 질문 - 레이어 타이밍, 사용자 정의 작업 비용,
bottlenecks, partition breakdown, host vs NPU split, and more.
병목 현상, 파티션 분석, 호스트 대 NPU 분할 등.

## Optimization Target (READ FIRST)
## 최적화 목표(먼저 읽어보세요)

**The number to optimize is `average_npu_time_usec` ("NPU Compute").**
**최적화할 숫자는 `average_npu_time_usec`("NPU 컴퓨팅")입니다.**
This is the actual time the NPU spends executing the model per inference.
이는 NPU가 추론당 모델을 실행하는 데 소비하는 실제 시간입니다.
Always report it FIRST and frame the rest of the analysis around it.
항상 이를 먼저 보고하고 이를 중심으로 나머지 분석을 구성하십시오.

Do NOT use the host-side "Kernel Launch Window" (formerly labeled "Kernel
호스트 측 "커널 시작 창"(이전에는 "커널"로 레이블이 지정됨)을 사용하지 마십시오.
Execution") as the optimization target.
실행")을 최적화 대상으로 지정합니다.
That value comes from
그 가치는 다음에서 비롯됩니다.
`get_performance_metrics()` host_profiling and measures the host's view of
`get_performance_metrics()` 호스트_프로파일링을 수행하고 호스트의 관점을 측정합니다.
the launch->completion window, so it includes NPU compute + DMA setup/
시작->완료 창이므로 NPU 컴퓨팅 + DMA 설정/이 포함됩니다.
teardown + driver/runtime sync overhead.
분해 + 드라이버/런타임 동기화 오버헤드.
It is useful for end-to-end
끝까지 유용합니다.
latency accounting, but it is NOT what custom-op or kernel work moves.
대기 시간 계산이지만 맞춤 작업이나 커널 작업이 이동하는 것은 아닙니다.

When you produce a report:
보고서를 생성할 때:
1. Lead with **NPU Compute** (`average_npu_time_usec`) -- this is the
1. **NPU 컴퓨팅**(`average_npu_time_usec`)으로 선두 -- 이것이
target and the first value in the summary.
대상과 요약의 첫 번째 값입니다.
2. Then show **Total inference time** per inference (divide
2. 그런 다음 추론별로 **총 추론 시간**을 표시합니다(나누기
`total_execution_time_us` by `inference_count` if needed) for context.
필요한 경우 `inference_count`로 `total_execution_time_us`)
3. Then the host-profiling breakdown, with the host kernel-window row
3. 그런 다음 호스트 커널 창 행을 사용하여 호스트 프로파일링 분석을 수행합니다.
labeled as a host-side measurement, not as the optimization target.
최적화 대상이 아닌 호스트 측 측정으로 라벨이 지정되었습니다.
4. Express custom-op and per-op costs as a percentage of NPU Compute, not
4. 맞춤형 작업 및 작업별 비용을 NPU 컴퓨팅의 백분율로 표시하지 않고
of the host kernel window.
호스트 커널 창의

## Parse Arguments
## 인수 구문 분석

This skill expects arguments in this form:
이 기술은 다음 형식의 인수를 기대합니다.

```
<run_dir> [question] [--compare <run_dir2>]
```

Extract from `$ARGUMENTS`:
`$ARGUMENTS`에서 추출:
- **run_dir**: **(MANDATORY)** Path to the directory containing profiling
- **run_dir**: **(필수)** 프로파일링이 포함된 디렉터리 경로
outputs (`summary.csv`, `record_timer_*.json`, `partition.json`, etc.).
출력(`summary.csv`, `record_timer_*.json`, `partition.json` 등).
- **question**: Optional natural language question about performance.
- **질문**: 성능에 대한 선택적인 자연어 질문입니다.
If omitted, generate a comprehensive performance summary.
생략된 경우 포괄적인 성능 요약을 생성합니다.
- `--compare <run_dir2>`: Optional second run directory for side-by-side
- `--compare <run_dir2>`: 병렬 실행을 위한 선택적 두 번째 실행 디렉터리
comparison.
비교.

Skill directory: the directory containing this SKILL.md file (referred to as
스킬 디렉터리: 이 SKILL.md 파일이 포함된 디렉터리(
`<skill_dir>` below).
아래 `<skill_dir>`).

**Validation**: `run_dir` must exist and contain at least one of:
**유효성 검사**: `run_dir`이 존재해야 하며 다음 중 하나 이상을 포함해야 합니다.
`summary.csv`, `record_timer_ts.json`, or a `record_timer_inference_*.json`
`summary.csv`, `record_timer_ts.json` 또는 `record_timer_inference_*.json`
file.
파일.
If missing, ask the user to provide the correct path.
누락된 경우 사용자에게 올바른 경로를 제공하도록 요청하세요.

## Script Reference
## 스크립트 참조

All analysis is done via `<skill_dir>/scripts/ai_extract.py`.
모든 분석은 `<skill_dir>/scripts/ai_extract.py`를 통해 수행됩니다.
It requires the Ryzen AI / Vitis AI Python environment to be available (so that `dlanalyzer` and friends are importable).
Ryzen AI / Vitis AI Python 환경을 사용할 수 있어야 합니다('dlanalyzer' 및 친구들을 가져올 수 있도록).

This skill supports two deployment models:
이 기술은 두 가지 배포 모델을 지원합니다.

1. **Sourced venv**: the user passes `--t <activate_path>` and every shell that runs `python3` must `source <activate_path>` first.
1. **Sourced venv**: 사용자는 `--t <activate_path>`를 전달하고 `python3`을 실행하는 모든 셸은 `source <activate_path>`를 먼저 수행해야 합니다.
2. **Pre-provisioned environment** (e.g. official Vitis AI Docker container): the Python packages are already on `PATH` / `PYTHONPATH`.
2. **사전 프로비저닝된 환경**(예: 공식 Vitis AI Docker 컨테이너): Python 패키지는 이미 'PATH' / 'PYTHONPATH'에 있습니다.
2. No `--t` is provided and no sourcing is needed.
2. `--t`가 제공되지 않으며 소싱이 필요하지 않습니다.

Wherever this document shows `<ENV_SETUP>` inside a bash snippet, expand it as follows:
이 문서가 bash 조각 내에서 `<ENV_SETUP>`을 표시할 때마다 다음과 같이 확장합니다.
- If `--t <path>` was provided: replace the `<ENV_SETUP>` line with `source <path> 2>/dev/null &&`.
- `--t <path>`가 제공된 경우: `<ENV_SETUP>` 줄을 `source <path> 2>/dev/null &&`로 바꿉니다.
- Otherwise: omit the `<ENV_SETUP>` token entirely (so the command starts directly with `python3`).
- 그렇지 않은 경우: `<ENV_SETUP>` 토큰을 완전히 생략합니다(따라서 명령은 `python3`으로 직접 시작됩니다).

Do NOT search the filesystem for an activate script if `--t` is absent -- treat its absence as a deliberate signal that the environment is already in place.
`--t`가 없으면 파일 시스템에서 활성화 스크립트를 검색하지 마십시오. 해당 부재가 환경이 이미 설치되어 있다는 의도적인 신호로 간주하십시오.

All commands follow this pattern:
모든 명령은 다음 패턴을 따릅니다.

```bash
bash -c '<ENV_SETUP> python3 <skill_dir>/scripts/ai_extract.py <run_dir> [FLAGS] 2>/dev/null' | grep -v "^2026\|^Loaded ONNX\|^Error getting schema"
```

### Modes
### 모드

| Flags<br>플래그 | Output<br>출력 | Use for<br>용도 |
|-------|--------|---------|
| `--compact --human`<br>`--컴팩트 --인간` | Human-readable summary<br>사람이 읽을 수 있는 요약 | General overview, host vs NPU split<br>일반 개요, 호스트와 NPU 분할 |
| `--top N --human`<br>`--top N --human` | Top N slowest nodes<br>가장 느린 상위 N개 노드 | Finding biggest offenders<br>최대 범죄자 찾기 |
| `--query "PATTERN" --human`<br>`--쿼리 "PATTERN" --human` | Matching nodes by name/type<br>이름/유형별로 노드 일치 | Custom ops, specific layers<br>맞춤형 작업, 특정 레이어 |
| `--compact`<br>`--컴팩트` | Compact JSON<br>컴팩트 JSON | Tool consumption<br>도구 소비 |
| `--top N`<br>`--상단 N` | Top-N JSON<br>상위 N JSON | Tool consumption<br>도구 소비 |
| `--query "PATTERN"`<br>`--쿼리 "패턴"` | Query JSON<br>JSON 쿼리 | Tool consumption<br>도구 소비 |
| Any of above + `-o FILE`<br>위 항목 중 하나 + `-o FILE` | JSON to file<br>파일로 JSON | Save for later or dual output<br>나중에 또는 이중 출력을 위해 저장 |
| `--human -o FILE`<br>`--인간 -o 파일` | Human to stdout + JSON to file<br>Human에서 stdout으로 + JSON에서 파일로 | Both at once<br>둘 다 동시에 |

### Examples
### 예

```bash
# Full summary
python3 ai_extract.py /path/to/run --compact --human

# Top 20 slowest individual NPU nodes
python3 ai_extract.py /path/to/run --top 20 --human

# All custom ops
python3 ai_extract.py /path/to/run --query "mydomain" --human

# A specific custom op by name (substring or regex)
python3 ai_extract.py /path/to/run --query "<op_name>" --human

# Just convolutions
python3 ai_extract.py /path/to/run --query "Conv" --human

# Human to terminal + JSON to file
python3 ai_extract.py /path/to/run --compact --human -o /tmp/perf.json
```

## Workflow
## 작업 흐름

### Step 1: Get the overview
### 1단계: 개요 보기

Run compact mode to get aggregated metrics:
집계된 측정항목을 얻으려면 압축 모드를 실행하세요.

```bash
bash -c '<ENV_SETUP> python3 <skill_dir>/scripts/ai_extract.py <run_dir> --compact --human 2>/dev/null' | grep -v "^2026\|^Loaded ONNX\|^Error getting schema"
```

This gives you: total inference time, NPU time, host profiling breakdown,
이를 통해 총 추론 시간, NPU 시간, 호스트 프로파일링 분석,
per-operator-type NPU breakdown (with custom ops highlighted), GMACS stats.
운영자 유형별 NPU 분석(사용자 정의 작업이 강조 표시됨), GMACS 통계.

### Step 2: Find the biggest offenders (if needed)
### 2단계: 가장 큰 범죄자 찾기(필요한 경우)

```bash
bash -c '<ENV_SETUP> python3 <skill_dir>/scripts/ai_extract.py <run_dir> --top 20 --human 2>/dev/null' | grep -v "^2026\|^Loaded ONNX\|^Error getting schema"
```

Shows the top N slowest individual nodes sorted by cycle count, with
주기 수를 기준으로 정렬된 가장 느린 상위 N개 개별 노드를 표시합니다.
estimated time, plus what percentage of total NPU time they cover.
예상 시간에 해당하는 총 NPU 시간의 몇 퍼센트를 더한 값입니다.

### Step 3: Drill into specifics (if needed)
### 3단계: 세부 사항 자세히 알아보기(필요한 경우)

Use `--query` to filter by operator name or type:
연산자 이름이나 유형별로 필터링하려면 `--query`를 사용하세요.

```bash
bash -c '<ENV_SETUP> python3 <skill_dir>/scripts/ai_extract.py <run_dir> --query "PATTERN" --human 2>/dev/null' | grep -v "^2026\|^Loaded ONNX\|^Error getting schema"
```

Pattern is regex or substring, case-insensitive.
패턴은 정규식 또는 하위 문자열이며 대소문자를 구분하지 않습니다.
Matches against both
둘 다 상대로 경기
the `Name` and `Type` fields.
'이름' 및 '유형' 필드.
Common patterns:
일반적인 패턴:
- `"mydomain"` -- all custom ops
- `"mydomain"` -- 모든 사용자 정의 작업
- `"Conv"` -- convolution layers
- ``Conv"` -- 컨볼루션 레이어
- `"<op_name>"` -- a specific custom op by its registered name
- `"<op_name>"` -- 등록된 이름을 기준으로 한 특정 사용자 정의 작업
- `"Sigmoid\|Tanh"` -- activation functions
- `"Sigmoid\|Tanh"` -- 활성화 함수

### Step 4: Answer the question
### 4단계: 질문에 답하기

Using the extracted data, answer the user's question directly with exact
추출된 데이터를 이용하여 사용자의 질문에 정확한 답변을 직접 제공
numbers and percentages.
숫자와 백분율.

### Comparing two runs
### 두 번의 실행 비교

Extract compact data from both directories separately and compare the
두 디렉토리에서 개별적으로 압축 데이터를 추출하고 비교합니다.
corresponding `operator_metrics` and summary values manually.
해당 'operator_metrics' 및 요약 값을 수동으로 지정합니다.

## Data Format Reference
## 데이터 형식 참조

### operator_metrics (from --compact)
### Operator_metrics(--compact에서)

Each entry has:
각 항목에는 다음이 포함됩니다.
- `category`: `"npu"` or `"host_profiling"`
- `카테고리`: `"npu"` 또는 `"host_profiling"`
- `name`: Operator type (e.g.
- `name`: 연산자 유형(예:
- `"Conv2DBf16"`, `"mydomain.<op_name>"`)
- `"Conv2DBf16"`, `"mydomain.<op_name>"`)
- `count`: Number of instances
- `count`: 인스턴스 수
- `value_usec`: Total aggregated time in microseconds
- `value_usec`: 총 집계 시간(마이크로초)
- `percent`: Percentage of timeline
- `percent`: 타임라인의 백분율
- `cycles_per_count`: Average cycles per invocation
- `cycles_per_count`: 호출당 평균 주기

### timing_data entries (from --query / --top)
### timing_data 항목(--query / --top에서)

Each entry has:
각 항목에는 다음이 포함됩니다.
- `Name`: Instance name (e.g.
- `이름`: 인스턴스 이름(예:
- `"node_Conv_179_Duplicated#1"`)
- `"node_Conv_179_Duplicated#1"`)
- `Type`: Operator type
- `유형`: 연산자 유형
- `Execution Time.Cycles`: Duration in cycles
- `Execution Time.Cycles`: 주기 기간
- `Approx Time.us` (in --top) or computed as `cycles / clock_mhz`
- `Approx Time.us`(--top에서) 또는 `cycles / clock_mhz`로 계산됨

### Common operator types
### 일반적인 연산자 유형

| Type<br>유형 | Description<br>설명 |
|------|-------------|
| `Conv2DBf16`<br>`Conv2DBf16` | 2D Convolution (bf16)<br>2D 컨볼루션(bf16) |
| `ClipBf16` / `ClipBf163D`<br>`ClipBf16` / `ClipBf163D` | ReLU / Clip activation<br>ReLU/클립 활성화 |
| `GemmBfp16`<br>'GemmBfp16' | Fully-connected / GEMM<br>완전 연결/GEMM |
| `SigmoidTemplatedBf163D`<br>`SigmoidTemplateBf163D` | Sigmoid activation<br>시그모이드 활성화 |
| `TanhTemplatedBf163D`<br>'TanhTemplateBf163D' | Tanh activation<br>탄 활성화 |
| `MulBf163D`<br>'MulBf163D' | Element-wise multiply<br>요소별 곱셈 |
| `AddBf163D`<br>`추가Bf163D` | Element-wise add<br>요소별 추가 |
| `Transpose4dAdf`<br>`Transpose4dAdf` | 4D Transpose<br>4D 전치 |
| `ReduceMeanTemplated`<br>'MeanTemplate 줄이기' | Mean reduction<br>평균 감소 |
| `mydomain.*`<br>`mydomain.*` | Custom operator types<br>맞춤 연산자 유형 |
