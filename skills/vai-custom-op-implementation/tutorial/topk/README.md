<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# TopK Custom Op Tutorial
# TopK 커스텀 작전 튜토리얼

This directory contains examples of implementing a TopK custom operation for VitisAI.
이 디렉토리에는 VitisAI에 대한 TopK 사용자 정의 작업을 구현하는 예가 포함되어 있습니다.

## What is TopK?
## 탑케이(TopK)란 무엇인가요?

The TopK operation selects the K largest elements from the input tensor.
TopK 연산은 입력 텐서에서 K개의 가장 큰 요소를 선택합니다.
Given an input tensor of shape `[batch, n]`, it outputs a tensor of shape `[batch, k]` containing the top K values from each batch.
'[batch, n]' 모양의 입력 텐서가 주어지면 각 배치의 상위 K 값을 포함하는 '[batch, k]' 모양의 텐서를 출력합니다.
