<!-- ko-translation: enabled -->
# pr_verify Checks for Segmented Configuration
# pr_verify 분할된 구성 확인

Use `pr_verify` to compare routed checkpoints from different design runs within a project or between projects to check if the NoC solution is identical.
NoC 솔루션이 동일한지 확인하기 위해 프로젝트 내에서 또는 프로젝트 간에 서로 다른 설계 실행의 라우팅된 체크포인트를 비교하려면 `pr_verify`를 사용하세요.
This is **required** for PL reload use cases. **Always** use  -segcfg_only option with pr_verify.
이는 PL 다시 로드 사용 사례에 **필수**입니다. **항상** pr_verify와 함께 -segcfg_only 옵션을 사용하세요.

## Syntax
## 구문

```tcl
pr_verify -segcfg_only -initial <first_design>_routed.dcp -additional <second_design>_routed.dcp 
```

The first design checkpoint (`-initial`) must be the "golden" design that established the NoC solution reused in subsequent projects/runs.
첫 번째 디자인 체크포인트(`-initial`)는 후속 프로젝트/실행에서 재사용되는 NoC 솔루션을 확립한 "골든" 디자인이어야 합니다.
This initial run must contain the greatest usage of PS/PL and NoC boundary connections to establish the superset of possible connectivity.
이 초기 실행에는 가능한 연결의 상위 집합을 설정하기 위해 PS/PL 및 NoC 경계 연결을 최대한 많이 사용해야 합니다.

## Check Details
## 세부정보 확인

### HDPRVerify-01, HDPRVerify-02, HDPRVerify-03, HDPRVerify-37: Part & Design Property
### HDPRVerify-01, HDPRVerify-02, HDPRVerify-03, HDPRVerify-37: 부품 및 설계 속성

**Checks:**
**검사:**
- The same part/package/speedgrade must be targeted in both checkpoints
- 두 체크포인트 모두에서 동일한 부품/패키지/속도등급을 목표로 삼아야 합니다.
- The `SEGMENTED_CONFIGURATION` property must be present in both designs
- `SEGMENTED_CONFIGURATION` 속성은 두 디자인 모두에 있어야 합니다.

### SegConfig-Validation-1, SegConfig-Validation-2: Boot Path Endpoints
### SegConfig-Validation-1, SegConfig-Validation-2: 부팅 경로 끝점

**Checks:**
**검사:**
- The same number of boot paths are required in both designs
- 두 설계 모두 동일한 수의 부팅 경로가 필요합니다.
- The same number of end points are required
- 동일한 수의 엔드포인트가 필요합니다.

### SegConfig-Validation-3: Boot Path Settings
### SegConfig-Validation-3: 부팅 경로 설정

**Checks:**
**검사:**
- Same endpoint type (master/slave)
- 동일한 엔드포인트 유형(마스터/슬레이브)
- Same protocol used
- 동일한 프로토콜이 사용됨
- Same destination ID
- 동일한 대상 ID
- Same slave addresses
- 동일한 슬레이브 주소
- Same readTC/writeTC
- 동일한 readTC/writeTC

### SegConfig-Validation-4: Boot Path Routing
### SegConfig-Validation-4: 부팅 경로 라우팅

**Checks:**
**검사:**
- Same endpoint location
- 동일한 엔드포인트 위치
- Same virtual channel
- 동일한 가상 채널
- Same physical routes
- 동일한 물리적 경로

### SegConfig-Validation-10: PS & CPM to PL Interface
### SegConfig-Validation-10: PS 및 CPM에서 PL 인터페이스로

**Checks:** Netlist connections from/to PS and CPM cells
**확인 사항:** PS 및 CPM 셀과의 넷리스트 연결
- The PLD design **cannot add** new connections from/to PS or CPM netlist cell
- PLD 설계는 PS 또는 CPM 넷리스트 셀과의 새로운 연결을 **추가할 수 없습니다**
- Error checking is limited to pins ending with `*RCLKCLK` or `*PL*VALID`; changes to other pin names are reported as warnings
- 오류 검사는 `*RCLKCLK` 또는 `*PL*VALID`로 끝나는 핀으로 제한됩니다. 다른 핀 이름에 대한 변경 사항은 경고로 보고됩니다.
- The PLD design **can have fewer** connections from/to PS or CPM cell in secondary runs; this is tolerated as a warning
- PLD 설계는 2차 실행에서 PS 또는 CPM 셀과의 연결이 **더 적을 수 있습니다**. 이것은 경고로 용납됩니다.

### SegConfig-Validation-12: Boot Partition NMU Configurations
### SegConfig-Validation-12: 부팅 파티션 NMU 구성

**Checks:** Configuration of a NMU relies on address map and ID of its connected NSUs.
**확인 사항:** NMU 구성은 연결된 NSU의 주소 맵과 ID에 따라 달라집니다.
For each NMU on a boot path:
부팅 경로의 각 NMU에 대해 다음을 수행합니다.
- All connected NSUs must have the same addresses and IDs
- 연결된 모든 NSU의 주소와 ID는 동일해야 합니다.
- The secondary PLD design **cannot add** new connections to a boot path NMU
- 보조 PLD 설계는 부팅 경로 NMU에 새 연결을 **추가할 수 없습니다**
- The secondary PLD design **can have fewer** connections to a boot path NMU
- 보조 PLD 설계는 부팅 경로 NMU에 **더 적은 수**의 연결을 가질 수 있습니다.

**Known Issue:** `read_noc_solution` does not report an error if there are changes to NSU addresses connected to boot path NMU instances.
**알려진 문제:** `read_noc_solution`은 부팅 경로 NMU 인스턴스에 연결된 NSU 주소가 변경된 경우 오류를 보고하지 않습니다.
If changes have been made, `pr_verify` will report `[Dfx 88-139] SegConfig-Validation-12`.
변경된 경우 `pr_verify`는 `[Dfx 88-139] SegConfig-Validation-12`를 보고합니다.
Review NoC boot path connectivity to ensure all NSU instances have the same addresses and IDs.
NoC 부팅 경로 연결을 검토하여 모든 NSU 인스턴스의 주소와 ID가 동일한지 확인하세요.

### SegConfig-Validation-13, 14, 15, 16, 3: All NoC Sites in Boot Partition
### SegConfig-Validation-13, 14, 15, 16, 3: 부팅 파티션의 모든 NoC 사이트

**Checks:**
**검사:**
- Same number of NoC sites in boot and PLD designs
- 부팅 및 PLD 설계에서 동일한 수의 NoC 사이트
- Same NoC endpoint type, protocol, destination ID, slave addresses
- 동일한 NoC 엔드포인트 유형, 프로토콜, 대상 ID, 슬레이브 주소
- Same readTC/writeTC
- 동일한 readTC/writeTC

### SegConfig-Validation-17: Boot Path Master Connections
### SegConfig-Validation-17: 부팅 경로 마스터 연결

**Checks:**
**검사:**
- The secondary PLD design **cannot add** new connections from a boot path NMU
- 보조 PLD 설계는 부팅 경로 NMU에서 새 연결을 **추가할 수 없습니다**

### SegConfig-Validation-18: Boot Partition IO Usage
### SegConfig-Validation-18: 부팅 파티션 IO 사용

**Checks:**
**검사:**
- The secondary PLD design **cannot add** new IO to banks included in the boot partition
- 보조 PLD 설계는 부팅 파티션에 포함된 뱅크에 새 IO를 **추가할 수 없습니다**
- These banks are used for DDRMC and are not reprogrammed with the PLD image
- 이 뱅크는 DDRMC에 사용되며 PLD 이미지로 다시 프로그래밍되지 않습니다.

### SegConfig-Validation-20: Boot NMU Remap Consistency
### SegConfig-Validation-20: 부팅 NMU 재매핑 일관성

**Checks:**
**검사:**
- New remap entries cannot be added in secondary PLD designs
- 보조 PLD 설계에는 새로운 리맵 항목을 추가할 수 없습니다.

## Known Limitations
## 알려진 제한 사항

- `pr_verify` does not check all possible mismatches in the processor domain by design.
- `pr_verify`는 의도적으로 프로세서 도메인에서 가능한 모든 불일치를 확인하지 않습니다.
- Users should ensure the same CIPS customization and non-PL NoC connectivity when using PL reload.
- 사용자는 PL 다시 로드를 사용할 때 동일한 CIPS 사용자 정의 및 비PL NoC 연결을 보장해야 합니다.
- An NMU included in an initial_boot path could have different destIDs between configurations, causing PLM error when loading a PLD PDI from a different design than the one that generated the resident Boot PDI.
- initial_boot 경로에 포함된 NMU는 구성 간에 서로 다른 destID를 가질 수 있으며, 상주 Boot PDI를 생성한 설계와 다른 설계에서 PLD PDI를 로드할 때 PLM 오류가 발생할 수 있습니다.
- These checks will be modified in future releases to align with new flexibility and expanded to confirm consistency on PL interfaces as needed.
- 이러한 검사는 향후 릴리스에서 새로운 유연성에 맞춰 수정되고 필요에 따라 PL 인터페이스의 일관성을 확인하도록 확장될 것입니다.

## Interpreting Results
## 결과 해석

A successful pr_verify run confirms that:
성공적인 pr_verify 실행은 다음을 확인합니다.
1. The NoC solutions are identical between the two designs
1. NoC 솔루션은 두 설계 간에 동일합니다.
2. The PLD images may be safely interchanged
2. PLD 이미지는 안전하게 교환될 수 있습니다.
3. Boot partition resources are consistent
3. 부팅 파티션 리소스가 일관됩니다.

If pr_verify fails, review the specific error code against the table above and correct the design accordingly before generating PDI images.
pr_verify가 실패하면 위 표에 대한 특정 오류 코드를 검토하고 이에 따라 PDI 이미지를 생성하기 전에 디자인을 수정하세요.
