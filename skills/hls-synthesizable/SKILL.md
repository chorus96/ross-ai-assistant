<!-- ko-translation: enabled -->
---
name: hls-synthesizable
description: 'Analyze whether user code is synthesizable for HLS'
license: MIT
argument-hint: "[<TOP_FUNCTION — top-level HLS function name e.g. 'Kernel'>]"
metadata:
  author: "Alex Li"
  version: "1.0.0"
---
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
You are focused on analyzing user's code to determine whether it is synthesizable.
사용자의 코드를 분석하여 합성 가능한지 여부를 결정하는 데 중점을 둡니다.
Strictly follow our rules 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 and do not exert yourself.
규칙 1, 2, 3, 4, 5, 6, 7, 8, 9, 10을 엄격히 따르고 너무 무리하지 마십시오.

#### Concepts:
#### 개념:
- The **top function** is the top-level HLS function, get it via argument `<TOP_FUNCTION>`, if not provided, call `/hls-component-basic-info` skill to get top_function which is the top-level function name
- **top 함수**는 최상위 HLS 함수입니다. `<TOP_FUNCTION>` 인수를 통해 가져옵니다. 제공되지 않은 경우 `/hls-comComponent-basic-info` 스킬을 호출하여 최상위 함수 이름인 top_function을 가져옵니다.

#### Scope:
#### 범위:
- Do NOT check code in test bench files.
- 테스트 벤치 파일의 코드를 확인하지 마십시오.
- Do NOT check code in the `main` function.
- `main` 함수의 코드를 확인하지 마세요.
- ONLY check code in the **top function** and code that is **called by the top function**.
- **상위 함수**의 코드와 **상위 함수에 의해 호출되는 코드**만 확인하세요.
- Otherwise, check the code step by step.
- 그렇지 않으면 코드를 단계별로 확인하세요.

#### Analysis Rules:
#### 분석 규칙:

1. **System Calls** — Functions like `printf()`, `fprintf(stdout,)`, `getc()`, `time()`, `sleep()`, `cin`, `cout`... are system calls.
1. **시스템 호출** — `printf()`, `fprintf(stdout,)`, `getc()`, `time()`, `sleep()`, `cin`, `cout`...과 같은 함수는 시스템 호출입니다.
If there is a system call in the code, validate that it is guarded by:
코드에 시스템 호출이 있는 경우 다음을 통해 보호되는지 확인하세요.
   ```cpp
   #ifndef __SYNTHESIS__
   #endif
   ```
1.1 If the system call is in some other macro guard, it is valid, but suggest to the user that they should use:
1.1 시스템 호출이 다른 매크로 가드에 있는 경우 유효하지만 사용자에게 다음을 사용해야 한다고 제안합니다.
   ```cpp
   #ifndef __SYNTHESIS__
   #endif
   ```

2. **Dynamic Memory** — Functions like `malloc()`, `alloc()`, `free()`, `new`, `delete`... are dynamic memory usage.
2. **동적 메모리** — `malloc()`, `alloc()`, `free()`, `new`, `delete`...와 같은 함수는 동적 메모리 사용량입니다.
If there is dynamic memory usage in the code, validate that it is guarded by:
코드에 동적 메모리 사용량이 있는 경우 다음을 통해 보호되는지 확인하세요.
   ```cpp
   #ifndef __SYNTHESIS__
   #endif
   ```
2.1 If the dynamic memory usage is in some other macro guard, it is valid, but suggest to the user that they should use:
2.1 동적 메모리 사용량이 다른 매크로 가드에 있는 경우 이는 유효하지만 사용자에게 다음을 사용해야 한다고 제안합니다.
   ```cpp
   #ifndef __SYNTHESIS__
   #endif
   ```

3. **No Virtual Functions** — Validate that there are no virtual functions.
3. **가상 기능 없음** — 가상 기능이 없는지 확인합니다.

4. **No Recursive Functions** — Validate that there are no recursive functions.
4. **재귀 함수 없음** — 재귀 함수가 없는지 확인합니다.

5. **STL Whitelist** — Only the following C++ Standard Template Library (STL) functions are allowed.
5. **STL 화이트리스트** — 다음 C++ 표준 템플릿 라이브러리(STL) 함수만 허용됩니다.
5. Any STL function NOT in this whitelist is forbidden:
5. 이 화이트리스트에 없는 모든 STL 기능은 금지됩니다.
   - std::abs, std::arg, std::norm, std::conj, std::cos, std::cosh, std::exp, std::log, std::log10, std::pow
   - std::abs, std::arg, std::norm, std::conj, std::cos, std::cosh, std::exp, std::log, std::log10, std::pow
   - std::sin, std::sinh, std::sqrt, std::tan, std::tanh, std::acos, std::asin, std::atan, std::acosh
   - std::sin, std::sinh, std::sqrt, std::tan, std::tanh, std::acos, std::asin, std::atan, std::acosh
   - std::asinh, std::atanh, std::fabs, std::max, std::min, std::complex, std::operator, std::real, std::imag
   - std::asinh, std::atanh, std::fabs, std::max, std::min, std::complex, std::operator, std::real, std::imag
   - std::norm, std::pow, std::proj, std::polar, std::conj, std::fpclassify, std::isfinite, std::isinf
   - std::norm, std::pow, std::proj, std::폴라, std::conj, std::fpclassify, std::isfinite, std::isinf
   - std::isnan, std::isnormal, std::signbit, std::isgreater, std::isgreaterequal, std::isless
   - std::isnan, std::isnormal, std::signbit, std::isgreater, std::isgreaterequal, std::isless
   - std::islessequal, std::islessgreater, std::isunordered, std::div, std::atan2, std::ceil, std::floor
   - std::islessequal, std::islessgreater, std::isunordered, std::div, std::atan2, std::ceil, std::floor
   - std::fmod, std::frexp, std::ldexp, std::modf, std::cbrt, std::copysign, std::erfc, std::exp2
   - std::fmod, std::frexp, std::ldexp, std::modf, std::cbrt, std::copysign, std::erfc, std::exp2
   - std::expm1, std::fdim, std::fma, std::fmax, std::fmin, std::hypot, std::ilogb, std::lgamma
   - std::expm1, std::fdim, std::fma, std::fmax, std::fmin, std::hypot, std::ilogb, std::lgamma
   - std::llrint, std::llround, std::log1p, std::log2, std::logb, std::lrint, std::lround, std::nan
   - std::llrint, std::llround, std::log1p, std::log2, std::logb, std::lrint, std::lround, std::nan
   - std::nearbyint, std::nextafter, std::nexttoward, std::remainder, std::remquo, std::rint, std::round
   - std::nearbyint, std::nextafter, std::nexttoward, std::remainder, std::remquo, std::rint, std::round
   - std::scalbln, std::scalbn, std::tgamma, std::trunc, std::__complex_sqrt, std::__complex_pow_unsigned
   - std::scalbln, std::scalbn, std::tgamma, std::trunc, std::__complex_sqrt, std::__complex_pow_unsigned
   - std::__complex_atan, std::__complex_acosh, std::__complex_asinh, std::__complex_atanh
   - std::__complex_atan, std::__complex_acosh, std::__complex_asinh, std::__complex_atanh
   - std::__complex_proj, std::equal_to, std::not_equal_to, std::less, std::greater, std::greater_equal
   - std::__complex_proj, std::equal_to, std::not_equal_to, std::less, std::greater, std::greater_equal
   - std::less_equal
   - 표준::less_equal

6. **No `long double` Type** — Validate that the types `long double` are forbidden.
6. **`long double` 유형 없음** — `long double` 유형이 금지되어 있는지 확인합니다.

7. **No Function Pointers** — Validate that there are no function pointers.
7. **함수 포인터 없음** — 함수 포인터가 없는지 확인합니다.

8. **No Virtual Function Pointers** — Validate that there are no virtual function pointers.
8. **가상 함수 포인터 없음** — 가상 함수 포인터가 없는지 확인합니다.

9. **No Pointer to Pointer** — Validate that there are no pointer-to-pointer usages.
9. **포인터 간 포인터 없음** — 포인터 간 사용이 없는지 확인합니다.

10. **Arbitrary Precision Type Pointer Casting** — `ap_int`, `ap_uint`, `ap_fixed`, `ap_ufixed`, `ap_float`, `half` are arbitrary precision types.
10. **임의 정밀도 유형 포인터 캐스팅** — `ap_int`, `ap_uint`, `ap_fixed`, `ap_ufixed`, `ap_float`, `half`는 임의 정밀도 유형입니다.
    - Only pointer casting between native C/C++ types is allowed.
    - 네이티브 C/C++ 유형 간의 포인터 캐스팅만 허용됩니다.
    - Pointer casting between arbitrary precision types is forbidden.
    - 임의의 정밀도 유형 간의 포인터 캐스팅은 금지됩니다.
    - Pointer casting between an arbitrary precision type and a C/C++ native type is forbidden.
    - 임의 정밀도 유형과 C/C++ 기본 유형 간의 포인터 캐스팅은 금지됩니다.

#### Verdict:
#### 평결:
If **all** criteria are met, the code forms a **synthesizable subset**.
**모든** 기준이 충족되면 코드는 **합성 가능한 하위 집합**을 형성합니다.
Otherwise, it is **not synthesizable**.
그렇지 않으면 **합성할 수 없습니다**.

