<!-- ko-translation: enabled -->
# Vivado Hardware Manager Guide for Segmented Configuration
# 분할된 구성을 위한 Vivado 하드웨어 관리자 가이드

## Overview
## 개요

Vivado Hardware Manager provides a GUI and Tcl-accessible interface for programming Versal devices and configuration flash memory.
Vivado Hardware Manager는 Versal 장치 프로그래밍 및 플래시 메모리 구성을 위한 GUI 및 Tcl 액세스 가능 인터페이스를 제공합니다.
For Segmented Configuration, it supports programming Boot and PLD PDIs sequentially, selective Boot-only or PLD-only programming, and flash programming with offset control.
세그먼트 구성의 경우 부트 및 PLD PDI의 순차적 프로그래밍, 선택적 부트 전용 또는 PLD 전용 프로그래밍, 오프셋 제어를 통한 플래시 프로그래밍을 지원합니다.

## Prerequisite: Enable Segmented Configuration Mode (Gen 1 Only)
## 전제 조건: 세그먼트 구성 모드 활성화(Gen 1만 해당)

For first-generation Versal devices, the Segmented Configuration programming dialog requires a parameter:
1세대 Versal 장치의 경우 세그먼트 구성 프로그래밍 대화 상자에는 매개변수가 필요합니다.

```tcl
# Run in the Vivado Tcl Console before opening Hardware Manager
set_param labtools.enableSegmentedConfigFlow 1
```

This is NOT required for second-generation Versal devices (Segmented Configuration is always active).
이는 2세대 Versal 장치에는 필요하지 않습니다(세그먼트 구성은 항상 활성화되어 있음).

## Connecting to Hardware
## 하드웨어에 연결

1. Open Hardware Manager: **Flow Navigator > Program and Debug > Open Hardware Manager**
1. 하드웨어 관리자 열기: **Flow Navigator > 프로그램 및 디버그 > 하드웨어 관리자 열기**
2. Click **Open Target > Auto Connect**
2. **대상 열기 > 자동 연결**을 클릭합니다.
3. The JTAG chain is scanned; the Versal device appears in the Hardware window
3. JTAG 체인이 스캔됩니다. Versal 장치가 하드웨어 창에 나타납니다.

```tcl
# Tcl equivalent:
open_hw_manager
connect_hw_server
open_hw_target
```

## Programming the Device (JTAG)
## 장치 프로그래밍(JTAG)

### GUI Method
### GUI 방식
1. Right-click the device in the Hardware window → **Program Device**
1. 하드웨어 창에서 장치를 마우스 오른쪽 버튼으로 클릭 → **프로그램 장치**
2. In the Program Device dialog:
2. 프로그램 장치 대화 상자에서:
   - Check **Enable segmented configuration mode** (first-gen devices)
   - **분할된 구성 모드 활성화**를 선택합니다(1세대 장치).
   - **Boot PDI File**: Browse to `<design>_boot.pdi`
   - **부팅 PDI 파일**: `<design>_boot.pdi`로 이동합니다.
   - **Boot LTX Probe File**: Browse to `<design>_boot.ltx` (optional, for ChipScope debug)
   - **부팅 LTX 프로브 파일**: `<design>_boot.ltx`로 이동합니다(선택 사항, ChipScope 디버그용).
   - **PLD PDI File**: Browse to `<design>_pld.pdi`
   - **PLD PDI 파일**: `<design>_pld.pdi`로 이동합니다.
   - **PLD LTX Probe File**: Browse to `<design>_pld.ltx` (optional)
   - **PLD LTX 프로브 파일**: `<design>_pld.ltx`로 이동합니다(선택 사항).
   - Click **Program**
   - **프로그램**을 클릭하세요.

### Tcl Method
### Tcl 방법
```tcl
# Get the hardware device handle
set hw_device [get_hw_devices]

# Program Boot PDI only
set_property PROGRAM.FILE {impl_1/<design>_boot.pdi} $hw_device
program_hw_devices $hw_device

# Program PLD PDI (after boot is loaded)
set_property PROGRAM.FILE {impl_1/<design>_pld.pdi} $hw_device
program_hw_devices $hw_device
```

## Boot-Only Programming
## 부팅 전용 프로그래밍

To load only the boot image (test PS domain in isolation):
부팅 이미지만 로드하려면(격리된 PS 도메인 테스트):
- In the Program Device dialog, fill in only the Boot PDI field
- Program Device 대화 상자에서 Boot PDI 필드만 입력합니다.
- Leave the PLD PDI field empty
- PLD PDI 필드를 비워 두세요.
- Click Program
- 프로그램을 클릭하세요

This enables isolated testing of the PS domain, DDR, and hard blocks.
이를 통해 PS 도메인, DDR 및 하드 블록을 격리하여 테스트할 수 있습니다.

## PLD-Only Programming (PL Reload)
## PLD 전용 프로그래밍(PL 리로드)

To load only the PLD image (boot must already be resident):
PLD 이미지만 로드하려면(부팅이 이미 상주해야 함):
- In the Program Device dialog, fill in only the PLD PDI field
- 프로그램 장치 대화 상자에서 PLD PDI 필드만 입력합니다.
- Leave the Boot PDI field empty
- Boot PDI 필드를 비워 두세요.
- Click Program
- 프로그램을 클릭하세요

Hardware Manager loads the PLD image; PLM handles isolation and PL reconfiguration automatically.
하드웨어 관리자가 PLD 이미지를 로드합니다. PLM은 격리 및 PL 재구성을 자동으로 처리합니다.

## Flash Programming (Configuration Memory)
## 플래시 프로그래밍(구성 메모리)

### Setup
### 설정
1. Right-click the device → **Add Configuration Memory Device**
1. 장치를 마우스 오른쪽 버튼으로 클릭 → **구성 메모리 장치 추가**
2. Select the appropriate flash part for your board:
2. 보드에 적합한 플래시 부품을 선택하십시오.
   - VCK190: MT25QU02GCBB (Micron 2Gb SPI Quad)
   - VCK190: MT25QU02GCBB(미크론 2Gb SPI 쿼드)
   - VEK280: check board documentation
   - VEK280: 보드 문서를 확인하세요
3. Right-click the memory device → **Program Configuration Memory Device**
3. 메모리 장치를 마우스 오른쪽 버튼으로 클릭 → **프로그램 구성 메모리 장치**

### Offset Configuration
### 오프셋 구성
When programming a Segmented Configuration design to flash:
세그먼트 구성 설계를 플래시하도록 프로그래밍하는 경우:
- **At least one offset must be non-zero** (Boot PDI and PLD PDI must be at different locations)
- **최소 하나의 오프셋은 0이 아니어야 합니다**(부트 PDI와 PLD PDI는 서로 다른 위치에 있어야 함)
- Offsets must be **multiples of the flash sector size**: 64KB (0x10000)
- 오프셋은 **플래시 섹터 크기의 배수**여야 합니다: 64KB(0x10000)
- Ensure enough space for Boot PDI before declaring PLD offset
- PLD 오프셋을 선언하기 전에 Boot PDI를 위한 충분한 공간을 확보하세요.

```
Example layout for a ~10MB boot PDI:
  Boot PDI offset:  0x00000000  (start of flash)
  PLD PDI offset:   0x00A00000  (10MB, 64KB-aligned)
```

### GUI Fields
### GUI 필드
- **Bitstream file**: `<design>_boot.pdi` with offset `0x00000000`
- **비트스트림 파일**: 오프셋 `0x00000000`을 포함한 `<design>_boot.pdi`
- **Second bitstream file**: `<design>_pld.pdi` with appropriate offset
- **두 번째 비트스트림 파일**: 적절한 오프셋이 있는 `<design>_pld.pdi`

## ChipScope Debug with Segmented Configuration
## 세그먼트 구성을 사용한 ChipScope 디버그

- **Boot partition** ILA/VIO cores: activated when boot PDI loads → use `<design>_boot.ltx`
- **부팅 파티션** ILA/VIO 코어: 부팅 PDI 로드 시 활성화 → `<design>_boot.ltx` 사용
- **PLD partition** ILA/VIO cores: activated when PLD PDI loads → use `<design>_pld.ltx`
- **PLD 파티션** ILA/VIO 코어: PLD PDI 로드 시 활성화 → `<design>_pld.ltx` 사용
- Refresh the Hardware window (right-click device → Refresh Hardware) after each PDI load
- 각 PDI 로드 후 하드웨어 창 새로 고침(장치 마우스 오른쪽 버튼 클릭 → 하드웨어 새로 고침)
- New debug cores appear automatically after refresh
- 새로 고침 후 새 디버그 코어가 자동으로 나타납니다.

## Reference
## 참고자료

[Vivado Design Suite User Guide: Programming and Debugging (UG908)](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging)
[Vivado Design Suite 사용자 가이드: 프로그래밍 및 디버깅(UG908)](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging)
