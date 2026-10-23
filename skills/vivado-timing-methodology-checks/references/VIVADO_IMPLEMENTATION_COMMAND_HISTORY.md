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
This explains how to see previous commands that have been run in Vivado.
Vivado에서 실행되었던 이전 명령어를 확인하는 방법을 설명합니다.

# Inside Vivado
# 비바도 내부
When running a command inside vivado use:
vivado 내에서 명령을 실행할 때 다음을 사용하십시오.
```tcl
report_design_analysis -qor_summary
```
**Example:**
**예:**

```
1. Tool Option Summary
----------------------

+--------------+--------------------------------+------------+
|   Task Name  |             Options            | Directives |
+--------------+--------------------------------+------------+
| synth_design | xcvc1902-viva1596-1LP-e-S      |            |
| opt_design   |                                | Explore    |
| opt_design   | -muxf_remap -control_set_merge |            |
+--------------+--------------------------------+------------+
* Data is available only for the fields shown and not collected for all fields.
```

In this example the following commands were run:
이 예에서는 다음 명령이 실행되었습니다.
* `synth_design -part xcvc1902-viva1596-1LP-e-S`
* `synth_design -part xcvc1902-viva1596-1LP-e-S`
* `opt_design -directive Explore`
* `opt_design -지시문 탐색`
* `opt_design -muxf_remap -control_set_merge`
* `opt_design -muxf_remap -control_set_merge`

### DO:
✅ Understand the part being targeted
✅ 타겟이 되는 부분을 이해하세요
✅ Understand the default options for each part.
✅ 각 부분의 기본 옵션을 이해하세요.
