<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Clock Tree Tracing Skill
# 클록 트리 추적 기술

## Summary
## 요약
This skill captures how to trace a clock tree from an output pin back to where the source clock constraint should be defined.
이 기술은 출력 핀에서 소스 클럭 제약 조건이 정의되어야 하는 위치까지 클럭 트리를 추적하는 방법을 캡처합니다.

## Key Concepts
## 주요 개념

### Valid Clock Definition Objects
### 유효한 시계 정의 객체
1. A port
1. 항구
2. An output pin that has no timing arcs
2. 타이밍 아크가 없는 출력 핀


# Timing arc relationship
# 타이밍 아크 관계

## Essential Tcl Commands
## 필수 Tcl 명령

### 1.
### Trace using timing arcs
### 타이밍 아크를 사용한 추적
**Preferred:** Trace using timing arcs is preferred inside Vivado
**선호:** Vivado 내부에서는 타이밍 아크를 사용한 트레이스가 선호됩니다.

**Example 1:** Tracing back to pins without a timing arc or port from an output pin
**예 1:** 출력 핀에서 타이밍 아크나 포트가 없는 핀으로 역추적
```tcl
proc trace_clock_back_to_source {pin} {
    set debug 1
    set pin [get_pins $pin]
    if {[get_property DIRECTION $pin] eq "OUT"} {
        set output_pins $pin
        if {$debug==1} {puts "-D: Starting trace from output pin $output_pins"}
    } else {
        set output_pins [get_pins -leaf -of [get_nets -of $pin] -filter {DIRECTION==OUT}]
        if {[llength $output_pins] == 0} {return [list $pin pin]}
    }
    
    set visited_pins {}
    set max_visits 5
    set pin_visit_count [dict create]
    
    while {1} {
        if {[llength [set arcs [get_timing_arcs -filter {TYPE=="Reg Clk to Q"||TYPE=="combinational"} -to $output_pins]]] == 0} {return [list $output_pins pin]}
        
        # Check if pin has been visited too many times
        if {[dict exists $pin_visit_count $output_pins]} {
            set count [dict get $pin_visit_count $output_pins]
            if {$count >= $max_visits} {return [list $output_pin pin]}
            dict set pin_visit_count $output_pins [expr {$count + 1}]
        } else {
            dict set pin_visit_count $output_pins 1
        }
        
        set input_pins [get_pins [get_property FROM_PIN $arcs]]
        set net [get_nets -parent -of $input_pins]
        set output_pins [get_pins -leaf -filter {DIRECTION==OUT} -of $net]
        if {[llength $output_pins] == 0} {
            if {[llength [set ports [get_ports -of $net]]] > 0} {return [list $ports port]}
        }
    }
}

set output_pin [get_pins ios_0/mmcm_0/CLKOUT2]
set sp [trace_clock_back_to_source $output_pin]

```

### DO:
✅ Consider there could be multiple timing arcs per pin
✅ 핀당 여러 개의 타이밍 아크가 있을 수 있다는 점을 고려하세요.
✅ Consider there could be multiple clocks per pin
✅ 핀당 여러 개의 클럭이 있을 수 있다는 점을 고려하세요.

### DO NOT:
### 하지 말아야 할 것:
❌ Run indefinitely in a loop.
❌ 루프에서 무한정 실행됩니다.
Break after iterating on the same object
동일한 객체를 반복한 후 중단

## References
## 참고자료

### Xilinx Documentation
### 자일링스 문서
- **UG906**: Vivado Design Suite User Guide: Design Analysis and Closure Techniques
- **UG906**: Vivado Design Suite 사용자 가이드: 설계 분석 및 폐쇄 기술
  - [Invalid Primary Clock on CMB](https://docs.xilinx.com/r/en-US/ug906-vivado-design-analysis/Example)
  - [CMB의 잘못된 기본 시계](https://docs.xilinx.com/r/en-US/ug906-vivado-design-analytic/Example)
