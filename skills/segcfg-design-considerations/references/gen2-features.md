<!-- ko-translation: enabled -->
# Gen 2 Device-Specific Features for Segmented Configuration
# 분할된 구성을 위한 Gen 2 장치별 기능

Applies to: Versal AI Edge Gen 2 (2VExxxx) and Versal Prime Gen 2 (2VMxxxx)
적용 대상: Versal AI Edge Gen 2(2VExxxx) 및 Versal Prime Gen 2(2VMxxxx)

## Key Differences from Gen 1
## Gen 1과의 주요 차이점

- Segmented Configuration is **always enabled** — cannot be disabled
- 분할된 구성은 **항상 활성화**되어 있으며 비활성화할 수 없습니다.
- Uses `versal_ps` IP (PS Wizard) instead of `versal_cips`
- `versal_cips` 대신 `versal_ps` IP(PS 마법사)를 사용합니다.
- Uses `axi_noc2` instead of `axi_noc`
- `axi_noc` 대신 `axi_noc2`를 사용합니다.
- Software build uses **Yocto** (meta-xilinx-tools), not PetaLinux
- 소프트웨어 빌드에서는 PetaLinux가 아닌 **Yocto**(meta-xilinx-tools)를 사용합니다.
- Single PDI concatenation with two `include` statements is supported
- 두 개의 'include' 문을 사용한 단일 PDI 연결이 지원됩니다.

## VCU (Video Codec Unit) and ISP (Image Signal Processor)
## VCU(비디오 코덱 장치) 및 ISP(이미지 신호 프로세서)

These hard IP blocks in Gen 2 devices depend on clock resources located in the PL domain.
Gen 2 장치의 이러한 하드 IP 블록은 PL 도메인에 있는 클럭 리소스에 따라 달라집니다.

- VCU and ISP are **bundled with the PL domain** in the pld.pdi image
- VCU 및 ISP는 pld.pdi 이미지에 **PL 도메인과 함께 번들로 제공**됩니다.
- They are **unavailable** until pld.pdi is loaded
- pld.pdi가 로드될 때까지는 **사용할 수 없습니다**.
- Applications requiring VCU or ISP must wait for PL configuration
- VCU 또는 ISP가 필요한 애플리케이션은 PL 구성을 기다려야 합니다.

## ASU Soft Crypto Extension
## ASU 소프트 암호화 확장

The Application Security Unit (ASU) supports programmable soft cryptographic algorithms.
ASU(Application Security Unit)는 프로그래밍 가능한 소프트 암호화 알고리즘을 지원합니다.

- Enable in PS Wizard: check **Enable PL extension** checkbox
- PS 마법사에서 활성화: **PL 확장 활성화** 확인란을 선택합니다.
- The PL extension is programmed as part of pld.pdi
- PL 확장은 pld.pdi의 일부로 프로그래밍됩니다.
- Unavailable until pld.pdi is loaded
- pld.pdi가 로드될 때까지 사용할 수 없습니다.
- Allows adding or updating crypto functions post-deployment
- 배포 후 암호화 기능을 추가하거나 업데이트할 수 있습니다.

## 10GbE FIFO and TSU (Time Stamp Unit)
## 10GbE FIFO 및 TSU(타임 스탬프 장치)

The 10GbE Controller in Gen 2 has two PL-dependent features:
Gen 2의 10GbE 컨트롤러에는 두 가지 PL 종속 기능이 있습니다.

### FIFO Interface
### FIFO 인터페이스
- Can be sourced from PL for direct Ethernet packet access
- 직접 이더넷 패킷 액세스를 위해 PL에서 소싱 가능
- Runtime selection: GEM register selects between DMA and external FIFO input
- 런타임 선택: GEM 레지스터는 DMA와 외부 FIFO 입력 중에서 선택합니다.
- Enable in PS Wizard: **MMI Peripherals** section when 10GbE is enabled
- PS 마법사에서 활성화: 10GbE가 활성화된 경우 **MMI 주변 장치** 섹션
- Unavailable until PL is configured
- PL이 구성될 때까지 사용할 수 없음

### Time Stamp Unit (TSU)
### 타임 스탬프 단위(TSU)
- Can be clocked from PL clocking resources
- PL 클로킹 리소스에서 클럭킹 가능
- Runtime selection: `TSU_CLK_LB_SEL` register selects PS or PL source
- 런타임 선택: `TSU_CLK_LB_SEL` 레지스터가 PS 또는 PL 소스를 선택합니다.
- Enable in PS Wizard: **MMI Peripherals** section when 10GbE is enabled
- PS 마법사에서 활성화: 10GbE가 활성화된 경우 **MMI 주변 장치** 섹션
- Unavailable until PL is configured
- PL이 구성될 때까지 사용할 수 없음

Reference: [AM026 - Versal AI Edge Series Gen 2 and Prime Series Gen 2 TRM](https://docs.amd.com/r/en-US/am026-versal-ai-edge-prime-gen2-trm/10G-Ethernet-MAC)
참조: [AM026 - Versal AI Edge 시리즈 Gen 2 및 Prime 시리즈 Gen 2 TRM](https://docs.amd.com/r/en-US/am026-versal-ai-edge-prime-gen2-trm/10G-Ethernet-MAC)

## DPDC (Display Port Controller)
## DPDC(디스플레이 포트 컨트롤러)

Certain display modes require PL-sourced clocks.
특정 디스플레이 모드에는 PL 소스 클럭이 필요합니다.

### Affected Modes
### 영향을 받는 모드
- Live mode
- 라이브 모드
- Mixed mode
- 혼합 모드

### "DP Required Before PL Config" Feature
### "PL 구성 전 DP 필요" 기능
- Enable in PS Wizard → DPDC Configuration → check **DP required before PL Config**
- PS 마법사에서 활성화 → DPDC 구성 → **PL 구성 전에 DP 필요** 확인
- With this option, DPDC uses PS-based GPU_CLK initially
- 이 옵션을 사용하면 DPDC는 처음에 PS 기반 GPU_CLK를 사용합니다.
- After pld.pdi is loaded, hardware switches to the PL-based clock
- pld.pdi가 로드된 후 하드웨어는 PL 기반 시계로 전환됩니다.
- A momentary glitch in video output will occur during the clock switch (DisplayPort retraining)
- 시계 전환(DisplayPort 재교육) 중에 비디오 출력에 순간적인 결함이 발생합니다.
- Full driver support for this switchover is scheduled for a future release
- 이 전환에 대한 전체 드라이버 지원은 향후 릴리스에서 예정되어 있습니다.

Reference: [AM026 - Display Controller section](https://docs.amd.com/r/en-US/am026-versal-ai-edge-prime-gen2-trm/Display-Controller)
참조: [AM026 - 디스플레이 컨트롤러 섹션](https://docs.amd.com/r/en-US/am026-versal-ai-edge-prime-gen2-trm/Display-Controller)

## Known Issues for Gen 2 Devices (2025.2)
## Gen 2 장치의 알려진 문제(2025.2)

### Master Bank Restriction
### 마스터 은행 제한
- If the boot partition uses any DDRMC X5IO banks, the master bank (bank 700, leftmost along bottom) is recommended to be one of them
- 부팅 파티션이 DDRMC X5IO 뱅크를 사용하는 경우 마스터 뱅크(뱅크 700, 맨 왼쪽 하단)를 그 중 하나로 사용하는 것이 좋습니다.
- Unsupported: connecting PS to the right-most DDRMC when the left-most DDRMC (and master X5IO bank) is connected to the PL domain
- 지원되지 않음: 가장 왼쪽 DDRMC(및 마스터 X5IO 뱅크)가 PL 도메인에 연결된 경우 PS를 가장 오른쪽 DDRMC에 연결
- Fix planned for a future Vivado release
- 향후 Vivado 릴리스에 대한 수정 계획

### DDRMC5 PL Reload Restrictions
### DDRMC5 PL 재로드 제한 사항
- If any DDRMC5 site has both ports connected to PL domains, the controller can lose connection to PS after a PL Reload event
- DDRMC5 사이트에 두 포트가 모두 PL 도메인에 연결되어 있는 경우 PL 다시 로드 이벤트 후 컨트롤러가 PS에 대한 연결이 끊어질 수 있습니다.
- See [AR39028](https://adaptivesupport.amd.com/s/article/000039208) for supported and unsupported DDRMC5 configurations
- 지원되거나 지원되지 않는 DDRMC5 구성은 [AR39028](https://adaptivesupport.amd.com/s/article/000039208)을 참조하세요.

### DRC: SEGCONFIG-5
### DRC: SEGCONFIG-5
- Master bank X5IO port used in secondary PLD partition but not in boot partition
- 보조 PLD 파티션에는 사용되지만 부팅 파티션에는 사용되지 않는 마스터 뱅크 X5IO 포트
- Resolution: Ensure the master bank is either in the boot partition or completely unused
- 해결 방법: 마스터 뱅크가 부팅 파티션에 있거나 완전히 사용되지 않는지 확인하세요.
