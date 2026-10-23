<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# Custom Op example: negate
# 맞춤 작업 예시: 부정

The examples in this directory computes the negation of its inputs, e.g., like the [onnx negate op](https://onnx.ai/onnx/operators/onnx__Neg.html).
이 디렉터리의 예는 [onnx 부정 연산](https://onnx.ai/onnx/operators/onnx__Neg.html)과 같은 입력의 부정을 계산합니다.
It includes both a simpler "emulated" 1x1, and a more complex 4x4 tiling example.
여기에는 더 간단한 "에뮬레이트된" 1x1과 더 복잡한 4x4 타일링 예제가 모두 포함되어 있습니다.
You can use these examples as an initial reference when building your custom op.
사용자 정의 작업을 구축할 때 이러한 예를 초기 참조로 사용할 수 있습니다.
