<!-- ko-translation: enabled -->
---
name: vai-dequantize-model
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
description: Strip quantization from an ONNX model to produce a non-quantized (float)
  version suitable for re-quantization. Removes QDQ pairs, Extended QDQ operators,
  and BFP quantization nodes while preserving the computational graph structure. Supports
  legacy QDQ (vint8), Extended QDQ (EQDQ/bf16/fp16), and MX6/BFP quantization.
---
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->

# Dequantize Model Skill
# 모델 기술의 역양자화

## Access command
## 액세스 명령

This skill must always be accessed through the worker, never in a different way.
이 기술은 항상 작업자를 통해 접근해야 하며 다른 방식으로는 접근할 수 없습니다.
Allowed worker: `vai-mixed-precision-worker`.
허용되는 작업자: `vai-mixed-precision-worker`.

## Description
## 설명

Takes an already-quantized ONNX model and produces a non-quantized (float32)
이미 양자화된 ONNX 모델을 가져와 양자화되지 않은(float32) 모델을 생성합니다.
version.
버전.
This is useful for:
이는 다음과 같은 경우에 유용합니다.
- Re-quantization with a different configuration
- 다른 구성으로 재양자화
- Switching from legacy quantization (multi-island fp op flags) to extended
- 레거시 양자화(다중 아일랜드 fp op 플래그)에서 확장된 양자화로 전환
quantization (EQDQ)
양자화(EQDQ)
- Switching from extended to legacy quantization
- 확장 양자화에서 레거시 양자화로 전환
- Model analysis and debugging
- 모델 분석 및 디버깅

## Parse Arguments
## 인수 구문 분석

This skill expects arguments in this form:
이 기술은 다음 형식의 인수를 기대합니다.

```
--input <quantized.onnx> --output <float_model.onnx> [--keep-weights] [--preserve-structure]
```

- `--input <path>`: **(MANDATORY)** Path to the quantized ONNX model (.onnx or .onnxtxt)
- `--input <path>`: **(필수)** 양자화된 ONNX 모델의 경로(.onnx 또는 .onnxtxt)
- `--output <path>`: **(MANDATORY)** Path for the output non-quantized model
- `--output <path>`: **(필수)** 출력 비양자화 모델의 경로
- `--keep-weights`: **(OPTIONAL)** Preserve weight values as float32 (default: true)
- `--keep-weights`: **(선택 사항)** 가중치 값을 float32로 유지합니다(기본값: true)
- `--preserve-structure`: **(OPTIONAL)** Keep graph topology unchanged except for Q/DQ removal
- `--preserve-structure`: **(선택 사항)** Q/DQ 제거를 제외하고 그래프 토폴로지를 변경하지 않고 유지합니다.

## Quantization Patterns to Remove
## 제거할 양자화 패턴

### 1.
### Standard QDQ (QuantizeLinear / DequantizeLinear)
### 표준 QDQ(QuantizeLinear/DeQuantizeLinear)

**Input pattern**:
**입력 패턴**:
```
tensor → QuantizeLinear(scale, zero_point) → DequantizeLinear(scale, zero_point) → consumer
```

**Output (dequantized)**:
**출력(역양자화)**:
```
tensor → consumer
```

**Steps**:
**단계**:
1. Identify all QuantizeLinear → DequantizeLinear pairs
1. 모든 QuantizeLinear → DeQuantizeLinear 쌍을 식별합니다.
2. For activation Q/DQ: rewire the DQ output consumers to use the Q input directly
2. Q/DQ 활성화의 경우: Q 입력을 직접 사용하도록 DQ 출력 소비자를 다시 연결합니다.
3. For weight Q/DQ: dequantize weight values back to float32:
3. 가중치 Q/DQ의 경우: 가중치 값을 다시 float32로 역양자화합니다.
   ```python
   float_weight = (int8_weight - zero_point) * scale
   ```
4. Remove the Q and DQ nodes
4. Q 및 DQ 노드 제거
5. Update initializer dtypes from int8 to float32
5. int8에서 float32로 초기화 dtype 업데이트

### 2.
### Extended QDQ (ExtendedQuantizeLinear / ExtendedDequantizeLinear)
### 확장된 QDQ(ExtendedQuantizeLinear / ExtendedDeQuantizeLinear)

**Input pattern** (com.amd.quark domain):
**입력 패턴** (com.amd.quark 도메인):
```
tensor → ExtendedQuantizeLinear(scale, zp) → ExtendedDequantizeLinear(scale, zp) → consumer
```

**Output (dequantized)**:
**출력(역양자화)**:
```
tensor → consumer
```

**Steps**:
**단계**:
1. Identify all ExtendedQuantizeLinear → ExtendedDequantizeLinear pairs
1. 모든 ExtendedQuantizeLinear → ExtendedDeQuantizeLinear 쌍을 식별합니다.
2. Since EQDQ represents bf16/fp16 precision, the values are already essentially
2. EQDQ는 bf16/fp16 정밀도를 나타내므로 값은 이미 본질적으로
float — simply remove the EQ/EDQ wrapper nodes
float — 간단히 EQ/EDQ 래퍼 노드를 제거합니다.
3. Rewire: EQ input tensor → directly to EDQ consumers
3. 재연결: EQ 입력 텐서 → EDQ 소비자에게 직접 연결
4. Remove unused scale/zero_point initializers
4. 사용하지 않는 scale/zero_point 이니셜라이저 제거
5. Remove `com.amd.quark` from opset_import if no quark ops remain
5. quark 연산이 남아 있지 않으면 opset_import에서 `com.amd.quark`를 제거하세요.

### 3.
### BFP / MX6 Quantization
### BFP / MX6 양자화

**Input pattern**:
**입력 패턴**:
```
tensor → BFPQuantizeLinear(params) → BFPDequantizeLinear(params) → consumer
```

**Output (dequantized)**:
**출력(역양자화)**:
```
tensor → consumer
```

**Steps**:
**단계**:
1. Identify BFP Q/DQ pairs (look for ops with BFP attributes or in quark domain)
1. BFP Q/DQ 쌍 식별(BFP 속성이 있거나 쿼크 도메인에서 작업 찾기)
2. Remove BFP quantization nodes
2. BFP 양자화 노드 제거
3. Rewire inputs directly to consumers
3. 입력을 소비자에게 직접 다시 연결
4. Remove BFP-specific initializers and attributes
4. BFP 관련 이니셜라이저 및 속성 제거

### 4.
### Standalone Q or DQ (Boundary Nodes)
### 독립형 Q 또는 DQ(경계 노드)

At model input/output boundaries, there may be standalone Q or DQ:
모델 입력/출력 경계에는 독립형 Q 또는 DQ가 있을 수 있습니다.

**Input boundary** (standalone DQ at beginning):
**입력 경계**(시작 시 독립형 DQ):
```
input(int8) → DequantizeLinear → float_ops
```
→ Change input type to float32, remove DQ
→ 입력 유형을 float32로 변경하고 DQ를 제거합니다.

**Output boundary** (standalone Q at end):
**출력 경계**(끝에 독립형 Q):
```
float_ops → QuantizeLinear → output(int8)
```
→ Remove Q, change output type to float32
→ Q를 제거하고 출력 유형을 float32로 변경합니다.

## Edge Cases
## 엣지 케이스

1. **Shared scales**: Multiple Q/DQ nodes may share scale/zp initializers.
1. **공유 스케일**: 여러 Q/DQ 노드가 scale/zp 이니셜라이저를 공유할 수 있습니다.
Only remove an initializer when ALL referencing nodes are removed.
모든 참조 노드가 제거된 경우에만 초기화 프로그램을 제거하십시오.

2. **Subgraph Q/DQ**: Q/DQ nodes may appear inside subgraphs (If, Loop).
2. **하위 그래프 Q/DQ**: Q/DQ 노드는 하위 그래프(If, Loop) 내에 나타날 수 있습니다.
Process all subgraphs recursively.
모든 하위 그래프를 재귀적으로 처리합니다.

3. **Fused ops**: Some quantized models have fused ops (e.g., QLinearConv).
3. **퓨즈된 작업**: 일부 양자화 모델에는 융합된 작업이 있습니다(예: QLinearConv).
These must be replaced with their float equivalents (Conv).
이는 해당 부동소수점(Conv)으로 대체되어야 합니다.

4. **Mixed domains**: A model may have both standard QDQ and EQDQ.
4. **혼합 도메인**: 모델에는 표준 QDQ와 EQDQ가 모두 있을 수 있습니다.
Process both types.
두 가지 유형을 모두 처리합니다.

5. **Dynamic quantization**: Externally-quantized models may use per-channel
5. **동적 양자화**: 외부 양자화 모델은 채널별 사용 가능
quantization.
양자화.
Handle both per-tensor and per-channel QDQ correctly during
텐서별 및 채널별 QDQ를 모두 올바르게 처리하는 동안
stripping.
스트리핑.
Note: VINT8 applied by this toolchain is always per-tensor.
참고: 이 툴체인에서 적용되는 VINT8은 항상 텐서별로 적용됩니다.

## Output Verification
## 출력 검증

After dequantization:
역양자화 후:
1. Run ONNX shape inference on the output model
1. 출력 모델에서 ONNX 모양 추론 실행
2. Verify all tensors are float32
2. 모든 텐서가 float32인지 확인하세요.
3. Run the model on CPUExecutionProvider to confirm it produces valid outputs
3. CPUExecutionProvider에서 모델을 실행하여 유효한 출력이 생성되는지 확인합니다.
4. Compare with the quantized model's CPU output (should be very close for
4. 양자화된 모델의 CPU 출력과 비교하십시오(
properly dequantized weights)
적절하게 역양자화된 가중치)

## Usage in Re-quantization Workflow
## 재양자화 작업 흐름에서의 사용

```bash
# Step 1: Dequantize existing model
/vai-dequantize --input model_vint8.onnx --output model_float.onnx

# Step 2: Re-quantize with different config (e.g., move some layers to bf16)
/vai-quantize --input model_float.onnx --config backbone_int8_other_bf16.json

# Step 3: Or re-quantize with extended quantization for interleaved precision
/vai-quantize --input model_float.onnx --config eqdq_config.json
```
