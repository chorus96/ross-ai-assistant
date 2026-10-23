<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# vai-flag-configuration
# vai 플래그 구성
:external_skills/flag_configuration/skills/flag-configuration/README.md
:external_skills/flag_configuration/skills/flag-configuration/README.md

Orchestration skill that iterates propose-flags -> compile -> validate to improve
제안 플래그 -> 컴파일 -> 검증을 반복하여 개선하는 오케스트레이션 기술
a model's VAIML configuration.
모델의 VAIML 구성.
It is its own entry point -- start it by name;
이는 자체 진입점입니다. 이름으로 시작합니다.
see `SKILL.md` for the procedure.
절차는 `SKILL.md`를 참조하세요.

## Layout
## 레이아웃
- `SKILL.md` -- orchestrator instructions.
- `SKILL.md` -- 오케스트레이터 지침.
- `scripts/flag_catalog.py` -- enumerates legal non-default flags
- `scripts/flag_catalog.py` -- 기본값이 아닌 합법적인 플래그를 열거합니다.
(vitisai_config fields + ``fe_args`` frontend flags with ``public: true``) and applies the
(vitisai_config 필드 + ``public: true``를 포함한 ``fe_args`` 프런트엔드 플래그)
relevance policy: flags that cannot change the compiled result, need an
관련성 정책: 컴파일된 결과를 변경할 수 없는 플래그,
artifact the skill cannot synthesize, are deprecated, trade accuracy,
스킬이 합성할 수 없는 아티팩트, 더 이상 사용되지 않음, 거래 정확도,
or fight the selected objective are tagged (never dropped) with a cited
또는 선택한 목표와 싸우려면 인용된 태그가 지정됩니다(삭제되지 않음).
reason.
이유.
Use `proposable(...)` for candidates and `excluded_flags(...)` to
후보에는 `proposable(...)`을 사용하고 후보에는 `excluded_flags(...)`를 사용하세요.
explain a rule-out.
룰아웃에 대해 설명해보세요.
- `scripts/ledger.py` -- records iterations (change proposal + config + results
- `scripts/ledger.py` -- 반복 기록(변경 제안 + 구성 + 결과)
  + the termination metrics from `evaluate_stop`), prevents repeats, picks the
  + `evaluate_stop`의 종료 측정항목), 반복을 방지하고
best, and writes a self-describing `result.json` per iteration.
최고이며 반복마다 자체 설명하는 'result.json'을 작성합니다.
- `scripts/offload_gap.py` -- reads the `CpuBecause` subgraphs from a compile's
- `scripts/offload_gap.py` -- 컴파일 파일에서 `CpuBecause` 하위 그래프를 읽습니다.
`*.chained_kernel.tosa.mlir` to report which ops are still on the CPU and what
CPU에 아직 남아 있는 작업과 작업을 보고하는 `*.chained_kernel.tosa.mlir`
the compiler said about each, routing only on citable markers.
컴파일러는 각각에 대해 인용 가능한 마커에만 라우팅한다고 말했습니다.
Turns "maximize
"최대화"로 바뀐다
offload" into "close this specific gap".
오프로드"를 "이 특정 격차를 해소"합니다.
- `scripts/spill_check.py` -- flags FM-driven L3 spills (FM footprint > device L2,
- `scripts/spill_check.py` -- FM 기반 L3 유출에 플래그를 지정합니다(FM 공간 > 장치 L2,
per column: STX/T50/T20 = 512 KB) from `DetailedSpillingAnalysis.csv`.
열당: STX/T50/T20 = 512KB) `DetailedSpillingAnalytic.csv`에서.
- `scripts/model_ops.py` -- censuses ONNX operator types and matches them to
- `scripts/model_ops.py` -- ONNX 연산자 유형을 조사하고 이를 다음과 일치시킵니다.
catalog flags (op-type-aware seed proposal); also inspects shapes/sizes
카탈로그 플래그(op 유형 인식 시드 제안); 모양/크기도 검사합니다.
pre-compile (`input_shapes`, `output_shapes` via shape inference,
사전 컴파일(모양 추론을 통한 `input_shapes`, `output_shapes`,
`large_layers`) for latency-bottleneck reasoning.
대기 시간 병목 현상 추론을 위한 `large_layers`)
- `scripts/ai_analyzer_timing.py` -- board E2E / NPU / CPU split via the AI
- `scripts/ai_analyzer_timing.py` -- AI를 통해 보드 E2E/NPU/CPU 분할
Analyzer SDK (`get_timing_dataframe_batch`); never parse raw `record_timer_*.json`.
분석기 SDK(`get_timing_dataframe_batch`); 절대로 원시 `record_timer_*.json`을 구문 분석하지 마세요.
- `scripts/normalize_timer_json.py` -- repairs schema-2.0 list timer files so the SDK reads them.
- `scripts/normalize_timer_json.py` -- SDK가 읽을 수 있도록 스키마 2.0 목록 타이머 파일을 복구합니다.

## Reused building blocks
## 재사용된 빌딩 블록
Components are named, not linked: the harness resolves a skill or agent by its
구성 요소는 연결되지 않고 이름이 지정됩니다. 하네스는 해당 요소에 따라 스킬이나 에이전트를 해결합니다.
name.
이름.
Scripts of a sibling skill are addressed relative to this skill's
형제 스킬의 스크립트는 이 스킬의 스크립트와 관련하여 처리됩니다.
directory, where the skills are siblings:
스킬이 형제인 디렉토리:
- Optional quantization: `vai-quantization-guide` skill. **The only entry point
- 선택적 양자화: `vai-Quantization-guide` 스킬. **유일한 진입점
to the mixed-precision skill is through `vai-quantization-guide`** — do not
혼합 정밀도 기술은 `vai-Quantization-guide`**를 통해 이루어집니다. — 하지 마세요.
invoke leaf mixed-precision skills directly. **Experimental:** print a warning
리프 혼합 정밀도 기술을 직접 호출합니다. **실험적:** 경고 인쇄
before offering it.
제공하기 전에.
- Optional custom-op development when CPU-resident ops block offload after flag
- CPU 상주 작업이 플래그 후 오프로드를 차단하는 경우 선택적 맞춤형 작업 개발
exploration: `vai-custom-op` skill — **the only entry point for the custom-op
탐색: `vai-custom-op` 스킬 — **custom-op의 유일한 진입점
skill is through `vai-custom-op`**; **user consent required**; **experimental**
스킬은 `vai-custom-op`**을 통해 이루어집니다. **사용자 동의가 필요합니다**; **실험적**
(print the same warning before offering); opens a separate session and may
(제공하기 전에 동일한 경고를 인쇄하십시오) 별도의 세션을 열고 다음을 수행할 수 있습니다.
take substantial time.
상당한 시간이 걸립니다.
- Flag analysis: `vai-fe-args` skill.
- 플래그 분석: 'vai-fe-args' 스킬.
- Compilation: `compile.py` in the scripts folder of the
- 컴파일: scripts 폴더의 `compile.py`
`vai-custom-op-implementation` skill.
`vai-custom-op-implementation` 스킬.
- Validation: `vai-perf-analysis` skill for offload % and SDK-backed operator
- 검증: 오프로드 % 및 SDK 지원 연산자에 대한 'vai-perf-analytic' 기술
metrics; `scripts/ai_analyzer_timing.py` for board timing.
측정항목; 보드 타이밍을 위한 `scripts/ai_analyzer_timing.py`

## Transparency
## 투명성
Every proposal/decision cites source material the user can actually open --
모든 제안/결정은 사용자가 실제로 열 수 있는 소스 자료를 인용합니다.
`FlagSpec.path` (the shipped `fe-args.html` public catalog anchor), the published
`FlagSpec.path`(제공된 `fe-args.html` 공개 카탈로그 앵커), 게시됨
Vitis AI EP-configuration doc, `DetailedSpillingAnalysis.csv`, the
Vitis AI EP 구성 문서 `DetailedSpillingAnalytic.csv`,
`*.chained_kernel.tosa.mlir`, or SDK timing JSON from `ai_analyzer_timing.py` /
`*.chained_kernel.tosa.mlir` 또는 `ai_analyzer_timing.py`의 SDK 타이밍 JSON /
`ai_extract.py` -- and explains the rationale for the choice.
`ai_extract.py` -- 선택 이유를 설명합니다.

## Setup (clean virtual environment)
## 설정(클린 가상 환경)

Running the skill needs only `skill_requirements.txt` (`onnx`).
스킬을 실행하려면 `skill_requirements.txt`(`onnx`)만 필요합니다.
Working on it in
에서 작업 중
a compiler source tree, and running the unit tests, additionally needs
컴파일러 소스 트리 및 단위 테스트를 실행하려면 추가로 필요합니다.
`requirements-dev.txt` (`pyyaml`, `pytest`) — install that one below, since it
`requirements-dev.txt` (`pyyaml`, `pytest`) — 아래에 설치하세요.
pulls in `skill_requirements.txt` too.
`skill_requirements.txt`도 가져옵니다.
A dedicated `.venv` is the cleanest,
전용 `.venv`가 가장 깨끗합니다.
isolated way, and it avoids PEP 668 "externally-managed-environment" errors on
격리된 방식으로 PEP 668 "외부 관리 환경" 오류를 방지합니다.
system Python:
시스템 파이썬:

```bash
cd <this skill's directory>   # the one containing SKILL.md
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements-dev.txt
```

When finished: `deactivate`.
완료되면: `비활성화`.

> Running the actual **compile** and **profiling** steps (Phase 2/3) requires the
> 실제 **컴파일** 및 **프로파일링** 단계(2/3단계)를 실행하려면 다음이 필요합니다.
> **Vitis AI product environment** — the official Docker image or an equivalent
> **Vitis AI 제품 환경** — 공식 Docker 이미지 또는 이에 상응하는 이미지
> install where `flexml`, `onnxruntime` with the `VitisAIExecutionProvider`, and
> `flexml`, `onnxruntime`과 `VitisAIExecutionProvider`를 설치하고
> `dlanalyzer` are already importable.
> `dlanalyzer`는 이미 가져올 수 있습니다.
> Supply them via `--t <activate>` when using
> 사용할 때 `--t <활성화>`를 통해 제공하세요.
> a product virtualenv, or run inside the Vitis AI container with no `--t`.
> virtualenv 제품을 사용하거나 '--t' 없이 Vitis AI 컨테이너 내에서 실행합니다.
> The Vitis AI Docker image provides `onnx` but **does not ship pytest**; it is
> Vitis AI Docker 이미지는 `onnx`를 제공하지만 **pytest를 제공하지 않습니다**. 그것은이다
> not where you run this skill's unit tests.
> 이 기술의 단위 테스트를 실행하는 곳이 아닙니다.

## Running the tests (compiler source tree only)
## 테스트 실행(컴파일러 소스 트리만 해당)

The **delivered skill bundle does not ship pytest or the test suite** — only
**제공된 스킬 번들은 pytest 또는 테스트 스위트를 제공하지 않습니다** — 단지
`skill_requirements.txt` (`onnx`) is installed with the skill.
`skill_requirements.txt`(`onnx`)는 스킬과 함께 설치됩니다.
`requirements-dev.txt` (`pytest`, `pyyaml`) and the tests under
`requirements-dev.txt`(`pytest`, `pyyaml`) 및 아래의 테스트
`python/test/flag_configuration/` live in the **vitis_flexml source tree** for
`python/test/flag_configuration/`는 **vitis_flexml 소스 트리**에 있습니다.
skill development and CI, not in the ai_utils install.
기술 개발 및 CI는 ai_utils 설치가 아닙니다.

From a vitis_flexml checkout, create the dev venv above, then from the
vitis_flexml 체크아웃에서 위의 dev venv를 생성한 다음
**repository root**:
**저장소 루트**:

```bash
python -m pytest --noconftest -o addopts="" \
  python/test/flag_configuration/ -v
```

The repository root's `pytest`/`conftest.py` config pulls in unrelated plugins
저장소 루트의 `pytest`/`conftest.py` 구성이 관련 없는 플러그인을 가져옵니다.
(e.g. torch, xdist), so `--noconftest -o addopts=""` keeps these tests
(예: torch, xdist) `--noconftest -o addopts=""`는 이러한 테스트를 유지합니다.
standalone.
독립형.
Do not expect this command to work inside the Vitis AI Docker image
이 명령이 Vitis AI Docker 이미지 내에서 작동할 것으로 예상하지 마세요.
unless you pip-install `requirements-dev.txt` there yourself (normally unnecessary).
`requirements-dev.txt`를 직접 pip 설치하지 않는 한(일반적으로 불필요함).
