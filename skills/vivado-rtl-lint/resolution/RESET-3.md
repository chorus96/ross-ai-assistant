<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# RESET-3: Incomplete Synchronous Reset Coverage
# RESET-3: 불완전한 동기식 재설정 범위

**Rule:** RESET-3 | **Severity:** CRITICAL WARNING | **Standard:** UG901 Ch.2
**규칙:** RESET-3 | **심각도:** 심각한 경고 | **표준:** UG901 Ch.2

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 8-XXX] [RESET-3] Register 'out2' lacks reset assignment
while other registers in same block are reset. File 'top.v', Line 25.
```

## Root Cause
## 근본 원인

Register in synchronous reset block lacks reset assignment while other registers in the same block are reset — causing unpredictable initial state and non-deterministic post-reset behavior.
동기식 재설정 블록의 레지스터에는 재설정 할당이 없지만 동일한 블록의 다른 레지스터는 재설정되어 예측할 수 없는 초기 상태와 비결정적인 재설정 후 동작이 발생합니다.

**Common mistakes:** Copy-paste without updating reset, **missing `begin...end`** (only first statement runs), incremental signal additions
**일반적인 실수:** 업데이트 재설정 없이 복사-붙여넣기, **`begin...end` 누락**(첫 번째 명령문만 실행), 증분 신호 추가

## Fix: Add Reset for All Registers
## 수정: 모든 레지스터에 재설정 추가

```verilog
// Before: RESET-3 violation
always @(posedge clk) begin
   if(rst)
      out1 <= 0;        // out1 reset
   else if (en) begin
      out1 <= in1;
      out2 <= in2;      // out2 NOT reset
   end 
end

// After: All registers reset (note begin...end)
always @(posedge clk) begin
   if(rst) begin         // begin...end for multiple statements!
      out1 <= 0;
      out2 <= 0;         // FIXED: out2 now reset
   end
   else if (en) begin
      out1 <= in1;
      out2 <= in2;
   end 
end
```

**Critical pitfall:** Without `begin...end`, only the first statement after `if(rst)` executes — remaining registers silently miss reset.
**중요한 함정:** `begin...end`가 없으면 `if(rst)` 이후의 첫 번째 문만 실행됩니다. 나머지 레지스터는 자동으로 재설정을 놓치게 됩니다.

## VHDL Pattern
## VHDL 패턴

```vhdl
-- Before: out2 not reset
process(clk) begin
   if rising_edge(clk) then
      if rst = '1' then
         out1 <= '0';     -- Only out1 reset
      elsif en = '1' then
         out1 <= in1;
         out2 <= in2;     -- out2 NOT reset
      end if;
   end if;
end process;

-- After: Both registers reset
process(clk) begin
   if rising_edge(clk) then
      if rst = '1' then
         out1 <= '0';
         out2 <= '0';     -- FIXED
      elsif en = '1' then
         out1 <= in1;
         out2 <= in2;
      end if;
   end if;
end process;
```

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint
# Verify: RESET-3 = 0
```

## References
## 참고자료
**Xilinx:** UG901 Ch.2 "Synchronous Reset Coding", UG906 Appendix B, AR# 52988
**Xilinx:** UG901 Ch.2 "동기 재설정 코딩", UG906 부록 B, AR# 52988
**Related:** RESET-1 (multiple async resets), RESET-2 (incomplete async reset)
**관련:** RESET-1(다중 비동기 재설정), RESET-2(불완전한 비동기 재설정)
