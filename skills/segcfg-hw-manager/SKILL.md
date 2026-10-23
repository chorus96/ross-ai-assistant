<!-- ko-translation: enabled -->
---
name: segcfg-hw-manager
description: Program Versal hardware using Vivado Hardware Manager in Segmented Configuration mode. Use when programming boot.pdi and pld.pdi through the Vivado GUI, programming configuration flash memory, or enabling the segmented configuration dialog for first-generation Versal devices.
license: MIT
compatibility: Requires Vivado Hardware Manager, access to AMD Versal hardware, and matching boot.pdi and pld.pdi images.
metadata:
   version: "1.0.0"
   stage: beta
allowed-tools: Read Bash Write
---

Guide the user through programming Versal hardware via the Vivado Hardware Manager in Segmented Configuration mode.
Segmented Configuration 모드에서 Vivado Hardware Manager를 통해 Versal 하드웨어를 프로그래밍하는 과정을 사용자에게 안내합니다.

## Instructions
## 지침

1. Read `references/hw-manager-guide.md` for Hardware Manager dialog details and flash programming specifics.
1. 하드웨어 관리자 대화 상자 세부 정보 및 플래시 프로그래밍 세부 사항은 `references/hw-manager-guide.md`를 읽어보세요.

2. Ask the user:
2. 사용자에게 다음과 같이 질문하세요.
   - Which board they are targeting
   - 그들이 어떤 보드를 목표로 삼고 있는지
   - Whether they need JTAG programming only, or flash (configuration memory) programming
   - JTAG 프로그래밍만 필요한지 아니면 플래시(구성 메모리) 프로그래밍이 필요한지 여부
   - Whether they are using a first-generation or second-generation Versal device
   - 1세대 Versal 장치를 사용하는지 아니면 2세대 Versal 장치를 사용하는지 여부

3. **Enable Segmented Configuration mode in Hardware Manager** (first-gen devices only):
3. **하드웨어 관리자에서 세그먼트 구성 모드 활성화**(1세대 장치만 해당):
In the Vivado Tcl Console before opening Hardware Manager:
하드웨어 관리자를 열기 전 Vivado Tcl 콘솔에서:
   ```tcl
   set_param labtools.enableSegmentedConfigFlow 1
   ```
This exposes the Segmented Configuration programming dialog for first-generation Versal devices.
그러면 1세대 Versal 장치에 대한 세그먼트 구성 프로그래밍 대화 상자가 표시됩니다.
For second-generation devices, Segmented Configuration is always active — no parameter needed.
2세대 장치의 경우 세그먼트 구성은 항상 활성화되어 있으며 매개변수가 필요하지 않습니다.

4. **Connect to the hardware target**:
4. **하드웨어 대상에 연결**:
   - Open Vivado Hardware Manager: **Flow Navigator > Program and Debug > Open Hardware Manager**
   - Vivado 하드웨어 관리자 열기: **Flow Navigator > Program and Debug > Open Hardware Manager**
   - Click **Open Target > Auto Connect** to discover the JTAG chain
   - JTAG 체인을 검색하려면 **Open Target > Auto Connect**를 클릭하세요.
   - The Versal device will appear in the Hardware window
   - Versal 장치가 하드웨어 창에 나타납니다.

5. **Program the device via JTAG**:
5. **JTAG를 통해 장치 프로그래밍**:
   - Right-click the device in the Hardware window → **Program Device**
   - 하드웨어 창에서 장치를 마우스 오른쪽 버튼으로 클릭 → **프로그램 장치**
   - In the Program Device dialog, the Segmented Configuration-specific fields appear:
   - 프로그램 장치 대화 상자에는 세그먼트 구성 관련 필드가 나타납니다.
     - **Enable segmented configuration mode** checkbox (first-gen devices)
     - **분할된 구성 모드 활성화** 확인란(1세대 장치)
     - **Boot PDI File**: browse to `<design>_boot.pdi`
     - **부팅 PDI 파일**: `<design>_boot.pdi`를 찾습니다.
     - **Boot LTX Probe File**: browse to `<design>_boot.ltx` (if ChipScope cores in boot partition)
     - **부팅 LTX 프로브 파일**: `<design>_boot.ltx`로 이동합니다(부팅 파티션에 ChipScope 코어가 있는 경우).
     - **PLD PDI File**: browse to `<design>_pld.pdi`
     - **PLD PDI 파일**: `<design>_pld.pdi`를 찾습니다.
     - **PLD LTX Probe File**: browse to `<design>_pld.ltx` (if ChipScope cores in PLD)
     - **PLD LTX 프로브 파일**: `<design>_pld.ltx`로 이동합니다(PLD에 ChipScope 코어가 있는 경우).
   - Hardware Manager loads Boot and PLD images sequentially
   - 하드웨어 관리자는 부팅 및 PLD 이미지를 순차적으로 로드합니다.

6. **Boot-only programming** (test PS domain without PL):
6. **부팅 전용 프로그래밍**(PL 없이 PS 도메인 테스트):
   - In the Program Device dialog, populate only the Boot PDI field
   - 프로그램 장치 대화 상자에서 부팅 PDI 필드만 채웁니다.
   - Leave the PLD PDI field empty
   - PLD PDI 필드를 비워 두세요.
   - This enables testing the processing domain independently
   - 이를 통해 처리 도메인을 독립적으로 테스트할 수 있습니다.

7. **PLD-only programming** (for PL Reload — boot PDI already loaded):
7. **PLD 전용 프로그래밍**(PL 재로드용 - 부팅 PDI가 이미 로드됨):
   - Populate only the PLD PDI field
   - PLD PDI 필드만 채웁니다.
   - The boot PDI must already be resident on the device
   - 부팅 PDI는 이미 장치에 상주해야 합니다.
   - Used for PL Reload scenarios and iterative PL debug
   - PL 다시 로드 시나리오 및 반복적인 PL 디버그에 사용됩니다.

8. **Programming configuration flash memory**:
8. **프로그래밍 구성 플래시 메모리**:
   - Right-click the device → **Add Configuration Memory Device**
   - 장치를 마우스 오른쪽 버튼으로 클릭 → **구성 메모리 장치 추가**
   - Select the flash part for your board (e.g., MT25QU02GCBB for VCK190)
   - 보드에 맞는 플래시 부품을 선택하세요(예: VCK190의 경우 MT25QU02GCBB)
   - Right-click the memory device → **Program Configuration Memory Device**
   - 메모리 장치를 마우스 오른쪽 버튼으로 클릭 → **프로그램 구성 메모리 장치**
   - In the dialog, set offsets for both boot and PLD PDIs:
   - 대화 상자에서 부팅 및 PLD PDI 모두에 대한 오프셋을 설정합니다.
     - **Boot PDI offset**: `0x00000000` (start of flash)
     - **부팅 PDI 오프셋**: `0x00000000`(플래시 시작)
     - **PLD PDI offset**: non-zero value, multiple of flash sector size (64KB = 0x10000)
     - **PLD PDI 오프셋**: 0이 아닌 값, 플래시 섹터 크기의 배수(64KB = 0x10000)
     - Ensure sufficient room for boot PDI before declaring PLD offset
     - PLD 오프셋을 선언하기 전에 부팅 PDI를 위한 충분한 공간을 확보하세요.
   - Example: if boot PDI is ~10MB, use PLD offset `0x00A00000` (10MB aligned to 64KB)
   - 예: 부팅 PDI가 ~10MB인 경우 PLD 오프셋 `0x00A00000`을 사용합니다(10MB는 64KB로 정렬됨).

9. **ChipScope debugging with Segmented Configuration**:
9. **세그먼트 구성을 사용한 ChipScope 디버깅**:
   - Boot partition ILA/VIO are activated when boot PDI loads → LTX file from boot.pdi
   - 부팅 PDI가 로드될 때 부팅 파티션 ILA/VIO가 활성화됨 → boot.pdi에서 LTX 파일
   - PLD partition ILA/VIO are activated when pld PDI loads → LTX file from pld.pdi
   - pld PDI가 로드되면 PLD 파티션 ILA/VIO가 활성화됩니다. → pld.pdi에서 LTX 파일
   - Refresh hardware after each PDI load to discover new debug cores
   - 새로운 디버그 코어를 발견하기 위해 각 PDI 로드 후 하드웨어를 새로 고칩니다.

10. For more information, refer to the [Vivado Design Suite User Guide: Programming and Debugging (UG908)].
10. 자세한 내용은 [Vivado Design Suite 사용자 가이드: 프로그래밍 및 디버깅(UG908)]을 참조하세요.
