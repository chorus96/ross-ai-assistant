<!-- ko-translation: enabled -->
---
name: segcfg-dfx
description: Explain how Segmented Configuration interacts with Dynamic Function eXchange (DFX) on AMD Versal devices. Use this if someone asks about using DFX with Segmented Configuration.
license: MIT
compatibility: Requires Vivado 2026.1+ and an AMD Versal Segmented Configuration design using Dynamic Function eXchange.
metadata:
     version: "1.0.0"
     stage: beta
allowed-tools: Read Bash Write
---

## Segmented Configuration + DFX overview
## 분할된 구성 + DFX 개요
Support for Segmented Configuration feature on top of DFX design flow will be available from the 2026.1 Vivado release.
DFX 설계 흐름 외에 세그먼트 구성 기능에 대한 지원은 2026.1 Vivado 릴리스부터 제공됩니다.
Both PS/PL partition and reconfigurable partitions are to be supported in the same design.
PS/PL 파티션과 재구성 가능한 파티션이 모두 동일한 설계에서 지원됩니다.
For produced configuration images, each reconfigurable partition will still have one reconfigurable image; the full device configuration image of DFX flow is to be replaced by two Segmented Configuration images, such are boot and pld PDI files.
생성된 구성 이미지의 경우 각 재구성 가능한 파티션에는 여전히 하나의 재구성 가능한 이미지가 있습니다. DFX 흐름의 전체 장치 구성 이미지는 boot 및 pld PDI 파일과 같은 두 개의 세그먼트화된 구성 이미지로 대체됩니다.
The reconfigurable partitions must be within the PL partition, e.g., the PL partition includes all reconfigurable partitions, logically and physically.
재구성 가능한 파티션은 PL 파티션 내에 있어야 합니다. 예를 들어 PL 파티션에는 논리적으로나 물리적으로 재구성 가능한 모든 파티션이 포함됩니다.
The PL partition image is the child of the boot partition image and it is the parent of the reconfigurable partition images.
PL 파티션 이미지는 부팅 파티션 이미지의 하위 이미지이며 재구성 가능한 파티션 이미지의 상위 이미지입니다.
Abstract shell design is treated as pure DFX design, no additional constraint is expected.
추상 셸 디자인은 순수 DFX 디자인으로 처리되므로 추가적인 제약이 예상되지 않습니다.

## Pre-requisites
## 전제조건
1. For basic Segmented Configuration flow creation, load the `segcfg-overview` skill and enable Segmented Configuration before converting the project to a DFX design with the instructions below.
1. 기본 세그먼트 구성 흐름 생성의 경우 아래 지침에 따라 프로젝트를 DFX 디자인으로 변환하기 전에 'segcfg-overview' 스킬을 로드하고 세그먼트 구성을 활성화합니다.

## Instructions
## 지침
1. NoC logical paths in "IP Integrator (block design) RM container " **can't** be set to initial_boot, since end point connected to NoC AXI interface can't be placed in boot partition.
1. NoC AXI 인터페이스에 연결된 엔드포인트를 부팅 파티션에 배치할 수 없기 때문에 "IP Integrator(블록 설계) RM 컨테이너"의 NoC 논리 경로는 **initial_boot로 설정할 수 없습니다**.
1. The user  needs to ensure that the reconfigurable partitions are properly defined and placed within the PL partition.
1. 사용자는 재구성 가능한 파티션이 PL 파티션 내에 올바르게 정의되고 배치되었는지 확인해야 합니다.

2. After block design creation and validation, project should be converted to DFX.
2. 블록 설계 생성 및 검증 후 프로젝트를 DFX로 변환해야 합니다.
2. Used the following command to convert the project to DFX:
2. 다음 명령을 사용하여 프로젝트를 DFX로 변환했습니다.
```tcl
     set_property PR_FLOW 1 [current_project]
```

Set this property **before** enabling reconfigurability on the RM container (do this before steps 3 and 4).
RM 컨테이너에서 재구성 기능을 활성화하기 **전에** 이 속성을 설정하십시오(3단계와 4단계 전에 이 작업을 수행하십시오).

3. Reconfigurable module in block design is expected to be container type, since RM container is required for DFX design.
3. DFX 설계에는 RM 컨테이너가 필요하므로 블록 설계의 재구성 가능한 모듈은 컨테이너 유형이 될 것으로 예상됩니다.
3. If user creates RM as non-container type, this will cause issue during implementation, since DFX flow expects RM to be container type.
3. 사용자가 RM을 컨테이너가 아닌 유형으로 생성하는 경우 DFX 흐름에서는 RM이 컨테이너 유형일 것으로 예상하므로 구현 중에 문제가 발생합니다.
a.
To create RM container, follow the steps below:
RM 컨테이너를 생성하려면 다음 단계를 따르세요.

- In the top-level block design, create a hierarchy that contains the instances you want inside the reconfigurable region:
- 최상위 블록 디자인에서 재구성 가능 영역 내에 원하는 인스턴스를 포함하는 계층 구조를 만듭니다.
- Select the IP blocks → right-click → Create Hierarchy
- IP 블록 선택 → 마우스 오른쪽 버튼 클릭 → 계층 생성
- Right-click the created hierarchy → Create Block Design Container
- 생성된 계층을 우클릭 → 블록 디자인 컨테이너 생성
- Provide the BDC name → Vivado creates a new child .bd source in the Sources window
- BDC 이름 제공 → Vivado는 Sources 창에서 새로운 하위 .bd 소스를 생성합니다.
(“The BDC name specified … is assigned to the .bd source file … e.g. rp_container.bd”.)
(“지정된 BDC 이름은 … .bd 소스 파일에 할당됩니다 … 예: rp_container.bd”.)

b.
To create container instance in top level block design, follow the steps below:
최상위 블록 디자인에서 컨테이너 인스턴스를 생성하려면 아래 단계를 따르십시오.
- Once the BDC is created, the hierarchy in the top canvas is replaced by a BDC cell instance that instantiates the new .bd you created.
- BDC가 생성되면 상단 캔버스의 계층 구조는 생성된 새 .bd를 인스턴스화하는 BDC 셀 인스턴스로 대체됩니다.
- You’ll see the BDC cell on the top BD canvas (and you can double-click to open the customize dialog later).
- 상단 BD 캔버스에 BDC 셀이 표시됩니다(나중에 두 번 클릭하여 사용자 정의 대화 상자를 열 수 있습니다).

- Convert RM  instance into container.
- RM 인스턴스를 컨테이너로 변환합니다.
- Refer to following example command to convert RM instance into container type:
- RM 인스턴스를 컨테이너 유형으로 변환하려면 다음 예제 명령을 참조하세요.
```tcl
create_bd_design -cell [get_bd_cells /rp_container] rp_container
set new_cell [create_bd_cell -type container -reference rp_container rp_container_temp]
replace_bd_cell [get_bd_cells /rp_container] $new_cell
delete_bd_objs  [get_bd_cells /rp_container]
set_property name rp_container $new_cell
```
- Go to step 4. to make the container reconfigurable.
- 4단계로 이동하여 컨테이너를 재구성 가능하게 만듭니다.

4. To make RM container reconfigurable the following command should be executed:
4. RM 컨테이너를 재구성 가능하게 만들려면 다음 명령을 실행해야 합니다.
```tcl
     set_property CONFIG.ENABLE_DFX {true} [get_bd_cells rp_container]
```
This is required for the tool to recognize the partition as reconfigurable and to generate the appropriate images and constraints for it.
이는 도구가 파티션을 재구성 가능한 것으로 인식하고 이에 대한 적절한 이미지와 제약 조건을 생성하는 데 필요합니다.
The user also needs to ensure that the reconfigurable partitions are properly defined and placed within the PL partition, and that they are connected to the necessary interfaces for their intended functionality.
또한 사용자는 재구성 가능한 파티션이 PL 파티션 내에 올바르게 정의 및 배치되었는지, 의도한 기능에 필요한 인터페이스에 연결되어 있는지 확인해야 합니다.

5. Create pblock constraint that will contain several clock regions from project device, and add RM reconfigurable module cells to it.
5. 프로젝트 장치의 여러 클록 영역을 포함하는 pblock 제약 조건을 생성하고 여기에 RM 재구성 가능 모듈 셀을 추가합니다.
5. Example of pblock constraint for RM container:
5. RM 컨테이너에 대한 pblock 제약 조건의 예:
```tcl
     create_pblock pblock_rp
     add_cells_to_pblock [get_pblocks pblock_rp_container] [get_cells -hierarchical \
     path/to/rp_container]
     resize_pblock [get_pblocks pblock_rp] -add {CLOCKREGION_X2Y2:CLOCKREGION_X3Y3}
```
Add constraint to project.This is required for the tool to recognize the partition as reconfigurable and to generate the appropriate images and constraints for it.
프로젝트에 제약 조건을 추가합니다. 이는 도구가 파티션을 재구성 가능한 것으로 인식하고 이에 대한 적절한 이미지와 제약 조건을 생성하는 데 필요합니다.

6. The boot image is expected to be identical to the one generated with DFX feature turned off.
6. 부팅 이미지는 DFX 기능을 끈 상태에서 생성된 이미지와 동일할 것으로 예상됩니다.
6. Exceptions are expected only in rare cases, such as HSR master bank special handling for blackbox design.
6. 블랙박스 설계를 위한 HSR 마스터 뱅크 특별 처리 등 드문 경우에만 예외가 예상됩니다.

7. The pld image is expected to use pure DFX design bitmap and everything else should be identical to the one generated with DFX feature turned off.
7. pld 이미지는 순수한 DFX 디자인 비트맵을 사용할 것으로 예상되며 그 밖의 모든 것은 DFX 기능을 끈 상태에서 생성된 이미지와 동일해야 합니다.
7. Comparing to files generated with DFX feature off, no rnpi configuration difference is expected.
7. DFX 기능을 끈 상태에서 생성된 파일과 비교할 때 rnpi 구성 차이는 예상되지 않습니다.
7. PLM nodes and design ID related with DFX partitions are exceptions.
7. DFX 파티션과 관련된 PLM 노드 및 디자인 ID는 예외입니다.
7. Comparing to files generated with DFX feature off, rcdo bitmap differences are expected only in RCLK clock buffer, since DFX design instead of flat design bitmap is used.
7. DFX 기능을 끈 상태에서 생성된 파일과 비교하면 플랫 디자인 비트맵 대신 DFX 디자인이 사용되므로 RCLK 클럭 버퍼에서만 rcdo 비트맵 차이가 예상됩니다.
7. PLM node and design ID handling wise, the pld image should include DFX partitions as children.
7. PLM 노드 및 디자인 ID를 현명하게 처리하려면 pld 이미지에 DFX 파티션이 하위 항목으로 포함되어야 합니다.
