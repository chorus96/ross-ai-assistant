<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# Custom Op example: 1-D convolution
# Custom Op 예: 1차원 컨볼루션

The examples in this directory demonstrate a 1-D, single-channel convolution
이 디렉토리의 예는 1D, 단일 채널 컨볼루션을 보여줍니다.
implemented as a custom op.
맞춤형 작전으로 구현되었습니다.
The operator computes, for each output index `o`:
연산자는 각 출력 인덱스 'o'에 대해 다음을 계산합니다.

```
out[o] = sum_{k=0..K-1} in[o + k] * kernel[k]
```

This is the same operation as a stride-1, non-padded 1-D convolution with a
이는 stride-1, 패딩되지 않은 1D 컨볼루션과 동일한 작업입니다.
single input/output channel, equivalent to ONNX `Conv` with a 1-D kernel.
단일 입력/출력 채널, 1D 커널이 있는 ONNX `Conv`와 동일합니다.

The shape convention used in the examples is `out[L_out] = in[L_in] *
예제에 사용된 모양 규칙은 `out[L_out] = in[L_in] *입니다.
kernel[K]` with `L_out = L_in - K + 1`.
커널[K]`와 `L_out = L_in - K + 1`.

This tutorial is used as an example in the [tiling guide](../../tiling.md).
이 튜토리얼은 [타일링 가이드](../../tiling.md)의 예로 사용됩니다.
