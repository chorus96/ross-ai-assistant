<!-- ko-translation: enabled -->
# IO Bank and Clocking Constraints for Segmented Configuration
# 세그먼트 구성에 대한 IO 뱅크 및 클로킹 제약 조건

## Mixed IO Banks
## 혼합 IO 뱅크

An IO bank is "mixed" (shared) when it contains pins used by both the PS/boot domain and the PL domain.
IO 뱅크는 PS/부트 도메인과 PL 도메인 모두에서 사용되는 핀을 포함하는 경우 "혼합"(공유)됩니다.

### Behavior
### 행동
- The entire IO bank is programmed when **boot.pdi** is loaded (full-bank granularity in 2025.2)
- 전체 IO 뱅크는 **boot.pdi**가 로드될 때 프로그래밍됩니다(2025.2에서는 전체 뱅크 세분화).
- IO pins connected to PL logic become **electrically active** before the PL domain is configured
- PL 로직에 연결된 IO 핀은 PL 도메인이 구성되기 전에 **전기적으로 활성화**됩니다.
- These pins **cannot change functionality** during PL Reload — they remain active until a full device reconfiguration
- 이 핀은 PL 재로드 중에 **기능을 변경할 수 없습니다**. 전체 장치 재구성이 완료될 때까지 활성 상태로 유지됩니다.

### Design Guidance
### 디자인 지침
- Minimize or eliminate mixed IO banks when PL Reload is planned
- PL 재로드가 계획된 경우 혼합 IO 뱅크를 최소화하거나 제거합니다.
- Accepted: IO in mixed banks can be used as simple GPIO before PL loads
- 허용됨: 혼합 뱅크의 IO는 PL 로드 전에 간단한 GPIO로 사용될 수 있습니다.
- Not recommended: complex PL logic behind IO in a shared bank if PL Reload is planned
- 권장되지 않음: PL 재로드가 계획된 경우 공유 뱅크의 IO 뒤에 복잡한 PL 논리

### DRC Checks
### DRC 수표
- **SEGCONFIG-1**: Reports any shared IO bank (warning)
- **SEGCONFIG-1**: 공유 IO 뱅크를 보고합니다(경고).
- **SEGCONFIG-2**: Error if a PL port in a shared bank uses non-LVCMOS IOSTANDARD
- **SEGCONFIG-2**: 공유 뱅크의 PL 포트가 LVCMOS가 아닌 IOSTANDARD를 사용하는 경우 오류

### Resolution for SEGCONFIG-2
### SEGCONFIG-2에 대한 해결 방법
```tcl
# Change IOSTANDARD of the offending PL port to LVCMOS
set_property IOSTANDARD LVCMOS18 [get_ports <port_name>]
```

## X5IO Bank Granularity (Gen 2 Devices Only)
## X5IO 뱅크 세분성(Gen 2 장치만 해당)

In Versal AI Core Gen 2 and Prime Gen 2 devices, X5IO banks can be selectively shared.
Versal AI Core Gen 2 및 Prime Gen 2 장치에서는 X5IO 뱅크를 선택적으로 공유할 수 있습니다.

### Supported Split Configuration
### 지원되는 분할 구성
The third X5IO bank from a DDR site can be allocated to the PL partition when:
다음과 같은 경우 DDR 사이트의 세 번째 X5IO 뱅크를 PL 파티션에 할당할 수 있습니다.
- Only two half-banks of DDR5 are used in the BOOT partition
- BOOT 파티션에는 DDR5의 하프 뱅크 2개만 사용됩니다.
- Supported DDR configurations:
- 지원되는 DDR 구성:
  - Comp(DDR5) x4: 16-bit or 20-bit data, 16/17/18-bit address
  - Comp(DDR5) x4: 16비트 또는 20비트 데이터, 16/17/18비트 주소
  - Comp(LPDDR5) x16: 16-bit data, any address width
  - Comp(LPDDR5) x16: 16비트 데이터, 모든 주소 폭
  - Comp(LPDDR5) x32: 32-bit data, any address width
  - Comp(LPDDR5) x32: 32비트 데이터, 모든 주소 폭

### Master Bank Requirement (Gen 2)
### 마스터 뱅크 요구 사항(Gen 2)
- Master bank (bank 700 — leftmost along device bottom) must be in the boot partition if used
- 마스터 뱅크(뱅크 700 - 장치 하단을 따라 가장 왼쪽)는 사용되는 경우 부팅 파티션에 있어야 합니다.
- **DRC SEGCONFIG-5**: Reports if the master X5IO bank is used in PLD but not in boot
- **DRC SEGCONFIG-5**: 마스터 X5IO 뱅크가 PLD에서 사용되지만 부팅에서는 사용되지 않는지 보고합니다.
- Workaround: Use the master bank for DDR in boot, or leave it completely unused
- 해결 방법: 부팅 시 DDR용 마스터 뱅크를 사용하거나 완전히 사용하지 않은 채로 두십시오.
- AR39028: Additional restrictions on DDRMC5 usage with PL Reload (see Known Issues)
- AR39028: PL Reload와 함께 DDRMC5 사용에 대한 추가 제한 사항(알려진 문제 참조)

## Clocking Constraints
## 클러킹 제약

### SEGCONFIG-3: Clock resource in boot design
### SEGCONFIG-3: 부트 설계의 클록 리소스
Clocking tiles (BUFGCE, MMCM, PLL, etc.) used by the boot domain are reported as a warning.
부팅 도메인에서 사용되는 클러킹 타일(BUFGCE, MMCM, PLL 등)은 경고로 보고됩니다.
In most designs, clocking resources should reside in the PLD domain.
대부분의 설계에서 클로킹 리소스는 PLD 도메인에 있어야 합니다.

**Resolution**: Move clocking resources out of the boot partition.
**해결 방법**: 클럭 리소스를 부팅 파티션 밖으로 이동합니다.
Ensure no CIPS-related logic drives CLK resources that are shared with PL.
CIPS 관련 로직이 PL과 공유되는 CLK 리소스를 구동하지 않는지 확인하세요.

### SEGCONFIG-4: Shared XPLL or CLK_PLL_AND_PHY tile
### SEGCONFIG-4: 공유 XPLL 또는 CLK_PLL_AND_PHY 타일
XPLL and CLK_PLL_AND_PHY tiles must not be shared between boot and PLD partitions.
XPLL 및 CLK_PLL_AND_PHY 타일은 부팅 파티션과 PLD 파티션 간에 공유되어서는 안 됩니다.

**Resolution**: Reorganize clock planning so each tile is dedicated to one partition.
**해결 방법**: 각 타일이 하나의 파티션에 전용되도록 클록 계획을 재구성합니다.

## General IO Planning Recommendations
## 일반 IO 계획 권장 사항

1. Dedicate full IO banks to either boot (PS/DDR) or PL — avoid mixed banks
1. 전체 IO 뱅크를 부팅(PS/DDR) 또는 PL 전용으로 지정 - 혼합 뱅크 방지
2. Assign DDR IO first; remaining banks available for PL
2. DDR IO를 먼저 할당하십시오. PL에 사용 가능한 나머지 은행
3. Use the `get_dfx_footprint -seg_config_boot` command to see the boot partition tile footprint:
3. 부팅 파티션 타일 공간을 보려면 `get_dfx_footprint -seg_config_boot` 명령을 사용하세요.
   ```tcl
   get_dfx_footprint -seg_config_boot
   ```
4. Use this footprint to define pblock constraints that keep PL logic out of boot-partition tiles
4. 이 공간을 사용하여 부팅 파티션 타일에서 PL 로직을 유지하는 pblock 제약 조건을 정의하세요.
