<!-- ko-translation: enabled -->
# Segmented Configuration Design Rule Checks (DRCs)
# 분할된 구성 설계 규칙 검사(DRC)

Design Rule Checks (DRC) can be called interactively for designs in Vivado.
Vivado의 디자인에 대해 DRC(Design Rule Check)를 대화형으로 호출할 수 있습니다.
Several checks are specifically targeted to Segmented Configuration requirements, focusing on IO bank usage, clocking resources, and DDRMC sharing.
여러 검사는 특히 IO 뱅크 사용량, 클로킹 리소스 및 DDRMC 공유에 중점을 두고 세그먼트 구성 요구 사항을 대상으로 합니다.

Run these DRCs on a **post-synthesis checkpoint** to analyze the design.
**합성 후 체크포인트**에서 이러한 DRC를 실행하여 설계를 분석합니다.

## Running DRCs
## DRC 실행

```tcl
# Run all Segmented Configuration DRCs
report_drc -checks {SEGCONFIG-1 SEGCONFIG-2 SEGCONFIG-3 SEGCONFIG-4 SEGCONFIG-5 SEGCONFIG-6 SEGCONFIG-7}

# Run a specific check
report_drc -checks {SEGCONFIG-1}

# Run via GUI: Reports > Report DRC, then select "Segmented Configuration" category
```

## DRC Check Details
## DRC 확인 세부정보

### SEGCONFIG-1: Shared IO Bank
### SEGCONFIG-1: 공유 IO 뱅크

**Type:** Shared IO bank between boot and PLD partitions
**유형:** 부팅 파티션과 PLD 파티션 간의 공유 IO 뱅크

**Message:** IO bank `<name>` is shared by the initial boot design and pld design.
**메시지:** IO 뱅크 `<name>`은 초기 부팅 설계와 PLD 설계에서 공유됩니다.
In Segmented Configuration flow, tiles and sites in the boot image will only be loaded once, this IO bank will not be reprogrammed when pld image is loaded.
세그먼트 구성 흐름에서 부팅 이미지의 타일과 사이트는 한 번만 로드되며, 이 IO 뱅크는 pld 이미지가 로드될 때 다시 프로그래밍되지 않습니다.

**Impact:** When an IO bank contains IO for both PS and PL domains, the entire bank is configured during boot image delivery and is NOT reconfigured when the PL image loads.
**영향:** IO 뱅크에 PS 및 PL 도메인 모두에 대한 IO가 포함되어 있으면 부팅 이미지 전달 중에 전체 뱅크가 구성되며 PL 이미지가 로드될 때 재구성되지 않습니다.
PL-connected IO in shared banks become active before the PL logic they connect to, and these IO cannot change functionality during PL Reload.
공유 뱅크의 PL 연결 IO는 연결되는 PL 로직 이전에 활성화되며 이러한 IO는 PL 재로드 중에 기능을 변경할 수 없습니다.

**Resolution:** Avoid mixed IO banks when possible.
**해결 방법:** 가능하면 혼합 IO 뱅크를 사용하지 마세요.
If unavoidable, understand that PL IO in shared banks is fixed for the lifetime of the boot image.
불가피한 경우 공유 뱅크의 PL IO는 부팅 이미지의 수명 동안 고정된다는 점을 이해하세요.

### SEGCONFIG-2: Non-LVCMOS PLD Port in Shared IO Bank
### SEGCONFIG-2: 공유 IO 뱅크의 비LVCMOS PLD 포트

**Type:** IOSTANDARD violation for PLD ports in shared banks
**유형:** 공유 은행의 PLD 포트에 대한 IOSTANDARD 위반

**Message:** Pld port `<name>` is found in IO bank `<name>`, which contains another port `<name>` of the initial boot design.
**메시지:** Pld 포트 `<name>`이 IO 뱅크 `<name>`에서 발견되었습니다. 이 뱅크에는 초기 부팅 설계의 또 다른 포트 `<name>`이 포함되어 있습니다.
In a Segmented Configuration design, pld port in an IO bank that is shared with boot design must be configured as LVCMOS, please change IOSTANDARD property of `<name>` from `<current>` to LVCMOS.
세그먼트 구성 설계에서 부팅 설계와 공유되는 IO 뱅크의 pld 포트는 LVCMOS로 구성되어야 합니다. `<name>`의 IOSTANDARD 속성을 `<current>`에서 LVCMOS로 변경하십시오.

**Impact:** PLD ports sharing an IO bank with boot design ports must use LVCMOS IOSTANDARD for compatibility.
**영향:** 부팅 설계 포트와 IO 뱅크를 공유하는 PLD 포트는 호환성을 위해 LVCMOS IOSTANDARD를 사용해야 합니다.

**Resolution:** Change the IOSTANDARD property of the affected port(s) to LVCMOS:
**해결 방법:** 영향을 받는 포트의 IOSTANDARD 속성을 LVCMOS로 변경합니다.
```tcl
set_property IOSTANDARD LVCMOS18 [get_ports <port_name>]
```

### SEGCONFIG-3: Clock Resource in Boot Design
### SEGCONFIG-3: 부트 설계의 클록 리소스

**Type:** Clocking tile used by boot design
**유형:** 부츠 디자인에 사용되는 클러킹 타일

**Message:** Clocking tile `<name>` is used by the initial boot design, this may lead to problem in a Segmented Configuration flow.
**메시지:** 클럭킹 타일 `<name>`은 초기 부팅 설계에서 사용되며 이로 인해 세그먼트 구성 흐름에 문제가 발생할 수 있습니다.
Clocking resources are typically used by pld designs.
클로킹 리소스는 일반적으로 pld 설계에 사용됩니다.

**Impact:** Clocking resources are typically part of the PLD partition.
**영향:** 클로킹 리소스는 일반적으로 PLD 파티션의 일부입니다.
Having them in the boot partition can cause issues if they need to be reconfigured during PL reload.
이를 부팅 파티션에 두면 PL 다시 로드 중에 재구성해야 하는 경우 문제가 발생할 수 있습니다.

**Resolution:** Review the design to ensure clocking resources are placed in the PLD partition unless they are genuinely needed for the boot partition.
**해결 방법:** 부팅 파티션에 꼭 필요한 경우를 제외하고 클럭킹 리소스가 PLD 파티션에 배치되도록 설계를 검토하세요.

### SEGCONFIG-4: Shared XPLL/CLK_PLL_AND_PHY Tile
### SEGCONFIG-4: 공유 XPLL/CLK_PLL_AND_PHY 타일

**Type:** Shared clocking tile between boot and PLD partitions
**유형:** 부팅 파티션과 PLD 파티션 간의 공유 클록 타일

**Message:** Tile `<name>` is shared by the initial boot design and pld design.
**메시지:** `<name>` 타일은 초기 부팅 디자인과 pld 디자인에 의해 공유됩니다.
XPLL and CLK_PLL_AND_PHY tiles should not be shared by the initial boot design and pld design.
XPLL 및 CLK_PLL_AND_PHY 타일은 초기 부팅 설계 및 pld 설계에서 공유되어서는 안 됩니다.

**Impact:** These clocking tiles cannot be reliably shared between partitions.
**영향:** 이러한 클럭 타일은 파티션 간에 안정적으로 공유될 수 없습니다.
The boot partition programs them first; the PLD partition cannot reprogram them.
부팅 파티션이 먼저 프로그래밍합니다. PLD 파티션은 이를 다시 프로그래밍할 수 없습니다.

**Resolution:** Reorganize the design so that XPLL and CLK_PLL_AND_PHY tiles are used exclusively by one partition.
**해결 방법:** XPLL 및 CLK_PLL_AND_PHY 타일이 하나의 파티션에서 독점적으로 사용되도록 디자인을 재구성합니다.

### SEGCONFIG-5: Master Bank X5IO Not in Boot Partition
### SEGCONFIG-5: 마스터 뱅크 X5IO가 부팅 파티션에 없음

**Type:** Used master bank X5IO not in boot partition (Gen 2 devices)
**유형:** 부팅 파티션에 없는 사용된 마스터 뱅크 X5IO(Gen 2 장치)

**Message:** Master bank X5IO port `<name>` in tile `<name>` is used in the secondary pld partition but not in the initial boot partition.
**메시지:** `<name>` 타일의 마스터 뱅크 X5IO 포트 `<name>`은 보조 pld 파티션에서 사용되지만 초기 부팅 파티션에서는 사용되지 않습니다.
In Segmented Configuration flow, used master bank X5IO must be in the boot partition or remain unused.
세그먼트 구성 흐름에서 사용된 마스터 뱅크 X5IO는 부팅 파티션에 있거나 사용되지 않은 상태로 유지되어야 합니다.

**Impact:** For Versal AI Edge Gen 2 and Prime Gen 2 devices, the master bank (bank 700, left-most bank along bottom of device) must either be in the boot partition or remain completely unused.
**영향:** Versal AI Edge Gen 2 및 Prime Gen 2 장치의 경우 마스터 뱅크(뱅크 700, 장치 하단의 가장 왼쪽 뱅크)가 부팅 파티션에 있거나 완전히 사용되지 않은 상태로 유지되어야 합니다.

**Resolution:** Move the master bank usage to the boot partition, or leave the master bank unused entirely.
**해결 방법:** 마스터 뱅크 사용을 부팅 파티션으로 이동하거나 마스터 뱅크를 완전히 사용하지 않은 상태로 두세요.

### SEGCONFIG-6: Reconfigurable Pblock Using Boot Resources
### SEGCONFIG-6: 부팅 리소스를 사용하여 재구성 가능한 Pblock

**Type:** Reconfigurable pblock contains boot partition resources
**유형:** 재구성 가능한 pblock에는 부팅 파티션 리소스가 포함되어 있습니다.

**Message:** Reconfigurable pblock `<name>` contains boot partition resources.
**메시지:** 재구성 가능한 pblock `<name>`에는 부팅 파티션 리소스가 포함되어 있습니다.
Reconfigurable pblock cannot use boot partition resources.
재구성 가능한 pblock은 부팅 파티션 리소스를 사용할 수 없습니다.
Please resize this pblock to remove boot partition tiles.
부팅 파티션 타일을 제거하려면 이 p블록의 크기를 조정하세요.
To see the boot partition footprint, use command: `get_dfx_footprint -seg_config_boot`.
부팅 파티션 공간을 보려면 `get_dfx_footprint -seg_config_boot` 명령을 사용하세요.

**Impact:** A reconfigurable pblock cannot overlap with boot partition resources, as these are not reprogrammed during PL reload.
**영향:** 재구성 가능한 pblock은 PL 다시 로드 중에 다시 프로그래밍되지 않으므로 부팅 파티션 리소스와 겹칠 수 없습니다.

**Resolution:** Resize the pblock to exclude boot partition tiles:
**해결 방법:** 부팅 파티션 타일을 제외하도록 pblock의 크기를 조정합니다.
```tcl
# View the boot partition footprint
get_dfx_footprint -seg_config_boot

# Adjust pblock constraints accordingly
resize_pblock [get_pblocks <pblock_name>] -remove <boot_tiles>
```

### SEGCONFIG-7: DDRMC Subsystem Tile Sharing
### SEGCONFIG-7: DDRMC 하위 시스템 타일 공유

**Type:** DDRMC subsystem tile shared between boot and PLD partitions
**유형:** 부팅 파티션과 PLD 파티션 간에 공유되는 DDRMC 하위 시스템 타일

**Message:** DDRMC subsystem tile `<name>` at boot partition instance `<name>`, site `<name>` is shared with PLD partition instance `<name>`, site `<name>`.
**메시지:** 부팅 파티션 인스턴스 `<name>`, 사이트 `<name>`의 DDRMC 하위 시스템 타일 `<name>`은 PLD 파티션 인스턴스 `<name>`, 사이트 `<name>`과 공유됩니다.
PLD can't load in this case.
이 경우 PLD를 로드할 수 없습니다.
Please change design to avoid resources sharing of boot partition DDRMC subsystems and PLD partition DDRMC subsystems.
부팅 파티션 DDRMC 하위 시스템과 PLD 파티션 DDRMC 하위 시스템의 리소스 공유를 방지하려면 설계를 변경하십시오.

**Impact:** If a DDRMC subsystem tile is shared between partitions, the PLD image cannot be loaded.
**영향:** DDRMC 하위 시스템 타일이 파티션 간에 공유되는 경우 PLD 이미지를 로드할 수 없습니다.
This is a critical error.
이는 심각한 오류입니다.

**Resolution:** Redesign the DDR connectivity so boot partition and PLD partition DDRMC subsystems use separate tiles.
**해결 방법:** 부팅 파티션과 PLD 파티션 DDRMC 하위 시스템이 별도의 타일을 사용하도록 DDR 연결을 다시 설계합니다.
Ensure that each DDRMC instance is fully contained within one partition.
각 DDRMC 인스턴스가 하나의 파티션 내에 완전히 포함되어 있는지 확인하십시오.
