<!-- ko-translation: enabled -->
# Address Aperture Flexibility for Segmented Configuration PL Reload
# 분할된 구성 PL 재로드를 위한 주소 조리개 유연성

## Problem
## 문제

Traditional NoC designs use static NMU decode and fixed address apertures.
기존 NoC 설계에서는 정적 NMU 디코드와 고정 주소 조리개를 사용합니다.
When using PL Reload with different PL images, aperture mismatches between the initial and reload designs can cause verification failures or routing incompatibilities.
서로 다른 PL 이미지와 함께 PL Reload를 사용하는 경우 초기 설계와 재로드 설계 간의 애퍼처 불일치로 인해 검증 실패 또는 라우팅 비호환성이 발생할 수 있습니다.

## Solution: Address Aperture Flexibility
## 솔루션: 주소 조리개 유연성

Segmented Configuration introduces controlled aperture adjustment across PL reload variants:
세그먼트 구성은 PL 리로드 변형 전체에 제어된 조리개 조정을 도입합니다.

- The **golden design** defines the **maximum aperture** for all anticipated reload cases
- **골든 디자인**은 예상되는 모든 재장전 케이스에 대한 **최대 조리개**를 정의합니다.
- **Variant designs** may use a **smaller subset** of that reserved range
- **변형 디자인**은 예약된 범위의 **더 작은 하위 집합**을 사용할 수 있습니다.
- Variant designs may NOT exceed the golden aperture
- 변형 디자인은 골든 조리개를 초과할 수 없습니다.
- This enables flexibility without breaking `pr_verify` checks
- 이를 통해 'pr_verify' 검사를 중단하지 않고도 유연성을 확보할 수 있습니다.

## Key Capabilities
## 주요 기능

| Capability<br>능력 | Description<br>설명 |
|-----------|-------------|
| Dynamic Aperture Management<br>동적 조리개 관리 | Reload designs define apertures fitting their PL slave address space, within the golden's pre-allocated range<br>재로드 설계는 골든의 사전 할당 범위 내에서 PL 슬레이브 주소 공간에 맞는 조리개를 정의합니다. |
| Static Decode Preservation<br>정적 디코드 보존 | Base address and NMU decode remain consistent (similar to DFX schemes)<br>기본 주소 및 NMU 디코드는 일관성을 유지합니다(DFX 방식과 유사). |
| Traffic Spec Integration<br>트래픽 사양 통합 | Aperture constraints annotated in traffic spec for consistent address assignment<br>일관된 주소 할당을 위해 트래픽 사양에 주석이 달린 조리개 제약 |
| User-Specified Apertures<br>사용자 지정 조리개 | Explicit aperture definition on INI ports to prevent overwrite during auto address assignment<br>자동 주소 할당 중 덮어쓰기를 방지하기 위해 INI 포트에 명시적인 조리개 정의 |

## Implementation
## 구현

### Step 1: Golden Design — Define Maximum Aperture
### 1단계: 골든 디자인 - 최대 조리개 정의
In the initial design, set the largest possible aperture for each PL slave interface.
초기 설계에서는 각 PL 슬레이브 인터페이스에 대해 가능한 가장 큰 조리개를 설정합니다.
This reserves the full address range needed by any reload variant.
이는 다시 로드 변형에 필요한 전체 주소 범위를 예약합니다.

Example: Reserve 256K for an AXI BRAM controller that may only need 64K in some variants.
예: 일부 변형에서는 64K만 필요할 수 있는 AXI BRAM 컨트롤러에 대해 256K를 예약합니다.

```
Golden design address map:
  /axi_bram_ctrl_0/S_AXI: base = 0xa4000000, aperture = 256K (0x40000)
```

### Step 2: Aperture Propagation
### 2단계: 조리개 전파
The aperture settings propagate through the NoC solution file (`.ncr`) to subsequent designs.
조리개 설정은 NoC 솔루션 파일(`.ncr`)을 통해 후속 설계로 전파됩니다.
Apertures for boot-path NMU instances are locked — they remain consistent across all designs.
부팅 경로 NMU 인스턴스의 조리개는 잠겨 있으며 모든 설계에서 일관성을 유지합니다.

### Step 3: Variant Design — Use Subset Aperture
### 3단계: 변형 디자인 - 하위 세트 조리개 사용
In the reload variant, the aperture can be reduced to match the actual PL slave size:
재장전 변형에서는 실제 PL 슬레이브 크기에 맞게 조리개를 줄일 수 있습니다.

```
Variant design address map:
  /axi_bram_ctrl_0/S_AXI: base = 0xa4000000, aperture = 64K (0x10000)
```

The base address and NMU decode remain the same; only the aperture size shrinks.
기본 주소와 NMU 디코드는 동일하게 유지됩니다. 조리개 크기만 줄어듭니다.

### Step 4: Verify with pr_verify
### 4단계: pr_verify로 확인
`pr_verify` checks that variant apertures do not exceed golden apertures:
`pr_verify`는 변형 조리개가 황금 조리개를 초과하지 않는지 확인합니다.
```tcl
pr_verify -segcfg_only -initial <golden>_routed.dcp -additional <variant>_routed.dcp
```

If the variant aperture is within the golden range, `pr_verify` passes.
변형 조리개가 황금 범위 내에 있으면 'pr_verify'가 통과됩니다.
If the variant aperture exceeds the golden range, `pr_verify` reports an error.
변형 조리개가 황금 범위를 초과하면 'pr_verify'가 오류를 보고합니다.

## Tcl / GUI Options
## Tcl / GUI 옵션

### Set INI Aperture via Tcl
### Tcl을 통해 INI 조리개 설정
```tcl
# Set explicit aperture on an INI (NoC Interface) to prevent auto-overwrite
set_property -dict {CONFIG.APERTURES {0xa4000000 256K}} \
    [get_bd_intf_pins /axi_noc_0/S00_INI]
```

### GUI
### GUI
In the AXI NoC IP's address editor, you can set the aperture size directly on AXI slave ports.
AXI NoC IP의 주소 편집기에서는 AXI 슬레이브 포트에서 조리개 크기를 직접 설정할 수 있습니다.

## Example Scenario
## 예시 시나리오

```
Golden:  BRAM aperture = 256K → covers future 64K, 128K, 256K variants
Variant A: 64K BRAM   → aperture = 64K  (within 256K → pr_verify PASSES)
Variant B: 128K BRAM  → aperture = 128K (within 256K → pr_verify PASSES)
Variant C: 512K BRAM  → aperture = 512K (EXCEEDS 256K → pr_verify FAILS)
```

For Variant C, the golden design must be updated to define the maximum required aperture (512K), and all other PDIs must be recompiled.
변형 C의 경우 최대 요구 조리개(512K)를 정의하기 위해 골든 디자인을 업데이트해야 하며 다른 모든 PDI를 다시 컴파일해야 합니다.

## Notes
## 메모

- Aperture flexibility applies to PL slave address spaces; PS/DDR address apertures must remain identical across all designs
- 조리개 유연성은 PL 슬레이브 주소 공간에 적용됩니다. PS/DDR 주소 구멍은 모든 설계에서 동일하게 유지되어야 합니다.
- This feature is tracked by `pr_verify` — verify before deploying to hardware
- 이 기능은 'pr_verify'로 추적됩니다. 하드웨어에 배포하기 전에 확인하세요.
- For more details on Vivado NoC address assignment, refer to UG1387 (Versal Development Methodology Guide)
- Vivado NoC 주소 할당에 대한 자세한 내용은 UG1387(Versal Development Methodology Guide)을 참조하세요.
