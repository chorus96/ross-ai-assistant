<!-- ko-translation: enabled -->
---
name: segcfg-design-considerations
description: Understand Segmented Configuration design constraints for IO banks, CPM/PCIe use cases, and Gen 2 device-specific features (VCU, ISP, ASU, DPDC, 10GbE). Use before finalizing a design that straddles PS and PL domains.
license: MIT
compatibility: Applies to AMD Versal and Versal Gen 2 Segmented Configuration designs in Vivado.
metadata:
   version: "1.0.0"
   stage: beta
allowed-tools: Read Bash Write
---

Explain Segmented Configuration design constraints and help the user evaluate their design against known restrictions.
분할된 구성 설계 제약 조건을 설명하고 사용자가 알려진 제한 사항에 대해 설계를 평가하도록 돕습니다.

## Instructions
## 지침

1. Read `references/io-bank-guide.md` for IO bank and clocking constraints.
1. IO 뱅크 및 클럭킹 제약 조건에 대해서는 `references/io-bank-guide.md`를 읽어보세요.
Read `references/cpm-pcie-guide.md` for CPM/PCIe-specific requirements.
CPM/PCIe 관련 요구 사항은 'references/cpm-pcie-guide.md'를 읽어보세요.
Read `references/gen2-features.md` for Gen 2 device-specific features.
Gen 2 장치별 기능에 대해서는 `references/gen2-features.md`를 읽어보세요.

2. Ask the user which aspects of their design they want to review:
2. 사용자에게 디자인의 어떤 측면을 검토하고 싶은지 물어보세요.
   - IO bank sharing (PS and PL on same bank)
   - IO 뱅크 공유(동일 뱅크의 PS 및 PL)
   - CPM/PCIe use cases
   - CPM/PCIe 사용 사례
   - Gen 2 device features (VCU, ISP, ASU, DPDC, 10GbE)
   - 2세대 장치 기능(VCU, ISP, ASU, DPDC, 10GbE)
   - General PS-PL boundary awareness
   - 일반 PS-PL 경계 인식

3. Explain **Mixed IO Banks** (applies to all Versal):
3. **혼합 IO 뱅크**에 대해 설명하세요(모든 Versal에 적용됨):
   - The boot image programs all IO in any bank required for DDR access (entire bank granularity)
   - 부팅 이미지는 DDR 액세스에 필요한 모든 뱅크의 모든 IO를 프로그래밍합니다(전체 뱅크 세분화).
   - IO connecting to PL logic becomes active *before* PL domain is configured
   - PL 로직에 연결하는 IO는 PL 도메인이 구성되기 *전에* 활성화됩니다.
   - These IO cannot change functionality during PL Reload (they remain active until full-device reconfiguration)
   - 이러한 IO는 PL 다시 로드 중에 기능을 변경할 수 없습니다(전체 장치를 재구성할 때까지 활성 상태로 유지됨).
   - Run SEGCONFIG-1 DRC to identify shared IO banks
   - SEGCONFIG-1 DRC를 실행하여 공유 IO 뱅크 식별
   - PL ports in shared IO banks must use LVCMOS IOSTANDARD (SEGCONFIG-2 DRC)
   - 공유 IO 뱅크의 PL 포트는 LVCMOS IOSTANDARD(SEGCONFIG-2 DRC)를 사용해야 합니다.

4. Explain **X5IO Granularity** (Gen 2 devices only):
4. **X5IO 세분성** 설명(2세대 장치에만 해당):
   - The third X5IO bank from a DDR site can be allocated to PL when only two half-banks are used by DDR5
   - DDR5에서 2개의 하프 뱅크만 사용하는 경우 DDR 사이트의 세 번째 X5IO 뱅크를 PL에 할당할 수 있습니다.
   - Supported only for specific DDR5 configurations: Comp(DDR5) x4 with 16/20 data width, LPDDR5 x16/x32
   - 특정 DDR5 구성에 대해서만 지원됨: Comp(DDR5) x4, 16/20 데이터 폭, LPDDR5 x16/x32
   - Master bank (bank 700, left-most) must be in the boot partition if used by DDR
   - DDR에서 사용하는 경우 마스터 뱅크(뱅크 700, 가장 왼쪽)가 부팅 파티션에 있어야 합니다.
   - Run SEGCONFIG-5 DRC for master bank validation
   - 마스터 뱅크 검증을 위해 SEGCONFIG-5 DRC 실행

5. Explain **CPM Use Cases**:
5. **CPM 사용 사례** 설명:
   - CPM Root Port mode: must be held in reset until PLD PDI is loaded; also must be held in reset during PL Reload
   - CPM 루트 포트 모드: PLD PDI가 로드될 때까지 재설정을 유지해야 합니다. PL 재로드 중에도 재설정을 유지해야 합니다.
   - CPM end point (PCIe): Segmented Configuration auto-enables Tandem Configuration for 120ms link training
   - CPM 엔드포인트(PCIe): 세그먼트 구성으로 120ms 링크 트레이닝을 위한 직렬 구성이 자동 활성화됩니다.
   - Do NOT select Tandem Configuration manually in CPM customization
   - CPM 사용자 정의에서 직렬 구성을 수동으로 선택하지 마십시오.
   - To load PLD PDI over QDMA (Gen 1 CIPS): add `boot_device { pcie }` to the boot.bif (line after `id = 0x2`)
   - QDMA(Gen 1 CIPS)를 통해 PLD PDI를 로드하려면 boot.bif에 `boot_device { pcie }`를 추가합니다(`id = 0x2` 다음 줄).
   - To load PLD PDI over QDMA (Gen 2 PS Wizard): set Secondary Boot = MMI_PCIe in Boot Mode (PLM) settings
   - QDMA(Gen 2 PS Wizard)를 통해 PLD PDI를 로드하려면: 부팅 모드(PLM) 설정에서 보조 부팅 = MMI_PCIe를 설정합니다.
   - PCIe Extended Configuration Space, QDMA multi-function: NOT available until PL is configured
   - PCIe 확장 구성 공간, QDMA 다기능: PL이 구성될 때까지 사용할 수 없음

6. Explain **Gen 2 device-specific features** that depend on PL:
6. PL에 의존하는 **Gen 2 장치별 기능**을 설명하세요.
   - **VCU & ISP**: Bundled with PL domain — unavailable until PLD PDI is loaded
   - **VCU 및 ISP**: PL 도메인과 함께 번들로 제공됨 - PLD PDI가 로드될 때까지 사용할 수 없음
   - **10GbE FIFO and TSU**: FIFO can be sourced from PL; TSU can use PL clock.
   - **10GbE FIFO 및 TSU**: FIFO는 PL에서 소싱될 수 있습니다. TSU는 PL 클럭을 사용할 수 있습니다.
   - Both require PL to be loaded
   - 둘 다 PL을 로드해야 합니다.
   - **ASU Soft Crypto Extension**: PL extension enabled in PS Wizard; programmed by PLD PDI
   - **ASU Soft Crypto Extension**: PS Wizard에서 PL 확장이 활성화되었습니다. PLD PDI로 프로그래밍
   - **DPDC (Display Controller)**: Live/Mixed modes require PL clock.
   - **DPDC(디스플레이 컨트롤러)**: 라이브/혼합 모드에는 PL 클럭이 필요합니다.
   - Enable "DP required before PL Config"
   - "PL 구성 전에 DP 필요"를 활성화합니다.
checkbox to use PS-based GPU_CLK initially, then switch after PL loads (driver support in future release)
처음에는 PS 기반 GPU_CLK를 사용하고 PL 로드 후에 전환하려면 확인란을 선택합니다(향후 릴리스에서 드라이버 지원).
   - **SYSMON auxiliary ports**: MIO/HDIO pins for external channel monitoring not available until PL is configured
   - **SYSMON 보조 포트**: PL이 구성될 때까지 외부 채널 모니터링을 위한 MIO/HDIO 핀을 사용할 수 없습니다.

7. Explain **CIPS-inferred PL logic** (all Gen 1 Versal):
7. **CIPS 추론 PL 로직** 설명(모두 Gen 1 Versal):
   - Some CIPS features are not entirely in hard blocks
   - 일부 CIPS 기능은 완전히 하드 블록에 포함되어 있지 않습니다.
   - SYSMON auxiliary port measurements from external pins not possible until pins are configured
   - 핀이 구성될 때까지 외부 핀의 SYSMON 보조 포트 측정이 불가능합니다.

8. Provide guidance: for each concern identified, recommend whether to:
8. 지침 제공: 식별된 각 우려사항에 대해 다음을 권장합니다.
   - Redesign to avoid the constraint
   - 제약을 피하기 위해 재설계
   - Accept and document the behavior gap during the PS-only phase
   - PS 전용 단계에서 행동 격차를 수용하고 문서화합니다.
   - Use DRC checks to validate before PDI generation
   - PDI 생성 전 DRC 검사를 사용하여 검증
