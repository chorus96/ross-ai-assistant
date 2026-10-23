<!-- ko-translation: enabled -->
---
name: segcfg-overview
description: Explain Segmented Configuration concepts for AMD Versal devices — what it is, why it exists, the two-phase boot model, supported device families, and how it compares to standard boot. Use this if someone asks what Segmented Configuration is, how it works, or whether it applies to their device.
license: MIT
compatibility: Applies to AMD Versal and Versal Gen 2 devices and their supported Vivado Segmented Configuration flows.
metadata:
   version: "1.0.0"
   stage: beta
allowed-tools: Read Bash Write
---

Explain the AMD Versal Segmented Configuration feature at a conceptual level and orient the user to the overall flow.
개념적 수준에서 AMD Versal 세그먼트 구성 기능을 설명하고 사용자에게 전반적인 흐름을 안내합니다.

## Instructions
## 지침

1. Explain **The Need** — why Segmented Configuration exists:
1. **필요** 설명 — 분할된 구성이 존재하는 이유:
   - Demand for faster boot and flexible PL configuration
   - 더 빠른 부팅과 유연한 PL 구성에 대한 요구
   - Boot the system to U-Boot/Linux OS before PL is loaded, optimizing time and flash size
   - PL이 로드되기 전에 시스템을 U-Boot/Linux OS로 부팅하여 시간과 플래시 크기를 최적화합니다.
   - Load PL image dynamically from a centralized image repository or boot device
   - 중앙 집중식 이미지 저장소 또는 부팅 장치에서 PL 이미지를 동적으로 로드합니다.
   - Deliver PL updates on the fly without full device reprogramming
   - 전체 장치 재프로그래밍 없이 즉시 PL 업데이트 제공

2. Explain the **fundamental design goals**:
2. **기본 디자인 목표**를 설명하세요.
   - Approach alignment to silicon features and boundaries, not legacy tool structures
   - 레거시 도구 구조가 아닌 실리콘 기능 및 경계에 대한 접근 방식 조정
   - Collaboration between Versal silicon, Vivado, Vitis, and embedded software groups
   - Versal 실리콘, Vivado, Vitis 및 임베디드 소프트웨어 그룹 간의 협력
   - Build on a standard "flat" design flow — no feature loss, no new project type
   - 표준 "플랫" 설계 흐름을 기반으로 구축 — 기능 손실 없음, 새로운 프로젝트 유형 없음
   - Minimize restrictions, complexity, and user intervention
   - 제한사항, 복잡성, 사용자 개입 최소화
   - Ensure compatibility with related solutions (Linux, PCIe Tandem, DFX, fallback, security)
   - 관련 솔루션(Linux, PCIe Tandem, DFX, fallback, 보안)과의 호환성 보장

3. Explain **Booting process split into two distinct phases**:
3. **부팅 프로세스가 두 단계로 나누어짐**에 대해 설명하세요.
   - **boot.pdi**: Contains PS, HNoC (Hard NoC), and DDRMC only.
   - **boot.pdi**: PS, HNoC(하드 NoC) 및 DDRMC만 포함합니다.
   - Boots processors and memory.
   - 프로세서와 메모리를 부팅합니다.
   - **pld.pdi**: Contains PL, AIE, VCU, ISP, VNoC, and any remaining resources.
   - **pld.pdi**: PL, AIE, VCU, ISP, VNoC 및 나머지 리소스를 포함합니다.
   - Loaded at runtime.
   - 런타임 시 로드됩니다.
   - PS-DDR NoC paths are automatically flagged for the boot partition
   - PS-DDR NoC 경로는 부팅 파티션에 대해 자동으로 플래그가 지정됩니다.
   - Standard flat Vivado implementation flow generates both PDI files from a single design
   - 표준 플랫 Vivado 구현 흐름은 단일 디자인에서 두 PDI 파일을 모두 생성합니다.
   - Isolation exists on the PS-PL boundary until pld.pdi is loaded
   - pld.pdi가 로드될 때까지 PS-PL 경계에 격리가 존재합니다.

4. Explain the **Software Stack**:
4. **소프트웨어 스택**에 대해 설명하세요.
   - Boot partition brings up: PLM, PSM, RPU (optional), NoC, DDR, ATF, U-Boot, Linux
   - 부팅 파티션 표시: PLM, PSM, RPU(옵션), NoC, DDR, ATF, U-Boot, Linux
   - PL partition adds: PL logic, ISP/VCU (Gen 2), ASU ELF, AIE array
   - PL 파티션 추가: PL 로직, ISP/VCU(Gen 2), ASU ELF, AIE 어레이
   - Each phase is self-contained — the OS is running before PL is ever configured
   - 각 단계는 독립적입니다. PL이 구성되기 전에 OS가 실행됩니다.

5. Present **Device Support** as of Vivado 2025.2:
5. Vivado 2025.2 기준 **장치 지원** 제공:

**Production — Segmented Configuration optional:**
**프로덕션 - 분할된 구성 옵션:**
| Device Family<br>장치 제품군 | Example Parts<br>예시 부품 |
   |---------------|---------------|
| Versal Prime Series<br>Versal 프라임 시리즈 | VM1102–VM2902<br>VM1102~VM2902 |
| Versal AI Core Series<br>Versal AI 코어 시리즈 | VC1502–VC2802<br>VC1502~VC2802 |
| Versal AI Edge Series<br>Versal AI 엣지 시리즈 | VE1752–VE2802<br>VE1752~VE2802 |
| Versal Premium Series<br>Versal 프리미엄 시리즈 | VP1002–VP2802 (SSI)<br>VP1002–VP2802(SSI) |
| Versal HBM Series<br>Versal HBM 시리즈 | VH1522–VH1782 (SSI)<br>VH1522~VH1782(SSI) |
| Development Kits<br>개발 키트 | VCK190, VEK280, VHK158, VMK180, VPK120, VPK180, Alveo V70/V80<br>VCK190, VEK280, VHK158, VMK180, VPK120, VPK180, Alveo V70/V80 |

**Production — Segmented Configuration always enabled (cannot be disabled):**
**프로덕션 — 분할된 구성은 항상 활성화됩니다(비활성화할 수 없음):**
| Device Family<br>장치 제품군 | Parts<br>부품 |
   |---------------|-------|
| Versal AI Edge Series Gen 2<br>Versal AI Edge 시리즈 Gen 2 | 2VE3804, 2VE3858<br>2VE3804, 2VE3858 |
| Versal Prime Series Gen 2<br>Versal 프라임 시리즈 Gen 2 | 2VM3858 |

**Early Access:**
**사전 체험:**
   - Additional Versal AI Edge Gen 2, Prime Gen 2 parts (check release notes for current list)
   - 추가 Versal AI Edge Gen 2, Prime Gen 2 부품(현재 목록은 릴리스 노트 확인)

Note: Versal RF Series is the LAST family where Segmented Configuration is optional.
참고: Versal RF 시리즈는 세그먼트 구성이 선택 사항인 마지막 제품군입니다.
All future adaptive SoC devices will have it as the only supported configuration flow.
향후 모든 적응형 SoC 장치에서는 이를 지원되는 유일한 구성 흐름으로 갖게 됩니다.

   
6. Explain **enabling Segmented Configuration**:
6. **세그먼트 구성 활성화**에 대해 설명하세요.

For optional (Gen 1) devices — two methods:
옵션(1세대) 장치의 경우 - 두 가지 방법:
   ```
   GUI: Tools > Settings > General > check "Project is a Segmented Configuration project"
   ```
   ```tcl
   set_property SEGMENTED_CONFIGURATION true [current_project]
   ```

For mandatory (Gen 2) devices:
필수(Gen 2) 장치의 경우:
   - Property is pre-set and cannot be disabled
   - 속성은 사전 설정되어 비활성화할 수 없습니다.
   - No Tcl command needed; segmented config is always active
   - Tcl 명령이 필요하지 않습니다. 분할된 구성은 항상 활성화되어 있습니다.

For non-project mode:
비프로젝트 모드의 경우:
   ```tcl
   link_design -part <part>
   set_property segmented_configuration true [current_design]
   ```

7. Summarize the **Vivado tool flow** at a high level:
7. 높은 수준에서 **Vivado 도구 흐름**을 요약합니다.
   - Design entry: block design (IP Integrator) or RTL (Modular NoC)
   - 설계 항목: 블록 설계(IP Integrator) 또는 RTL(Modular NoC)
   - Set NoC `initial_boot` paths for DDR connectivity in boot partition
   - 부팅 파티션에서 DDR 연결을 위한 NoC 'initial_boot' 경로 설정
   - Run standard synthesis → implementation → `write_device_image`
   - 표준 합성 실행 → 구현 → `write_device_image`
   - Two PDI files are generated: `boot.pdi` and `pld.pdi`
   - 두 개의 PDI 파일인 'boot.pdi' 및 'pld.pdi'가 생성됩니다.
   - Validate with Segmented Configuration DRCs (SEGCONFIG-1 through SEGCONFIG-7)
   - 세그먼트화된 구성 DRC(SEGCONFIG-1~SEGCONFIG-7)로 검증

8. Point to next steps based on user intent:
8. 사용자 의도에 따라 다음 단계를 가리킵니다.
   - Starting a new design → follow the project setup and enablement steps above
   - 새 디자인 시작 → 위의 프로젝트 설정 및 활성화 단계를 따르세요.
   - Setting up NoC boot paths → set and verify the NoC `initial_boot` paths in Vivado
   - NoC 부팅 경로 설정 → Vivado에서 NoC 'initial_boot' 경로 설정 및 확인
   - Understanding design constraints → use `/segcfg-design-considerations`
   - 설계 제약 이해 → `/segcfg-design-considerations` 사용
   - Generating PDI images → run implementation and `write_device_image` as described above
   - PDI 이미지 생성 → 위에 설명된 대로 구현 및 `write_device_image` 실행
   - PL reload across multiple images → use `/segcfg-pl-reload`
   - 여러 이미지에 걸쳐 PL 다시 로드 → `/segcfg-pl-reload` 사용
   - DFX + Segmented Configuration → use `/segcfg-dfx`
   - DFX + 세그먼트 구성 → `/segcfg-dfx` 사용
   - Programming devices → use `/segcfg-hw-manager`
   - 프로그래밍 장치 → `/segcfg-hw-manager` 사용
   - Building system software and runtime deployment → use the appropriate system-software workflow
   - 시스템 소프트웨어 구축 및 런타임 배포 → 적절한 시스템-소프트웨어 워크플로 사용
