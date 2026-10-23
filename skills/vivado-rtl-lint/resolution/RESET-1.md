<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# RESET-1: Multiple Asynchronous Resets
# RESET-1: 다중 비동기 재설정

**Rule:** RESET-1 | **Severity:** CRITICAL WARNING | **Standard:** UG901 Ch.2 | **Correlation:** ROAS-1
**규칙:** RESET-1 | **심각도:** 심각한 경고 | **표준:** UG901 Ch.2 | **상관관계:** ROAS-1

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 8-XXX] [RESET-1] Multiple asynchronous resets specified
for register 'reg_name'. Hierarchy 'top', File 'top.v', Line 42.
```

## Root Cause
## 근본 원인

Multiple async controls (reset/set) in sensitivity list create race conditions and unpredictable behavior.
민감도 목록의 여러 비동기 제어(재설정/설정)는 경쟁 조건과 예측할 수 없는 동작을 만듭니다.
FPGA flip-flops support a single async reset; dual async set+reset is not recommended.
FPGA 플립플롭은 단일 비동기 재설정을 지원합니다. 이중 비동기 설정+재설정은 권장되지 않습니다.

**Key rule:** Sequential sensitivity list should contain **only** clock + **one** async reset (if needed).
**핵심 규칙:** 순차 민감도 목록에는 **시계만** + **1** 비동기 재설정(필요한 경우)이 포함되어야 합니다.

## RESET-1 / ROAS-1 Correlation
## RESET-1 / ROAS-1 상관관계

| Rule<br>규칙 | Tool<br>도구 | Level<br>레벨 | Message<br>메시지 |
|------|------|-------|---------|
| RESET-1<br>리셋-1 | `synth_design -lint`<br>`synth_design -lint` | RTL sensitivity list<br>RTL 민감도 목록 | "Multiple async resets"<br>"다중 비동기 재설정" |
| ROAS-1<br>ROAS-1 | `report_methodology`<br>`보고_방법론` | Synthesized primitive<br>합성된 프리미티브 | "Both async set and reset"<br>"비동기 설정 및 재설정 모두" |

Fix both simultaneously: single async reset.
두 가지를 동시에 수정합니다. 단일 비동기 재설정.

## Fix: Single Async Reset
## 수정: 단일 비동기 재설정

```verilog
// Before: Multiple async controls
always @(posedge clk or posedge rst or posedge en1) begin  // Both rst, en1 async
   if(rst) out1 <= 4'b0;
   else if(en1) out1 <= 4'hF;     // Race if rst=1, en1=1
   else out1 <= in1;
end

// After: Single async reset
always @(posedge clk or posedge rst) begin                 // Only rst async
   if(rst) out1 <= 4'b0;
   else if(en1) out1 <= 4'hF;     // Now synchronous
   else out1 <= in1;
end
```

### Active-Low Variant
### 액티브 로우 변형
```verilog
always @(posedge clk or negedge rstn) begin
   if(!rstn) out1 <= 4'b0;
   else if(en1) out1 <= 4'hF;     // Synchronous
   else out1 <= in1;
end
```

## VHDL Pattern
## VHDL 패턴

```vhdl
-- Before: Multiple async controls
process (rst, clk, en1) begin
   if(rst = '1') then out1 <= (others => '0');
   elsif(en1 = '1') then out1 <= (others => '1');  -- Async set
   elsif rising_edge(clk) then out1 <= in1;
   end if;
end process;

-- After: Single async reset
process (clk, rst) begin
   if(rst = '1') then out1 <= (others => '0');
   elsif rising_edge(clk) then
      if(en1 = '1') then out1 <= (others => '1');  -- Now synchronous
      else out1 <= in1;
      end if;
   end if;
end process;
```

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint
# Verify: RESET-1 = 0
synth_design -top <module> -part <part> -rtl
report_methodology
# Verify: ROAS-1 = 0
report_utilization -hierarchical
# Expect: FDRE/FDCE (not FDPE — async preset indicates async set still present)
```

## References
## 참고자료
**Xilinx:** UG901 Ch.2 "Async Reset Coding", UG906 Appendix B, UG949 "Reset Strategies", AR# 52988
**Xilinx:** UG901 Ch.2 "비동기 재설정 코딩", UG906 부록 B, UG949 "재설정 전략", AR# 52988
**IEEE:** 1364-2005 §9.2, 1800-2017 §9.4
**IEEE:** 1364-2005 §9.2, 1800-2017 §9.4
**Related:** RESET-2 (incomplete async coverage), RESET-3 (incomplete sync coverage), ROAS-1
**관련:** RESET-2(불완전한 비동기 적용 범위), RESET-3(불완전한 동기화 적용 범위), ROAS-1
