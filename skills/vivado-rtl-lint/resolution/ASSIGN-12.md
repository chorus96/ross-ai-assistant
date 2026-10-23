<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-12: Unconnected/Tri-State Port Connection
# ASSIGN-12: 연결되지 않음/삼중 상태 포트 연결

**Rule:** ASSIGN-12 | **Severity:** CRITICAL WARNING | **Standard:** IEEE 1364-2005 §12.3.3
**규칙:** ASSIGN-12 | **심각도:** 심각한 경고 | **표준:** IEEE 1364-2005 §12.3.3

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 8-XXXX] [ASSIGN-12]
Port 'port_name' of instance 'instance_name' is connected to tri-state value (1'bz).
```

## Root Cause
## 근본 원인

Module instance port connected to `1'bz` (high-impedance).
'1'bz'(고임피던스)에 연결된 모듈 인스턴스 포트입니다.
Tri-state is only valid for bidirectional I/O pads — never for internal logic ports.
3상태는 양방향 I/O 패드에만 유효하며 내부 로직 포트에는 유효하지 않습니다.

**Common sources:** Copy-paste placeholder, incomplete instantiation, misunderstanding of tri-state usage
**공통 소스:** 복사-붙여넣기 자리 표시자, 불완전한 인스턴스화, 3상태 사용에 대한 오해

## Fix Options
## 수정 옵션

### Option 1: Tie to Constant (Recommended)
### 옵션 1: 상수에 연결(권장)

```verilog
// Before
sub u2 (.clk(clk), .din(1'bz), .dout(dout[1]));     // Tri-state

// After
sub u2 (.clk(clk), .din(1'b0), .dout(dout[1]));     // Tied to logic 0
```
Choose `1'b0` or `1'b1` based on sub-module reset/default requirements.
하위 모듈 재설정/기본 요구 사항에 따라 `1'b0` 또는 `1'b1`을 선택합니다.

### Option 2: Connect to Signal (Multi-instance designs)
### 옵션 2: Signal에 연결(다중 인스턴스 설계)

```verilog
genvar i;
generate
   for (i = 0; i < 2; i = i + 1) begin : channel
      sub u (.clk(clk), .din(din[i]), .dout(dout[i]));
   end
endgenerate
```

### Option 3: Remove Unused Instance
### 옵션 3: 사용하지 않는 인스턴스 제거

If the instance itself is unnecessary, remove it entirely and adjust the module interface.
인스턴스 자체가 불필요한 경우에는 완전히 제거하고 모듈 인터페이스를 조정합니다.

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-12 count = 0
```

## References
## 참고자료
**Xilinx:** UG901 Ch.2 "Port Connection Rules", UG906 Appendix B
**Xilinx:** UG901 Ch.2 "포트 연결 규칙", UG906 부록 B
**IEEE:** 1364-2005 §12.3.3, 1800-2017 §23.3.2
**IEEE:** 1364-2005 §12.3.3, 1800-2017 §23.3.2
