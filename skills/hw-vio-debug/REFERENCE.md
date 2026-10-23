<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# VIO Debug — Reference Data
# VIO 디버그 - 참조 데이터

## VIO Properties (UG912 — HW_VIO)
## VIO 속성(UG912 — HW_VIO)

| Property<br>재산 | Type<br>유형 | R/W | Description<br>설명 |
|----------|------|-----|-------------|
| CORE_REFRESH_RATE_MS<br>CORE_REFRESH_RATE_MS | int<br>정수 | R/W | Auto-refresh interval (ms). 0=disabled. Recommended ≥500<br>자동 새로 고침 간격(ms)입니다. 0=비활성화됨. 권장 ≥500 |
| INSTANCE_NAME<br>INSTANCE_NAME | string<br>문자열 | R/O | RTL instance name<br>RTL 인스턴스 이름 |
| IS_ACTIVITY_SUPPORTED<br>IS_ACTIVITY_지원됨 | bool<br>부울 | R/O | Activity detection available?<br>활동 감지가 가능합니까? |

## VIO Probe Properties
## VIO 프로브 속성

**Input probes (TYPE == vio_input):**
**입력 프로브(TYPE == vio_input):**

| Property<br>재산 | R/W | Description<br>설명 |
|----------|-----|-------------|
| INPUT_VALUE<br>INPUT_VALUE | R/O | Current value (after `refresh_hw_vio`)<br>현재 값(`refresh_hw_vio` 이후) |
| ACTIVITY_VALUE<br>ACTIVITY_VALUE | R/O | Transition indicators (↑↓↕)<br>전환지표(↑↓↕) |
| ACTIVITY_PERSISTENCE<br>ACTIVITY_PERSISTENCE | R/W | INFINITE, LONG (80 samples), SHORT (8 samples)<br>INFINITE, LONG(80개 샘플), SHORT(8개 샘플) |
| INPUT_VALUE_RADIX<br>INPUT_VALUE_RADIX | R/W | BIN, OCT, HEX, UNSIGNED, SIGNED<br>BIN, OCT, HEX, 서명되지 않음, 서명됨 |

**Output probes (TYPE == vio_output):**
**출력 프로브(TYPE == vio_output):**

| Property<br>재산 | R/W | Description<br>설명 |
|----------|-----|-------------|
| OUTPUT_VALUE<br>OUTPUT_VALUE | R/W | Value to drive (commit with `commit_hw_vio`)<br>추진할 가치(`commit_hw_vio`로 커밋) |
| OUTPUT_VALUE_RADIX<br>OUTPUT_VALUE_RADIX | R/W | BIN, OCT, HEX, UNSIGNED, SIGNED<br>BIN, OCT, HEX, 서명되지 않음, 서명됨 |

## Tcl Command Quick Reference — VIO
## Tcl 명령 빠른 참조 - VIO

| Command<br>명령 | Purpose<br>목적 |
|---------|---------|
| `get_hw_vios`<br>`get_hw_vios` | List all VIO debug cores on current device<br>현재 장치의 모든 VIO 디버그 코어 나열 |
| `get_hw_probes -of_objects $vio -filter {TYPE == vio_input}`<br>`get_hw_probes -of_objects $vio -filter {TYPE == vio_input}` | List input probes<br>입력 프로브 나열 |
| `get_hw_probes -of_objects $vio -filter {TYPE == vio_output}`<br>`get_hw_probes -of_objects $vio -filter {TYPE == vio_output}` | List output probes<br>출력 프로브 나열 |
| `refresh_hw_vio $vio`<br>`refresh_hw_vio $vio` | Read input values from hardware<br>하드웨어에서 입력 값 읽기 |
| `refresh_hw_vio -update_output_values $vio`<br>`refresh_hw_vio -update_output_values $vio` | Sync output values from hardware<br>하드웨어의 출력 값 동기화 |
| `set_property OUTPUT_VALUE <val> $probe`<br>`set_property OUTPUT_VALUE <값> $프로브` | Set value on output probe<br>출력 프로브에 값 설정 |
| `commit_hw_vio $vio`<br>`commit_hw_vio $vio` | Drive output values to hardware<br>출력 값을 하드웨어로 구동 |
| `reset_hw_vio_outputs $vio`<br>`reset_hw_vio_outputs $vio` | Reset outputs to design-time initial values<br>출력을 디자인 타임 초기 값으로 재설정 |
| `reset_hw_vio_activity $vio`<br>`reset_hw_vio_activity $vio` | Clear activity detectors<br>명확한 활동 감지기 |
| `set_property CORE_REFRESH_RATE_MS <ms> $vio`<br>`set_property CORE_REFRESH_RATE_MS <ms> $vio` | Set auto-refresh interval<br>자동 새로고침 간격 설정 |

## vio_snapshot.json Schema
## vio_snapshot.json 스키마

```json
{
  "metadata": {
    "skill": "hw-vio-debug", "version": "1.0.0",
    "mode": "vio", "timestamp": "<ISO8601>",
    "device": "<part>", "vio_core": "hw_vio_1"
  },
  "input_probes": [
    { "name": "status_led", "value": "0011", "activity": "both", "width": 4 }
  ],
  "output_probes": [
    { "name": "rst_n", "value": "0", "width": 1 }
  ],
  "actions_taken": ["Set rst_n=1, committed", "Set rst_n=0, committed"],
  "observations": ["status_led toggling (activity detected)"]
}
```

## References
## 참고자료

- **UG908**: Vivado Programming and Debugging — hw_vio, hw_probe Tcl commands
- **UG908**: Vivado 프로그래밍 및 디버깅 — hw_vio, hw_probe Tcl 명령
- **UG912**: Vivado Properties Reference — HW_VIO, HW_PROBE properties
- **UG912**: Vivado 속성 참조 — HW_VIO, HW_PROBE 속성
- **UG835**: Vivado Tcl Commands Reference — commit_hw_vio, refresh_hw_vio, etc.
- **UG835**: Vivado Tcl 명령 참조 — commit_hw_vio, Refresh_hw_vio 등
- **UG936**: Vivado Tutorial: Programming and Debugging — step-by-step VIO lab
- **UG936**: Vivado 튜토리얼: 프로그래밍 및 디버깅 — 단계별 VIO 랩
- **PG159**: Virtual Input/Output v3.0 Product Guide — VIO core parameters, ports
- **PG159**: 가상 입력/출력 v3.0 제품 가이드 - VIO 핵심 매개변수, 포트
