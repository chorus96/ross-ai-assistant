<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# TopK Indices Custom Op Tutorial
# TopK 지수 맞춤형 운영 튜토리얼

This directory contains an example of implementing a TopK Indices custom operation for VitisAI, which is similar to the [ONNX TopK operation](https://onnx.ai/onnx/operators/onnx__TopK.html) but returns only the indices output.
이 디렉터리에는 VitisAI에 대한 TopK 지수 사용자 지정 작업을 구현하는 예가 포함되어 있습니다. 이는 [ONNX TopK 작업](https://onnx.ai/onnx/operators/onnx__TopK.html)과 유사하지만 인덱스 출력만 반환합니다.

## What is TopK Indices?
## TopK 지수란 무엇인가요?

The TopK Indices operation identifies the K largest elements from a 1D input tensor and returns their **indices** (not the values).
TopK Indices 작업은 1D 입력 텐서에서 K개의 가장 큰 요소를 식별하고 해당 **인덱스**(값이 아님)를 반환합니다.
Given:
주어진:
- An input tensor (values) of shape `[N]` containing numerical values
- 숫자 값을 포함하는 `[N]` 형태의 입력 텐서(값)
- A parameter K specifying how many top elements to find
- 찾을 상위 요소 수를 지정하는 매개변수 K

It outputs a tensor of shape `[K]` containing the indices of the K largest elements, sorted in descending order by their corresponding values.
이는 K개의 가장 큰 요소의 인덱스를 포함하고 해당 값에 따라 내림차순으로 정렬된 `[K]` 모양의 텐서를 출력합니다.

This operation is commonly used in feature detection, ranking, and selection tasks where you need to know the positions of the highest-scoring elements rather than the scores themselves.
이 작업은 점수 자체보다는 가장 높은 점수를 받은 요소의 위치를 ​​알아야 하는 특징 검색, 순위 지정 및 선택 작업에 ​​일반적으로 사용됩니다.

## Overview
## 개요

The implementation in `0_untiled` demonstrates:
`0_untiled`의 구현은 다음을 보여줍니다.
- **Single-Core Processing**: Processes the entire input on a single AIE core
- **단일 코어 처리**: 단일 AIE 코어에서 전체 입력을 처리합니다.
- **In-Memory Operation**: Fits the full input tensor in L1 memory (up to 10K elements for bfloat16)
- **인메모리 작업**: L1 메모리의 전체 입력 텐서에 적합합니다(bfloat16의 경우 최대 10K 요소)
- **Index Tracking**: Maintains a sorted list of top K indices using an insertion sort algorithm
- **인덱스 추적**: 삽입 정렬 알고리즘을 사용하여 상위 K개 인덱스의 정렬된 목록을 유지합니다.
- **Output Encoding**: Returns indices as int16 values.
- **출력 인코딩**: 인덱스를 int16 값으로 반환합니다.

Note: The runtime zero-pads the 16-bit results by the right to 32 bits before returning them to the user.
참고: 런타임은 16비트 결과를 사용자에게 반환하기 전에 32비트 오른쪽으로 0으로 채웁니다.
The kernel's results are therefore the upper 16 bits of each 32-bit result value.
따라서 커널의 결과는 각 32비트 결과 값의 상위 16비트입니다.
