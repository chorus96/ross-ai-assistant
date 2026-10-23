<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Summary
# 요약
This page captures reporting clock differences either between before and after constraint changes or for conflicting constraints
이 페이지는 제약 조건 변경 전후 또는 제약 조건 충돌에 대한 보고 클럭 차이를 캡처합니다.

# Example
# 예
#### Before and After Comparison
#### 전후 비교

| Aspect<br>측면 | Period<br>기간 | Waveform<br>파형 | Clock Definition Object<br>시계 정의 객체 | Constraint<br>제약 |
|--------|--------|----------|------------------------|------------|
| **Before**<br>**이전** | 5.000 ns | {0 2.500} | GTX_CLK (primary)<br>GTX_CLK(기본) | /`create_clock -period 5.000 -name GTX_CLK [get_pins ios_0/mmcm_0/CLKOUT2]/`<br>/`create_clock -기간 5.000 -name GTX_CLK [get_pins ios_0/mmcm_0/CLKOUT2]/` |
| **After**<br>**이후** | 12.800 ns | {0 6.400} | GTX_CLK (generated)<br>GTX_CLK(생성됨) | /`create_generated_clock -name GTX_CLK -source [get_pins ios_0/mmcm_0/CLKIN1] -multiply_by 30 -divide_by 48 [get_pins ios_0/mmcm_0/CLKOUT2]/`<br>/`create_generated_clock -name GTX_CLK -source [get_pins ios_0/mmcm_0/CLKIN1] -multiply_by 30 -divide_by 48 [get_pins ios_0/mmcm_0/CLKOUT2]/` |


#### Conflicting constraints
#### 제약조건 충돌

| Aspect<br>측면 | Period<br>기간 | Waveform<br>파형 | Clock Definition Object<br>시계 정의 객체 | Constraint<br>제약 |
|--------|--------|----------|------------------------|------------|
| **Primary**<br>**기본** | 5.000 ns | {0 2.500} | GTX_CLK (primary)<br>GTX_CLK(기본) | /`create_clock -period 5.000 -name GTX_CLK [get_ports CLKIN]/`<br>/`create_clock -기간 5.000 -name GTX_CLK [get_ports CLKIN]/` |
| **Redefinition#1**<br>**재정의#1** | 12.800 ns | {0 6.400} | NEW_CLK (generated)<br>NEW_CLK(생성됨) | /`create_clock -name NEW_CLK -period 12.800 [get_pins ios_0/mmcm_0/CLKOUT2]/`<br>/`create_clock -name NEW_CLK -기간 12.800 [get_pins ios_0/mmcm_0/CLKOUT2]/` |
