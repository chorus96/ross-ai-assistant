<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# INFER-1: Latch Inference
# INFER-1: 래치 추론

**Rule:** INFER-1 | **Severity:** CRITICAL WARNING | **Standard:** UG901 Ch.4.3.1
**규칙:** 추론-1 | **심각도:** 심각한 경고 | **표준:** UG901 Ch.4.3.1

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 8-327] [INFER-1] Latch inferred for signal 'out1_reg'.
RTL Name 'out1', Hierarchy 'top', File 'top.v', Line 11.
```
**Parse:** Strip `_reg` suffix from signal name for RTL correlation.
**분석:** RTL 상관 관계를 위해 신호 이름에서 '_reg' 접미사를 제거합니다.

## Root Cause
## 근본 원인

Combinational `always@(*)` blocks with incomplete signal assignments create unintended latches — causing timing failures, glitch propagation, and simulation-synthesis mismatch.
신호 할당이 불완전한 조합형 'always@(*)' 블록은 의도하지 않은 래치를 생성하여 타이밍 오류, 글리치 전파 및 시뮬레이션-합성 불일치를 유발합니다.

## Pattern Recognition Table
## 패턴 인식 테이블

| ID | Pattern<br>패턴 | Root Cause<br>근본 원인 |
|----|---------|------------|
| P1 | `if(c) s<=v;` (no else)<br>`if(c) s<=v;` (다른 것은 없음) | Missing else<br>그 외 누락 |
| P2 | `case(s) ... endcase` (no default)<br>`case(s) ... endcase` (기본값 없음) | Missing default<br>기본값 누락 |
| P3 | `else s=s;`<br>`그렇지 않으면 s=s;` | Self-reference<br>자기 참조 |
| P4 | `if(e) begin r1=a; r2=b; end`<br>`if(e) 시작 r1=a; r2=b; 끝` | Variable assigned in one path only<br>하나의 경로에만 할당된 변수 |
| P5 | `assign o=e?v:o;`<br>`o=e 할당?v:o;` | Continuous assign self-ref<br>연속 할당 자체 참조 |
| P6 | `o<=e?r:i; r<=e?o:i2;`<br>`o<=e?r:i; r<=e?o:i2;` | Circular dependency<br>순환 의존성 |
| P7 | `o<=0; case(s)... endcase`<br>`o<=0; 케이스... 엔드케이스` | Pre-init false security<br>사전 초기화 허위 보안 |
| P8 | `output reg o=0; ... case`<br>`출력 레지스터 o=0; ... 사건` | Port init false security<br>포트 초기화 허위 보안 |
| P9 | `o[3:2]<=0; case ... o<=v`<br>`o[3:2]<=0; 경우 ... o<=v` | Partial bits + incomplete<br>부분 비트 + 불완전 |
| P10 | Comb block + unused clk<br>콤블럭 + 미사용 clk | Sequential intent unclear<br>순차적 의도 불분명 |
| P11 | `<=` in `always@(*)`<br>`always@(*)`의 `<=` | Wrong assignment type<br>잘못된 할당 유형 |
| P12 | UDP with hold state (`-`)<br>보류 상태(`-`)가 있는 UDP | Level-sensitive primitive latch<br>레벨 감지 기본 래치 |

**Critical notes:**
**중요 사항:**
- **P7/P8:** Pre-initialization does NOT prevent latches — must have explicit `else`/`default`
- **P7/P8:** 사전 초기화는 래치를 방지하지 않습니다. 명시적인 `else`/`default`가 있어야 합니다.
- **P3/P5:** Self-reference (`sig=sig` or `?v:sig`) creates same latch as missing else
- **P3/P5:** 자체 참조(`sig=sig` 또는 `?v:sig`)는 누락된 것과 동일한 래치를 생성합니다.
- **P12:** UDP hold state (`-`) is level-sensitive latch — change to constant or edge notation
- **P12:** UDP 홀드 상태(`-`)는 레벨 감지 래치입니다. 상수 또는 에지 표기법으로 변경됩니다.
- **P1/P2:** Cover ~80% of cases — check these first
- **P1/P2:** 사례의 최대 80%에 해당 - 먼저 확인하세요.

## Fix Options
## 수정 옵션

### Option 1: Complete Combinational Coverage (Default — no value retention needed)
### 옵션 1: 완전한 조합 보장(기본값 - 가치 보존이 필요하지 않음)

```verilog
// P1 fix: Missing else
always@(*) begin
   if(en) out1 = in1;
   else   out1 = 1'b0;    // Add explicit else
end

// P2 fix: Missing default
always@(*) begin
   case(sel)
      2'b00: out = a;  2'b01: out = b;  2'b10: out = c;
      default: out = 1'b0;              // Add default
   endcase
end

// P3/P5 fix: Self-reference
// WRONG: else out = out;   ← still a latch!
// RIGHT: else out = 1'b0;  ← constant, no latch
```

**Rules:** Use blocking `=` (not `<=`) in combinational blocks.
**규칙:** 조합 블록에서는 `=` 차단(`<=` 아님)을 사용하세요.
Never self-reference.
절대 자기 참조를 하지 마세요.
Use `always_comb` (SV) for compile-time latch detection.
컴파일 타임 래치 감지에는 'always_comb'(SV)을 사용하세요.

### Option 2: Convert to Sequential (Value retention intended)
### 옵션 2: 순차로 변환(가치 보존 의도)

```verilog
// BEFORE: always@(*) begin if(en) out1 <= in1; end    ← latch
// AFTER:
always@(posedge clk) begin
   if(en) out1 <= in1;  // Register holds when en=0
end
```

**Use when:** Signal must retain value between updates.
**사용 시기:** Signal은 업데이트 간에 가치를 유지해야 합니다.
Keep `<=` (non-blocking).
`<=`(비차단)을 유지하세요.
Add reset if needed.
필요한 경우 재설정을 추가하십시오.

### P12 Fix: UDP Latch
### P12 수정: UDP 래치

```verilog
// Option A: Change hold to clear (minimal)
table
     0   ?  : ?   : 0  ;  // Force 0 instead of hold (-)
     1   0  : ?   : 0  ;
     1   1  : ?   : 1  ;
endtable

// Option B: Convert to edge-sensitive FF
table
    (01) 0  : ?   : 0  ;  // Rising edge triggered
    (01) 1  : ?   : 1  ;
    (0?) ?  : ?   : -  ;  // Non-edge hold OK in sequential
    (?0) ?  : ?   : -  ;
endtable
```
If latch is intentional, use `create_waiver` instead.
래치가 의도적인 경우 대신 'create_waiver'를 사용하세요.

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint
# Verify: INFER-1 count = 0
get_cells -hierarchical -filter {REF_NAME =~ LD*}
# Expected: {} (no LD/LDC/LDE/LDCE primitives)
```

## References
## 참고자료
**Xilinx:** UG901 Ch.4.3.1, UG949 RTL guidelines, UG906 TIMING-10, Answer 41851
**자일링스:** UG901 Ch.4.3.1, UG949 RTL 지침, UG906 TIMING-10, 답변 41851
**IEEE:** 1364-2005 §9.5, 1800-2017 §9.2.2
**IEEE:** 1364-2005 §9.5, 1800-2017 §9.2.2
**Related:** INFER-2 (incomplete case), ASSIGN-5 (undriven signal)
**관련:** INFER-2(불완전한 경우), ASSIGN-5(구동되지 않은 신호)
