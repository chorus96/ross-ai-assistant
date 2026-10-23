<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# RESET-2: Incomplete Asynchronous Reset Coverage
# RESET-2: 불완전한 비동기식 재설정 적용 범위

**Rule:** RESET-2 | **Severity:** WARNING | **Standard:** UG901 Ch.2
**규칙:** RESET-2 | **심각도:** 경고 | **표준:** UG901 Ch.2

## Vivado Message
## 비바도 메시지
```
WARNING: [Synth 8-XXX] [RESET-2] Register 'temp' in always/process block
lacks asynchronous reset while other registers have it. File 'top.v', Line 30.
```

## Root Cause
## 근본 원인

Register(s) in async reset block lack reset assignment while other registers in the same block are reset — causing inconsistent reset behavior, X-states in simulation, and unpredictable power-on values.
비동기 재설정 블록의 레지스터에는 재설정 할당이 부족하고 동일한 블록의 다른 레지스터는 재설정되어 일관되지 않은 재설정 동작, 시뮬레이션의 X 상태 및 예측할 수 없는 전원 켜기 값이 발생합니다.

**Common mistakes:** Copy-paste without updating reset, incremental signal additions, partial bit reset
**일반적인 실수:** 업데이트 재설정 없이 복사-붙여넣기, 증분 신호 추가, 부분 비트 재설정

**Rule:** All registers in same block must have **consistent reset behavior** (all async OR all sync, never mixed).
**규칙:** 동일한 블록의 모든 레지스터는 **일관된 재설정 동작**을 가져야 합니다(모두 비동기 또는 모두 동기화, 절대 혼합되지 않음).

## Fix: Complete Async Reset Coverage
## 수정: 완전한 비동기 재설정 적용 범위

```verilog
// Before: RESET-2 violation
always @(posedge clk or posedge rst) begin
   if(rst) begin
      out1 <= 1'b0;     // Has async reset
      out2 <= 1'b0;     // Has async reset
                        // temp missing!
   end else begin
      out1 <= in1;
      out2 <= temp;
      temp <= in2;      // Assigned but not reset
   end
end

// After: Complete coverage
always @(posedge clk or posedge rst) begin
   if(rst) begin
      out1 <= 1'b0;
      out2 <= 1'b0;
      temp <= 1'b0;     // FIXED: Added async reset
   end else begin
      out1 <= in1;
      out2 <= temp;
      temp <= in2;
   end
end
```

### Multi-bit Registers
### 다중 비트 레지스터
```verilog
// WRONG: Partial reset
if(rst) counter[7:4] <= 4'b0;  // Upper bits only!

// RIGHT: Complete reset
if(rst) counter <= 8'b0;       // Full width
```

## VHDL Pattern
## VHDL 패턴

```vhdl
-- Before: q1_r missing from reset
process(clk, rst) begin
   if(rst = '1') then
      q <= '0'; q1 <= '0';           -- q1_r missing!
   elsif rising_edge(clk) then
      q <= d; q1 <= q1_r; q1_r <= not d;
   end if;
end process;

-- After: Complete coverage
process(clk, rst) begin
   if(rst = '1') then
      q <= '0'; q1 <= '0'; q1_r <= '0';  -- All registers reset
   elsif rising_edge(clk) then
      q <= d; q1 <= q1_r; q1_r <= not d;
   end if;
end process;
```

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint
# Verify: RESET-2 = 0
report_utilization -hierarchical
# Expect: All same FF type (FDRE/FDCE) — no mixed primitives
```

## References
## 참고자료
**Xilinx:** UG901 Ch.2 "Reset Coding Styles", UG906 Appendix B, UG949 "Reset Distribution", AR# 53273
**Xilinx:** UG901 Ch.2 "코딩 스타일 재설정", UG906 부록 B, UG949 "배포 재설정", AR# 53273
**IEEE:** 1364-2005 §9.2.2, 1800-2017 §9.4.2, 1076-2008 §8.1
**IEEE:** 1364-2005 §9.2.2, 1800-2017 §9.4.2, 1076-2008 §8.1
**Related:** RESET-1 (multiple async resets), RESET-3 (incomplete sync reset)
**관련:** RESET-1(다중 비동기 재설정), RESET-3(불완전한 동기화 재설정)
