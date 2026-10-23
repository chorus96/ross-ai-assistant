<!-- ko-translation: enabled -->
---
name: vai-partition-analysis
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
description: Inspect the result of a VAIML compilation of a mixed-precision ONNX model
  and reduce the number of NPU partitions toward a single partition. Parses the compiler
  operation statistics (matched kernels / templated graphs), the partition count,
  and the AIE/CPU offload split; when more than one partition is produced, reads the
  CpuBecause messages and compilation warnings, classifies them, and proposes remediation.
  Remediation prefers the correct frontend (fe_args) flags over model patching; when
  the partition break originates from quantization, it proposes Quark configuration
  changes instead. Every proposed change is followed by a re-compilation to evaluate
  the result.
---

# Partition-Analysis Skill
# 파티션 분석 기술

## Access command
## 액세스 명령

This skill must always be accessed through the worker, never in a different way.
이 기술은 항상 작업자를 통해 접근해야 하며 다른 방식으로는 접근할 수 없습니다.
Allowed worker: `vai-mixed-precision-worker`.
허용되는 작업자: `vai-mixed-precision-worker`.

## Description
## 설명

Inspect a VAIML compilation of a mixed-precision model and drive it toward a
혼합 정밀도 모델의 VAIML 컴파일을 검사하고 이를
**single NPU partition**.
**단일 NPU 파티션**.
This skill does not compile by itself: it reads the
이 스킬은 자체적으로 컴파일되지 않습니다.
artifacts of a compilation that already ran, reports the partitioning it
이미 실행된 컴파일의 아티팩트, 분할을 보고합니다.
achieved, and — when the model did not collapse to one partition — proposes the
달성되었으며 모델이 하나의 파티션으로 축소되지 않았을 때 다음을 제안합니다.
**smallest change that can remove a partition boundary** and asks to re-compile
**파티션 경계를 제거할 수 있는 가장 작은 변경** 및 재컴파일 요청
to evaluate it.
그것을 평가하기 위해.

> **Scope rule (non-negotiable).** This skill reduces partitions by **choosing
> **범위 규칙(협상 불가능).** 이 기술은 **선택하여 파티션을 줄입니다.
> the right compiler frontend (`fe_args`) flags** and, when the break is
> 올바른 컴파일러 프런트엔드(`fe_args`) 플래그** 그리고 중단이
> caused by quantization, by **adjusting the Quark quantization configuration**.
> **Quark 양자화 구성 조정**으로 인해 양자화로 인해 발생합니다.
> It does **NOT** patch the ONNX graph, rewrite nodes, change layer
> ONNX 그래프 패치, 노드 재작성, 레이어 변경 **아닙니다**
> shapes/counts, or prune the model to force a pass.
> 모양/개수를 지정하거나 모델을 잘라내어 강제로 통과시킵니다.
> Model patching and model
> 모델 패치 및 모델
> modification are out of scope — if nothing in the flag surface or the Quark
> 수정은 범위를 벗어납니다. 플래그 표면이나 Quark에 아무것도 없는 경우
> config can close the gap, say so and stop.
> 구성을 통해 격차를 해소하고 그렇게 말하고 중지할 수 있습니다.

> **Config key aliases.** `fe_args` and `fe_experiment` are the same option, as
> **구성 키 별칭.** `fe_args` 및 `fe_experiment`는 다음과 같은 옵션입니다.
> are `group_args` and `experiments`.
> 'group_args' 및 'experiments'입니다.
> Compiler logs and older configs may use either
> 컴파일러 로그 및 이전 구성은 다음 중 하나를 사용할 수 있습니다.
> spelling — read both, write `fe_args` and `group_args`.
> 철자법 - 둘 다 읽고 `fe_args`와 `group_args`를 씁니다.

## Parse Arguments
## 인수 구문 분석

This skill expects arguments in this form:
이 기술은 다음 형식의 인수를 기대합니다.

```
--compile-dir <vaiml_compile_dir> [--log <compile.log>] [--model <onnx>] [--config <vitisai_config.json>] [--device stx|ve2] [--quark-config <quark.json>]
```

- `--compile-dir <path>`: **(MANDATORY)** Directory containing the VAIML
- `--compile-dir <path>`: **(필수)** VAIML이 포함된 디렉터리
compilation artifacts (the cache dir produced by `compile.py` /
컴파일 아티팩트(`compile.py`에 의해 생성된 캐시 디렉토리/
`compile_probe.py`, e.g.
`compile_probe.py`, 예:
`<model_stem>/` with `cache/vaiml_par_*` inside).
내부에 `cache/vaiml_par_*`가 있는 `<model_stem>/`).
The compile must have run with `keep_outputs: true` so the
컴파일은 `keep_outputs: true`로 실행되어야 하므로
`*.chained_kernel.tosa.mlir` and the pass summaries are on disk.
`*.chained_kernel.tosa.mlir` 및 패스 요약은 디스크에 있습니다.
- `--log <path>`: **(OPTIONAL)** The captured compiler stdout/stderr log.
- `--log <경로>`: **(선택 사항)** 캡처된 컴파일러 stdout/stderr 로그입니다.
- If not
- 그렇지 않은 경우
given, look for `compile.log`, `AIECompiler.log`, or the ORT capture under
주어진 경우 `compile.log`, `AIECompiler.log` 또는 아래에서 ORT 캡처를 찾으세요.
`--compile-dir`.
`--컴파일-디렉터리`.
The `[VAIML-COMPILE ...]` operation-statistics and
`[VAIML-COMPILE ...]` 작업 통계 및
partition-count lines are read from here.
파티션 수 행은 여기에서 읽혀집니다.
- `--model <path>`: **(OPTIONAL)** The compiled ONNX model.
- `--model <path>`: **(선택 사항)** 컴파일된 ONNX 모델입니다.
- Needed only when the
- 다음과 같은 경우에만 필요합니다.
remediation requires re-examining the graph (e.g. reading a `QuantizeLinear`
문제를 해결하려면 그래프를 다시 검토해야 합니다(예: `QuantizeLinear` 읽기
scale/zero_point to decide whether `match-power-of-two-quant-kernel` can fire).
scale/zero_point는 `2양 커널 일치 전력`이 실행될 수 있는지 여부를 결정합니다).
- `--config <path>`: **(OPTIONAL)** The `vitisai_config.json` that was compiled.
- `--config <path>`: **(선택 사항)** 컴파일된 `vitsai_config.json`입니다.
New flag proposals are written on top of this file, never from scratch —
새로운 깃발 제안은 이 파일 위에 작성되며 처음부터 작성되지 않습니다.
every flag already present is carried forward.
이미 존재하는 모든 플래그가 전달됩니다.
- `--device <stx|ve2>`: **(OPTIONAL, default: ve2)** Target device.
- `--device <stx|ve2>`: **(선택 사항, 기본값: ve2)** 대상 장치.
- Governs
- 통치하다
whether BF16 (`enable_f32_to_bf16_conversion`) or FP16
BF16(`enable_f32_to_bf16_conversion`) 또는 FP16인지 여부
(`enable_f32_to_f16_conversion`) conversion is the right lever.
(`enable_f32_to_f16_conversion`) 변환은 오른쪽 레버입니다.
- `--quark-config <path>`: **(OPTIONAL)** The Quark JSON config that produced the
- `--quark-config <경로>`: **(선택 사항)**
quantized model.
양자화 모델.
Required only when the remediation is a quantization change
교정이 양자화 변경인 경우에만 필요합니다.
(see [Quark-config remediation](#remediation-b-quark-configuration-when-the-break-is-quantization)).
([Quark-config 교정](#remediation-b-quark-configuration-when-the-break-is-Quarkization)을 참조하세요).

Skill directory: the directory containing this `SKILL.md` (referred to below as
스킬 디렉터리: 이 `SKILL.md`를 포함하는 디렉터리(아래에서
`<skill_dir>`).
`<skill_dir>`).
The reused CpuBecause/offload parser is `offload_gap.py` in the
재사용된 CpuBecause/offload 파서는 'offload_gap.py'입니다.
scripts folder of the `vai-flag-configuration` skill.
`vai-flag-configuration` 스킬의 scripts 폴더입니다.

Scripts of that skill are addressed through a shell variable: before running any
해당 기술의 스크립트는 쉘 변수를 통해 처리됩니다.
snippet below, set `VAI_FLAG_CONFIGURATION_SCRIPTS` to the script directory of
아래 스니펫에서 `VAI_FLAG_CONFIGURATION_SCRIPTS`를 스크립트 디렉토리로 설정합니다.
the `vai-flag-configuration` skill, and set
'vai-flag-configuration' 스킬을 설정하고
`VAI_CUSTOM_OP_IMPLEMENTATION_SCRIPTS` to the script directory of the
`VAI_CUSTOM_OP_IMPLEMENTATION_SCRIPTS`를 스크립트 디렉토리에
`vai-custom-op-implementation` skill.
`vai-custom-op-implementation` 스킬.

## Phase 1 — Read the partitioning the compiler achieved
## 1단계 - 컴파일러가 달성한 분할 읽기

The compiler reports the partitioning in its log.
컴파일러는 로그에 분할을 보고합니다.
Extract three facts, in order.
세 가지 사실을 순서대로 추출합니다.

### 1a.
### Operation statistics (what matched, and as what)
### 운영통계(무엇이 일치하는지, 무엇으로 일치하는지)

The compiler prints an operation census after frontend matching:
컴파일러는 프런트엔드 일치 후 작업 인구 조사를 인쇄합니다.

```
INFO: [VAIML-COMPILE 1000] Operation statistics for the model:
 Operations encountered:
-----------------------
  kernel:AddBf163D (bf16, bf16) -> (bf16)                             , 25
  kernel:DequantizeLinear3D (i8) -> (bf16)                            , 1
  kernel:GeluLUTQi83D (i8, ui8) -> (i8)                               , 12
  kernel:Gemm (i8, i8, i16) -> (bf16)                                 , 12
  kernel:LayerNormRowMajor1Pass (i8, i8, i8) -> (i8)                  , 25
  kernel:QuantizeLinear3D (bf16) -> (i8)                              , 25
  templatedGraph:MX9MHAs256d512m2048Adf (i8, f32, f32, ...) -> (bf16) , 12
```

Parse each line into `(match_kind, name, in_dtypes, out_dtypes, count)` where
각 줄을 `(match_kind, name, in_dtypes, out_dtypes, count)`로 구문 분석합니다.
`match_kind` is `kernel` or `templatedGraph`.
'match_kind'는 'kernel' 또는 'templatedGraph'입니다.
Report this as a table.
이를 테이블로 보고합니다.
The dtypes
dtype
matter: a `kernel:...Bf16...` line means the op was placed on the NPU in bf16; a
문제: `kernel:...Bf16...` 줄은 작업이 bf16의 NPU에 배치되었음을 의미합니다. 에
line that still carries `f32`/`float32` in its signature is a red flag (a naked
여전히 서명에 `f32`/`float32`를 포함하는 라인은 빨간색 플래그(네이키드 플래그)입니다.
float op the frontend did not convert — see the `[MLLIB]` `config.dtype` failures
float op 프런트엔드가 변환되지 않았습니다. `[MLLIB]` `config.dtype` 실패를 참조하세요.
below).
아래).
The exact kernels and templated graphs differ per model; do not assume a
정확한 커널과 템플릿 그래프는 모델마다 다릅니다. 가정하지 마십시오
fixed set.
고정 세트.

### 1b.
### Partition count (the number to drive to 1)
### 파티션 수(1로 구동할 수)

```
INFO: [VAIML-COMPILE 1000] Partition completed with 1 partition
INFO: [VAIML-COMPILE 1046] 100.00% of operations will run on AIE, 0.00% of operations will run on CPU.
```

Extract:
추출:
- **partition count** from `Partition completed with N partition`;
- `N 파티션으로 완성된 파티션`의 **파티션 수**;
- **AIE / CPU split** from `[VAIML-COMPILE 1046] X% ... on AIE, Y% ... on CPU`.
- **AIE/CPU 분할** `[VAIML-COMPILE 1046] X% ... on AIE, Y% ... on CPU`.

**Success criterion:** exactly **1 partition** and **100.00% on AIE**.
**성공 기준:** 정확히 **1개의 파티션** 및 **AIE에서 100.00%**.
When both
둘 다일 때
hold, report success (with the operation-statistics table for the record) and
보류, 성공 보고(기록용 작업 통계 테이블 사용) 및
stop — there is nothing to remediate.
중지 - 수정할 것이 없습니다.

### 1c.
### Where each partition breaks (only when count > 1)
### 각 파티션이 중단되는 위치(개수 > 1인 경우에만)

For a multi-partition result the compiler names the boundary op of each break:
다중 파티션 결과의 경우 컴파일러는 각 중단의 경계 작업 이름을 지정합니다.

```
Partition 0 breaks at: CpuBecause("/model.6/.../Transpose_2_output_0_DequantizeLinear") / CpuBecause(".../Split_output_1_DequantizeLinear")
Partition 1 breaks at: CpuBecause("/model.6/.../Transpose_3_output_0_DequantizeLinear")
```

Collect every `Partition N breaks at: CpuBecause(...)` line.
모든 `파티션 N 중단 위치: CpuBecause(...)` 라인을 수집합니다.
The op named inside
내부에 명명된 작전
each `CpuBecause(...)` is the boundary that split the graph — these are the ops
각 `CpuBecause(...)`는 그래프를 분할하는 경계입니다. 이것이 작업입니다.
to remove from the CPU.
CPU에서 제거합니다.
`graph_partition_trace.csv` under `cache/` carries the
`cache/` 아래의 `graph_partition_trace.csv`에는
same information in tabular form when present.
동일한 정보가 있는 경우 표 형식으로 표시됩니다.

## Phase 2 — Read the CpuBecause messages and warnings (when count > 1)
## 2단계 - CpuBecause 메시지 및 경고 읽기(개수 > 1인 경우)

Every operator the compiler pushed to the CPU becomes an `xten_nn.subgraph` with
컴파일러가 CPU에 푸시한 모든 연산자는 다음과 같은 `xten_nn.subgraph`가 됩니다.
`Reason = "CpuBecause"`, an `Op` attribute (the responsible operation) and a
`Reason = "CpuBecause"`, `Op` 속성(책임 있는 작업) 및
`Message` (the compiler's own explanation) in the
`Message`(컴파일러 자체 설명)
`*.chained_kernel.tosa.mlir`. **Do not hand-parse the MLIR** — reuse the parser
`*.chained_kernel.tosa.mlir`. **MLIR을 직접 구문 분석하지 마세요** — 파서를 재사용하세요
that already ships in the `vai-flag-configuration` skill:
이미 `vai-flag-configuration` 스킬에 포함되어 있습니다:

```bash
python3 - <<'PY'
import sys
import os
sys.path.insert(0, os.environ["VAI_FLAG_CONFIGURATION_SCRIPTS"])
from offload_gap import find_chained_kernel_mlir, offload_gap, classify_message
mlir = find_chained_kernel_mlir("<compile-dir>")
for entry in offload_gap(mlir):                 # ranked by node count
    route = {m: classify_message(m).route for m in entry.messages}
    print(entry.op, entry.count, entry.messages, route)
print("MLIR:", mlir)
PY
```

`offload_gap()` returns, biggest-first, every op type still on the CPU with the
`offload_gap()`은 CPU에 여전히 있는 모든 작업 유형을 가장 큰 것부터 반환합니다.
compiler's verbatim `Message` and a routing tag from `classify_message()`.
컴파일러의 축어적 `Message`와 `classify_message()`의 라우팅 태그입니다.
Also
또한
read the compiler's **warnings** (`WARNING: [VAIML-...]` in the log and
로그에서 컴파일러의 **경고**(`WARNING: [VAIML-...]`)를 읽고
`preliminary-`/`final-vaiml-pass-summary.txt` under `cache/`) — a warning often
`cache/` 아래 `preliminary-`/`final-vaiml-pass-summary.txt`) — 종종 경고
names the same boundary from the FE side.
FE 측에서 동일한 경계의 이름을 지정합니다.

### CpuBecause routing table (from `offload_gap.classify_message`)
### CpuBecause 라우팅 테이블(`offload_gap.classify_message`에서)

Only these markers are attributable to a stage; anything else is
이러한 마커만 단계에 귀속됩니다. 다른 것은
`unclassified` and must be reported **verbatim**, never guessed at.
'분류되지 않음'이며 **동시에** 보고해야 하며, 추측할 수 없습니다.

| Marker in `Message`<br>`메시지`의 마커 | Route<br>경로 | What it means<br>의미 | First thing to try<br>가장 먼저 시도해 볼 사항 |
|---|---|---|---|
| `User requested this layer to be partitioned out`<br>`사용자가 이 레이어를 분할하도록 요청했습니다` | `user-config`<br>`사용자 구성` | This run's own config pushed the layer to CPU.<br>이 실행의 자체 구성은 레이어를 CPU로 푸시했습니다. | Review the active flags in `--config` before proposing new ones — you may be causing the break.<br>새로운 플래그를 제안하기 전에 `--config`에서 활성 플래그를 검토하세요. 중단이 발생할 수 있습니다. |
| `Unwrapping layer with ...`<br>`...로 레이어 래핑 해제` | `tiling-unwrap`<br>`타일링 풀기` | The tiling engine could not place the named kernel.<br>타일링 엔진이 명명된 커널을 배치할 수 없습니다. | Kernel-specific chaining flag or spilling analysis (hand to `vai-flag-configuration` / `vai-perf-analysis`), not a quantization change.<br>양자화 변경이 아닌 커널별 체이닝 플래그 또는 유출 분석(`vai-flag-configuration` / `vai-perf-analytic`으로 이동). |
| `... [FE]` | `fe` (frontend)<br>`fe`(프런트엔드) | The frontend did not match the operator.<br>프런트엔드가 운영자와 일치하지 않습니다. | A frontend flag that **decomposes or rewrites** the op, else it is a quantization-boundary or custom-op issue.<br>작업을 **분해하거나 다시 작성**하는 프런트엔드 플래그입니다. 그렇지 않으면 양자화 경계 또는 사용자 정의 작업 문제입니다. |
| `... [BE]` | `be` (backend)<br>`be`(백엔드) | No backend kernel for this shape/dtype.<br>이 모양/dtype에 대한 백엔드 커널이 없습니다. | A rewrite flag (e.g. conv↔gemm) that reaches a supported kernel.<br>지원되는 커널에 도달하는 재작성 플래그(예: conv⇔gemm)입니다. |
| `... [MLLIB]`<br>`... [MLLIB]` | `mllib`<br>`mllib` | A kernel-library constraint rejected the layer.<br>커널 라이브러리 제약 조건이 레이어를 거부했습니다. | Read the constraint; commonly a `config.dtype` mismatch → a naked float op that must be converted (see below).<br>제약 조건을 읽으십시오. 일반적으로 'config.dtype' 불일치 → 변환해야 하는 기본 부동 연산입니다(아래 참조). |

### Common CpuBecause messages and their cause (cite the message verbatim)
### 일반적인 CpuBecause 메시지와 그 원인(메시지를 그대로 인용)

These are frequently seen on mixed-precision models.
이는 혼합 정밀도 모델에서 자주 볼 수 있습니다.
Match on the message text,
메시지 텍스트와 일치합니다.
then follow the remediation in Phase 3.
그런 다음 3단계의 해결 방법을 따르세요.

| Message (substring)<br>메시지(하위 문자열) | Cause<br>원인 | Primary remediation class<br>기본 교정 수업 |
|---|---|---|
| `Standalone tosa.cast is not supported [FE]`<br>`독립형 tosa.cast는 지원되지 않습니다. [FE]` | A naked float cast left at an island/function edge.<br>섬/기능 가장자리에 왼쪽으로 캐스팅된 네이키드 플로트입니다. | Frontend flag — dtype conversion (Remediation A).<br>프런트엔드 플래그 - dtype 변환(수정 사항 A). |
| `Standalone dequantizeLinear is not supported [FE]` / `Standalone quantizeLinear is not supported [FE]`<br>`독립형 DeQuantizeLinear는 지원되지 않습니다. [FE]` / `독립형 QuantizeLinear는 지원되지 않습니다. [FE]` | A lone Q or DQ sits on a partition/function edge (the classic partition-break op in `Partition N breaks at`).<br>단독 Q 또는 DQ는 파티션/기능 가장자리에 위치합니다('파티션 N 중단'의 고전적인 파티션 중단 작업). | Frontend flag — edge/power-of-two quant kernel (A). If the Q/DQ cannot be matched, it is a **quantization** issue (B).<br>프런트엔드 플래그 — 에지/2의 거듭제곱 퀀트 커널(A). Q/DQ가 일치하지 않으면 **양자화** 문제입니다(B). |
| `YAML constraints failed [MLLIB]: ... 'config.dtype' ... float32 ... incompatible`<br>`YAML 제약 조건 실패 [MLLIB]: ... 'config.dtype' ... float32 ... 호환되지 않음` | A kernel (e.g. `MulBf16`, `GemmBfp16`, `AddBf163D`) was asked to run in `float32` because a naked float op was not converted.<br>Naked float 작업이 변환되지 않았기 때문에 커널(예: `MulBf16`, `GemmBfp16`, `AddBf163D`)이 `float32`에서 실행되도록 요청되었습니다. | Frontend flag — enable bf16/f16 conversion (A).<br>프런트엔드 플래그 - bf16/f16 변환(A)을 활성화합니다. |
| `Dataformat Edge to Op can not be fixed`<br>'Dataformat Edge to Op를 수정할 수 없습니다' | Layout mismatch at a function edge.<br>기능 가장자리의 레이아웃 불일치. | Frontend flag — data-storage / layout (A). Not model patching.<br>프런트엔드 플래그 — 데이터 저장/레이아웃(A). 모델 패치가 아닙니다. |
| `Reshape with start garbage cannot be merged into non-AIE consumer` / `PseudoOp:Reshape only supports bf16, fp16, int8, and uint8`<br>`시작 가비지를 사용한 Reshape는 AIE가 아닌 소비자로 병합될 수 없습니다` / `PseudoOp:Reshape는 bf16, fp16, int8 및 uint8만 지원합니다` | A reshape feeds a CPU consumer or carries an unsupported dtype.<br>모양 변경은 CPU 소비자에게 먹이를 주거나 지원되지 않는 dtype을 전달합니다. | Frontend flag; if dtype is float32, converge it via dtype conversion (A).<br>프런트엔드 플래그; dtype이 float32인 경우 dtype 변환(A)을 통해 수렴합니다. |
| `Small tensor (threshold 100) in function edge`<br>`함수 가장자리의 작은 텐서(임계값 100)` | A tiny tensor at an edge is cheaper on CPU by the threshold heuristic.<br>가장자리에 있는 작은 텐서는 임계값 휴리스틱에 의해 CPU에서 더 저렴합니다. | Frontend flag — small-tensor threshold (A), **but this conflicts with a max-offload goal**: only unwrap when a single partition is the explicit goal.<br>프런트엔드 플래그 — 작은 텐서 임계값(A), **이것은 최대 오프로드 목표와 충돌합니다**: 단일 파티션이 명시적인 목표인 경우에만 래핑 해제됩니다. |
| `onnx.Transpose in function edge`<br>`함수 가장자리의 onnx.Transpose` | A transpose landed on a partition edge.<br>전치(transpose)가 파티션 가장자리에 도착했습니다. | Frontend flag (A). Never insert/rewrite transposes in the ONNX.<br>프런트엔드 플래그(A). ONNX에 전치를 삽입하거나 다시 쓰지 마십시오. |
| `Datatype Unsupported [FE]: tosa.select : i1` / `Unsupported gather with constant data input and non-scalar indices [FE]`<br>`지원되지 않는 데이터 유형 [FE]: tosa.select : i1` / `상수 데이터 입력 및 비 스칼라 인덱스로 지원되지 않는 수집 [FE]` | A genuinely unsupported op/dtype at the frontend.<br>프런트엔드에서 실제로 지원되지 않는 op/dtype입니다. | Often not fixable by a public flag — report verbatim; may need quantization change (B) or is a hard limit.<br>공개 플래그로 해결할 수 없는 경우가 많습니다. 그대로 보고하세요. 양자화 변경(B)이 필요할 수 있거나 엄격한 제한이 있습니다. |

## Phase 3 — Propose remediation (frontend flags first)
## 3단계 - 수정 제안(프런트엔드 플래그 먼저)

Decide the remediation **class** per gap entry from its route and message, then
해당 경로 및 메시지에서 간격 항목별로 교정 **클래스**를 결정한 다음
propose the concrete change.
구체적인 변화를 제안합니다.
Always prefer Remediation A.
항상 수정 A를 선호합니다.
Only use Remediation B
교정 B만 사용하십시오.
when the break is caused by quantization.
양자화로 인해 중단이 발생한 경우.
Never patch the model.
모델을 패치하지 마십시오.

### Remediation A — frontend (`fe_args`) flags (default)
### 해결 A — 프런트엔드(`fe_args`) 플래그(기본값)

This is the primary path for reducing partitions.
이는 파티션을 줄이기 위한 기본 경로입니다.
Reuse the analysis already
이미 분석을 재사용
implemented by the sibling skills:
형제 기술에 의해 구현됨:

1. **Delegate dtype/island reasoning to `/vai-fe-dtype`**
1. **dtype/island 추론을 `/vai-fe-dtype`에 위임**
(`vai-fe-args`): it classifies the model's island
(`vai-fe-args`): 모델의 섬을 분류합니다.
topology and returns the required conversion + kernel-matching flags.
토폴로지를 반환하고 필요한 변환 + 커널 일치 플래그를 반환합니다.
The two
둘
levers it produces are the ones that collapse most mixed-precision breaks:
그것이 생성하는 레버는 대부분의 혼합 정밀도 나누기를 축소하는 레버입니다.
   - **Naked float ops** (`config.dtype ... float32 ... incompatible`, standalone
   - **Naked float ops** (`config.dtype ... float32 ... 호환되지 않음`, 독립형
`tosa.cast`): set `enable_f32_to_bf16_conversion: true` (STX/BF16) or
`tosa.cast`): `enable_f32_to_bf16_conversion: true` 설정(STX/BF16) 또는
`enable_f32_to_f16_conversion: true` with `device: ve2` (Telluride/FP16).
`enable_f32_to_f16_conversion: true`와 `device: ve2`(Telluride/FP16).
   - **Standalone Q/DQ at edges** (`Standalone [de]quantizeLinear ...
   - **가장자리의 독립형 Q/DQ** (`독립형 [de]QuantizeLinear ...
   - [FE]`,
`Partition N breaks at: CpuBecause(...DequantizeLinear)`): add
`파티션 N이 다음 위치에서 중단됩니다: CpuBecause(...DequalizeLinear)`): 추가
`match-power-of-two-quant-kernel=1` so the boundary Q/DQ is matched as an
`match-power-of-two-Quant-kernel=1`이므로 경계 Q/DQ는 다음과 같이 일치됩니다.
NPU kernel, and/or offload the boundary conversion to the runtime with
NPU 커널 및/또는 다음을 사용하여 경계 변환을 런타임으로 오프로드합니다.
`edge-quantization-in-rt=1` (or the one-sided `input-quantization-in-rt=1` /
`edge-Quantization-in-rt=1` (또는 일방적인 `input-Quantization-in-rt=1` /
`output-dequantization-in-rt=1`).
'출력-역양자화-in-rt=1').
These are equivalent to the mixed-precision
이는 혼합 정밀도와 동일합니다.
`group_args`/`match-power-of-two-quant-kernel` settings the orchestrator
`group_args`/`match-power-of-two-Quant-kernel`이 오케스트레이터를 설정합니다.
already documents.
이미 문서.
2. **Turn the CpuBecause gap into candidate flags** with the
2. **CpuBecause 간격을 후보 플래그로 전환**
`vai-flag-configuration` catalog.
`vai-flag-configuration` 카탈로그.
From the same gap entries produced in
에서 생성된 동일한 간격 항목에서
Phase 2:
2단계:
   ```bash
   python3 - <<'PY'
   import os, sys; sys.path.insert(0, os.environ["VAI_FLAG_CONFIGURATION_SCRIPTS"])
   from offload_gap import find_chained_kernel_mlir, offload_gap, match_gap_to_flags
   from flag_catalog import load_catalog, proposable
   gap = offload_gap(find_chained_kernel_mlir("<compile-dir>"))
   cat = proposable(load_catalog(), objectives=["offload"], autonomous=False)
   for spec, reason in match_gap_to_flags(gap, cat):
       print(spec.name, "::", reason, "::", spec.path)
   PY
   ```
`match_gap_to_flags` returns only flags that target an op **still on the CPU**,
`match_gap_to_flags`는 **아직 CPU에 있는** 작업을 대상으로 하는 플래그만 반환합니다.
ranked so a flag naming the missing op comes before a generic element-wise
누락된 작업의 이름을 지정하는 플래그가 일반 요소 단위 앞에 오도록 순위가 매겨졌습니다.
match.
일치합니다.
Treat it as a **shortlist**: read each flag's catalog description and
**후보 목록**으로 처리: 각 플래그의 카탈로그 설명을 읽고
discard any whose mechanism does not apply to the cited op.
인용된 작업에 적용되지 않는 메커니즘을 삭제합니다.
Only propose
제안만
`public: true` flags — never a flag outside the public catalog.
`public: true` 플래그 — 공개 카탈로그 외부의 플래그는 절대 아닙니다.
3. **Backend `[BE]` gaps**: propose a rewrite flag that reaches a supported
3. **백엔드 `[BE]` 간격**: 지원되는 수준에 도달하는 재작성 플래그를 제안합니다.
kernel (e.g.
커널(예:
`enable-conv-to-im2col-and-gemm=1` / `prefer-conv-to-gemm-conversion=true`
`enable-conv-to-im2col-and-gemm=1` / `prefer-conv-to-gemm-conversion=true`
for a Conv with no backend kernel).
백엔드 커널이 없는 Conv의 경우).
Cite the flag's catalog path.
플래그의 카탈로그 경로를 인용하세요.
4. **`tiling-unwrap` gaps**: this is a placement/spilling problem, not a
4. **`타일링-풀기` 간격**: 이는 배치/흘림 문제이지
frontend or quantization one.
프론트엔드 또는 양자화.
Hand the compile dir to `/vai-flag-configuration`
컴파일 디렉토리를 `/vai-flag-configuration`에 전달하세요.
(or `/vai-perf-analysis`) for a kernel-chaining/spilling flag rather than
(또는 `/vai-perf-analytic`) 대신 커널 체인/유출 플래그에 대해
changing dtype or quantization.
dtype 또는 양자화 변경.
5. **Respect existing flags and the budget.** Write proposals on top of
5. **기존 플래그와 예산을 존중합니다.** 그 위에 제안서를 작성하세요.
`--config`, carrying every flag already set.
`--config`, 이미 설정된 모든 플래그를 전달합니다.
Propose at most a **few** flag
최대 **몇 개** 플래그 제안
changes per iteration so the effect stays interpretable, and record for each
반복마다 변경되므로 효과를 해석 가능한 상태로 유지하고 각각에 대해 기록합니다.
flag its catalog source path, the default it deviates from, and a one-line
카탈로그 소스 경로, 벗어난 기본값 및 한 줄 플래그를 지정합니다.
rationale drawn from the compiler message.
컴파일러 메시지에서 도출된 근거.
A proposal without a cited source
인용된 출처가 없는 제안서
and reason is incomplete — do not present it.
이성은 불완전합니다. 제시하지 마세요.

**`needs-consent` flags:** never enable an accuracy-trading or
**`needs-consent` 플래그:** 절대 정확도 거래 또는
contract-changing flag (e.g.
계약 변경 플래그(예:
`small-tensor-threshold-unwrapping`,
'작은 텐서 임계값 풀기',
`use-accurate-mode`, `indices-are-positive`) on the user's behalf.
사용자를 대신하여 `use-accurate-mode`, `indices-are-Positive`)를 수행합니다.
Offer it with
함께 제공하세요
the trade-off stated and wait for consent.
절충안을 명시하고 동의를 기다립니다.

### Remediation B — Quark configuration (when the break is quantization)
### 교정 B - Quark 구성(중단이 양자화인 경우)

Use this only when the partition break originates from **how the model was
**모델이 어떻게 만들어졌는지에 따라 파티션 나누기가 발생한 경우에만 이 옵션을 사용하세요.
quantized**, i.e. the gap is a standalone Q/DQ or a quant-boundary that a
양자화**, 즉 간격은 독립형 Q/DQ이거나 양자 경계입니다.
frontend flag cannot match.
프런트엔드 플래그가 일치할 수 없습니다.
Typical signals:
일반적인 신호:

- `match-power-of-two-quant-kernel=1` cannot fire because the boundary
- 'match-power-of-two-Quant-kernel=1'은 경계 때문에 실행될 수 없습니다.
`QuantizeLinear` has a **non-power-of-two scale** or a **non-zero zero_point**
`QuantizeLinear`에는 **2의 거듭제곱이 아닌 스케일** 또는 **0이 아닌 zero_point**가 있습니다.
(read the initializers off `--model`; Quark symmetric-int8 usually satisfies
(`--model`에서 초기화 프로그램을 읽어보세요. Quark symmetric-int8은 일반적으로 다음을 만족합니다.
both).
둘 다).
Fix it at the source: quantize **symmetric int8 with power-of-two
소스에서 수정: **대칭 int8을 2의 거듭제곱으로 양자화
scales and zero_point 0** (`"ActivationSymmetric": true`, symmetric weight/act
scales 및 zero_point 0** (`"ActivationSymmetric": true`, 대칭 가중치/행위
specs) so the boundary Q/DQ becomes matchable.
사양) 경계 Q/DQ가 일치 가능해집니다.
- The mixed-precision assignment creates **many small float islands**, each
- 혼합 정밀도 할당은 **많은 작은 부동 섬**을 생성합니다.
bounded by a DQ→float→Q pair that becomes a partition break.
파티션 나누기가 되는 DQ→float→Q 쌍으로 경계가 지정됩니다.
Reduce the number
숫자를 줄이세요
of boundaries by moving the float/int8 cut so contiguous regions share one
인접한 영역이 하나를 공유하도록 float/int8 컷을 이동하여 경계를 만듭니다.
precision (fewer transitions), or promote/demote the offending layers — via the
정밀도(전환 횟수 감소) 또는 문제가 되는 레이어 승격/강등 —
Quark config's `specific_layer_config` / `target_node_names`, **not** by
Quark 구성의 `특정 레이어_구성` / `대상_노드 이름`, **아님**
editing the graph.
그래프를 편집하고 있습니다.
Fewer precision transitions ⇒ fewer partitions.
더 적은 정밀도 전환 ⇒ 더 적은 파티션.
- Naked float ops leaked because casts were not emitted as QDQ: ensure
- 캐스트가 QDQ로 방출되지 않았기 때문에 Naked float 작업이 유출되었습니다.
`"BF16QDQToCast": true` and that `enable_f32_to_bf16_conversion` is consistent
`"BF16QDQToCast": true` 및 `enable_f32_to_bf16_conversion`이 일관됨
with the config, and keep `EnableDualQuantNodePairs` where dual QDQ pairs are
구성을 사용하고 이중 QDQ 쌍이 있는 `EnableDualQuantNodePairs`를 유지합니다.
needed at boundaries.
경계에 필요합니다.

Apply the change by re-running quantization through the orchestrator's
오케스트레이터를 통해 양자화를 다시 실행하여 변경 사항을 적용합니다.
`mixed_precision_quantize.py` with the adjusted Quark JSON (hand back to
조정된 Quark JSON이 포함된 `mixed_precision_Quantize.py`(다시
`/vai-quantize` or `/vai-quantization-guide --mode requantize`). **Do not** hand-edit Q/DQ
`/vai-퀀트화` 또는 `/vai-퀀트화-가이드 --모드 재양자화`). **하지 마세요** Q/DQ를 직접 편집하지 마세요
nodes in the ONNX — change the Quark config and re-quantize.
ONNX의 노드 — Quark 구성을 변경하고 다시 양자화합니다.

> If a break is neither a frontend-flag case nor a quantization case (e.g.
> 중단이 프런트엔드 플래그 사례도 아니고 양자화 사례도 아닌 경우(예:
> `Datatype Unsupported [FE]: tosa.select : i1`, a genuine unsupported op),
> `지원되지 않는 데이터 유형 [FE]: tosa.select : i1`, 지원되지 않는 실제 작업),
> report it verbatim and state that it is outside this skill's flag/quant
> 이를 문자 그대로 보고하고 이 스킬의 플래그/퀀트 외부에 있음을 명시합니다.
> surface.
> 표면.
> Do not invent a fix and do not patch the model.
> 수정 사항을 고안하거나 모델을 패치하지 마십시오.

## Phase 4 — Re-compile to evaluate (every time)
## 4단계 — 재컴파일하여 평가(매번)

**A proposal is not a result.** Every change this skill proposes MUST be followed
**제안은 결과가 아닙니다.** 이 기술이 제안하는 모든 변경 사항은 반드시 따라야 합니다.
by a VAIML re-compilation so its effect on the partition count is measured, not
VAIML 재컴파일을 통해 파티션 수에 미치는 영향을 측정합니다.
assumed.
가정.
After writing the new `vitisai_config.json` (Remediation A) or producing
새로운 `vitsai_config.json`(수정 A)을 작성하거나 생성한 후
a re-quantized model (Remediation B):
재양자화된 모델(개선 B):

1. Ask the user to re-compile (a choice), or state that you are re-compiling in
1. 사용자에게 재컴파일(선택)을 요청하거나 다음에서 재컴파일 중임을 명시하십시오.
autonomous mode, and run the compile with `keep_outputs: true`.
자율 모드로 전환하고 'keep_outputs: true'로 컴파일을 실행하세요.
Use the same
같은 것을 사용하세요
compiler entry point the orchestrator uses:
오케스트레이터가 사용하는 컴파일러 진입점:
   ```bash
   # Fast frontend/partition probe (no AIE binary) — enough to read the new
   # partition count and CpuBecause set:
   python3 "$VAI_FLAG_CONFIGURATION_SCRIPTS/compile_probe.py" <model>.onnx \
       --vitisai-config <updated_vitisai_config.json> \
       --mode capability --cache-dir <model_stem> -o compile_probe.json

   # Full board-ready compile (when validating the final result):
   python3 "$VAI_CUSTOM_OP_IMPLEMENTATION_SCRIPTS/compile.py" <model>.onnx \
       --vitisai-config <updated_vitisai_config.json>
   ```
The fast `capability` probe reaches the same frontend/partition stage that
빠른 '능력' 프로브는 동일한 프런트엔드/파티션 단계에 도달합니다.
emits the operation statistics and `CpuBecause` set, so it is the cheap way to
작업 통계와 'CpuBecause' 세트를 내보냅니다. 따라서 이는 저렴한 방법입니다.
check whether a partition boundary was removed; switch to the full
파티션 경계가 제거되었는지 확인하십시오. 풀로 전환하다
`compile.py` before claiming a board-ready single-partition result.
보드 준비 단일 파티션 결과를 주장하기 전 `compile.py`.
2. Re-run Phase 1 on the fresh artifacts.
2. 새로운 아티팩트에 대해 1단계를 다시 실행합니다.
2. Report the **partition-count delta**
2. **파티션 수 델타** 보고
(e.g.
`3 → 1`) and the new AIE/CPU split, and whether each previously-broken
`3 → 1`) 및 새로운 AIE/CPU 분할, 그리고 각각이 이전에 깨졌는지 여부
`CpuBecause` boundary is gone.
'CpuBecause' 경계가 사라졌습니다.
3. **Loop.** If more than one partition remains, take the next-largest gap entry
3. **루프.** 두 개 이상의 파티션이 남아 있으면 다음으로 큰 간격 항목을 가져옵니다.
and repeat Phases 2–4.
2~4단계를 반복합니다.
Stop when the model reaches **1 partition / 100% AIE**,
모델이 **1개 파티션/100% AIE**에 도달하면 중지합니다.
when no untried proposable flag or Quark change remains, or when the residual
시도되지 않은 제안 가능한 플래그나 Quark 변경 사항이 남아 있지 않은 경우 또는 잔여
break is outside the flag/quant surface — and say which of these ended the loop.
break는 플래그/퀀트 표면 외부에 있습니다. 그리고 이들 중 어느 것이 루프를 종료했는지 말해보세요.

## Output
## 출력

Produce a **Partition Analysis report**:
**파티션 분석 보고서**를 생성합니다.
- the operation-statistics table (matched kernels / templated graphs with dtypes
- 작업 통계 테이블(dtypes와 일치하는 커널/템플릿 그래프)
and counts);
및 카운트);
- the **partition count** and **AIE/CPU %** split, with the success verdict
- **파티션 수** 및 **AIE/CPU %** 분할, 성공 판정
(1 partition / 100% AIE, or not);
(파티션 1개 / 100% AIE 여부)
- when > 1 partition: the ranked CpuBecause gap (op, node count, verbatim
- > 1 파티션인 경우: 순위가 매겨진 CpuBecause 간격(op, 노드 수, 축어적)
message, route) and, per entry, the proposed remediation (frontend flag with
메시지, 경로) 및 항목별로 제안된 수정(프런트엔드 플래그
its catalog path + rationale, or Quark-config change), each cited to the
카탈로그 경로 + 근거 또는 Quark-config 변경), 각각 인용됨
compiler message it addresses;
그것이 다루는 컴파일러 메시지;
- the **re-compilation** you ran (or propose to run) to evaluate the change, and
- 변경 사항을 평가하기 위해 실행한(또는 실행을 제안한) **재컴파일**
the resulting partition-count delta once available.
사용 가능한 경우 결과 파티션 수 델타입니다.

Cite every proposal to a source the user can open (the compiler `Message`, the
사용자가 열 수 있는 소스(컴파일러 '메시지',
flag's catalog path via `FlagSpec.path`, the doc anchor for a `vaiml_config`
`vaiml_config`에 대한 문서 앵커인 `FlagSpec.path`를 통한 플래그의 카탈로그 경로
field).
필드).
A proposal without a citation and a reason is incomplete.
인용 및 이유가 없는 제안은 불완전합니다.
