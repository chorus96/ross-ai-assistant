<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-6: Dead Code / Unused Signal
# ASSIGN-6: 데드 코드/미사용 신호

**Rule:** ASSIGN-6 | **Severity:** WARNING | **Standard:** UG901
**규칙:** ASSIGN-6 | **심각도:** 경고 | **표준:** UG901

## Vivado Message
## 비바도 메시지
```
WARNING: [Synth 37-124] [ASSIGN-6] Signal 'unread_wire' was assigned but not read. 
First unread bit index is 0. RTL Name 'unread_wire', Hierarchy 'top', File 'top.v', Line 12.
```

## Root Cause
## 근본 원인

Signal is assigned but never read.
신호가 할당되었지만 읽혀지지 않았습니다.
Dead code with no functional impact — indicates incomplete implementation, refactoring artifacts, or leftover debug logic.
기능에 영향을 주지 않는 데드 코드 - 불완전한 구현, 리팩터링 아티팩트 또는 남은 디버그 논리를 나타냅니다.

## Fix: Remove Unused Signal
## 수정: 사용하지 않는 신호 제거

1. Verify signal is truly unused (search entire file for read references)
1. 신호가 실제로 사용되지 않았는지 확인합니다(읽기 참조에 대해 전체 파일 검색).
2. Remove all assignments to the signal
2. 신호에 대한 모든 할당 제거
3. Remove signal declaration
3. 신호 선언 제거
4. Clean up empty blocks left behind
4. 남겨진 빈 블록을 정리하세요
5. Check for **signal chains**: if this signal was the only reader of another signal, that upstream signal may now trigger ASSIGN-6 too
5. **신호 체인** 확인: 이 신호가 다른 신호의 유일한 판독기인 경우 해당 업스트림 신호는 이제 ASSIGN-6도 트리거할 수 있습니다.

```diff
-wire [7:0] wire1;
-assign wire1 = b[0];      // Dead code — never read

 always@(posedge clk) begin
    result <= a + b;
 end
```

**Chain cleanup example:**
**체인 정리 예:**
```verilog
// If temp3 is unused, and temp2 feeds only temp3, and temp1 feeds only temp2:
// Remove entire chain: temp1, temp2, temp3
```

If downstream signal has other readers, stop cleanup at that point.
다운스트림 신호에 다른 판독기가 있는 경우 해당 지점에서 정리를 중지합니다.

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-6 count = 0
# Check: No new ASSIGN-5 violations from broken chains
```

## References
## 참고자료
**Xilinx:** UG901 "Unused Logic Optimization", UG949
**Xilinx:** UG901 "사용되지 않은 논리 최적화", UG949
**Related:** ASSIGN-5 (opposite: used but not assigned)
**관련:** ASSIGN-5(반대: 사용되었지만 할당되지 않음)
