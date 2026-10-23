<!-- ko-translation: enabled -->
# Detecting X5IO bank sharing conflict between PL and Boot partition in Gen2 devices, used when PL is using OCTAD resources from the bank, where boot partition includes DDRMC
# PL이 부팅 파티션에 DDRMC가 포함된 은행의 OCTAD 리소스를 사용할 때 사용되는 Gen2 장치의 PL과 부팅 파티션 간의 X5IO 뱅크 공유 충돌 감지

## Debug Instructions for Gen2 devices that are using X5IO OCTADs
## X5IO OCTAD를 사용하는 Gen2 장치에 대한 디버그 지침

> **IMPORTANT — Site vs Tile naming:**
> **중요 — 사이트 및 타일 이름 지정:**
> `SegConfig_BootTiles.tcl` contains **TILE** names (e.g.
> `SegConfig_BootTiles.tcl`에는 **TILE** 이름이 포함되어 있습니다(예:
> `CMPHY_OCTAD_CORE_X75Y0`).
> `CMPHY_OCTAD_CORE_X75Y0`).
> PLM error logs and register address maps report **SITE** names (e.g.
> PLM 오류 로그 및 레지스터 주소 맵은 **SITE** 이름(예:
> `CMPHY_OCTAD_X52Y0`).
> `CMPHY_OCTAD_X52Y0`).
> Site X-indices and tile X-indices use **different numbering schemes** — they are NOT the same.
> 사이트 X-색인과 타일 X-색인은 **다른 번호 매기기 체계**를 사용하며 동일하지 않습니다.
> Never construct a tile name by prepending `CMPHY_OCTAD_CORE_` to a site X-index, or vice versa.
> 사이트 X-색인 앞에 `CMPHY_OCTAD_CORE_`를 추가하여 타일 이름을 구성하지 마십시오. 반대의 경우도 마찬가지입니다.
> Always use the Vivado API to convert:
> 변환하려면 항상 Vivado API를 사용하십시오.
> - **Tile → Site**: `get_sites -of_objects [get_tiles $tile]`
> - **타일 → 사이트**: `get_sites -of_objects [get_tiles $tile]`
> - **Site → Tile**: `get_tiles -of_objects [get_sites $site]`
> - **사이트 → 타일**: `get_tiles -of_objects [get_sites $site]`

---

### Step 1 — Get boot partition OCTAD tiles from SegConfig_BootTiles.tcl
### 1단계 - SegConfig_BootTiles.tcl에서 부팅 파티션 OCTAD 타일 가져오기

`SegConfig_BootTiles.tcl` is in the `hd_visual/` folder of the impl run.
`SegConfig_BootTiles.tcl`은 impl 실행의 `hd_visual/` 폴더에 있습니다.
The grep returns **TILE** names.
grep은 **TILE** 이름을 반환합니다.

```bash
grep PHY_OCTAD_CORE ./SegConfig_BootTiles.tcl
```

---

### Step 2 — For each boot OCTAD tile, find its IO bank via the RIU_CK0 network
### 2단계 - 각 부팅 OCTAD 타일에 대해 RIU_CK0 네트워크를 통해 해당 IO 뱅크를 찾습니다.

CMPHY_OCTAD tiles do not have a direct `IOBANK` property.
CMPHY_OCTAD 타일에는 직접적인 'IOBANK' 속성이 없습니다.
The correct path is:
올바른 경로는 다음과 같습니다.
tile → site → `XCV2RIU_CK0` pin → nodes → PIPs → nodes → connected X5IO tiles → sites → IO bank.
타일 → 사이트 → `XCV2RIU_CK0` 핀 → 노드 → PIP → 노드 → 연결된 X5IO 타일 → 사이트 → IO 뱅크.

```tcl
# $tile is a TILE name from SegConfig_BootTiles.tcl (e.g. CMPHY_OCTAD_CORE_X75Y0)

# Step 2a: TILE -> SITE
set site [get_sites -of_objects [get_tiles $tile]]

# Step 2b: SITE -> connected X5IO tiles via RIU_CK0 network
set tileList [get_tiles -of_objects \
  [get_nodes -of_objects \
    [get_pips -of_objects \
      [get_nodes -of_objects \
        [get_site_pins $site/XCV2RIU_CK0]]]]]

# Step 2c: From those tiles -> sites -> IO bank
foreach item $tileList {
    set siteList [get_sites -of_objects [get_tiles $item]]
    foreach itemSite $siteList {
        puts [get_iobanks -of_objects [get_sites $itemSite]]
        break
    }
}
```

---

### Step 3 — Convert a PLM-reported site name to its tile (for partition membership check)
### 3단계 - PLM에서 보고한 사이트 이름을 타일로 변환합니다(파티션 멤버십 확인용).

When a PLM error or register address map gives a **site** name (e.g.
PLM 오류 또는 등록 주소 맵이 **사이트** 이름을 제공하는 경우(예:
`CMPHY_OCTAD_X52Y0`),
`CMPHY_OCTAD_X52Y0`),
use the API to get the corresponding **tile** before checking against the boot tile list:
부팅 타일 목록을 확인하기 전에 API를 사용하여 해당 **타일**을 가져옵니다.

```tcl
# site -> tile (correct — do NOT guess the tile name from the site X-index)
set tile [get_tiles -of_objects [get_sites CMPHY_OCTAD_X52Y0]]
# Example result: CMPHY_OCTAD_CORE_X75Y0  (X-index differs from site!)

# Check if that tile is in the boot partition tile list
set in_boot [expr {[lsearch $boot_octad_tiles $tile] >= 0}]
```

---

### Step 4 — Find banks of PL ports (e.g.
### 4단계 - PL 포트 뱅크 찾기(예:
### MIPI)
### 미피)

MIPI ports are always in the PL partition.
MIPI 포트는 항상 PL 파티션에 있습니다.
`IOBANK` is a direct property on IO ports.
'IOBANK'는 IO 포트의 직접적인 속성입니다.

```tcl
get_property IOBANK [get_ports MIPI*]
```

---

### Step 5 — Compare: MIPI port banks vs boot OCTAD banks
### 5단계 - 비교: MIPI 포트 뱅크와 부트 OCTAD 뱅크

1. If both lists contain the same bank number → **conflict**: boot and PL partitions share the same DDR bank resources.
1. 두 목록 모두 동일한 뱅크 번호를 포함하는 경우 → **충돌**: 부팅 및 PL 파티션은 동일한 DDR 뱅크 리소스를 공유합니다.
2. This usually triggers DRC violation `SegConfig-Validation-18`.
2. 이는 일반적으로 DRC 위반 'SegConfig-Validation-18'을 유발합니다.

---

### Solution
### 솔루션

Separate boot and PL logic so they do not share the same bank resources: X5PLL, OCTAD, I/O package pins, etc.
X5PLL, OCTAD, I/O 패키지 핀 등 동일한 뱅크 리소스를 공유하지 않도록 부팅 및 PL 로직을 분리합니다.
