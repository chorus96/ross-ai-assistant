<!-- ko-translation: enabled -->
# CPM/PCIe Use Cases for Segmented Configuration
# 세그먼트 구성을 위한 CPM/PCIe 사용 사례

## Overview
## 개요

Versal devices with CPM (Coherent Processing Module) blocks — CPM4, CPM5 (Gen 1), or MMI (Gen 2) — support PCIe endpoints as a method for delivering the PLD PDI image (QDMA-based loading).
CPM(Coherent Process Module) 블록(CPM4, CPM5(Gen 1) 또는 MMI(Gen 2))이 있는 Versal 장치는 PLD PDI 이미지(QDMA 기반 로딩)를 전달하기 위한 방법으로 PCIe 엔드포인트를 지원합니다.

Segmented Configuration **automatically** applies Tandem Configuration for CPM endpoints to meet the PCIe 120ms link training deadline.
세그먼트 구성 **자동으로** CPM 엔드포인트에 직렬 구성을 적용하여 PCIe 120ms 링크 교육 기한을 충족합니다.

## CPM Root Port Mode
## CPM 루트 포트 모드

- CPM Root Port is **not ready** until the PL domain is configured
- CPM 루트 포트는 PL 도메인이 구성될 때까지 **준비되지 않음**입니다.
- The CPM must be held in reset until the pld.pdi has been delivered
- CPM은 pld.pdi가 전달될 때까지 재설정 상태로 유지되어야 합니다.
- The CPM must also be held in reset **during** any PL Reload event and released afterward
- CPM은 PL 재로드 이벤트 **중에** 재설정 상태로 유지되어야 하며 이후에 해제되어야 합니다.
- Drivers must be aware of this startup sequence
- 운전자는 이 시동 순서를 알고 있어야 합니다.

## CPM Endpoint Mode (PCIe Device)
## CPM 엔드포인트 모드(PCIe 장치)

### Automatic Tandem Configuration
### 자동 탠덤 구성
When CPM is configured as a PCIe endpoint, Segmented Configuration automatically structures boot.pdi as a Tandem PROM image:
CPM이 PCIe 끝점으로 구성되면 세그먼트 구성은 자동으로 boot.pdi를 Tandem PROM 이미지로 구성합니다.
- CPM and required elements are programmed and released first
- CPM 및 필수 요소를 먼저 프로그래밍하여 출시합니다.
- Remaining boot content loads subsequently
- 이후에 나머지 부팅 콘텐츠가 로드됩니다.
- This meets the 120ms link training goal without manual Tandem Configuration selection
- 이는 수동으로 직렬 구성을 선택하지 않고도 120ms 링크 훈련 목표를 충족합니다.

**IMPORTANT**: Do NOT enable Tandem Configuration manually in CPM customization.
**중요**: CPM 사용자 정의에서 직렬 구성을 수동으로 활성화하지 마십시오.
An error is issued during `write_device_image` if both are selected simultaneously.
둘 다 동시에 선택하면 'write_device_image' 중에 오류가 발생합니다.

### QDMA-Based PLD Delivery
### QDMA 기반 PLD 전달

To load pld.pdi over the PCIe QDMA interface, PCIe must be declared as a secondary boot interface.
PCIe QDMA 인터페이스를 통해 pld.pdi를 로드하려면 PCIe를 보조 부팅 인터페이스로 선언해야 합니다.

#### First-Generation Versal (CIPS IP — CPM4/CPM5)
#### 1세대 Versal(CIPS IP — CPM4/CPM5)
Manual BIF edit is required:
수동 BIF 편집이 필요합니다.

1. After `write_device_image`, locate `<design>_boot.bif` in the implementation run directory
1. `write_device_image` 뒤 구현 실행 디렉터리에서 `<design>_boot.bif`를 찾습니다.
2. Open the file and find the line: `id = 0x2`
2. 파일을 열고 `id = 0x2` 줄을 찾으세요.
3. Insert `boot_device { pcie }` on the line **after** `id = 0x2`:
3. `id = 0x2` **다음** 줄에 `boot_device { pcie }`를 삽입하세요.
   ```
   ...
   id = 0x2
   boot_device { pcie }    <-- INSERT THIS LINE
   ...
   ```
4. Regenerate the boot PDI with the edited BIF:
4. 편집된 BIF를 사용하여 부팅 PDI를 재생성합니다.
   ```bash
   bootgen -arch versal -image <design>_boot.bif -w -o <design>_boot.pdi
   ```

Without this modification, PLM will report the following error when loading via QDMA:
이 수정이 없으면 PLM은 QDMA를 통해 로드할 때 다음 오류를 보고합니다.
```
/dev/qdma01000-MM-0, W off 0x102100000, 0x578730 failed -1. write file: Input/output error.
```

#### Second-Generation Versal (PS Wizard — MMI)
#### 2세대 Versal(PS 마법사 - MMI)
No manual BIF edit required.
수동 BIF 편집이 필요하지 않습니다.
Configure in the PS Wizard IP:
PS 마법사 IP에서 구성:
- Navigate to: **Boot Mode (PLM)** category
- **부팅 모드(PLM)** 범주로 이동합니다.
- Set **Secondary Boot** pulldown to: **MMI_PCIe**
- **보조 부팅** 풀다운을 **MMI_PCIe**로 설정합니다.

## CPM4/CPM5 Feature Restrictions Until PL Loads
## PL 로드까지 CPM4/CPM5 기능 제한

The following PCIe features are NOT available until pld.pdi is loaded:
다음 PCIe 기능은 pld.pdi가 로드될 때까지 사용할 수 없습니다:
- PCIe Extended Configuration Space (requires PL logic)
- PCIe 확장 구성 공간(PL 로직 필요)
- QDMA multi-function (uses PL mailbox, probed during driver load)
- QDMA 다기능(PL 메일박스 사용, 드라이버 로드 중에 검색됨)

Drivers must be structured to defer probing for these capabilities until after the PL domain is configured.
드라이버는 PL 도메인이 구성될 때까지 이러한 기능에 대한 검색을 연기하도록 구성되어야 합니다.

## Design Reference
## 디자인 참조

For CPM5 QDMA example design using Segmented Configuration, see:
세그먼트 구성을 사용한 CPM5 QDMA 예제 설계는 다음을 참조하세요.
- Vivado Example Designs: "Versal CPM5 QDMA Based Acceleration System Design"
- Vivado 예제 설계: "Versal CPM5 QDMA 기반 가속 시스템 설계"
- CEDStore: `https://github.com/Xilinx/XilinxCEDStore/tree/2025.2/ced/Xilinx/IPI/Versal_CPM_QDMA_Accel_Sys_Design`
- CED스토어: `https://github.com/Xilinx/XilinxCEDStore/tree/2025.2/ced/Xilinx/IPI/Versal_CPM_QDMA_Accel_Sys_Design`
