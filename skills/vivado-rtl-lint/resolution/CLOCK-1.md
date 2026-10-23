<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# CLOCK-1: Mixed Clock Edges
# CLOCK-1: 혼합 클록 에지

**Rule:** CLOCK-1 | **Severity:** CRITICAL WARNING | **Standard:** UG901 Ch.3
**규칙:** CLOCK-1 | **심각도:** 심각한 경고 | **표준:** UG901 Ch.3

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 37-94] [CLOCK-1]
Module '(module_name)' has registers with different clock edges.
Clock name: '(clock_signal)'. First register: '(reg1)'. Second register: '(reg2)'.
```

## Root Cause
## 근본 원인

Module has registers clocked by both `posedge` and `negedge` of the same clock.
모듈에는 동일한 클록의 'posege'와 'negedge' 모두에 의해 클록된 레지스터가 있습니다.
Increases control sets, reduces placement flexibility, and complicates timing analysis.
제어 세트를 늘리고 배치 유연성을 감소시키며 타이밍 분석을 복잡하게 만듭니다.

**Common sources:** Unintentional mixing (99% of cases), copy-paste from different timing domain
**공통 소스:** 의도하지 않은 혼합(99% 사례), 다른 타이밍 도메인에서 복사하여 붙여넣기

## Fix: Change Edge (One-line change)
## 수정: Edge 변경(한 줄 변경)

Change `negedge` → `posedge` on the offending always block.
문제가 있는 항상 블록에서 `negedge` → `posedge`를 변경하세요.
Do NOT merge blocks or change assignment styles — keep the fix minimal.
블록을 병합하거나 할당 스타일을 변경하지 말고 수정 사항을 최소화하십시오.

```diff
  always@(posedge clk)
    y1 = a;

- always@(negedge clk)      // Wrong edge
+ always@(posedge clk)      // FIXED: unified to posedge
    y2 <= b;                // Keep original assignment style
```

**Bit-slicing pattern:**
**비트 분할 패턴:**
```diff
  always@(posedge clk) y[3:0] <= a[3:0];
- always@(negedge clk) y[7:4] <= a[7:4];
+ always@(posedge clk) y[7:4] <= a[7:4];  // FIXED
```

### True DDR Design (Rare)
### 진정한 DDR 설계(희귀)

If both edges are intentionally required for DDR I/O, use ODDR/IDDR primitives instead of mixed always blocks:
DDR I/O에 의도적으로 두 에지가 모두 필요한 경우 항상 혼합된 블록 대신 ODDR/IDDR 기본 형식을 사용하세요.
```verilog
ODDR #(.DDR_CLK_EDGE("OPPOSITE_EDGE")) ODDR_inst (
   .Q(ddr_out), .C(clk), .D1(posedge_data), .D2(negedge_data), ...);
```
See UG953 (7-Series) or UG974 (UltraScale) for primitive details.
기본 세부 사항은 UG953(7 시리즈) 또는 UG974(UltraScale)를 참조하십시오.

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: CLOCK-1 count = 0
report_control_sets -verbose
# Before: 2 control sets (posedge + negedge). After: 1
```

## References
## 참고자료
**Xilinx:** UG901 Ch.3 "Clocking Guidelines", UG949 "Control Set Optimization"
**Xilinx:** UG901 Ch.3 "클러킹 지침", UG949 "컨트롤 세트 최적화"
**Related:** CDC-1 (different clocks, not different edges)
**관련:** CDC-1(다른 클럭, 다른 에지가 아님)
