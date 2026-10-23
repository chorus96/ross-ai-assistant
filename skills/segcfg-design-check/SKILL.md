<!-- ko-translation: enabled -->
---
name: segcfg-design-check
description: Run Segmented Configuration DRCs and pr_verify design verification checks. Use when validating a design for IO bank conflicts, clocking issues, DDRMC sharing violations, or when comparing routed checkpoints for PL reload compatibility.
license: MIT
compatibility: Requires Vivado and routed design checkpoints for pr_verify comparisons.
metadata:
   version: "1.0.0"
   stage: beta
allowed-tools: Read Bash Write
---

Guide the user through running Segmented Configuration DRCs and pr_verify, and generate the corresponding Tcl commands.
세그먼트 구성 DRC 및 pr_verify 실행을 통해 사용자를 안내하고 해당 Tcl 명령을 생성합니다.

## Instructions
## 지침

1. Read `references/segmented-drcs.md` for the full DRC list with error messages.
1. 오류 메시지가 포함된 전체 DRC 목록을 보려면 'references/segmented-drcs.md'를 읽어보세요.
Read `references/pr-verify-checks.md` for pr_verify error codes and what they check.
pr_verify 오류 코드와 확인 내용은 `references/pr-verify-checks.md`를 읽어보세요.

2. Ask the user which checks they need:
2. 사용자에게 어떤 확인이 필요한지 물어보세요.
   - **Interactive DRCs** (post-synthesis): Validate IO banks, clocking, DDRMC resource sharing
   - **대화형 DRC**(합성 후): IO 뱅크 검증, 클로킹, DDRMC 리소스 공유
   - **pr_verify** (post-route): Confirm NoC solution compatibility between a golden and variant design for PL reload
   - **pr_verify**(포스트 경로): PL 재로드를 위한 골든 설계와 변형 설계 간의 NoC 솔루션 호환성을 확인합니다.

3. For **DRC checks**, generate Tcl:
3. **DRC 확인**의 경우 Tcl을 생성합니다.
   ```tcl
   # Run all Segmented Configuration-specific DRCs on post-synthesis checkpoint
   report_drc -checks {SEGCONFIG-1 SEGCONFIG-2 SEGCONFIG-3 SEGCONFIG-4 SEGCONFIG-5 SEGCONFIG-6 SEGCONFIG-7}
   ```

4. Explain each DRC and guidance on resolution:
4. 각 DRC와 해결 지침을 설명하세요.

| DRC<br>콩고민주공화국 | Checks<br>수표 | Resolution<br>해상도 |
   |-----|--------|-----------|
| SEGCONFIG-1<br>SEGCONFIG-1 | Shared IO bank (boot + PLD use same bank)<br>공유 IO 뱅크(부팅 + PLD는 동일한 뱅크 사용) | IO will not be reprogrammed on PL Reload; accept or move PL IO to a dedicated bank<br>IO는 PL 재로드 시 다시 프로그래밍되지 않습니다. PL IO를 수락하거나 전용 은행으로 이동 |
| SEGCONFIG-2<br>SEGCONFIG-2 | Non-LVCMOS PLD port in a shared IO bank<br>공유 IO 뱅크의 비LVCMOS PLD 포트 | Change PLD port IOSTANDARD to LVCMOS<br>PLD 포트 IOSTANDARD를 LVCMOS로 변경 |
| SEGCONFIG-3<br>SEGCONFIG-3 | Clocking tile used by boot design<br>부츠 디자인에 사용되는 클러킹 타일 | Move clocking resources out of boot domain; typically should be in PLD<br>클럭킹 리소스를 부팅 도메인 외부로 이동합니다. 일반적으로 PLD에 있어야 합니다. |
| SEGCONFIG-4<br>SEGCONFIG-4 | XPLL or CLK_PLL_AND_PHY tile shared by boot and PLD<br>부팅 및 PLD에서 공유되는 XPLL 또는 CLK_PLL_AND_PHY 타일 | Reorganize clock planning to dedicate tiles to one partition<br>타일을 하나의 파티션에 전용으로 사용하도록 시계 계획을 재구성합니다. |
| SEGCONFIG-5<br>SEGCONFIG-5 | Used master X5IO bank not in boot partition (Gen 2)<br>부팅 파티션에 없는 사용된 마스터 X5IO 뱅크(Gen 2) | Move master bank DDR connection to boot partition or leave unused<br>마스터 뱅크 DDR 연결을 부팅 파티션으로 이동하거나 사용하지 않은 상태로 둡니다. |
| SEGCONFIG-6<br>SEGCONFIG-6 | Reconfigurable pblock contains boot partition resources<br>재구성 가능한 pblock에는 부팅 파티션 리소스가 포함되어 있습니다. | Resize pblock: `get_dfx_footprint -seg_config_boot` shows boot tile footprint<br>pblock 크기 조정: `get_dfx_footprint -seg_config_boot`가 부팅 타일 공간을 표시합니다. |
| SEGCONFIG-7<br>SEGCONFIG-7 | DDRMC subsystem tile shared between boot and PLD partition<br>부팅 및 PLD 파티션 간에 공유되는 DDRMC 하위 시스템 타일 | Redesign DDR connectivity to avoid cross-partition DDRMC subsystem sharing<br>파티션 간 DDRMC 하위 시스템 공유를 방지하기 위해 DDR 연결을 재설계합니다. |

5. For **pr_verify** (PL Reload compatibility), generate Tcl:
5. **pr_verify**(PL 다시 로드 호환성)의 경우 Tcl을 생성합니다.
   ```tcl
   # Compare golden (initial) design against a PL variant (additional)
   # The -initial DCP must be the golden design containing the superset of connectivity
   pr_verify -segcfg_only -initial <golden_design>_routed.dcp -additional <variant_design>_routed.dcp
   ```

6. Explain key pr_verify error codes from the reference file:
6. 참조 파일의 주요 pr_verify 오류 코드를 설명하세요.
   - **HDPRVerify-01/02/03/37**: Part, package, speed grade, and SEGMENTED_CONFIGURATION property must match
   - **HDPRVerify-01/02/03/37**: 부품, 패키지, 속도 등급 및 SEGMENTED_CONFIGURATION 속성이 일치해야 합니다.
   - **SegConfig-Validation-1/2**: Same number of boot paths and endpoints required
   - **SegConfig-Validation-1/2**: 동일한 수의 부팅 경로 및 엔드포인트가 필요합니다.
   - **SegConfig-Validation-3**: Boot path settings (protocol, dest ID, slave addresses, TC values) must match
   - **SegConfig-Validation-3**: 부팅 경로 설정(프로토콜, 대상 ID, 슬레이브 주소, TC 값)이 일치해야 합니다.
   - **SegConfig-Validation-4**: Boot path routing (location, virtual channel, physical routes) must match
   - **SegConfig-Validation-4**: 부팅 경로 라우팅(위치, 가상 채널, 물리적 경로)이 일치해야 합니다.
   - **SegConfig-Validation-10**: PS/CPM-to-PL interface connections — PLD cannot add new connections
   - **SegConfig-Validation-10**: PS/CPM-PL 인터페이스 연결 — PLD는 새 연결을 추가할 수 없습니다.
   - **SegConfig-Validation-12**: Boot partition NMU configurations — NSU addresses and IDs must match
   - **SegConfig-Validation-12**: 부팅 파티션 NMU 구성 - NSU 주소와 ID가 일치해야 합니다.
   - **SegConfig-Validation-13–16**: All NoC sites in boot partition must match
   - **SegConfig-Validation-13–16**: 부팅 파티션의 모든 NoC 사이트는 일치해야 합니다.
   - **SegConfig-Validation-17**: PLD cannot add new connections from a boot-path NMU
   - **SegConfig-Validation-17**: PLD는 부팅 경로 NMU에서 새 연결을 추가할 수 없습니다.
   - **SegConfig-Validation-18**: PLD cannot add new IO to boot-partition banks
   - **SegConfig-Validation-18**: PLD는 부팅 파티션 뱅크에 새 IO를 추가할 수 없습니다.
   - **SegConfig-Validation-20**: Boot NMU remap entries cannot be added in PLD designs
   - **SegConfig-Validation-20**: PLD 설계에 부팅 NMU 재매핑 항목을 추가할 수 없습니다.

7. If pr_verify fails with **SegConfig-Validation-12**, check that all NSU instances connected to boot NMUs have identical addresses and IDs between golden and variant designs.
7. **SegConfig-Validation-12**로 인해 pr_verify가 실패하는 경우 부팅 NMU에 연결된 모든 NSU 인스턴스가 골든 설계와 변형 설계 간에 동일한 주소와 ID를 가지고 있는지 확인하세요.
7. Use `read_noc_solution` to import the golden .ncr file before implementation.
7. 구현 전에 `read_noc_solution`을 사용하여 골든 .ncr 파일을 가져옵니다.


8. If pr_verify fails with **SegConfig-Validation-18**, check that the PLD design has not added new IO usages in banks included in the boot partition.
8. **SegConfig-Validation-18**과 함께 pr_verify가 실패하는 경우 PLD 설계가 부팅 파티션에 포함된 뱅크에 새로운 IO 사용을 추가하지 않았는지 확인하세요.
8. These banks are used for DDRMC and are not reprogrammed with the PLD image, so adding new IOs here would cause runtime errors.
8. 이러한 뱅크는 DDRMC에 사용되며 PLD 이미지로 다시 프로그래밍되지 않으므로 여기에 새 IO를 추가하면 런타임 오류가 발생합니다.
   - Read `references/x5io-bank-sharing-debug.md` for detailed diagnosing, guidance and resolving IO bank sharing issues in Segmented Configuration designs.
   - 세그먼트 구성 설계의 IO 뱅크 공유 문제에 대한 자세한 진단, 지침 및 해결 방법은 'references/x5io-bank-sharing-debug.md'를 참조하세요.

9. Remind the user that `pr_verify` is **required** for all PL Reload use cases and must be run on post-route checkpoints.
9. 'pr_verify'는 모든 PL Reload 사용 사례에 **필수**이며 경로 후 체크포인트에서 실행되어야 함을 사용자에게 상기시킵니다.

10. Examples of checks performed represented in below table:
10. 수행된 검사의 예는 아래 표에 나와 있습니다.

| Type of Check<br>수표 유형 | Conditions Required<br>필수 조건 |
|----------|-------------|
| Boot path endpoints<br>부팅 경로 끝점 | The same number of booth paths and end points are required<br>동일한 수의 부스 경로와 종료 지점이 필요합니다. |
| Boot path settings​<br>부팅 경로 설정​  |  Same endpoint type (master/slave)​, Same protocol used​, Same destination ID​, Same slave addresses​<br>동일한 엔드포인트 유형(마스터/슬레이브)​, 동일한 프로토콜 사용​, 동일한 대상 ID​, 동일한 슬레이브 주소​ |
| Boot path routing<br>부팅 경로 라우팅 | Same endpoint location​, Same virtual channel​, Same physical routes<br>동일한 엔드포인트 위치, 동일한 가상 채널, 동일한 물리적 경로 |
| Boot path masters connections​<br>부팅 경로 마스터 연결​ | The secondary PLD design cannot add new connection from a boot path NMU​<br>보조 PLD 설계는 부팅 경로 NMU에서 새 연결을 추가할 수 없습니다. |
| Boot partition IO usage<br>부팅 파티션 IO 사용량 | The secondary PLD design cannot add new IO to banks included in the boot partition.<br>보조 PLD 설계는 부팅 파티션에 포함된 뱅크에 새 IO를 추가할 수 없습니다. |
| Boot partition IO usage<br>부팅 파티션 IO 사용량 |     These banks are used for DDRMC and are not reprogrammed with the PLD image<br>이 뱅크는 DDRMC에 사용되며 PLD 이미지로 다시 프로그래밍되지 않습니다. |




