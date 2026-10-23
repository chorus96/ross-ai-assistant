<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# IP_CONSTRAINTS
# IP_제약
IP constraints are important because users can not easily edit them.
IP 제약조건은 사용자가 쉽게 편집할 수 없기 때문에 중요합니다.
They are also scoped.
또한 범위가 지정됩니다.
Searching for hierarchy names needs to account for this.
계층 구조 이름을 검색할 때 이를 고려해야 합니다.

## Identifying IP constraints
## IP 제약조건 식별
There are a few ways to confirm a file is from an IP and not a user source.
파일이 사용자 소스가 아닌 IP에서 가져온 것인지 확인하는 몇 가지 방법이 있습니다.

### Scoping
### 범위 지정
IP constraints will have their own constraints file.
IP 제약조건에는 자체 제약조건 파일이 있습니다.
IP constraint files are scoped to an instance using `current_instance <hierarchy>`
IP 제약 조건 파일은 `current_instance <hierarchy>`를 사용하여 인스턴스로 범위가 지정됩니다.
Ignore `current_instance -quiet` where no hierarchy is specified.
계층 구조가 지정되지 않은 `current_instance -quiet`을 무시하십시오.

In constraints.xdc this will have a header section and a current_instance defined inside the section
Constraints.xdc에는 헤더 섹션과 섹션 내부에 정의된 current_instance가 있습니다.
```xdc
# The next 3 lines are the header
####################################################################################
# Constraints from file : 'hwtest_harness_top_clk_wizard_0_0.xdc'
####################################################################################

# The next line is NOT the current instance definition
current_instance -quiet
# The next line IS the current instance definition
current_instance hwtest_harness_top_i/Clock_Gen/clk_wizard_0/inst
create_clock -period 3.000 [get_ports -scoped_to_current_instance clk_in1]
set_input_jitter [get_clocks -of_objects [get_ports -scoped_to_current_instance clk_in1]] 0.100

```

### Hierarchy
### 계층 구조
Another way of telling if a cell is an IP is to look at the hierarchy:
셀이 IP인지 확인하는 또 다른 방법은 계층 구조를 살펴보는 것입니다.
```
hwtest_harness_top_i/Clock_Gen/clk_wizard_0/inst/clock_primitive_inst/MMCME5_inst
```
`clk_wizard_0` is the default name of the IP instance
`clk_wizard_0`은 IP 인스턴스의 기본 이름입니다.
`inst` is the hierarchy wrapper of the IP when genertated in Verilog
'inst'는 Verilog에서 생성될 때 IP의 계층 구조 래퍼입니다.
`u1` is the hierarchy wrapper of the IP when genertated in VHDL
'u1'은 VHDL에서 생성될 때 IP의 계층 구조 래퍼입니다.
`clock_primitive_inst` is next wrapper level in the clocking wizard IP
`clock_primitive_inst`는 클로킹 마법사 IP의 다음 래퍼 레벨입니다.

### Compile Order
### 컴파일 주문
IP runs run OOC will have their own fileset.
IP 실행 실행 OOC에는 자체 파일 세트가 있습니다.
Run `report_compile_order`
`report_compile_order`를 실행하세요.

```
# The following has fileset clk_wizard_0
# The constraints files are scoped to ref and scoped to cells
Constraint evaluation order for 'synthesis' with fileset 'clk_wizard_0':
Index  File Name               Used_In       Scoped_To_Ref  Scoped_To_Cells  Processing_Order  Out_Of_Context  Full Path Name
-----  ----------------------  ------------  -------------  ---------------  ----------------  --------------  ----------------------------------------------------------------------------------
1      clk_wizard_0_ooc.xdc    Synth & Impl  clk_wizard_0   inst             EARLY             yes             c:/Temp/project_28/project_28.gen/sources_1/ip/clk_wizard_0/clk_wizard_0_ooc.xdc
2      clk_wizard_0_board.xdc  Synth & Impl  clk_wizard_0   inst             EARLY                             c:/Temp/project_28/project_28.gen/sources_1/ip/clk_wizard_0/clk_wizard_0_board.xdc
3      clk_wizard_0.xdc        Synth & Impl  clk_wizard_0   inst             EARLY                             c:/Temp/project_28/project_28.gen/sources_1/ip/clk_wizard_0/clk_wizard_0.xdc
4      clk_wizard_0_late.xdc   Synth & Impl  clk_wizard_0   inst             LATE                              c:/Temp/project_28/project_28.gen/sources_1/ip/clk_wizard_0/clk_wizard_0_late.xdc
```


## IP Specific Resolutions
## IP별 해상도

### Clocking Wizard
### 클럭킹 마법사
Configuring the ip differently can remove the constraint file
IP를 다르게 구성하면 제약 조건 파일이 제거될 수 있습니다.

**Resolution:**
**해결 방법:**
1. Remove the input buffer
1. 입력 버퍼 제거
```tcl
# Example clocking wizard configuration change to remove the create clock IP constraint
create_ip -name clk_wizard -vendor xilinx.com -library ip -version 1.0 -module_name clk_wizard_0
set_property CONFIG.PRIM_SOURCE {No_buffer} [get_ips clk_wizard_0]

```
