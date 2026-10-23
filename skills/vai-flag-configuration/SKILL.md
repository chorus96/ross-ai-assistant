<!-- ko-translation: enabled -->
---
name: vai-flag-configuration
description: Iteratively improve the VAIML compilation configuration for an ONNX model.
  Analyzes the model and the flag catalog, proposes a non-default vitisai_config.json,
  compiles it, validates NPU offload (and board inference time when available), and
  loops until the user's stop condition is met. Use when the user wants to tune compiler
  flags, improve NPU offloading, or reduce inference time for a model.
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
---
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->

# Flag-Configuration Skill
# 플래그 구성 스킬

## Description
## 설명

Orchestrates a three-phase loop to improve a model's VAIML configuration:
모델의 VAIML 구성을 개선하기 위해 3단계 루프를 조정합니다.
**(1) analyze & propose flags -> (2) compile -> (3) validate**, repeating until
**(1) 플래그 분석 및 제안 -> (2) 컴파일 -> (3) 검증**, 까지 반복
the user is satisfied or no further improvement is possible.
사용자가 만족하거나 더 이상의 개선이 불가능합니다.
This skill is an
이 스킬은
orchestrator: it reuses existing skills/scripts and owns only the loop, the flag
오케스트레이터: 기존 기술/스크립트를 재사용하고 루프, 플래그만 소유합니다.
catalog, user interaction, and bookkeeping.
카탈로그, 사용자 상호 작용 및 장부.

## Parse Arguments
## 인수 구문 분석

This skill expects arguments in this form:
이 기술은 다음 형식의 인수를 기대합니다.

```
--model <onnx> [--t <activate>] [--device <ep-device-id>] [--vitisai-config <json>] [--board] [--work-dir <dir>] [--spill-workdir <dir>]
```

- `--model <path>`: **(MANDATORY)** ONNX model to optimize.
- `--model <path>`: **(필수)** 최적화할 ONNX 모델입니다.
- `--t <path>`: **(OPTIONAL)** product virtualenv activate script.
- `--t <경로>`: **(선택 사항)** 제품 virtualenv 활성화 스크립트.
- Omit when running
- 실행시 생략
inside the **official Vitis AI Docker image** or another pre-provisioned install
**공식 Vitis AI Docker 이미지** 또는 사전 프로비저닝된 다른 설치 내부
where `flexml`, `onnxruntime`, and `dlanalyzer` already import.
여기서 `flexml`, `onnxruntime` 및 `dlanalyzer`는 이미 가져왔습니다.
Do NOT search the
검색하지 마세요
filesystem for an activate script when `--t` is absent.
`--t`가 없을 때 활성화 스크립트를 위한 파일 시스템.
- `--device <value>`: **(OPTIONAL, no default)** target for `vaiml_config.device`
- `--device <값>`: **(선택 사항, 기본값 없음)** `vaiml_config.device`의 대상
when generating a config.
구성을 생성할 때.
If omitted, take `device` from `--vitisai-config` or
생략된 경우 `--vitsai-config`에서 `device`를 가져오거나
ask the user — never assume `stx` or any other part.
사용자에게 물어보십시오. `stx`나 다른 부분을 가정하지 마십시오.
Use the **Supported Devices
**지원 장치 사용
for Compilation** table in the public Vitis AI docs (values like
공개 Vitis AI 문서의 Compilation** 테이블(예:
`ve2-xc2ve3858`, `ve2-xc2ve3558`, …).
`ve2-xc2ve3858`, `ve2-xc2ve3558`, …).
Some installs also
일부 설치도
-  accept short names (`t50`, `t20`); VAIP may normalize or override the
-  짧은 이름(`t50`, `t20`)을 허용합니다. VAIP는 정규화하거나 재정의할 수 있습니다.
value for the active product context — echo back what the compile actually used.
활성 제품 컨텍스트에 대한 값 - 컴파일에서 실제로 사용한 내용을 다시 반향합니다.
- `--vitisai-config <path>`: **(OPTIONAL)** starting `vitisai_config.json`.
- `--vitisai-config <경로>`: **(선택 사항)** `vitsai_config.json`을 시작합니다.
- When
- 언제
given, **load it and treat every flag already set there as fixed** — carry
주어진 경우 **로드하고 이미 고정된 것으로 설정된 모든 플래그를 처리합니다** — 캐리
them forward into every iteration and only add or tweak flags on top; never
모든 반복에 전달되고 맨 위에 플래그만 추가하거나 조정합니다. 결코
drop, reset, or silently omit a user-provided value unless the user explicitly
사용자가 명시적으로 명시하지 않는 한 사용자가 제공한 값을 삭제, 재설정 또는 자동으로 생략합니다.
asks to remove it.
제거해달라고 요청합니다.
It must include top-level `target` and `targets`; a file
최상위 'target' 및 'targets'를 포함해야 합니다. 파일
containing only `passes` is incomplete and makes VAIP target discovery fail.
'passes'만 포함하면 불완전하며 VAIP 대상 검색이 실패하게 됩니다.
When omitted, generate the complete baseline config shown below.
생략하는 경우 아래 표시된 전체 기준 구성을 생성합니다.
- `--board`: **(OPTIONAL)** a board run is available; enables the inference-time
- `--board`: **(선택 사항)** 보드 실행이 가능합니다. 추론 시간을 활성화합니다
objective.
목표.
- `--work-dir <path>`: **(OPTIONAL, default: ./flag_opt_work)** where iteration
- `--work-dir <path>`: **(선택 사항, 기본값: ./flag_opt_work)** 여기서 반복
artifacts and the ledger are stored.
유물과 원장이 저장됩니다.
- `--spill-workdir <path>`: **(OPTIONAL)** an existing compile output dir that
- `--spill-workdir <path>`: **(선택 사항)** 기존 컴파일 출력 디렉토리
already contains a `DetailedSpillingAnalysis.csv`.
이미 `DetailedSpillingAnalytic.csv`가 포함되어 있습니다.
If given, the L3 spilling
주어진다면, L3 유출
check reads it directly instead of relying on the loop's own compile.
check는 루프 자체 컴파일에 의존하는 대신 직접 읽습니다.

Skill directory: the directory containing this SKILL.md (referred to as
스킬 디렉터리: 이 SKILL.md가 포함된 디렉터리(라고 함)
`<skill_dir>`).
`<skill_dir>`).

## Environment Setup Convention
## 환경 설정 규칙

Wherever a bash snippet shows `<ENV_SETUP>`: if `--t <path>` was given, replace
Bash 조각에 `<ENV_SETUP>`이 표시되는 경우: `--t <path>`가 제공된 경우 교체
it with `source <path> 2>/dev/null &&`; otherwise omit the token.
`source <path> 2>/dev/null &&`; 그렇지 않으면 토큰을 생략합니다.
Never
절대로
auto-detect an activate script.
활성화 스크립트를 자동 감지합니다.

## Transparency: cite sources and explain choices (ALWAYS)
## 투명성: 출처 인용 및 선택 설명(항상)

Every recommendation, proposal, or decision you present MUST include both:
귀하가 제시하는 모든 권장 사항, 제안 또는 결정에는 다음 두 가지가 모두 포함되어야 합니다.

1. **The source material, when available** -- a source the user can actually
1. **사용 가능한 경우 소스 자료** -- 사용자가 실제로 사용할 수 있는 소스
open.
오픈.
For flags, this is `FlagSpec.path` from `scripts/flag_catalog.py`, which
플래그의 경우 `scripts/flag_catalog.py`의 `FlagSpec.path`입니다.
resolves to whatever exists in the current installation: the shipped
현재 설치에 존재하는 모든 항목으로 확인됩니다.
public flag catalog anchor (`fe-args.html#fe-flag-<name>`).
공개 플래그 카탈로그 앵커(`fe-args.html#fe-flag-<name>`).
For validation, cite
검증을 위해 인용
the SDK timing JSON from `scripts/ai_analyzer_timing.py` and the
`scripts/ai_analyzer_timing.py`의 SDK 타이밍 JSON 및
`ai_extract.py` extract from the `vai-perf-analysis` skill (both
`vai-perf-analytic` 스킬에서 `ai_extract.py` 추출(둘 다
call the AI Analyzer SDK — never raw timer JSON or on-disk operator_metrics);
AI 분석기 SDK를 호출합니다. 원시 타이머 JSON이나 온디스크 Operator_metrics는 사용하지 마세요.
for spilling, the `DetailedSpillingAnalysis.csv` path; for the offload gap,
유출의 경우 `DetailedSpillingAnalytic.csv` 경로; 오프로드 격차에 대해
the `*.chained_kernel.tosa.mlir` path.
`*.chained_kernel.tosa.mlir` 경로.
For handoffs, cite the agent or skill
핸드오프의 경우 상담원이나 기술을 인용하세요.
by name.
이름으로.
2. **The rationale for the AI's choice** -- why this flag/iteration/handoff, in
2. **AI 선택의 근거** -- 플래그/반복/인계 이유
plain language, grounded in the cited source (e.g. the flag's `description`,
인용된 소스를 기반으로 한 일반 언어(예: 플래그의 '설명',
the offload/latency numbers, the spill driver).
오프로드/대기 시간 번호, 유출 드라이버).

If no source material exists for a claim, say so explicitly ("no source
주장에 대한 원본 자료가 존재하지 않는 경우 명시적으로 명시하십시오("출처 없음
available; this is a heuristic") rather than presenting it as authoritative.
사용 가능; 이것은 권위 있는 것으로 제시하는 것이 아니라 경험적 방법입니다."
A
proposal without a citation and a reason is INCOMPLETE -- do not present it.
인용문이 없고 이유가 불완전한 제안서 - 제출하지 마세요.

> **Cite only what the user can open.** Prefer, in order: the public Vitis AI
> **사용자가 열 수 있는 것만 인용하세요.** 우선 순서대로: 공개 Vitis AI
> documentation below; `fe-args.html#fe-flag-<name>` via `FlagSpec.path`; files
> 아래 문서; `FlagSpec.path`를 통한 `fe-args.html#fe-flag-<name>`; 파일
> in the user's own work/compile directory; and this skill's own `scripts/`.
> 사용자 자신의 작업/컴파일 디렉토리에 있습니다. 그리고 이 스킬 자체의 `scripts/`가 있습니다.

### Authoritative documentation sources (cite these first)
### 신뢰할 수 있는 문서 소스(먼저 인용)

For any `vaiml_config` / EP-config option, cite the **AMD Vitis AI EP
`vaiml_config` / EP-config 옵션의 경우 **AMD Vitis AI EP를 인용하세요.
Configuration File** doc -- it is the authoritative public source for types,
구성 파일** 문서 - 유형에 대한 신뢰할 수 있는 공개 소스입니다.
supported values, and defaults -- with the specific anchor:
지원되는 값 및 기본값 - 특정 앵커 사용:

- Base URL:
- 기본 URL:
  <https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/ep-config-file.html>
- `optimize_level` (1/2/3; default 2; O3 stacks on `tp_size`): `#optimize-level`
- `optimize_level`(1/2/3; 기본값 2; `tp_size`의 O3 스택): `#optimize-level`
- `dp_size` (data parallelism; 1-6 for T20, 1-9 for T50;
- `dp_size`(데이터 병렬 처리, T20의 경우 1-6, T50의 경우 1-9;
default 1): `#dp-size`
기본값 1): `#dp-size`
- `tp_size` (tensor parallelism; 0-6 / 0-9; default 0 = auto, resolves to 6 on
- `tp_size`(텐서 병렬성; 0-6 / 0-9; 기본값 0 = 자동, 6으로 해석됨
T50): `#tp-size`
T50): `#tp-크기`
- `threshold_gops_percent` (0-100; default 20; ops above the threshold go to the
- `threshold_gops_percent`(0-100; 기본값 20; 임계값을 초과하는 작업은
NPU, below to the CPU): `#threshold-gops-percent`
NPU, 아래 CPU까지): `#threshold-gops-percent`
- `keep_outputs` (bool; default false in the EP doc; **always set true** in this
- `keep_outputs`(bool; EP 문서에서는 기본값이 false입니다. 이 문서에서는 **항상 true로 설정**됩니다.
skill unless the user opts out; retains the vaiml dir for MLIR/spilling/partition
사용자가 옵트아웃하지 않는 한 기술; MLIR/유출/파티션에 대한 vaiml 디렉토리를 유지합니다.
artifacts): `#keep-outputs`
아티팩트): `#keep-outputs`
- `preferred_data_storage` (`auto` | `vectorized` | `unvectorized`; default
- `preferred_data_storage` (`auto` | `벡터화` | `벡터화되지 않음`; 기본값
`auto`): `#preferred-data-storage` — prefer **vectorized** for conv and stencil
`auto`): `#preferred-data-storage` — 전환 및 스텐실에 **벡터화** 선호
reductors (MaxPool, AveragePool, GlobalAveragePool); **unvectorized** for
환원기(MaxPool, AveragePool, GlobalAveragePool); **벡터화되지 않음**
GEMM/MatMul-heavy graphs
GEMM/MatMul이 많은 그래프
- `device` (mandatory; EP doc string, e.g.
- `device`(필수, EP 문서 문자열, 예:
- `ve2-xc2ve3858` for XC2VE3858):
- XC2VE3858의 경우 've2-xc2ve3858'):
`#device` — full list in
`#device` — 전체 목록
[Supported Devices for Compilation](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/compiling.html#supported-devices-for-compilation)
[컴파일 지원 장치](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/compiling.html#supported-devices-for-compilation)

When you propose one of these, cite the doc anchor AND state the value + default
이들 중 하나를 제안할 때 문서 앵커를 인용하고 값 + 기본값을 명시하세요.
from it.
그것으로부터.

## Board timing via the AI Analyzer SDK (never raw timer JSON)
## AI 분석기 SDK를 통한 보드 타이밍(원시 타이머 JSON 아님)

All inference-time numbers come from the **AI Analyzer SDK**, not from hand-parsed
모든 추론 시간 수치는 직접 구문 분석한 것이 아닌 **AI 분석기 SDK**에서 가져옵니다.
`record_timer_*.json`, `board_run.log` medians, or on-disk `operator_metrics`
`record_timer_*.json`, `board_run.log` 중앙값 또는 디스크 상의 `operator_metrics`
exports.
수출.
Raw files lie (schema-2.0 list timers read empty), double-count (summing
원시 파일 거짓말(스키마-2.0 목록 타이머가 비어 있음), 이중 카운트(합계
the timing dataframe), or inflate E2E (`run.py` perf_counter vs board VAIP).
타이밍 데이터 프레임) 또는 E2E를 확장합니다(`run.py` perf_counter 대 보드 VAIP).

**Primary script** (ships with this skill):
**기본 스크립트**(이 기술과 함께 제공됨):

```bash
MPLBACKEND=Agg python3 <skill_dir>/scripts/ai_analyzer_timing.py \
  <board_work_dir> -o <work-dir>/iteration_<i>/sdk_timing.json
```

`measure()` / the CLI returns:
`measure()` / CLI는 다음을 반환합니다.

| Field<br>필드 | Meaning<br>의미 |
|---|---|
| `board_e2e_ms`<br>`board_e2e_ms` | **Primary latency**: the SDK `Inference N` row (host + device), emitted only when `sdk_status` is `OK`<br>**기본 지연 시간**: SDK `추론 N` 행(호스트 + 장치), `sdk_status`가 `OK`인 경우에만 발생 |
| `board_e2e_usec`<br>`board_e2e_usec` | The same E2E in microseconds (`board_e2e_ms * 1000`) — copy it into the ledger as-is<br>마이크로초 단위의 동일한 E2E(`board_e2e_ms * 1000`) - 그대로 원장에 복사합니다. |
| `npu_layer_ms` + `pm_load_ms`<br>`npu_layer_ms` + `pm_load_ms` | Device timeline (kernels + program loads)<br>장치 타임라인(커널 + 프로그램 로드) |
| `npu_device_total_ms`<br>`npu_device_total_ms` | `npu_layer_ms` + `pm_load_ms`<br>`npu_layer_ms` + `pm_load_ms` |
| `cpu_only_ms`<br>`cpu_only_ms` | **E2E − `npu_device_total_ms`** — host + CPU-fallback ops<br>**E2E − `npu_device_total_ms`** — 호스트 + CPU 대체 작업 |
| `cpu_host_rows`<br>`cpu_host_rows` | Top CPU-timeline SDK rows — use before mapping an op-type flag<br>상위 CPU 타임라인 SDK 행 - 작업 유형 플래그를 매핑하기 전에 사용 |
| `vaip_median_ms`, `board_ms`<br>`vaip_median_ms`, `board_ms` | Cross-checks only (VAIP partition timers, `board.log` VART line)<br>교차 확인만 가능(VAIP 파티션 타이머, `board.log` VART 라인) |

The cross-checks are **expected to disagree** with `board_e2e_ms`, and a lower
교차 점검은 `board_e2e_ms`와 **동의하지 않을 것으로 예상**되며, 더 낮은
number there is not a reason to prefer it: `vaip_median_ms` and the EP-side
숫자를 선호할 이유가 없습니다: `vaip_median_ms` 및 EP 측
`vaiml_par_*` row time the partition call, not the inference, so both exclude the
`vaiml_par_*` 행 시간은 추론이 아닌 파티션 호출이므로 둘 다 제외됩니다.
host pre/post work that `Inference N` includes.
'Inference N'에 포함된 사전/사후 작업을 호스팅합니다.

When `npu_device_total_ms` is a small fraction of `board_e2e_ms`, the model is
`npu_device_total_ms`가 `board_e2e_ms`의 작은 부분인 경우 모델은 다음과 같습니다.
host-bound.
호스트 바인딩.
A flat E2E across iterations then does **not** mean the flags did
반복 전반에 걸쳐 평평한 E2E는 플래그가 그랬음을 의미하지 **않습니다**
nothing — it means they moved device time that E2E barely depends on.
아무것도 없습니다. 이는 E2E가 거의 의존하지 않는 장치 시간을 이동했다는 의미입니다.
Judge such
그런 판사
runs on `npu_device_total_ms` and `cpu_host_rows` as well, and say so in the
`npu_device_total_ms` 및 `cpu_host_rows`에서도 실행되며
verdict.
평결.

**Bottleneck op types** (NPU vs host phases): run the sibling
**병목 현상 작업 유형**(NPU 대 호스트 단계): 형제 실행
`ai_extract.py` from the scripts folder of the `vai-perf-analysis` skill on the
'vai-perf-analytic' 스킬의 scripts 폴더에 있는 'ai_extract.py'
same directory.
같은 디렉토리.
It also calls the
그것은 또한
SDK (`get_performance_summary`, `get_performance_summaries`, `get_total_execution_time`, `get_timing_dataframe`).
SDK(`get_performance_summary`, `get_performance_summaries`, `get_total_execution_time`, `get_timing_dataframe`).
Read `operator_metrics`
`operator_metrics` 읽기
from its JSON stdout — **do not open perf JSON already on disk without going
JSON 표준 출력에서 — **이동하지 않고 이미 디스크에 있는 성능 JSON을 열지 마십시오.
through `ai_extract.py`**.
`ai_extract.py`**를 통해.

Five traps the SDK frame punishes (`metrics_from_frame` in `ai_analyzer_timing.py`
SDK 프레임이 처벌하는 5가지 트랩(`ai_analyzer_timing.py`의 `metrics_from_frame`)
handles all five):
다섯 가지를 모두 처리합니다):

1. Do not sum the timing dataframe (total + breakdown → ~N× inflation).
1. 타이밍 데이터프레임을 합산하지 마세요(전체 + 분석 → ~N× 인플레이션).
2. Do not parse `record_timer_*.json` by hand — use the SDK (+ `normalize_timer_json`
2. `record_timer_*.json`을 직접 구문 분석하지 마세요. SDK(+ `normalize_timer_json`을 사용하세요.
when `record_timer_subgraph_cpu_ts.json` is a schema list).
`record_timer_subgraph_cpu_ts.json`이 스키마 목록인 경우).
3. Strip trailing spaces on `Name` (`"Inference 1 "`).
3. `이름`(`"추론 1 "`)에서 후행 공백을 제거합니다.
4. Mean per `inference_id`, not grand-total ÷ layer count.
4. 총합계 ¼ 레이어 개수가 아닌 'inference_id'별 평균입니다.
5. Split `PM Load N` out of kernel time.
5. 커널 시간에서 `PM Load N`을 분할합니다.

Pass the original full board work directory to `ai_analyzer_timing.py`; do not
원래 전체 보드 작업 디렉터리를 `ai_analyzer_timing.py`에 전달합니다. 하지 마십시오
hand-reconstruct a timing directory.
타이밍 디렉토리를 직접 재구성합니다.
The script reads the selected timer
스크립트는 선택한 타이머를 읽습니다.
session's `context_init[].subgraph_path`, keeps that compile cache, and builds a
세션의 `context_init[].subgraph_path`는 해당 컴파일 캐시를 유지하고
lightweight SDK-readable shadow when needed.
필요한 경우 SDK에서 읽을 수 있는 경량 섀도우.
The shadow uses a real directory tree
섀도우는 실제 디렉토리 트리를 사용합니다.
whose payloads are file symlinks, not a symlink to the cache directory as a whole:
페이로드는 캐시 디렉토리 전체에 대한 심볼릭 링크가 아닌 파일 심볼릭 링크입니다.
`dlanalyzer` discovers reports with a non-following recursive walk, so files
`dlanalyzer`는 비추종 재귀적 탐색으로 보고서를 발견하므로 파일
below directory symlinks are invisible and `get_timing_dataframe_batch` returns
아래 디렉토리 심볼릭 링크는 보이지 않으며 `get_timing_dataframe_batch`가 반환됩니다.
`Failed to get reports`.
'보고서를 가져오지 못했습니다'.

Treat `sdk_status != "OK"` as a timing failure to diagnose.
`sdk_status != "OK"`를 타이밍 오류로 간주하여 진단하세요.
In particular, do
특히,
not promote `vaip_median_ms`, raw VAIP timer medians, or `board.log` timing to
`vaip_median_ms`, 원시 VAIP 타이머 중앙값 또는 `board.log` 타이밍을 승격하지 마세요.
the primary SDK result when report discovery fails; fix the input layout and
보고서 검색 실패 시 기본 SDK 결과 입력 레이아웃을 수정하고
rerun this script.
이 스크립트를 다시 실행하세요.

## Flag Relevance Policy (what is worth spending a compile on)
## 플래그 관련성 정책(컴파일할 가치가 있는 항목)

A `flexml-compile` run is slow, so `public: true` in the catalog is necessary but not
`flexml-compile` 실행은 느리므로 카탈로그의 `public: true`가 필요하지만 그렇지 않습니다.
sufficient: a flag is only worth an iteration if it can plausibly change the
충분함: 플래그는 그럴듯하게 변경될 수 있는 경우에만 반복할 가치가 있습니다.
compiled result **for this model**.
**이 모델**에 대해 컴파일된 결과입니다.
`scripts/flag_catalog.py` tags every flag
`scripts/flag_catalog.py`는 모든 플래그에 태그를 지정합니다.
with a `relevance` and a cited `exclusion_reason`:
'관련성' 및 인용된 '제외_이유' 포함:

| Relevance<br>관련성 | Meaning<br>의미 | How to use it<br>사용방법 |
|---|---|---|
| `proposable`<br>'제안 가능' | may change the compiled result<br>컴파일된 결과가 변경될 수 있습니다. | normal candidate<br>일반후보 |
| `needs-consent`<br>'필요 동의' | trades accuracy, or changes the deployment/runtime contract<br>정확성을 거래하거나 배포/런타임 계약을 변경합니다. | never in autonomous mode; in human-in-the-loop, offer it WITH the trade-off stated<br>절대 자율 모드가 아닙니다. 인간 참여 루프에서는 명시된 절충안을 사용하여 제공합니다. |
| `excluded`<br>'제외' | cannot change the compiled result or needs an artifact you cannot synthesize<br>컴파일된 결과를 변경할 수 없거나 합성할 수 없는 아티팩트가 필요함 | never propose<br>절대 프로포즈하지 마세요 |

Use `proposable(catalog, objectives=<objectives>, autonomous=<bool>)` to get the
`proposable(catalog, 목표=<목표>, 자율적=<bool>)`을 사용하여
candidate set, never the raw catalog.
후보 세트는 절대 원시 카탈로그가 아닙니다.
It also drops flags whose
또한 플래그를 떨어뜨립니다.
`goal_conflicts` include a selected objective (for example,
`목표_충돌`에는 선택된 목표가 포함됩니다(예:
`small-tensor-threshold-unwrapping` works against an offload goal).
'small-tensor-threshold-unwrapping'은 오프로드 목표에 대해 작동합니다).

## Graph evidence (propose against the model, not from memory)
## 그래프 증거(기억이 아닌 모델에 대해 제안)

Every proposed flag needs **graph evidence**: something in this model, or in
제안된 모든 플래그에는 **그래프 증거**가 필요합니다. 이 모델 또는
this compile, that the flag acts on.
이 컴파일에서는 플래그가 작동합니다.
The two sources are:
두 가지 소스는 다음과 같습니다.

- **the ONNX graph** (pre-compile) -- `model_ops.op_type_counts`,
- **ONNX 그래프**(사전 컴파일) -- `model_ops.op_type_counts`,
`detect_custom_ops`, `input_shapes` / `output_shapes` / `large_layers`;
`Detect_custom_ops`, `input_shapes` / `output_shapes` / `large_layers`;
- **the compiled graph** (post-compile) -- `offload_gap.offload_gap`, i.e. what
- **컴파일된 그래프**(컴파일 후) -- `offload_gap.offload_gap`, 즉 무엇
the compiler actually failed to place on the NPU.
컴파일러가 실제로 NPU에 배치하지 못했습니다.

A flag with no evidence in either is not a candidate, however promising its
어느 쪽에도 증거가 없는 깃발은 후보가 아니지만,
description sounds.
설명 소리.
Say so rather than proposing it speculatively.
추측으로 제안하기보다는 그렇게 말하십시오.

## Orchestrator-only handoffs (do not invoke leaf skills directly)
## Orchestrator 전용 핸드오프(리프 기술을 직접 호출하지 않음)

The mixed-precision and custom-op **leaf skills** are not entry points of this
혼합 정밀도 및 사용자 정의 작업 **리프 기술**은 이 항목의 진입점이 아닙니다.
orchestrator. **Do not invoke them directly.** Always go through the matching
오케스트레이터. **직접 호출하지 마세요.** 항상 매칭을 진행하세요.
orchestrator skill after printing the experimental-feature warning:
실험적 기능 경고를 인쇄한 후 조정자 기술:

- **The only entry point to the mixed-precision skill is through the
- **혼합 정밀도 기술에 대한 유일한 진입점은
`vai-quantization-guide`**.
`vai-양자화 가이드`**.
That orchestrator may then run quantization /
그런 다음 해당 오케스트레이터는 양자화를 실행할 수 있습니다.
dequantization / patching.
역양자화/패칭.
Never call `vai-quantization-guide`'s leaf skills
`vai-Quantization-guide`의 리프 스킬을 호출하지 마세요.
(`vai-dequantize-model`, `vai-vaip-patching`, or similar) on their own from
(`vai-deQuantize-model`, `vai-vaip-patching` 또는 유사)
this loop. (`vai-fe-args` is island *analysis* only — it is not a
이 루프. (`vai-fe-args`는 섬 *분석*일 뿐입니다.
mixed-precision skill entry point.)
혼합 정밀도 기술 진입점.)
- **The only entry point for the custom-op skill is through the
- **커스텀 작전 스킬의 유일한 진입점은
`vai-custom-op`**.
`vai-custom-op`**.
Never invoke `vai-custom-op-implementation` to create or
생성하거나 생성하기 위해 `vai-custom-op-implementation`을 호출하지 마십시오.
integrate an op except by handing off to that orchestrator.
해당 오케스트레이터에게 전달하는 것을 제외하고 작업을 통합합니다.
Reusing
재사용
`compile.py` from the `vai-custom-op-implementation` skill for a compile in
컴파일을 위한 `vai-custom-op-implementation` 스킬의 `compile.py`
Phase 2 is not a custom-op skill entry; it is a compile helper.
2단계는 맞춤형 작전 기술 항목이 아닙니다. 컴파일 도우미입니다.

## Public configuration surface (the only flags this skill explores)
## 공개 구성 표면(이 기술이 탐색하는 유일한 플래그)

The shipped `fe-args.html` (the public flag catalog under `docs/`) is the
제공된 `fe-args.html`(`docs/` 아래의 공개 플래그 카탈로그)은
authoritative catalog.
권위 있는 카탈로그.
It lists every public frontend flag and the
여기에는 모든 공개 프런트엔드 플래그와
vaiml_config fields below. **Do not propose flags outside this surface.**
아래 viml_config 필드. **이 표면 외부에 깃발을 제안하지 마십시오.**

### Config key aliases (logs and older configs)
### 구성 키 별칭(로그 및 이전 구성)

When reading compiler logs or an existing `vitisai_config.json`, treat these
컴파일러 로그나 기존 `vitsai_config.json`을 읽을 때 다음을 처리하십시오.
pairs as the same option:
동일한 옵션으로 쌍:

| Preferred key (write in new configs)<br>기본 키(새 구성에 쓰기) | Alias (still valid)<br>별칭(아직 유효함) |
|---|---|
| `fe_args`<br>`fe_args` | `fe_experiment`<br>`fe_실험` |
| `group_args`<br>`그룹_인수` | `experiments`<br>'실험' |

A log line may say `fe_experiment args:` even when the JSON used `fe_args`.
JSON이 `fe_args`를 사용한 경우에도 로그 줄에 `fe_experiment args:`가 표시될 수 있습니다.

### vaiml_config fields
### vaiml_config 필드

| Field<br>필드 | Default<br>기본값 | Role<br>역할 | Exploration<br>탐험 |
|---|---|---|---|
| `device`<br>`장치` | *(required — no skill default)*<br>*(필수 — 스킬 기본값 없음)* | Target part; EP doc ID or `--device` / existing config<br>대상부분; EP 문서 ID 또는 `--device` / 기존 구성 | Never — set once at startup<br>안 함 - 시작 시 한 번 설정 |
| `keep_outputs`<br>`출력 유지` | false (EP doc)<br>거짓(EP 문서) | Retain compile artifacts<br>컴파일 아티팩트 유지 | **Always true** unless user opts out<br>**항상 true** 사용자가 선택 해제하지 않는 한 |
| `optimize_level`<br>`최적화_수준` | 2 | O3 for aggressive latency after tp/dp tuning<br>tp/dp 조정 후 공격적인 대기 시간을 위한 O3 | Yes<br>예 |
| `threshold_gops_percent`<br>`임계값_gops_퍼센트` | 20 | Lower → more ops on NPU<br>NPU에서 더 낮은 → 더 많은 작업 | Yes, for offload goal<br>예, 오프로드 목표의 경우 |
| `dp_size`<br>`dp_크기` | 1 | Data parallelism (throughput)<br>데이터 병렬성(처리량) | Yes, for latency goal<br>예, 지연 시간 목표의 경우 |
| `tp_size`<br>`tp_size` | 0 (auto)<br>0(자동) | Tensor parallelism (per-request latency)<br>Tensor 병렬 처리(요청별 대기 시간) | Yes, for latency goal<br>예, 지연 시간 목표의 경우 |
| `preferred_data_storage`<br>`preferred_data_storage` | auto<br>자동 | Data layout: **vectorized** for conv/pool CNNs<br>데이터 레이아웃: 전환/풀 CNN을 위해 **벡터화됨** | Yes, when conv or pool ops dominate<br>예, 전환 또는 풀 운영이 지배적인 경우 |

Also set `enable_f32_to_bf16_conversion` when mixed-precision island analysis
혼합 정밀도 섬 분석 시 `enable_f32_to_bf16_conversion`도 설정합니다.
(from `vai-fe-args`) requires it — it is not a frontend (`fe_args`) flag.
(`vai-fe-args`에서) 이 플래그가 필요합니다. 이는 프런트엔드(`fe_args`) 플래그가 아닙니다.

### Baseline `vitisai_config.json`
### 기준 `vitsai_config.json`

The public [AMD Vitis AI EP configuration
공개 [AMD Vitis AI EP 구성
reference](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/ep-config-file.html)
참조](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/ep-config-file.html)
requires both the pass definitions and the top-level target graph.
패스 정의와 최상위 목표 그래프가 모두 필요합니다.
Every
매
generated iteration config must have top-level `target` and `targets`; never
생성된 반복 구성에는 최상위 'target' 및 'targets'가 있어야 합니다. 결코
write a passes-only file.
패스 전용 파일을 작성합니다.
The `targets[].pass` list names the entries in
`targets[].pass` 목록은 항목의 이름을 지정합니다.
`passes` in execution order.
실행 순서대로 '통과'합니다.

Write each config to `<work-dir>/iteration_<i>/vitisai_config.json`.
각 구성을 `<work-dir>/iteration_<i>/vitsai_config.json`에 작성합니다.
Compilation
편집
runs with `<work-dir>/iteration_<i>` as its current directory and passes the
`<work-dir>/iteration_<i>`를 현재 디렉토리로 사용하여 실행되고
same file as `--vitisai-config vitisai_config.json`.
`--vitsai-config vitisai_config.json`과 동일한 파일입니다.
When the user supplies
사용자가 공급할 때
`--vitisai-config <path>`, copy that complete config into the iteration
`--vitsai-config <path>`, 전체 구성을 반복에 복사합니다.
directory, preserve its target graph and existing settings, then apply only the
디렉터리에 포함하고 해당 대상 그래프와 기존 설정을 보존한 다음
approved iteration changes.
승인된 반복 변경.

Minimal baseline for `ve2-xc2ve3858`:
`ve2-xc2ve3858`에 대한 최소 기준:

```json
{
  "passes": [
    {
      "name": "init",
      "plugin": "vaip-pass_init"
    },
    {
      "name": "vaiml_partition",
      "plugin": "vaip-pass_vaiml_partition",
      "vaiml_config": {
        "device": "ve2-xc2ve3858",
        "keep_outputs": true
      }
    }
  ],
  "target": "VAIML",
  "targets": [
    {
      "name": "VAIML",
      "pass": ["init", "vaiml_partition"]
    }
  ]
}
```

For **frontend (`fe_args`) flags**, read the shipped `fe-args.html` public catalog.
**프런트엔드(`fe_args`) 플래그**의 경우 제공된 `fe-args.html` 공개 카탈로그를 읽어보세요.
Each flag's description, defaults, and optional **`use_case`** routing token live
각 플래그의 설명, 기본값 및 선택적 **`use_case`** 라우팅 토큰이 실시간으로 제공됩니다.
there (YAML `use_case` → catalog `data-use-case` / **Use case** metadata).
거기에 있습니다(YAML `use_case` → 카탈로그 `data-use-case` / **사용 사례** 메타데이터).
Cite `fe-args.html#fe-flag-<name>` via `FlagSpec.path` for mechanism
메커니즘에 대해서는 `FlagSpec.path`를 통해 `fe-args.html#fe-flag-<name>`을 인용하세요.
details — do not duplicate descriptions from memory.
세부정보 — 메모리에서 설명을 복제하지 마세요.

When choosing candidates, map the user's goal and compile evidence to a use
후보자를 선택할 때 사용자의 목표를 매핑하고 용도에 맞는 증거를 수집합니다.
case, then list flags with:
그런 다음 다음을 사용하여 플래그를 나열합니다.

```python
flags_for_use_case("<token>", load_catalog())
```

where `<token>` is one of `push-to-cpu`, `offloading`, `accuracy`, or
여기서 `<token>`은 `push-to-cpu`, `offloading`, `accuracy` 중 하나입니다.
`coverage`.
'취재'.
Read each returned flag's catalog entry before proposing it.
제안하기 전에 반환된 각 플래그의 카탈로그 항목을 읽어보세요.
Only
만
flags with `public: true` in their YAML appear in `fe-args.html` and
YAML에 `public: true`가 있는 플래그는 `fe-args.html`에 표시되며
therefore in `load_catalog()`; if a flag is missing from the catalog, say it is
따라서 `load_catalog()`에서; 카탈로그에 플래그가 없으면 다음과 같이 말합니다.
not on the public surface yet — do not propose it.
아직 대중에게 공개되지 않았습니다. 제안하지 마세요.

### Use-case routing (from the catalog)
### 사용 사례 라우팅(카탈로그에서)

| Use case<br>사용 사례 | When to consider<br>고려해야 할 때 |
|---|---|
| **push-to-cpu**<br>**CPU로 푸시** | Host-side small tensors / edge ops are cheaper on CPU; *conflicts* with max-offload goals<br>호스트 측 소형 텐서/에지 작업은 CPU에서 더 저렴합니다. 최대 오프로드 목표와 *충돌* |
| **offloading**<br>**오프로딩 중** | Close NPU gaps: quantize/dequant edges, conv→gemm rewrites<br>NPU 격차 해소: 에지 양자화/역양자화, 전환→gemm 재작성 |
| **accuracy**<br>**정확성** | User accepts slower kernels for listed ops (`use-accurate-mode` needs consent)<br>사용자는 나열된 작업에 대해 느린 커널을 허용합니다(`use-accurate-mode`에는 동의가 필요함). |
| **coverage**<br>**보도** | User guarantees Gather indices are non-negative (`indices-are-positive` needs consent; compiler name — not `gather-indices-are-positive`)<br>사용자는 Gather 인덱스가 음수가 아님을 보장합니다(`indices-are-긍정적`에는 동의가 필요합니다. 컴파일러 이름은 `gather-indices-are-긍정적`이 아님). |

**Push-to-CPU vs offload objective:** flags tagged `push-to-cpu` carry
**Push-to-CPU 대 오프로드 목표:** 'push-to-CPU' 캐리 태그가 지정된 플래그
`goal_conflicts` with `offload` in `flag_catalog.py` — legitimate for latency /
`flag_catalog.py`의 `offload`가 포함된 `goal_con conflicts` — 대기 시간에 적합함 /
CPU-optimize flows, self-defeating when the objective is maximum NPU offload.
CPU 최적화 흐름은 목표가 최대 NPU 오프로드인 경우 자멸합니다.

## Startup Interview (ask BEFORE the loop)
## 스타트업 인터뷰(루프 전에 물어보세요)

**Always settle this interview first -- even when the skill is invoked explicitly
**항상 이 인터뷰를 먼저 해결하세요. 기술이 명시적으로 호출되는 경우에도 마찬가지입니다.
(`/vai-flag-configuration`) part-way through a conversation.** Do not assume
(`/vai-flag-configuration`) 대화 도중에.** 가정하지 마세요.
goals/mode from *earlier* conversation; the objectives, mode, and stop metrics
*이전* 대화의 목표/모드; 목표, 모드 및 중지 측정항목
must be settled for this run before Phase 0 or any compile.
0단계나 컴파일 전에 이 실행을 정산해야 합니다.

### First, read the answers the user already gave you
### 먼저 사용자가 이미 제공한 답변을 읽어보세요.

Settled is not the same as asked.
정착된 것은 요청한 것과 같지 않습니다.
The invocation itself is an answer: treat
호출 자체가 답변입니다.
anything stated in it as decided, and ask only about what is genuinely missing.
결정된 대로 거기에 명시된 모든 것을 확인하고 실제로 누락된 사항에 대해서만 질문하십시오.
Re-asking a question the user just answered is a bug, not diligence.
사용자가 방금 답변한 질문을 다시 묻는 것은 성실함이 아니라 버그입니다.
For
에 대한
example:
예:

> `/vai-flag-configuration Generate for the model /my/dir/awesome.onnx, targeting`
> `/vai-flag-configuration 모델 /my/dir/awesome.onnx에 대해 생성, 대상 지정`
> `t50, use a temporary dir.
> `t50, 임시 디렉토리를 사용하세요.
> I don't want to have quantization.
> 나는 양자화를 원하지 않습니다.
> The target`
> 목표`
> `should be full offloading.`
> '완전히 오프로드되어야 합니다.'

resolves the model, `--device t50`, the work dir, "no quantization", and an
모델 `--device t50`, 작업 디렉토리, "양자화 없음" 및
`offload` objective with a 100% target — leaving only mode (autonomous vs
100% 목표를 가진 '오프로드' 목표 — 모드만 남기고(자율 대
human-in-the-loop) and the remaining stop metrics to ask about.
Human-In-The-Loop) 및 나머지 중지 측정항목에 대해 물어보세요.
Write
쓰기
`ve2-xc2ve3858` (or the user's exact EP device string) into the
`ve2-xc2ve3858`(또는 사용자의 정확한 EP 장치 문자열)을
`vaiml_partition` pass's `vaiml_config.device` when generating the config.
`vaiml_partition`은 구성을 생성할 때 `vaiml_config.device`를 전달합니다.
If
neither `--device` nor `--vitisai-config` supplies a device, **ask** — do not
`--device`나 `--vitsai-config` 모두 장치를 제공하지 않습니다. **묻기** — 하지 마세요
default.
기본값.

Do this in three steps:
이 작업은 세 단계로 수행됩니다.

1. **Resolve** every setting you can from the invocation and its flags: model,
1. **해결** 호출 및 해당 플래그에서 가능한 모든 설정(모델,
`--device`, `--board`, `--vitisai-config`, `--work-dir`, quantization
`--device`, `--board`, `--vitsai-config`, `--work-dir`, 양자화
yes/no, objectives, target offload, mode, stop metrics.
예/아니요, 목표, 대상 오프로드, 모드, 측정항목 중지.
2. **Echo back** the resolved settings in one short block, marked as taken from
2. **에코백** 해결된 설정을 하나의 짧은 블록으로 표시하고 다음에서 가져온 것으로 표시합니다.
the request, so a misreading is visible before any compile starts.
요청이므로 컴파일이 시작되기 전에 잘못 읽은 부분이 표시됩니다.
3. **Ask only the unresolved ones**, using AskUserQuestion.
3. **해결되지 않은 문제만 질문**하세요. AskUserQuestion을 사용하세요.

Only the *earlier conversation* is off-limits as a source; the invocation is
*이전 대화*만 출처로 삼을 수 없습니다. 호출은
not.
아닙니다.
If the invocation is ambiguous rather than silent (e.g. "make it fast"
호출이 자동이 아닌 모호한 경우(예: "make it fast")
without `--board`), treat it as unresolved and ask.
`--board` 없이) 해결되지 않은 것으로 처리하고 질문하세요.

For any question that remains, ask the user, using AskUserQuestion, in this
궁금한 점이 있으면 AskUserQuestion을 사용하여 사용자에게 이 질문을 하세요.
order:
주문:

1. **Quantize the model first?** If you are about to offer or invoke
1. **모델을 먼저 양자화하시겠습니까?** 제안하거나 호출하려는 경우
quantization, **print this warning to the user first**, then ask:
양자화를 수행하려면 **먼저 이 경고를 사용자에게 인쇄**한 후 다음과 같이 질문하세요.

   > **Warning:** Calling this sub-skill for modification of the input model or creating a custom-op is an experimental feature of the flag-configuration skill.
   > **경고:** 입력 모델을 수정하거나 사용자 정의 작업을 생성하기 위해 이 하위 기술을 호출하는 것은 플래그 구성 기술의 실험적 기능입니다.

If the user wants quantization, the only entry point to the mixed-precision
사용자가 양자화를 원하는 경우 혼합 정밀도에 대한 유일한 진입점
skill is through the `vai-quantization-guide` (shipped alongside this skill in
스킬은 'vai-Quantization-guide'(이 스킬과 함께 제공됨)를 통해 이루어집니다.
the ai_utils bundle).
ai_utils 번들).
Hand the model to that orchestrator; do not invoke the
해당 오케스트레이터에게 모델을 넘겨주세요. 호출하지 마십시오
mixed-precision leaf skills directly.
혼합 정밀도 리프 스킬을 직접적으로 사용합니다.
The orchestrator runs the
오케스트레이터가 실행하는
quantization/dequantization/patching sub-skills, then continue the loop with
양자화/역양자화/하위 기술 패치를 수행한 다음 다음으로 루프를 계속합니다.
the returned quantized model.
반환된 양자화 모델
If declined, use the model as-is.
거부된 경우 모델을 있는 그대로 사용하세요.

2. **Objective(s)** (multi-select; state expectations):
2. **목표**(다중 선택, 상태 기대치):
   - *Compile successfully* (default) -- the default config is expected to
   - *컴파일 성공*(기본값) - 기본 구성은 다음과 같습니다.
compile; use this as the baseline.
컴파일; 이것을 기준으로 사용하십시오.
   - *Improve NPU offloading* -- maximize offloaded ops / minimize CPU fallback.
   - *NPU 오프로딩 개선* -- 오프로드된 작업을 최대화하고 CPU 폴백을 최소화합니다.
   - *Improve inference time* -- offered ONLY when `--board` is set.
   - *추론 시간 향상* -- `--board`가 설정된 경우에만 제공됩니다.
The selected objectives, in priority order, drive `Ledger.best(objectives)`
선택된 목표는 우선순위에 따라 `Ledger.best(objectives)`를 구동합니다.
with tokens `compile`, `offload`, `latency`.
토큰 `compile`, `offload`, `latency`를 사용합니다.
**Board pre-flight for `latency`:** a latency objective can only be measured
**`대기 시간`에 대한 보드 사전 비행:** 대기 시간 목표는 측정만 가능합니다.
by running the model on a board.
보드에서 모델을 실행하여
Before iterating, establish that a board run
반복하기 전에 보드 실행을 설정합니다.
actually works:
실제로 작동합니다:
   - Use the `scripts/run_on_board.py` script provided alongside the skill.
   - 스킬과 함께 제공되는 `scripts/run_on_board.py` 스크립트를 사용하세요.
   - If it is not there, or board access is not configured, STOP and ask the user
   - 거기에 없거나 보드 액세스가 구성되지 않은 경우 중지하고 사용자에게 문의하십시오.
how they run inference on their board (their own script, harness, or
보드에서 추론을 실행하는 방법(자체 스크립트, 하네스 또는
command) plus the connection/target details it needs.
명령)에 필요한 연결/대상 세부 정보를 추가합니다.
Only begin iterating
반복만 시작하세요
once a board run succeeds.
보드 실행이 성공하면.
Never start or mark an iteration 0 "complete"
반복을 시작하거나 0을 "완료"로 표시하지 마십시오.
for a latency goal you cannot measure this run.
대기 시간 목표의 경우 이 실행을 측정할 수 없습니다.
   - If the user cannot provide board access, offer to switch to a
   - 사용자가 보드 액세스를 제공할 수 없는 경우 보드로 전환하겠다고 제안하세요.
compile/offload objective, which is measurable on host.
호스트에서 측정 가능한 컴파일/오프로드 목표.
3. **Mode**:
3. **모드**:
   - *Fully autonomous exploration* -- iterate without stopping for approval.
   - *완전 자율 탐색* -- 승인을 위해 중단하지 않고 반복합니다.
   - *Human-in-the-loop* -- present each proposed flag set and wait for approval
   - *Human-in-the-loop* -- 제안된 각 플래그 세트를 제시하고 승인을 기다립니다.
before compiling.
컴파일하기 전에.
4. **Stop metrics** (suggest defaults, let the user confirm/edit):
4. **측정항목 중지**(기본값 제안, 사용자 확인/수정 가능):
   - NPU offload >= target %, OR
   - NPU 오프로드 >= 목표 %, 또는
   - N iterations with no improvement (default N = 2), OR
   - 개선되지 않은 N 반복(기본값 N = 2) 또는
   - no untried `public: true` flags remain, OR
   - 시도되지 않은 `public: true` 플래그가 남아 있지 않거나
   - user says stop.
   - 사용자가 중지라고 말합니다.
5. **Compile depth** (ask unless the user already chose):
5. **컴파일 깊이**(사용자가 이미 선택하지 않은 경우 질문):
   - *Fast — get_capability* — FE/partition probe only (~minutes, no AIE binary).
   - *빠름 — get_capability* — FE/파티션 프로브 전용(~분, AIE 바이너리 없음).
Use for early iterations, offload-gap discovery, and flag proposals that only
초기 반복, 오프로드 격차 발견 및 플래그 제안에 사용
affect partitioning/FE.
파티셔닝/FE에 영향을 미칩니다.
Run `scripts/compile_probe.py --mode capability`.
`scripts/compile_probe.py --mode Capability`를 실행하세요.
   - *Complete — full compile* — ORT VitisAI compile through AIE (`--mode compile`,
   - *완료 — 전체 컴파일* — AIE를 통한 ORT VitisAI 컴파일(`--mode compile`,
default).
기본값).
Required before board timing, spilling CSV, and final offload proof.
보드 타이밍, CSV 유출 및 최종 오프로드 증명 전에 필요합니다.

Default: **fast** while exploring flags; switch to **complete** before claiming
기본값: 플래그를 탐색하는 동안 **빠름**; 청구하기 전에 **완료**로 전환하세요.
board latency or a final offload number.
보드 대기 시간 또는 최종 오프로드 수.

## Compile depth and output types
## 컴파일 깊이 및 출력 유형

### Two modes this skill runs
### 이 스킬이 실행되는 두 가지 모드

| Mode<br>모드 | Command<br>명령 | Answers<br>답변 | Does NOT produce<br>생산하지 않음 |
|---|---|---|---|
| **Fast (`capability`)**<br>**빠름('능력')** | `compile_probe.py --mode capability`<br>`compile_probe.py --모드 기능` | Whole-model NPU capability, CpuBecause / unsupported ops, partition GOP split, `get_capability_v5.json` when `keep_outputs`<br>전체 모델 NPU 기능, CpuBecause/지원되지 않는 작업, 파티션 GOP 분할, `keep_outputs`인 경우 `get_capability_v5.json` | AIE binary, board-runnable xclbin, spilling CSV<br>AIE 바이너리, 보드 실행 가능 xclbin, CSV 유출 |
| **Complete (`compile`)**<br>**완료(`컴파일`)** | `compile_probe.py --mode compile` (or `compile.py`)<br>`compile_probe.py --mode compile`(또는 `compile.py`) | Everything above plus per-partition AIE compile, `final-vaiml-pass-summary.txt`, spilling artifacts<br>위의 모든 항목과 파티션별 AIE 컴파일, `final-vaiml-pass-summary.txt`, 유출 아티팩트 | Nothing — this is the validation compile<br>아무것도 없습니다. 이것은 검증 컴파일입니다. |

#### How fast mode works in a shipped Vitis AI environment
#### 출시된 Vitis AI 환경에서 빠른 모드가 작동하는 방식

The official **Vitis AI Docker image** and product install ship a Python environment
공식 **Vitis AI Docker 이미지** 및 제품 설치는 Python 환경을 제공합니다.
with the `flexml` package and its native **`pyflexmlcompile`** module already on
`flexml` 패키지와 기본 **`pyflexmlcompile`** 모듈이 이미 켜져 있는 경우
`PYTHONPATH`.
`PYTHONPATH`.
In that environment:
해당 환경에서:

- **`--t` is usually omitted** — same convention as the other Vitis AI skills:
- **`--t`는 일반적으로 생략됩니다** — 다른 Vitis AI 기술과 동일한 규칙:
packages are pre-provisioned inside the container or product venv.
패키지는 컨테이너 또는 제품 Venv 내부에 사전 프로비저닝됩니다.
- **`compile_probe.py --mode capability`** imports `flexml.experimental.ext.dynamic`
- **`compile_probe.py --mode Capability`**는 `flexml.experimental.ext.dynamic`을 가져옵니다.
and calls **`get_capability_v4`** (frontend/partition probe, no AIE binary).
**`get_capability_v4`**를 호출합니다(프런트엔드/파티션 프로브, AIE 바이너리 없음).
- Read **`compile_probe.json`**: when fast mode succeeded,
- **`compile_probe.json`** 읽기: 빠른 모드가 성공하면,
`flexml_lite.available` is `true` and there is **no** `fallback` key.
`flexml_lite.available`은 `true`이고 **no** `fallback` 키가 없습니다.

If `flexml` cannot be imported (for example a minimal venv used only to run
`flexml`을 가져올 수 없는 경우(예: 실행에만 사용되는 최소 venv)
`model_ops.py` with `onnx`), the script sets `fallback` in `compile_probe.json`
`model_ops.py`를 `onnx`로 사용), 스크립트는 `compile_probe.json`에 `fallback`을 설정합니다.
and runs a **full ORT VitisAI compile** instead — check that field; do not treat
대신 **전체 ORT VitisAI 컴파일**을 실행합니다. 해당 필드를 확인하세요. 치료하지 않는다
the iteration as a fast probe.
빠른 프로브로 반복합니다.

**Device strings on the two paths.** The EP config doc uses hyphenated IDs
**두 경로의 장치 문자열.** EP 구성 문서는 하이픈으로 연결된 ID를 사용합니다.
(`ve2-xc2ve3858`, …).
(`ve2-xc2ve3858`, …).
ORT/VAIP accepts those on the **complete compile** path.
ORT/VAIP는 **완전한 컴파일** 경로의 항목을 허용합니다.
The flexml **`get_capability`** API expects flexml device tokens (`ve2`, `stx`, …);
flexml **`get_capability`** API는 flexml 장치 토큰(`ve2`, `stx`, …)을 기대합니다.
`compile_probe.py` normalizes known EP IDs to `ve2` for capability-only calls.
`compile_probe.py`는 기능 전용 호출에 대해 알려진 EP ID를 `ve2`로 정규화합니다.
After any compile, cite the `device` value that landed in the iteration's
컴파일 후에는 반복의 결과에 도달한 `device` 값을 인용하세요.
`vitisai_config.json`.
`vitsai_config.json`.

Always write `compile_probe.json` beside the iteration config:
항상 반복 구성 옆에 `compile_probe.json`을 작성하세요.

```bash
python3 <skill_dir>/scripts/compile_probe.py <model> \
  --vitisai-config <work-dir>/iteration_<i>/vitisai_config.json \
  --mode capability --cache-dir <work-dir>/iteration_<i>/<model_stem> \
  -o <work-dir>/iteration_<i>/compile_probe.json
```

### FlexML `output_type` stages (when driving flexml-compile directly)
### FlexML `output_type` 단계(flexml-compile을 직접 구동하는 경우)

The bundled `compile.py` path always targets a **board-ready** compile (≈ `aie-exe`).
번들 `compile.py` 경로는 항상 **보드 준비** 컴파일(© `aie-exe`)을 대상으로 합니다.
When the user or a handoff uses **flexml-compile** / `flexml.compile()` instead,
사용자 또는 핸드오프가 대신 **flexml-compile** / `flexml.compile()`을 사용하는 경우,
`output_type` chooses how far down the pipeline to stop.
'output_type'은 파이프라인에서 중지할 단계를 선택합니다.
Ordered roughly by depth:
대략적으로 깊이에 따라 정렬됩니다.

| `output_type`<br>`출력_유형` | Stops after<br>다음 시간 이후에 중지 | Useful artifacts<br>유용한 유물 | Typical use<br>일반적인 사용 |
|---|---|---|---|
| `tosa-mlir`<br>'토사밀리르' | TOSA import/lowering<br>TOSA 가져오기/하강 | `*.tosa.mlir`, `unfused.viz.json`<br>`*.tosa.mlir`, `unfused.viz.json` | Import/debug only<br>가져오기/디버그 전용 |
| `frontend-mlir` / `mllib-kernel` / `incore-chaining`<br>`frontend-mlir` / `mllib-kernel` / `incore-chaining` | FE fusion & chaining<br>FE 융합 및 연결 | `fused.viz.json`, `*.chained_kernel.tosa.mlir`<br>`fused.viz.json`, `*.chained_kernel.tosa.mlir` | FE flags, CpuBecause — **same stage as get_capability**<br>FE 플래그, CpuBecause — **get_capability와 동일한 단계** |
| `partition-mlir`<br>`파티션-mlir` | Fail-safe partitioner<br>오류 방지 파티셔너 | `fs.fused.viz.json`, partition MLIR<br>`fs.fused.viz.json`, 파티션 MLIR | Partition-cap debugging<br>파티션 캡 디버깅 |
| `dse-mlir`<br>`dse-mlir` | Tiling / DSE<br>타일링 / DSE | `par.subgraph.dse.mlir`, `schedule.viz.json`<br>`par.subgraph.dse.mlir`, `schedule.viz.json` | TSG / tiling handoffs<br>TSG / 타일링 핸드오프 |
| `aie-graph` / `aie-compiler`<br>`aie-그래프` / `aie-컴파일러` | Backend codegen<br>백엔드 코드 생성 | ADF / compiler IR<br>ADF/컴파일러 IR | Backend-only issues<br>백엔드 관련 문제 |
| `aie-exe`<br>`aie-exe` | Full AIE binary<br>전체 AIE 바이너리 | Runnable partition tree<br>실행 가능한 파티션 트리 | **Complete compile** (default for validation)<br>**컴파일 완료**(검증을 위한 기본값) |

Do not quote a stage the run did not reach — read which files exist under
실행이 도달하지 못한 단계를 인용하지 마십시오. 아래에 어떤 파일이 있는지 읽어보십시오.
`cache/` and cite that path.
`cache/`를 입력하고 해당 경로를 인용하세요.

### Reading failures for the user (error-message usability)
### 사용자에 대한 읽기 실패(오류 메시지 유용성)

On every compile (either mode), read `compile_probe.json` → `failure_report` and
모든 컴파일(두 모드 모두)에서 `compile_probe.json` → `failure_report`를 읽고
**translate it**, do not paste raw logs first:
**번역**, 원시 로그를 먼저 붙여넣지 마세요.

1. **`user_message`** — one paragraph in plain language (what broke, for whom it matters).
1. **`user_message`** — 일반 언어로 된 한 문단(깨진 내용, 누구에게 중요한지).
2. **`actionable`** — concrete next config/flag checks tied to evidence.
2. **`실행 가능`** — 증거에 연결된 구체적인 다음 구성/플래그 확인.
3. **`evidence`** — cited log lines / summary fields the paragraph rests on.
3. **`evidence`** — 단락이 근거하는 인용 로그 줄/요약 필드입니다.

Source priority when building the story:
스토리 구성 시 소스 우선순위:

| Signal<br>신호 | Where<br>어디에 | Usability note<br>유용성 참고 사항 |
|---|---|---|
| Top-level exception<br>최상위 예외 | ORT / compile.py stderr<br>ORT / compile.py 표준 오류 | Often vague ("Failed") — always dig deeper<br>종종 모호함("실패") — 항상 더 깊이 파고들기 |
| `preliminary-vaiml-pass-summary.txt`<br>`예비-vaiml-pass-summary.txt` | `<cache>/cache/`<br>`<캐시>/캐시/` | **Best for threshold/partition story** — GOP % per `vaiml_par_*`<br>**임계값/파티션 스토리에 가장 적합** — `vaiml_par_*`당 GOP % |
| `final-vaiml-pass-summary.txt`<br>`최종-vaiml-pass-summary.txt` | same<br>같은 | Offloaded vs supported GOPs; subgraphs below threshold count<br>오프로드된 GOP와 지원되는 GOP 임계값 수 미만의 하위 그래프 |
| `get_capability_v5.json`<br>`get_capability_v5.json` | under cache when `keep_outputs: true`<br>`keep_outputs: true`일 때 캐시 아래에 있음 | Unsupported op list at original ONNX names<br>원래 ONNX 이름에서 지원되지 않는 작업 목록 |
| `aie_unsupported_original_ops_with_reasons.json`<br>`aie_unsupported_original_ops_with_reasons.json` | FE partition tree<br>FE 파티션 트리 | Per-op reason strings — cite verbatim<br>작업별 이유 문자열 - 그대로 인용 |
| `AIECompiler.log` / `aiecompiler-flexml.log`<br>`AIECompiler.log` / `aiecompiler-flexml.log` | under `vaiml_par_*`<br>`vaiml_par_*` 아래 | Backend failures after silent CPU fallback<br>자동 CPU 대체 후 백엔드 오류 |
| `compile.log` tail<br>`compile.log` 꼬리 | iteration dir<br>반복 디렉토리 | Last ERROR lines<br>마지막 ERROR 줄 |

**Silent failure trap:** ORT can print "Compilation successful" while AIE errors
**자동 실패 트랩:** ORT는 AIE 오류가 발생하는 동안 "컴파일 성공"을 인쇄할 수 있습니다.
force CPU fallback.
강제 CPU 대체.
`compile_probe.py` runs `scan_errors()` from `compile.py` for
`compile_probe.py`는 `compile.py`에서 `scan_errors()`를 실행합니다.
this reason — if `component_scan` evidence is non-empty, the compile **failed** for
이 이유 — `comComponent_scan` 증거가 비어 있지 않으면 컴파일이 **실패했습니다**.
loop purposes even when no exception was thrown.
예외가 발생하지 않은 경우에도 루프 목적.

Never invent a fix the compiler did not state; route `CpuBecause` / unsupported
컴파일러가 명시하지 않은 수정 사항을 절대로 고안하지 마십시오. `CpuBecause` 경로 / 지원되지 않음
reasons through `offload_gap.classify_message` when proposing the next flag.
다음 플래그를 제안할 때 `offload_gap.classify_message`를 통해 이유를 설명합니다.

## Phase 0 -- Pre-Compilation Analysis (HARD GATE before any compile)
## 0단계 - 사전 컴파일 분석(컴파일 전 HARD GATE)

This phase runs ONCE, before the loop, and **must complete and be presented to
이 단계는 루프 이전에 한 번 실행되며 **완료되어 대상에게 제시되어야 합니다.
the user before any compilation is started** -- including a default/baseline
컴파일이 시작되기 전의 사용자** - 기본값/기준 포함
compile.
컴파일.
Do the analysis (Step A), then present the consolidated results
분석(A 단계)을 수행한 후 통합된 결과를 제시합니다.
(Step C). **Do NOT run `compile.py`, suggest a default/baseline compile, or
(단계 C). **`compile.py`를 실행하지 마십시오. 기본/기준 컴파일을 제안하거나
enter the loop until the Step C results have been shown to the user** (and, in
C단계 결과가 사용자에게 표시될 때까지 루프를 입력합니다**(그리고
human-in-the-loop mode, acknowledged).
Human-In-The-Loop 모드, 인정됨).
Only after Step C do you suggest the next
C단계 이후에만 다음 단계를 제안합니다.
step (starting a compile).
단계(컴파일 시작).

### Step A: Operator-type census (pre-compilation)
### A 단계: 운영자 유형 인구조사(사전 컴파일)

Analyze the ONNX model(s) that will be compiled so the first proposal is grounded
첫 번째 제안이 근거가 되도록 컴파일될 ONNX 모델을 분석합니다.
in the model's actual contents:
모델의 실제 내용:

- Census operator types with `model_ops.op_type_counts(<model>)` (see
- `model_ops.op_type_counts(<model>)`를 사용한 인구 조사 연산자 유형(참조
`scripts/model_ops.py`).
`scripts/model_ops.py`).
Report the op-type histogram (e.g. how many Conv,
작업 유형 히스토그램을 보고합니다(예: 전환수,
Gemm, MatMul, Softmax, ...).
Gemm, MatMul, Softmax, ...).
- Match those op types to catalog flags with
- 해당 작업 유형을 다음과 같은 카탈로그 플래그와 일치시킵니다.
`model_ops.match_flags_to_ops(op_counts, load_catalog())`.
`model_ops.match_flags_to_ops(op_counts, load_catalog())`.
This
이
returns `public: true` flags whose name/description target a present op type,
이름/설명이 현재 작업 유형을 대상으로 하는 `public: true` 플래그를 반환합니다.
each with a cited reason (the op type + the flag path).
각각은 인용된 이유가 있습니다(작업 유형 + 플래그 경로).
These are the seed
이것들은 씨앗이다
candidates for Phase 1 iteration 0.
1단계 반복 후보 0.
- Inspect shapes/sizes **before any compile** with `scripts/model_ops.py`:
- `scripts/model_ops.py`를 사용하여 **컴파일하기 전에** 모양/크기를 검사하세요.
`input_shapes(<model>)` (declared graph-input shapes),
`input_shapes(<model>)`(선언된 그래프 입력 모양),
`output_shapes(<model>)` (per-tensor shapes via `onnx.shape_inference`), and
`output_shapes(<model>)`(`onnx.shape_inference`를 통한 텐서별 모양) 및
`large_layers(<model>, min_elements=...)` (nodes whose inferred output tensor
`large_layers(<model>, min_elements=...)`(유추된 출력 텐서가 있는 노드)
is large, biggest first).
큰 것부터 가장 큰 것부터).
This gives structured shape data for latency-
이는 대기 시간에 대한 구조화된 모양 데이터를 제공합니다.
bottleneck reasoning without needing a compile.
컴파일 없이 병목 현상을 추론합니다.
- **Detect custom ops** with `model_ops.detect_custom_ops(<model>)` (any node in
- **사용자 정의 작업 감지**`model_ops.Detect_custom_ops(<model>)`(
a non-standard operator domain).
비표준 운영자 도메인).
If any are found, the generated
발견된 경우 생성된
`vitisai_config.json` MUST carry a `custom_ops` block with an `op_config` entry
`vitisai_config.json`은 `op_config` 항목과 함께 `custom_ops` 블록을 전달해야 합니다.
per custom op, e.g.:
맞춤 작업당, 예:

  ```json
  "custom_ops": {
    "com.amd.quark:TestDummyOp": { "op_config": "<path>/dummy_op.yaml" }
  }
  ```

For the Quark quantization custom ops (`is_quant`: MX6 BFP QDQ,
Quark 양자화 사용자 정의 작업(`is_Quant`: MX6 BFP QDQ,
Extended[De]QuantizeLinear) the `op_config` YAML comes from the quantization
Extended[De]QuantizeLinear) 'op_config' YAML은 양자화에서 비롯됩니다.
flow and must be included or the model will not compile.
흐름에 포함되어야 하며 그렇지 않으면 모델이 컴파일되지 않습니다.
For a general custom
일반적인 관습의 경우
op, the user must supply its `op_config` YAML; authoring that support is out
op, 사용자는 `op_config` YAML을 제공해야 합니다. 지원이 종료된 것으로 작성
of scope for this skill, but the config must still reference it.
이 스킬의 범위는 제한적이지만 구성에서는 이를 참조해야 합니다.
- Present the op-type histogram, the shape summary (inputs + the largest layers),
- 연산 유형 히스토그램, 모양 요약(입력 + 가장 큰 레이어),
the matched flags (with source paths and reasons), and any detected custom ops
일치하는 플래그(소스 경로 및 이유 포함) 및 감지된 사용자 정의 작업
per Transparency.
투명성에 따라.
When Conv or pool ops (MaxPool, AveragePool,
전환 또는 풀 작업(MaxPool, AveragePool,
GlobalAveragePool) dominate the histogram, include
GlobalAveragePool)이 히스토그램을 지배하고 다음을 포함합니다.
`preferred_data_storage=vectorized` as a candidate and cite
`preferred_data_storage=벡터화`를 후보로 지정하고 인용
`#preferred-data-storage`.
`#선호-데이터 저장`.
If a model has an op type with no matching flag, say so; do
모델에 일치하는 플래그가 없는 op 유형이 있는 경우 그렇게 말하세요. 하다
not invent one.
하나도 발명하지 마세요.


### Step C: Present pre-compilation results, THEN suggest next steps (GATE)
### C 단계: 사전 컴파일 결과 제시 후 다음 단계 제안(GATE)

Before any compilation, present a single consolidated **Pre-Compilation Analysis
편집하기 전에 단일 통합 **사전 편집 분석을 제시합니다.
report** to the user containing:
다음 내용을 포함하여 사용자에게 보고**:

- the operator-type histogram (Step A) and the op-type -> flag matches, each with
- 연산자 유형 히스토그램(단계 A)과 op-type -> 플래그가 일치하며 각각 다음과 같습니다.
its cited source path and reason;
인용된 소스 경로 및 이유
- the mixed-precision/topology findings (from `vai-fe-args`);
- 혼합 정밀도/토폴로지 결과(`vai-fe-args`에서)
- the resulting **proposed starting configuration** (the candidate non-default
- 결과 **제안된 시작 구성**(후보는 기본값이 아님)
flags for iteration 0) with per-flag source + rationale.
플래그별 소스 + 이론적 근거가 있는 반복 0에 대한 플래그.

**This report is a gate.** Only AFTER it has been shown to the user do you
**이 보고서는 게이트입니다.** 사용자에게 표시된 후에만
suggest the next step, e.g.:
다음 단계를 제안합니다. 예:

> "Pre-compilation analysis complete (above).
> "사전 컴파일 분석이 완료되었습니다(위).
> Next I can start a **default/
> 다음으로 **default/를 시작할 수 있습니다.
> baseline compile** (no non-default flags) to establish the baseline, or a
> 기준선 컴파일**(기본값이 아닌 플래그 없음)을 사용하여 기준선을 설정하거나
> **first optimized compile** using the proposed configuration.
> 제안된 구성을 사용하여 **최초로 최적화된 컴파일**.
> Which would you
> 당신은 어느 쪽을 원하십니까?
> like?"
> 좋아?"

- In **human-in-the-loop** mode: wait for the user's choice before compiling.
- **human-in-the-loop** 모드: 컴파일하기 전에 사용자의 선택을 기다립니다.
- In **fully autonomous** mode: still print the report first, then state which
- **완전 자율** 모드: 보고서를 먼저 인쇄한 다음 어느 것을 명시할지 지정합니다.
compile you are starting and why, and proceed.
시작하는 이유와 이유를 컴파일하고 계속하십시오.

Never start `compile.py` before this report has been produced.
이 보고서가 생성되기 전에는 `compile.py`를 시작하지 마십시오.

## The Loop
## 루프

Enter the loop only after the Phase 0 gate (Step C) has been presented.
0단계 게이트(C단계)가 제시된 후에만 루프에 들어가십시오.
Maintain a
유지하다
`Ledger` at `<work-dir>/optimization_ledger.json` (see `scripts/ledger.py`).
`<work-dir>/optimization_ledger.json`의 `Ledger`(`scripts/ledger.py` 참조).
For
에 대한
each iteration `i` (starting at 0):
각 반복 `i`(0에서 시작):

### Phase 1 -- Analyze & propose (Skill 1)
### 1단계 - 분석 및 제안(기술 1)

- Build the legal flag catalog with `scripts/flag_catalog.py`
- `scripts/flag_catalog.py`를 사용하여 합법적인 플래그 카탈로그를 구축하세요.
(`load_catalog()`): the vaiml_config tuning fields plus every
(`load_catalog()`): vaiml_config 튜닝 필드와 모든
`fe_args` flag with `public: true`, read from the shipped
`public: true`가 포함된 `fe_args` 플래그, 배송된 파일에서 읽음
`fe-args.html` public catalog.
`fe-args.html` 공개 카탈로그.
The catalog loader
카탈로그 로더
exposes each flag's `default`, so you only ever propose NON-default values.
각 플래그의 '기본값'을 노출하므로 기본값이 아닌 값만 제안합니다.
Then narrow it with `proposable(catalog, objectives=..., autonomous=...)`
그런 다음 `제안 가능(카탈로그, 목표=..., 자율적=...)`으로 범위를 좁힙니다.
per the Flag Relevance Policy -- propose only from that set.
플래그 관련성 정책에 따라 해당 세트에서만 제안합니다.
**Every iteration config MUST set `keep_outputs: true`** unless the user
**모든 반복 구성은 사용자가 아닌 이상 `keep_outputs: true`를 설정해야 합니다.
explicitly opted out — offload-gap and spilling analysis depend on it.
명시적으로 옵트아웃됨 - 오프로드 격차 및 유출 분석이 이에 따라 달라집니다.
- On iteration 0, seed the candidate set from the Phase 0 pre-compilation
- 반복 0에서는 0단계 사전 컴파일에서 설정된 후보를 시드합니다.
analysis (`model_ops.match_flags_to_ops`) so the op types present in the model
분석(`model_ops.match_flags_to_ops`) 모델에 존재하는 작업 유형
drive the first proposal.
첫 번째 제안을 추진합니다.
Analyze the model's topology and reuse the
모델의 토폴로지를 분석하고 재사용
`vai-fe-args` skill for mixed-precision/island reasoning.
혼합 정밀도/섬 추론을 위한 'vai-fe-args' 기술.
- Choose a candidate set of non-default flags not already in the ledger
- 아직 원장에 없는 기본이 아닌 플래그의 후보 세트를 선택합니다.
(`Ledger.has_tried`).
(`Ledger.has_tried`).
Write them into `<work-dir>/iteration_<i>/vitisai_config.json`.
`<work-dir>/iteration_<i>/vitsai_config.json`에 작성하세요.
- **Respect existing flags.** When `--vitisai-config` was supplied, iteration 0
- **기존 플래그를 존중합니다.** `--vitsai-config`가 제공된 경우 반복 0
starts from that file — not from catalog defaults alone.
카탈로그 기본값만으로는 시작되지 않고 해당 파일에서 시작됩니다.
Echo inherited
에코가 상속됨
non-default flags in the Phase 0 report; each later iteration must retain every
0단계 보고서의 기본값이 아닌 플래그 이후의 각 반복에서는 다음을 유지해야 합니다.
flag the user (or a prior iteration) already set unless the user explicitly
사용자가 명시적으로 명시하지 않는 한 이미 설정된 사용자(또는 이전 반복)에 플래그를 지정합니다.
asks to remove it.
제거해달라고 요청합니다.
- **Exploration guardrail.** Do not change many flags at once.
- **탐사 가드레일.** 한 번에 많은 플래그를 변경하지 마세요.
- Propose at most
- 기껏해야 제안
**3 added/changed flags** relative to the current best config
현재 최상의 구성과 관련된 **3개의 추가/변경된 플래그**
(`ledger.flag_delta_count(best_flags, candidate) <= 3`, starting from the
(`ledger.flag_delta_count(best_flags, Candidate) <= 3`,
user's `--vitisai-config` on iteration 0 when provided, otherwise catalog
제공된 경우 반복 0에서 사용자의 `--vitsai-config`, 그렇지 않은 경우 카탈로그
defaults).
기본값).
Widen beyond that budget only with explicit user
명시적인 사용자로만 예산을 초과하여 확대
consent (human-in-the-loop) or an explicit autonomous budget -- this keeps the
동의(인간 참여형) 또는 명시적인 자율 예산 - 이를 통해
search interpretable and avoids exploring the full combinatorial flag space.
검색은 해석 가능하며 전체 조합 플래그 공간 탐색을 방지합니다.
- **Flags are typed, not all boolean.** Each `FlagSpec.type` is one of `bool`,
- **플래그는 모두 부울이 아닌 유형으로 입력됩니다.** 각 `FlagSpec.type`은 `bool` 중 하나입니다.
`int`, `int64_t`, `list[str]`, or `str`.
`int`, `int64_t`, `list[str]` 또는 `str`.
For a non-`bool` flag, propose a
`bool`이 아닌 플래그의 경우 다음을 제안하세요.
concrete valid *value* (not just on/off) and cite the allowed range / values
구체적으로 유효한 *값*(켜기/끄기뿐만 아니라) 허용되는 범위/값을 인용합니다.
from the flag's source doc (`FlagSpec.path`); never propose a bare toggle for a
플래그의 소스 문서(`FlagSpec.path`)에서; 절대 토글만 제안하지 마세요.
numeric or list/string flag.
숫자 또는 목록/문자열 플래그입니다.
- For EACH proposed flag, record a `FlagChange` (`scripts/ledger.py`): its cited
- 제안된 각 플래그에 대해 `FlagChange`(`scripts/ledger.py`)를 기록합니다.
source path (`FlagSpec.path`), the default it deviates from, the value you
소스 경로(`FlagSpec.path`), 기본값에서 벗어나는 값
chose, and a one-line rationale drawn from the flag's `description` and the
플래그의 '설명'과 플래그에서 도출된 한 줄의 근거
graph evidence (see Transparency).
그래프 증거(투명성 참조)
Attach the list to the iteration's
반복에 목록을 첨부하십시오.
`IterationRecord.proposal` -- this is what makes the iteration reconstructible
`IterationRecord.proposal` -- 이것이 반복을 재구성 가능하게 만드는 것입니다.
later.
나중에.
A flag with no citation + reason must not be proposed.
인용 + 이유가 없는 깃발은 제안할 수 없습니다.
- If human-in-the-loop: present the candidate set as a table (flag, source path,
- 인간이 루프에 참여하는 경우: 후보 세트를 테이블(플래그, 소스 경로,
default -> chosen, rationale) and wait for approval; if autonomous: proceed,
기본 -> 선택, 근거) 승인을 기다립니다. 자율적인 경우: 진행,
but still write the table to the iteration dir.
그러나 여전히 반복 디렉토리에 테이블을 작성합니다.

#### Goal-driven proposal strategy
#### 목표 중심 제안 전략

Pick candidate flags based on the selected objective(s).
선택한 목표에 따라 후보 플래그를 선택합니다.
Only propose flags that
다음과 같은 플래그만 제안하세요.
are in the catalog (`public: true`); if the ideal flag for a goal is NOT in the
카탈로그에 있습니다(`public: true`). 목표에 대한 이상적인 플래그가
catalog, surface it to the user with its path and say it is outside the
카탈로그에 경로와 함께 사용자에게 표시하고 카탈로그 외부에 있다고 말합니다.
proposable set rather than silently using it (Transparency).
조용히 사용하기보다는 제안 가능한 세트(투명성).

- **100% NPU offloading (`offload`) -- close the gap, do not tune generally.**
- **100% NPU 오프로딩(`offload`) -- 격차를 줄이고 일반적으로 조정하지 마세요.**
Maximizing offload is a *coverage* problem, not a performance problem: the
오프로드 최대화는 성능 문제가 아니라 *범위* 문제입니다.
only thing that matters is which operators the compiler failed to place on the
중요한 것은 컴파일러가 어떤 연산자를 배치하지 못했는가입니다.
NPU.
NPU.
Generic performance reasoning belongs to the `latency` goal, where board
일반적인 성능 추론은 '지연' 목표에 속합니다.
measurements can actually confirm it.
측정을 통해 실제로 확인할 수 있습니다.
So from iteration 1 onward, **start from
따라서 반복 1부터 **부터 시작하세요.
the offload gap, not from the op histogram**:
작업 히스토그램이 아닌 오프로드 간격**:

  1. Locate the compiled graph with
  1. 다음을 사용하여 컴파일된 그래프를 찾습니다.
`offload_gap.find_chained_kernel_mlir(<compile dir>)` -- the
`offload_gap.find_chained_kernel_mlir(<컴파일 디렉토리>)` --
`*.chained_kernel.tosa.mlir`, written when the compile ran with
`*.chained_kernel.tosa.mlir`, 컴파일이 다음과 같이 실행될 때 작성됨
`keep_outputs: true`.
`keep_outputs: true`.
  2. Run `offload_gap.offload_gap(<mlir>)`.
  2. `offload_gap.offload_gap(<mlir>)`을 실행합니다.
  2. It returns, ranked by node count,
  2. 노드 수에 따라 순위가 매겨져 반환됩니다.
every op type still on the CPU with the compiler's own `CpuBecause`
모든 작업 유형은 여전히 컴파일러 자체 `CpuBecause`를 사용하여 CPU에 있습니다.
`Message` for each (attribute contract:
각각에 대한 `메시지`(속성 계약:
the compiler records the responsible operation in `Op` and its explanation
컴파일러는 'Op'에 해당 작업과 그에 대한 설명을 기록합니다.
in `Message`).
'메시지'에서).
This IS the list of what is missing;
이것은 누락된 항목의 목록입니다.
cite the MLIR path.
MLIR 경로를 인용하세요.
  3. For each gap entry, read `routes` from `offload_gap.classify_message` and
  3. 각 간격 항목에 대해 `offload_gap.classify_message`에서 `routes`를 읽고
follow the returned `action` guidance — cite the MLIR path and the compiler
반환된 '작업' 지침을 따르세요. MLIR 경로와 컴파일러를 인용하세요.
message verbatim.
문자 그대로 메시지.
If the route is `unclassified`, report the message as-is
경로가 '분류되지 않음'인 경우 메시지를 있는 그대로 보고합니다.
and do not invent a cause or fix.
원인을 고안하거나 해결하지 마십시오.
  4. `offload_gap.match_gap_to_flags(gap, proposable(catalog, ...))` turns the
  4. `offload_gap.match_gap_to_flags(gap, proposable(catalog, ...))`는
gap into candidate flags for the residual ops only, ranked so flags naming
잔여 작업에 대해서만 후보 플래그와의 간격, 순위가 매겨져 플래그 이름 지정
a missing op come before generic element-wise matches.
누락된 작업은 일반적인 요소별 일치 앞에 옵니다.
Treat this as a
이것을 다음과 같이 취급하십시오.
**shortlist, not a recommendation**: it is keyword matching over flag
**후보 목록, 추천 아님**: 플래그보다 키워드 일치입니다.
descriptions, so read each candidate's `description` and discard any whose
설명이 있으므로 각 후보자의 '설명'을 읽고 해당 내용을 삭제하세요.
mechanism does not actually apply to the cited op.
메커니즘은 인용된 작업에 실제로 적용되지 않습니다.
Say which you discarded
당신이 버린 것을 말해 보세요
and why.
그리고 왜.

**When the offload target is still not met and CPU ops remain.** After at
**오프로드 목표가 여전히 충족되지 않고 CPU 작업이 유지되는 경우.** 이후
least one complete compile with gap evidence, check whether residual CPU work
갭 증거가 있는 하나 이상의 완전한 컴파일, 잔여 CPU 작동 여부 확인
is beyond what the public flag catalog can fix — for example:
공개 플래그 카탈로그가 수정할 수 있는 범위를 벗어났습니다. 예를 들면 다음과 같습니다.
  - `offload_gap.offload_gap(<mlir>)` still lists ops with no applicable
  - `offload_gap.offload_gap(<mlir>)`은 여전히 적용 가능한 작업이 없는 작업을 나열합니다.
proposable flag, or iterations stall with no offload improvement;
제안 가능한 플래그 또는 오프로드 개선 없이 반복이 중단됩니다.
  - `model_ops.detect_custom_ops(<model>)` finds non-standard-domain nodes or
  - `model_ops.Detect_custom_ops(<model>)`는 비표준 도메인 노드를 찾거나
Quark quant custom ops without a working `custom_ops` entry;
작동하는 `custom_ops` 항목이 없는 Quark 퀀트 사용자 정의 작업;
  - a gap route points at custom-op support (e.g. frontend `[FE]` messages that
  - 맞춤 작업 지원을 가리키는 갭 경로(예: 프런트엔드 `[FE]` 메시지
no catalog flag addresses).
카탈로그 플래그 주소 없음).

**Do not start custom-op work without explicit user consent.** Before you
**명시적인 사용자 동의 없이 맞춤 작업 작업을 시작하지 마세요.**
offer a custom-op handoff, **print this warning to the user**:
맞춤 작업 핸드오프를 제공하려면 **이 경고를 사용자에게 인쇄하세요**:

  > **Warning:** Calling this sub-skill for modification of the input model or creating a custom-op is an experimental feature of the flag-configuration skill.
  > **경고:** 입력 모델을 수정하거나 사용자 정의 작업을 생성하기 위해 이 하위 기술을 호출하는 것은 플래그 구성 기술의 실험적 기능입니다.

The only entry point for the custom-op skill is through the
맞춤형 작전 기술의 유일한 진입점은
`vai-custom-op` — do not invoke custom-op leaf skills directly.
`vai-custom-op` — custom-op 리프 기술을 직접 호출하지 마세요.
Then offer the
그런 다음
orchestrator with AskUserQuestion, citing the remaining gap entries (op type,
AskUserQuestion을 사용하여 나머지 공백 항목(작업 유형,
node count, compiler `Message` from the MLIR).
노드 수, MLIR의 컴파일러 '메시지').
Warn clearly that:
다음과 같이 분명히 경고하십시오.
  - accepting opens a **separate agentic session** via the **`vai-custom-op`**
  - 수락하면 **`vai-custom-op`**을 통해 **별도의 에이전트 세션**이 열립니다.
skill (`vai-custom-op`, shipped alongside this skill in the ai_utils
기술(`vai-custom-op`, ai_utils에서 이 기술과 함께 제공됨)
bundle);
번들);
  - custom-op development is a **complex, multi-phase workflow** (subgraph
  - 맞춤형 개발은 **복잡한 다단계 워크플로우**입니다(하위 그래프
extraction, kernel implementation, compile/simulation) and **may take
추출, 커널 구현, 컴파일/시뮬레이션) 및 **시간이 걸릴 수 있음
substantial time** — it is not another flag iteration;
상당한 시간** - 또 다른 플래그 반복이 아닙니다.
  - the vai-flag-configuration loop pauses until the custom op agentic session returns
  - 사용자 지정 작업 에이전트 세션이 반환될 때까지 vai-flag-configuration 루프가 일시 중지됩니다.
an integrated model/config or the user declines.
통합 모델/구성을 사용하지 않거나 사용자가 거부합니다.

If the user consents, hand off the model, the best `vitisai_config.json` so
사용자가 동의하면 가장 적합한 `vitsai_config.json` 모델을 전달합니다.
far, and the cited gap list to `vai-custom-op`, then resume flag tuning on
멀리, 인용된 간격 목록을 'vai-custom-op'로 지정한 다음 플래그 튜닝을 재개합니다.
the returned artifact.
반환된 유물.
If declined, continue reporting the gap and best
거부된 경우 계속해서 격차를 보고하고 최선의 결과를 얻으십시오.
achievable offload without implying a custom op is in progress.
사용자 지정 작업이 진행 중임을 암시하지 않고 달성 가능한 오프로드입니다.

If no `*.chained_kernel.tosa.mlir` exists yet (before the first compile, or
`*.chained_kernel.tosa.mlir`이 아직 존재하지 않는 경우(첫 번째 컴파일 전, 또는
`keep_outputs` was off), fall back to the Phase-0 op census and custom-op
'keep_outputs'가 꺼짐), 0단계 작업 인구 조사 및 맞춤 작업으로 돌아갑니다.
detection, and label that as pre-compile evidence rather than gap evidence.
탐지하고 이를 갭 증거가 아닌 사전 컴파일 증거로 분류합니다.

Beyond the gap, the one broad lever is **`threshold_gops_percent`**
격차 너머에 있는 하나의 광범위한 레버는 **`threshold_gops_percent`**입니다.
(`vaiml_config`; default 20; range 0-100): ops above the GOPS threshold run on
(`vaiml_config`; 기본값 20; 범위 0-100): GOPS 임계값을 초과하는 작업은 다음에서 실행됩니다.
the NPU, below on the CPU, so **lowering it pushes more ops onto the NPU**.
NPU는 CPU 아래에 있으므로 **이를 낮추면 더 많은 작업이 NPU에 푸시됩니다**.
Cite the doc anchor `#threshold-gops-percent` (value + default) as the reason.
원인으로 문서 앵커 '#threshold-gops-percent'(값 + 기본값)를 인용하세요.
To then enforce/verify the goal, add `full-offloading-required=true`
그런 다음 목표를 시행/확인하려면 'full-offloading-required=true'를 추가하세요.
(cite its catalog entry via `FlagSpec.path`) -- per
(`FlagSpec.path`를 통해 카탈로그 항목 인용) --
its description it makes compilation fail in `get_capability` if the model is
설명에 따르면 모델이 다음과 같은 경우 `get_capability`에서 컴파일이 실패합니다.
not fully offloaded, so it is the gate that proves 100% offload (it does not by
완전히 오프로드되지 않았으므로 100% 오프로드를 증명하는 게이트입니다.
itself increase offload).
자체적으로 오프로드가 증가합니다).
**Do not spend a separate iteration toggling `full-offloading-required` alone**
**'완전 오프로드 필요'를 단독으로 전환하는 데 별도의 반복을 소비하지 마십시오**
-- it changes only the compile exit code, not the graph, so on its own it never
-- 그래프가 아닌 컴파일 종료 코드만 변경하므로 그 자체로는 절대 변경되지 않습니다.
improves offload and just wastes an iteration.
오프로드를 개선하고 반복을 낭비할 뿐입니다.
Apply it alongside
같이 적용해 보세요
offload-increasing flags, or once as a final verification; exclude it from the
오프로드 증가 플래그 또는 최종 확인으로 한 번; 에서 제외하세요.
exploration delta budget.
탐사 델타 예산.
- **Reduce inference time (`latency`).** Run `ai_analyzer_timing.py` first for
- **추론 시간(`latency`)을 줄입니다.** 먼저 `ai_analyzer_timing.py`를 실행합니다.
E2E / NPU / CPU split, then `ai_extract.py --compact` of the
E2E/NPU/CPU 분할 후 `ai_extract.py --compact`
`vai-perf-analysis` skill
'vai-perf-analyse' 스킬
for SDK `operator_metrics`.
SDK `operator_metrics`의 경우.
Read `cpu_host_rows` and the host_profiling rows
`cpu_host_rows` 및 host_profiling 행 읽기
from the extract before mapping an op type — if a host phase dominates, check
작업 유형을 매핑하기 전에 추출에서 — 호스트 단계가 지배적인 경우 확인
*Host-side quantization* first.
*호스트 측 양자화* 먼저.
Then map the dominant **NPU** op type to a flag:
그런 다음 주요 **NPU** 작업 유형을 플래그에 매핑합니다.
  - **Conv-dominated** -> propose converting convs to GEMM:
  - **전환 중심** -> 전환을 GEMM으로 전환 제안:
`enable-conv-to-im2col-and-gemm=1`
`enable-conv-to-im2col-and-gemm=1`
(cite its catalog entry via `FlagSpec.path`)
(`FlagSpec.path`를 통해 카탈로그 항목을 인용하세요)
and/or `prefer-conv-to-gemm-conversion=true`
및/또는 `prefer-conv-to-gemm-conversion=true`
(likewise cited via `FlagSpec.path`),
(마찬가지로 `FlagSpec.path`를 통해 인용됨)
optionally tuning `conv-to-im2col-and-gemm-max-kernel-taps`.
선택적으로 'conv-to-im2col-and-gemm-max-kernel-taps'를 조정합니다.
When the op census shows Conv or pool ops (MaxPool, AveragePool,
운영 인구조사에 전환 또는 풀 운영(MaxPool, AveragePool,
GlobalAveragePool), also consider **`preferred_data_storage=vectorized`**
GlobalAveragePool), **`preferred_data_storage=벡터화`**도 고려하세요.
(cite `#preferred-data-storage`) — vectorized layout suits conv and stencil
(`#preferred-data-storage` 인용) — 벡터화된 레이아웃은 변환 및 스텐실에 적합합니다.
reductors; leave at `auto` or try `unvectorized` only for GEMM/MatMul-heavy
환원제; 'auto'로 두거나 GEMM/MatMul이 많은 경우에만 'unVectorized'를 시도하세요.
graphs.
그래프.
  - **When `tp_size` tuning is exhausted:** add **`optimize_level=3`** (O3), which
  - **`tp_size` 조정이 소진된 경우:** **`optimize_level=3`**(O3)을 추가하세요.
the doc says applies more aggressive latency optimizations *stacked on top of*
문서에 따르면 보다 공격적인 대기 시간 최적화를 *위에 쌓아서* 적용한다고 합니다.
any `tp_size`.
모든 `tp_size`.
Cite `#optimize-level` (EA option; use after standard paths).
'#optimize-level'(EA 옵션, 표준 경로 다음에 사용)을 인용하세요.
  - Always cite the bottleneck source (`sdk_timing.json` + `ai_extract` host/NPU
  - 항상 병목 현상 소스(`sdk_timing.json` + `ai_extract` 호스트/NPU를 인용하세요)
rows with numbers) as the reason for the mapping.
숫자가 있는 행)을 매핑 이유로 사용합니다.
- **Compile successfully (`compile`).** Prefer conservative, broadly-safe
- **성공적으로 컴파일됩니다(`compile`).** 보수적이고 광범위하게 안전한 것을 선호합니다
`public: true` flags; on a failure, use the Phase 3 error to constrain the next
`공개: true` 플래그; 실패 시 3단계 오류를 사용하여 다음 오류를 제한합니다.
proposal.
제안.

#### Host-side quantization (check before mapping an op type)
#### 호스트 측 양자화(작업 유형을 매핑하기 전에 확인)

The NPU op-type breakdown ranks only the work the NPU did, so it cannot show a
NPU 작업 유형 분류는 NPU가 수행한 작업에만 순위를 매기므로 다음을 표시할 수 없습니다.
`QuantizeLinear` that the compiler left on the CPU.
컴파일러가 CPU에 남긴 'QuantizeLinear'입니다.
A model can therefore look
따라서 모델은 다음을 볼 수 있습니다.
conv-dominated while the host spends longer quantizing the input than the whole
호스트가 전체보다 입력을 양자화하는 데 더 오랜 시간을 소비하는 동안 전환이 지배적입니다.
NPU partition takes, and every flag in the list above would be aimed at the
NPU 파티션은 위 목록의 모든 플래그를 목표로 합니다.
wrong thing.
잘못된 것.

**Detect it from two independent signals**, never from GOP share or raw timer files:
**GOP 공유 또는 원시 타이머 파일이 아닌 두 개의 독립적인 신호에서 감지**:

- **SDK host timeline** — `ai_analyzer_timing.py` `cpu_host_rows`, or
- **SDK 호스트 타임라인** — `ai_analyzer_timing.py` `cpu_host_rows` 또는
`ai_extract.py` `operator_metrics` with `category: host_profiling`, especially
특히 `카테고리: 호스트 프로파일링`이 포함된 `ai_extract.py` `operator_metrics`
`Input Quantization and Transformation` as a large share of inference;
추론의 큰 비중을 차지하는 '입력 양자화 및 변환';
- the offload gap — `offload_gap.offload_gap(<mlir>)`, or the compiler's own
- 오프로드 간격 — `offload_gap.offload_gap(<mlir>)` 또는 컴파일러 자체
`<cache>/cache/graph_partition_trace.csv`, still lists a `QuantizeLinear` on
`<cache>/cache/graph_partition_trace.csv`, 여전히 `QuantizeLinear`가 나열됩니다.
an input edge.
입력 가장자리.
Its GOP share is negligible, which is exactly why this gets
GOP 점유율은 미미합니다. 이것이 바로 이것이 얻는 이유입니다.
overlooked.
간과.

**The lever is `match-power-of-two-quant-kernel=1`.** Per its catalog entry it
**레버는 `match-power-of-two-Quant-kernel=1`입니다.** 카탈로그 항목에 따르면
matches the QuantizeLinear kernel for a BF16/F16 -> int8 quantize, which lets
BF16/F16 -> int8 양자화에 대한 QuantizeLinear 커널과 일치합니다.
the compiler place that quantize on the NPU while keeping the same float input
동일한 부동 입력을 유지하면서 NPU에서 양자화하는 컴파일러 위치
contract.
계약.

**It is in the public catalog** (`public: true`), but only when the
**공개 카탈로그에 있습니다**(`public: true`).
input quantize meets the preconditions below is it worth a compile.
입력 퀀타이즈는 아래 전제 조건을 충족하므로 컴파일할 가치가 있습니다.
Prefer it
선호한다
when the host-quantization signals fire; cite
호스트 양자화 신호가 실행될 때; 인용문
`fe-args.html#fe-flag-match-power-of-two-quant-kernel` and verify the scale/zero_point
`fe-args.html#fe-flag-match-power-of-two-Quant-kernel` 및 scale/zero_point 확인
before proposing it.
제안하기 전에.

**Check its two preconditions before spending a compile.** The kernel matches
**컴파일을 하기 전에 두 가지 전제 조건을 확인하세요.** 커널이 일치합니다.
only a quantize whose **scale is a power of two** and whose **zero_point is
**규모가 2의 거듭제곱**이고 **zero_point가 다음과 같은 양자화만 가능합니다.
zero**.
제로**.
Both are readable off the `QuantizeLinear` initializers in the ONNX
둘 다 ONNX의 `QuantizeLinear` 이니셜라이저에서 읽을 수 있습니다.
graph, so verify them there first -- Quark symmetric-int8 models typically
그래프를 먼저 확인하세요. -- Quark symmetric-int8 모델은 일반적으로
satisfy both.
둘 다 만족시키세요.
If either fails the flag cannot fire, and saying so is more useful
둘 중 하나라도 실패하면 깃발을 발사할 수 없으며 그렇게 말하는 것이 더 유용합니다.
than spending an iteration to discover it.
그것을 발견하기 위해 반복하는 것보다.

**Prove the arm actually did something.** An arm whose pattern matched nothing
**팔이 실제로 뭔가를 했다는 것을 증명하세요.** 패턴이 아무것도 일치하지 않는 팔
is indistinguishable from the baseline, so confirm all three before crediting
기준선과 구별할 수 없으므로 크레딧을 부여하기 전에 세 가지를 모두 확인하세요.
the flag:
깃발:

1. the compile resolved it -- the log shows `match-power-of-two-quant-kernel`
1. 컴파일로 문제가 해결되었습니다. 로그에 'match-power-of-two-Quant-kernel'이 표시됩니다.
as true, with no unknown-option warning naming it;
이름을 알 수 없는 옵션 경고 없이 true로 지정합니다.
2. the quantize moved -- the partition trace places it in a partition instead of
2. 퀀타이즈가 이동되었습니다. 파티션 추적은 이를 대신 파티션에 배치합니다.
leaving it unsupported, and the offloaded-operator count rises;
지원되지 않는 상태로 두면 오프로드된 운영자 수가 증가합니다.
3. the numerics did not move.
3. 숫자는 움직이지 않았습니다.
3. The flag relocates the quantize rather than
3. 플래그는 퀀타이즈 대신에 퀀타이즈를 재배치합니다.
approximating it, so a changed output is a bug to report, not a trade-off to
이를 근사화하므로 변경된 출력은 보고해야 할 버그이지 트레이드오프가 아닙니다.
accept.
받아들입니다.

### Phase 2 -- Compile (Skill 2)
### 2단계 - 컴파일(스킬 2)

Run compilation in a SUB-AGENT.
SUB-AGENT에서 컴파일을 실행합니다.
Use `compile_probe.py` so failures land in
실패가 발생하도록 `compile_probe.py`를 사용하십시오.
`compile_probe.json` with a usable `failure_report`:
사용 가능한 `failure_report`가 포함된 `compile_probe.json`:

```
Use the Agent tool (subagent_type="general-purpose") with a prompt like:
"Run this and report compile_probe.json status + failure_report.user_message:
bash -c '<ENV_SETUP> cd <work-dir>/iteration_<i> && \
  python3 <skill_dir>/scripts/compile_probe.py <model> \
  --vitisai-config vitisai_config.json \
  --mode capability|compile \
  --cache-dir <model_stem> -o compile_probe.json'"
```

Use **`--mode capability`** for exploration iterations unless the user chose
사용자가 선택하지 않는 한 탐색 반복을 위해 **`-모드 기능`**을 사용하십시오.
complete-only or you need spilling/board artifacts.
완전하거나 유출/보드 아티팩트가 필요합니다.
Switch to **`--mode compile`**
**`--모드 컴파일`**로 전환
before the final validation iteration.
최종 검증 반복 전에.

Capture: `compile_probe.json` (`status`, `failure_report`), and the cache directory.
캡처: `compile_probe.json`(`status`, `failure_report`) 및 캐시 디렉터리.

On failure, quote `failure_report.user_message` and `actionable` to the user before
실패 시 `failure_report.user_message` 및 `actionable`을 사용자에게 인용하기 전에
proposing the next flags — cite `evidence` lines, not undigested log walls.
다음 플래그 제안 — 소화되지 않은 통나무 벽이 아닌 '증거' 줄을 인용합니다.

### Phase 3 -- Validate & debug (Skill 3)
### 3단계 -- 검증 및 디버그(기술 3)

- **First, check whether the compile succeeded.** On compile failure, diagnose
- **먼저 컴파일 성공 여부를 확인합니다.** 컴파일 실패 시 진단
the error and derive a constraint for Phase 1 (e.g. avoid/adjust the offending
오류를 파악하고 1단계에 대한 제약 조건을 도출합니다(예: 위반 사항 방지/조정).
flag).
플래그).
Do not blindly retry, and do not run timing extraction on a failed compile.
맹목적으로 재시도하지 말고, 실패한 컴파일에 대해 타이밍 추출을 실행하지 마세요.
- On success, run `ai_extract.py --compact` of the `vai-perf-analysis` skill on the run
- 성공하면 실행 중인 `vai-perf-analytic` 스킬의 `ai_extract.py --compact`를 실행합니다.
dir for NPU offload % and SDK-backed `operator_metrics` (never read on-disk perf
NPU 오프로드 % 및 SDK 지원 `operator_metrics`에 대한 dir(디스크 성능을 읽지 않음)
JSON without going through this script).
이 스크립트를 거치지 않고 JSON).
- If `--board`, run `scripts/ai_analyzer_timing.py` on the board work directory
- `--board`인 경우 보드 작업 디렉터리에서 `scripts/ai_analyzer_timing.py`를 실행합니다.
established in pre-flight (same tree as the compile cache + `record_timer_*.json`).
비행 전 설정됩니다(컴파일 캐시와 동일한 트리 + `record_timer_*.json`).
Store `sdk_timing.json` beside the iteration. **Do not** use `run.py` wall-clock
반복 옆에 `sdk_timing.json`을 저장합니다. **run.py` 벽시계를 사용하지 마세요**
median or hand-parsed VAIP JSON as board E2E — the script applies the SDK + VAIP
보드 E2E로 중앙값 또는 수동으로 구문 분석된 VAIP JSON — 스크립트는 SDK + VAIP를 적용합니다.
rules from above.
위에서부터 규칙.
- Append an `IterationRecord` to the ledger -- carrying the `proposal` built in
- 내장된 `제안`을 포함하는 `IterationRecord`를 원장에 추가합니다.
Phase 1 -- and `Ledger.save()`.
1단계 -- 그리고 `Ledger.save()`.
The ledger scores `latency` on
원장은 '대기 시간'을 점수로 매깁니다.
`IterationRecord.board_inference_usec` (microseconds).
`IterationRecord.board_inference_usec`(마이크로초).
Copy it straight from
에서 바로 복사하세요
the `board_e2e_usec` field of `sdk_timing.json`, which the script already
스크립트가 이미 지정한 `sdk_timing.json`의 `board_e2e_usec` 필드
converted: `board_inference_usec = sdk_timing["board_e2e_usec"]`.
변환됨: `board_inference_usec = sdk_timing["board_e2e_usec"]`.
Do not
하지 마십시오
convert by hand and do not store the millisecond `board_e2e_ms` value in the
직접 변환하고 밀리초 `board_e2e_ms` 값을 저장하지 마세요.
microsecond field.
마이크로초 필드.

- Evaluate termination NOW, so its metrics are stored with the iteration that
- 지금 종료를 평가하여 메트릭이 반복과 함께 저장되도록 합니다.
produced them:
생산 :
`record.stop_check = Ledger.evaluate_stop(objectives, criteria).as_dict()`,
`record.stop_check = Ledger.evaluate_stop(목표, 기준).as_dict()`,
where `criteria` is the `StopCriteria` agreed in the Startup Interview
여기서 'criteria'는 스타트업 인터뷰에서 합의한 'StopCriteria'입니다.
(`target_offload_pct`, `max_no_improvement`, `max_iterations`, and
(`target_offload_pct`, `max_no_improvement`, `max_iterations` 및
`untried_candidates` = how many proposable, untried flags are left).
`untried_candidates` = 제안 가능하고 시도되지 않은 플래그가 몇 개 남아 있는지).
- Write the iteration's **result next to its config** with
- 반복의 **결과를 해당 구성 옆에** 다음과 같이 작성합니다.
`ledger.write_iteration_result("<work-dir>/iteration_<i>", record, objectives)`.
`ledger.write_iteration_result("<work-dir>/iteration_<i>", 레코드, 목표)`.
The resulting `result.json` holds the full picture of the iteration in one
결과 `result.json`은 반복의 전체 그림을 한 번에 보유합니다.
place -- the change **proposal** (each `FlagChange` with its source path,
place - 변경 **제안**(소스 경로가 포함된 각 `FlagChange`,
default -> value and rationale), the **config** that was compiled, the
기본값 -> 값 및 근거), 컴파일된 **config**,
**results** (offload %, npu/board time, verdict) and the **`stop_check`**
**결과**(오프로드 %, npu/보드 시간, 판정) 및 **`stop_check`**
metrics the termination decision was made on (`improvement`,
종료 결정이 내려진 지표 (`개선`,
`no_improvement_streak`, `target_gap`, `target_reached`, ...).
`no_improvement_streak`, `target_gap`, `target_reached`, ...).
This keeps each
이는 각각의
iteration dir self-describing, so a run can be re-evaluated later without
반복 디렉터리는 자체 설명되므로 나중에 실행을 재평가할 필요 없이 실행을 다시 평가할 수 있습니다.
replaying the agent log.
에이전트 로그를 재생합니다.

**L3 spilling check (Skill 3 extension).** Spill state is only visible from
**L3 유출 확인(스킬 3 확장).** 유출 상태는 다음에서만 볼 수 있습니다.
compile artifacts, so run this whenever a `DetailedSpillingAnalysis.csv` is
아티팩트를 컴파일하므로 `DetailedSpillingAnalytic.csv`가 있을 때마다 이를 실행하세요.
available -- either from `--spill-workdir`, or from this iteration's compile
사용 가능 - `--spill-workdir` 또는 이번 반복의 컴파일에서 사용 가능
(which must run with `keep_outputs: true` and `enable_cache_file_io_in_mem: "0"`
(`keep_outputs: true` 및 `enable_cache_file_io_in_mem: "0"`으로 실행해야 함
so the report lands on disk):
보고서가 디스크에 저장됩니다.)

- Locate the report with `spill_check.find_spilling_csv(<dir>)`.
- `spill_check.find_spilling_csv(<dir>)`를 사용하여 보고서를 찾습니다.
- `spill_check.load_spilling_layers(csv)` lists layers with `Spill or not == Yes`
- `spill_check.load_spilling_layers(csv)`는 `Spill or not == Yes`로 레이어를 나열합니다.
(column D); `spill_check.fm_driven_spills(csv)` keeps the FM-driven ones -- those
(D 열); `spill_check.fm_driven_spills(csv)`는 FM 기반 항목을 유지합니다.
whose `Layer Spill Reason` (column W) is an FM-size L2 overflow (marker
'Layer Spill Reason'(열 W)은 FM 크기 L2 오버플로(마커)입니다.
`l2FM Size Overflow`).
`l2FM 크기 오버플로`).
This uses the backend report directly instead of
이는 대신 백엔드 보고서를 직접 사용합니다.
reimplementing the L2-layout math, so no device/L2-capacity heuristic is needed.
L2 레이아웃 수학을 다시 구현하므로 장치/L2 용량 휴리스틱이 필요하지 않습니다.
- If FM-driven spilling layers exist, report them with their `Layer Spill Reason`
- FM 기반 유출 레이어가 있는 경우 'Layer Spill Reason'으로 보고합니다.
and note that spatial tiling of the OFM may reduce L3 spill — cite the CSV path.
OFM의 공간 타일링은 L3 유출을 줄일 수 있습니다. CSV 경로를 인용하세요.
- Layers that spill for any other reason (weights-driven, successor-forced,
- 다른 이유로 인해 유출되는 레이어(가중치 기반, 후계자 강제,
concat, ...) are reported separately.
concat, ...)은 별도로 보고됩니다.
- **No `DetailedSpillingAnalysis.csv` available** (e.g. pre-compile, or the
- **사용 가능한 `DetailedSpillingAnalytic.csv`가 없습니다**(예: 사전 컴파일 또는
report was not emitted): fall back to the pre-compile proxy
보고서가 내보내지지 않음): 사전 컴파일 프록시로 대체
`model_ops.estimate_fm_spill_candidates(<model>)`, which flags output feature
출력 기능에 플래그를 지정하는 `model_ops.estimate_fm_spill_candidates(<model>)`
maps whose estimated footprint exceeds L2 (512 KiB/column) from ONNX shape
예상 공간이 ONNX 모양의 L2(512KiB/열)를 초과하는 맵
inference alone. **Label any resulting recommendation clearly as a heuristic
추론만으로는. **결과 추천에 휴리스틱으로 명확하게 라벨을 지정하세요.
estimate, not a fact from the spilling report** — it ignores tiling/layout and
유출된 보고서의 사실이 아닌 추정치** — 타일링/레이아웃을 무시하고
can over- or under-report.
과대 또는 과소보고될 수 있습니다.
- A `DetailedSpillingAnalysis.csv` that lacks the `Layer Spill Reason` column
- 'Layer Spill Reason' 열이 없는 'DetailedSpillingAnalytic.csv'
still cannot be classified: state that FM-driven spill state is unknown.
여전히 분류할 수 없습니다. FM으로 인한 유출 상태를 알 수 없다고 명시합니다.

### Decide
### 결정하다

Compare against `Ledger.best(objectives)`.
`Ledger.best(objectives)`와 비교해보세요.
The stop metrics are applied by
중지 측정항목은 다음과 같이 적용됩니다.
`Ledger.evaluate_stop(objectives, criteria)` (already called in Phase 3): stop
`Ledger.evaluate_stop(objectives,criteria)`(이미 3단계에서 호출됨): 중지
when `should_stop` is true, and state which `reasons` fired
'should_stop'이 true이고 'reasons'가 실행된 상태인 경우
(`target-reached`, `no-improvement-limit`, `max-iterations`,
(`목표 도달`, `개선 제한 없음`, `최대 반복`,
`no-untried-flags`, `all-failed`) together with the metrics behind them.
'no-untried-flags', 'all-failed')와 그 뒤에 있는 측정항목이 함께 표시됩니다.
Do not
하지 마십시오
re-derive "did this improve?" by eye -- quote the recorded numbers, so the
"이것이 개선되었나요?"를 다시 파생시키세요. 눈으로 -- 녹음된 숫자를 인용하여
decision and the evidence for it cannot drift apart.
결정과 그에 대한 증거는 서로 분리될 수 없습니다.
In human-in-the-loop mode,
Human-In-The-Loop 모드에서는
ask whether to continue.
계속할지 물어보세요.
A ledger of tried flag combinations prevents repeats.
시도된 플래그 조합의 원장은 반복을 방지합니다.

**Offload target not reached (CPU ops remain).** When `offload` is an objective
**오프로드 목표에 도달하지 못했습니다(CPU 작업이 유지됨).** '오프로드'가 목표인 경우
and `Ledger.evaluate_stop` shows the target is not met (or `target_gap` > 0)
그리고 `Ledger.evaluate_stop`은 목표가 충족되지 않았음을 보여줍니다(또는 `target_gap` > 0).
while `offload_gap.offload_gap(<mlir>)` still lists CPU-resident ops, summarize
`offload_gap.offload_gap(<mlir>)`은 여전히 CPU 상주 작업을 나열하지만, 요약
the residual gap before proposing more flag iterations.
더 많은 플래그 반복을 제안하기 전의 잔여 간격입니다.
If the remaining ops are
남은 작업이 다음과 같은 경우
not addressable by untried proposable flags, **offer the custom-op path** (see
시도되지 않은 제안 가능한 플래그로 주소를 지정할 수 없는 경우 **맞춤 작업 경로를 제공**(참조
the offload goal section): print the experimental-feature warning first, then
오프로드 목표 섹션): 실험적 기능 경고를 먼저 인쇄한 다음
ask for consent, warn that `vai-custom-op` opens another agentic session and
동의를 요청하고 'vai-custom-op'이 다른 에이전트 세션을 열 것이라고 경고하고
may take substantial time, and only proceed on an explicit yes.
상당한 시간이 걸릴 수 있으며 명시적으로 '예'라고 응답한 경우에만 진행됩니다.
Do not spawn
스폰되지 않음
the custom ops worker autonomously.
Custom Ops 작업자가 자율적으로 수행합니다.

**Bail-out (baseline not compileable).** If the baseline config and every
**Bail-out(기준선 컴파일 불가).** 기준선 구성 및 모든
goal-relevant flag combination fail to compile -- `Ledger.all_failed()` once the
목표 관련 플래그 조합이 컴파일에 실패합니다 -- `Ledger.all_failed()`가 한 번
op/goal-relevant candidates are exhausted, an unsupported operator whose
작전/목표 관련 후보가 소진되고, 지원되지 않는 연산자
unwrapping is broken, or every combination forces full CPU offload -- STOP and
래핑 해제가 중단되거나 모든 조합으로 인해 전체 CPU 오프로드가 강제됩니다. -- 중지 및
report that **compilation fails under all relevant flag combinations**.
**모든 관련 플래그 조합에서 컴파일이 실패**한다고 보고합니다.
Do not
하지 마십시오
churn through irrelevant combinations.
관련 없는 조합을 휘젓습니다.
For unsupported operators, report the
지원되지 않는 운영자의 경우 다음을 보고하세요.
op name and compiler reason from the partition output or
파티션 출력의 작업 이름 및 컴파일러 이유 또는
`model_ops.detect_custom_ops`, and stop — do not prune the ONNX graph to force
`model_ops.Detect_custom_ops` 및 중지 — 강제로 ONNX 그래프를 정리하지 마세요.
a compile.
컴파일.
**Never reduce or prune the original ONNX to a compileable sub-network to force a
**원본 ONNX를 컴파일 가능한 하위 네트워크로 축소하거나 잘라내지 마십시오.
pass** -- that changes what is being compiled and hides the real gap.
pass** - 컴파일되는 내용을 변경하고 실제 차이를 숨깁니다.

## Final Report
## 최종 보고서

When the loop stops (including on bail-out):
루프가 중지된 경우(긴급 구제 포함):
1. Print `Ledger.summary_table(objectives)` (its `delta` column is the
1. `Ledger.summary_table(objectives)`를 인쇄합니다(`delta` 열은
per-iteration gain) and write the machine-readable run summary with
반복당 이득)을 사용하여 기계가 읽을 수 있는 실행 요약을 작성합니다.
`Ledger.write_summary("<work-dir>/optimization_summary.json", objectives)`.
`Ledger.write_summary("<work-dir>/optimization_summary.json", 목표)`.
This is produced **even when no configuration compiled** (`status:
**구성이 컴파일되지 않은 경우에도** 생성됩니다(`상태:
compile-failed`, `best: null`), so a failed run still yields structured output
compile-failed`, `best: null`), 따라서 실패한 실행은 여전히 구조화된 출력을 생성합니다.
of every attempt and its failure verdict for later re-evaluation or debugging.
이후의 재평가 또는 디버깅을 위한 모든 시도 및 실패 판정에 대한 정보입니다.
2. If a config compiled, copy the best iteration's `vitisai_config.json` to
2. 구성이 컴파일된 경우 최상의 반복의 `vitsai_config.json`을 복사하여
`<work-dir>/best_vitisai_config.json` and report its flags + metrics, each
`<work-dir>/best_vitsai_config.json` 및 해당 플래그 + 측정항목을 각각 보고합니다.
flag cited to its source path with the rationale it was chosen (Transparency).
선택한 근거(투명성)와 함께 소스 경로에 인용된 플래그입니다.
On a `compile-failed` run there is no best config: state the bail-out verdict
'컴파일 실패' 실행에는 최상의 구성이 없습니다. 구제 조치 평결을 명시하세요.
(compilation fails under all relevant flag combinations) and cite the failing
(모든 관련 플래그 조합에서 컴파일이 실패함) 그리고 실패한 내용을 인용합니다.
iterations' verdicts from the summary instead.
대신 요약의 반복 결과를 확인하세요.
3. **Evaluate whether each selected goal was achieved.** For every objective the
3. **선택한 각 목표가 달성되었는지 평가합니다.** 모든 목표에 대해
user chose at startup, state ACHIEVED / PARTIALLY / NOT ACHIEVED with the
사용자가 시작 시 ACHIEVED / PARTIALLY / NOT ACHIEVED 상태를 선택했습니다.
concrete cited metric from the best iteration:
최고의 반복에서 얻은 구체적인 인용 지표:
   - `offload` -> the NPU offload % (and, if `full-offloading-required=true` was
   - `offload` -> NPU 오프로드 %(`full-offloading-required=true`인 경우)
used, whether the compile passed = 100% offload proven, or failed = not
사용 여부(컴파일 통과 여부 = 100% 오프로드 입증됨, 실패 여부 = 아님)
fully offloadable).
완전히 오프로드 가능).
Cite the perf/partition source.
성능/파티션 소스를 인용하세요.
When NOT ACHIEVED or
달성되지 않은 경우 또는
PARTIALLY and CPU ops remain, cite `offload_gap.offload_gap(<mlir>)` and
부분적으로 CPU 작업은 그대로 유지됩니다. 'offload_gap.offload_gap(<mlir>)'을 인용하고
state whether the user was offered (or accepted) a **`vai-custom-op`**
사용자가 **`vai-custom-op`**을 제안(또는 수락)했는지 여부를 명시합니다.
handoff for the residual gap.
잔여 공백에 대한 핸드오프.
   - `latency` -> best `board_e2e_ms` from `sdk_timing.json` vs iteration-0
   - `latency` -> `sdk_timing.json`과 iteration-0의 가장 좋은 `board_e2e_ms`
baseline (absolute and %), plus dominant host/NPU row from the SDK extract.
기준선(절대 및 %)에 SDK 추출의 주요 호스트/NPU 행을 추가합니다.
Cite `sdk_timing.json` and the `ai_extract` output path.
`sdk_timing.json` 및 `ai_extract` 출력 경로를 인용하세요.
   - `compile` -> whether a compiling config was found.
   - `compile` -> 컴파일 구성이 발견되었는지 여부.
   - Cite the compile log.
   - 컴파일 로그를 인용하세요.
4. State why the loop stopped, citing the concrete stop metric and the ledger
4. 구체적인 중지 측정항목과 원장을 인용하여 루프가 중지된 이유를 설명합니다.
evidence.
증거.
Both are recorded: the summary's `final_stop_check` holds the
둘 다 기록됩니다. 요약의 `final_stop_check`에
`reasons` and the metrics they fired on, and `summary_table()`'s `delta`
`이유`와 그에 따른 측정항목, `summary_table()`의 `델타`
column shows which iterations improved or regressed and by how much.
열에는 개선되거나 회귀된 반복과 그 정도가 표시됩니다.
Quote
견적
those rather than recomputing them.
다시 계산하는 대신
If quantization was triggered during the
양자화가 도중에 트리거된 경우
run, cite the agent path and the source signal.
실행하고 에이전트 경로와 소스 신호를 인용합니다.
If a custom-op handoff was
사용자 지정 작업 핸드오프가 다음과 같은 경우
triggered, cite the `vai-custom-op` skill and whether it completed.
트리거되면 'vai-custom-op' 스킬과 완료 여부를 언급합니다.
5. If the user asks why a particular flag was never tried, answer from
5. 사용자가 특정 플래그가 시도되지 않은 이유를 묻는 경우
`excluded_flags(catalog)` -- its `relevance` and cited `exclusion_reason` --
`excluded_flags(catalog)` -- `관련성` 및 인용된 `제외_이유` --
or from the graph evidence it lacked. "It was not in the candidate set" alone
또는 그래프 증거에서 부족했습니다. "후보세트에 없었다" 혼자
is not an answer.
답변이 아닙니다.

## Non-goals
## 논골

No functional/numeric correctness checking, no new compiler, no new perf
기능/숫자 정확성 검사 없음, 새 컴파일러 없음, 새 성능 없음
extractor, no environment variables, and nothing the Flag Relevance Policy marks
추출기, 환경 변수 없음, 플래그 관련성 정책 표시 없음
`excluded`.
'제외'.
Because this skill does not check numerics, the accuracy-trading
이 스킬은 숫자를 확인하지 않기 때문에 정확도-트레이딩이
flags are `needs-consent` rather than `proposable` -- never enable one on the
플래그는 '제안 가능'이 아닌 '필요 동의'입니다.
user's behalf in autonomous mode.
자율 모드에서 사용자를 대신합니다.
Never reduce or prune the original ONNX to a
원래 ONNX를 축소하거나 잘라내지 마십시오.
compileable sub-network to force a pass -- if nothing compiles, bail out and
통과를 강제하기 위한 컴파일 가능한 하위 네트워크 - 아무것도 컴파일되지 않으면 구제 조치를 취하고
report (see Decide -> Bail-out).
보고합니다(결정 -> 구제금융 참조).

Do not infer a cause the compiler did not state.
컴파일러가 명시하지 않은 원인을 추론하지 마십시오.
`offload_gap` routes only the
`offload_gap`은 다음만 라우팅합니다.
`CpuBecause` messages that can be cited to the code emitting them; anything else
메시지를 내보내는 코드에 인용될 수 있는 'CpuBecause' 메시지. 다른 것
is `unclassified` and must be reported verbatim.
'미분류'이므로 문자 그대로 보고해야 합니다.
