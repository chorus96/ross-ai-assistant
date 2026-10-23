<!-- ko-translation: enabled -->
---
name: segcfg-pl-reload
description: >
  Guides the Versal Segmented Configuration PL reload workflow for creating and validating
  multiple independently compiled `pld.pdi` variants while keeping `boot.pdi` fixed. Use
  when a user asks how to set up a golden PL reload design, spawn a new PL variant project,
  preserve NoC boot paths with `write_noc_solution`, size address apertures for reload
  compatibility, run `pr_verify -segcfg_only`, or debug `unique_id` / `parent_unique_id`
  mismatches rejected by PLM.
license: MIT
compatibility: Requires Vivado 2025.2+ and routed checkpoints for AMD Versal Segmented Configuration PL reload designs.
metadata:
  version: "1.0.0"
  stage: beta
allowed-tools: Read Bash Write
---

# Segmented Configuration PL Reload
# 세그먼트화된 구성 PL 다시 로드

## Overview
## 개요

**Purpose:** Guide the user through golden project setup, variant spawning, NoC solution reuse, address aperture sizing, `pr_verify`, and UID compatibility for Versal PL reload flows.
**목적:** 골든 프로젝트 설정, 변형 생성, NoC 솔루션 재사용, 주소 조리개 크기 조정, 'pr_verify' 및 Versal PL 다시 로드 흐름을 위한 UID 호환성을 통해 사용자를 안내합니다.

**Output:** Guidance plus copy-pasteable Tcl and shell snippets for the user's current PL reload scenario.
**출력:** 사용자의 현재 PL 다시 로드 시나리오에 대한 지침과 복사하여 붙여 넣을 수 있는 Tcl 및 셸 조각입니다.
This skill does not generate `REPORT.md`.
이 스킬은 `REPORT.md`를 생성하지 않습니다.

**Supporting references:** Read `references/reload-methodology.md` for the full PL reload flow and `references/address-aperture.md` for aperture flexibility details.
**지원 참조:** 전체 PL 다시 로드 흐름을 보려면 `references/reload-methodology.md`를 읽고 조리개 유연성 세부 정보를 보려면 `references/address-aperture.md`를 읽어보세요.

---

## Prerequisites
## 전제조건

| Requirement<br>요구 사항 | Details<br>세부정보 |
|---|---|
| Vivado version<br>비바도 버전 | 2025.2 or later (2026.1 or later if combining DFX with Segmented Configuration)<br>2025.2 이상(DFX와 세그먼트화된 구성을 결합하는 경우 2026.1 이상) |
| Target family<br>대상가족 | Versal devices using Segmented Configuration<br>세그먼트 구성을 사용하는 Versal 장치 |
| Segmented configuration project setting<br>분할 구성 프로젝트 설정 | For first-generation Versal devices where this feature is optional, set `set_property segmented_configuration true [current_project]` before implementation and image generation<br>이 기능이 선택사항인 1세대 Versal 장치의 경우 구현 및 이미지 생성 전에 `set_property 세그먼트화 구성 true [현재_프로젝트]`를 설정하세요. |
| Design state<br>디자인 상태 | Golden design routed before generating the `.ncr` file; golden and variant designs routed before running `pr_verify`<br>`.ncr` 파일을 생성하기 전에 라우팅된 골든 디자인; `pr_verify`를 실행하기 전에 골든 및 변형 디자인이 라우팅되었습니다. |
| Open project<br>프로젝트 열기 | A Vivado project (`.xpr`) or routed checkpoint (`.dcp`) must be available for live validation<br>Vivado 프로젝트(`.xpr`) 또는 라우팅된 체크포인트(`.dcp`)를 실시간 검증에 사용할 수 있어야 합니다. |
| Vivado session<br>비바도 세션 | Connected via the MCP Vivado bridge or an interactive Tcl console<br>MCP Vivado 브리지 또는 대화형 Tcl 콘솔을 통해 연결됨 |

---

## Efficiency Guidelines
## 효율성 지침

- **Pass `session_id`** to every `vivado_execute` call when a Vivado session is active.
- **Vivado 세션이 활성화되면 모든 `vivado_execute` 호출에 `session_id`**를 전달합니다.
- **Write reports to file** — do not output full report content in chat; give a short summary only.
- **파일에 보고서 작성** — 채팅에서 전체 보고서 내용을 출력하지 않습니다. 간략한 요약만 제공하세요.
- **Read reports efficiently** — use `grep`, `sed`, or `awk` via terminal to extract specific sections instead of reading entire files into context.
- **보고서를 효율적으로 읽기** — 전체 파일을 컨텍스트로 읽는 대신 터미널을 통해 `grep`, `sed` 또는 `awk`를 사용하여 특정 섹션을 추출합니다.
- Use `wc -l` + `head` to check size first.
- 크기를 먼저 확인하려면 `wc -l` + `head`를 사용하세요.
- Full `read_file` is fine only for small reports (<200 lines).
- 전체 `read_file`은 작은 보고서(200줄 미만)에만 적합합니다.
- **Do NOT** use `shell ls`, `shell find`, or `shell glob` to locate files.
- **하지 마십시오** `shell ls`, `shell find` 또는 `shell glob`을 사용하여 파일을 찾습니다.
- **Do NOT** use Vivado Tcl (`exec cat`, `open`, `read`) to read files.
- **하지 마십시오** Vivado Tcl(`exec cat`, `open`, `read`)을 사용하여 파일을 읽습니다.
- Use `grep`/`sed` via terminal or `read_file` with line ranges.
- 터미널을 통해 `grep`/`sed`를 사용하거나 줄 범위와 함께 `read_file`을 사용하세요.
- **Do NOT** retry a failed Tcl command with different syntax.
- **하지 마십시오** 다른 구문으로 실패한 Tcl 명령을 다시 시도하십시오.
- Report the error and stop or proceed.
- 오류를 보고하고 중지하거나 계속 진행하세요.

---

## Workflow (Autonomous)
## 워크플로우(자율)

**⚠️ CRITICAL: Execute steps SEQUENTIALLY.
**⚠️ 중요: 단계를 순차적으로 실행하세요.
Wait for each command to complete.**
각 명령이 완료될 때까지 기다립니다.**

Segcfg PL Reload Progress:
Segcfg PL 다시 로드 진행률:
- [ ] Step 1: Read the bundled references and identify the user's PL reload scenario
- [ ] 1단계: 번들 참조를 읽고 사용자의 PL 다시 로드 시나리오 식별
- [ ] Step 2: Confirm or explain the golden design requirements
- [ ] 2단계: 골든 디자인 요구 사항 확인 또는 설명
- [ ] Step 3: Choose a safe variant spawning method
- [ ] 3단계: 안전한 변종 생성 방법 선택
- [ ] Step 4: Preserve NoC solution and address aperture compatibility
- [ ] 4단계: NoC 솔루션 보존 및 애퍼처 호환성 문제 해결
- [ ] Step 5: Run `pr_verify -segcfg_only` on routed designs
- [ ] 5단계: 라우팅된 설계에서 `pr_verify -segcfg_only` 실행
- [ ] Step 6: Check UID compatibility and summarize the next action
- [ ] 6단계: UID 호환성 확인 및 다음 작업 요약

### Step 1: Read the references and identify the scenario
### 1단계: 참고 자료를 읽고 시나리오 파악

Read `references/reload-methodology.md` and `references/address-aperture.md` before answering detailed methodology questions.
자세한 방법론 질문에 답하기 전에 'references/reload-methodology.md' 및 'references/address-aperture.md'를 읽어보세요.

If the user has not already made it clear, ask which scenario they need help with:
사용자가 아직 명확하게 설명하지 않은 경우 어떤 상황에서 도움이 필요한지 물어보세요.
- Setting up a golden project from scratch
- 골든 프로젝트를 처음부터 설정하기
- Spawning a new PL variant from an existing golden design
- 기존 골든 디자인에서 새로운 PL 변형 생성
- Debugging a `pr_verify` failure
- `pr_verify` 실패 디버깅
- Understanding address aperture requirements
- 주소 조리개 요구 사항 이해
- Debugging UID compatibility failures at runtime
- 런타임 시 UID 호환성 오류 디버깅

If live Vivado validation is needed, open the current design with a single-line Tcl command that handles either a checkpoint or a project workspace:
실시간 Vivado 검증이 필요한 경우 체크포인트나 프로젝트 작업공간을 처리하는 한 줄의 Tcl 명령을 사용하여 현재 디자인을 엽니다.

```tcl
set dcp [lindex [glob -nocomplain *.dcp] 0]; if {$dcp != ""} { open_checkpoint $dcp } elseif {[catch {current_design}]} { open_run impl_1 }; puts "Design: [current_design]"
```

Use `timeout_seconds: 18000` for long-running implementation commands such as `place_design` or `route_design`.
`place_design` 또는 `route_design`과 같은 장기 실행 구현 명령에는 `timeout_seconds: 18000`을 사용하세요.

### Step 2: Confirm the golden design requirements
### 2단계: 골든 디자인 요구 사항 확인

Explain and enforce these golden-design rules:
이러한 골든 디자인 규칙을 설명하고 시행하십시오.
- The golden design is the **superset** of all PS-PL boundary connectivity needed by any reload variant.
- 골든 디자인은 모든 재장전 변형에 필요한 모든 PS-PL 경계 연결의 **상위 세트**입니다.
- Enable every PS interface, NoC path, pin usage, and address aperture that any future variant may require.
- 향후 변형에 필요할 수 있는 모든 PS 인터페이스, NoC 경로, 핀 사용 및 주소 조리개를 활성화합니다.
- Keep the PS domain frozen across variants; only the PL domain may change.
- 변형 전반에 걸쳐 PS 도메인을 고정 상태로 유지합니다. PL 도메인만 변경될 수 있습니다.
- Compile the golden and all variants in the **same Vivado release**.
- **동일한 Vivado 릴리스**에서 골든 및 모든 변형을 컴파일합니다.
- For first-generation Versal devices where Segmented Configuration is optional, enable the project property before processing the design.
- 세그먼트 구성이 선택 사항인 1세대 Versal 장치의 경우 설계를 처리하기 전에 프로젝트 속성을 활성화합니다.

If needed, enable Segmented Configuration explicitly in project mode:
필요한 경우 프로젝트 모드에서 세그먼트 구성을 명시적으로 활성화합니다.

```tcl
set_property segmented_configuration true [current_project]
```

After the golden design reaches `route_design`, generate the NoC solution file:
골든 디자인이 'route_design'에 도달한 후 NoC 솔루션 파일을 생성합니다.

```tcl
set ncr [file normalize "[current_design].ncr"]; write_noc_solution -force $ncr; puts "Generated NoC solution: $ncr"
```

### Step 3: Choose a safe variant spawning method
### 3단계: 안전한 변종 생성 방법 선택

Generate Tcl for one of these supported methods:
지원되는 다음 방법 중 하나에 대해 Tcl을 생성합니다.

```tcl
# Option A: Clone the entire project
save_project_as new_variant_project
```

```tcl
# Option B: Import the golden block design directory into a new project
import_files <path_to_golden>/<golden_project>.gen/sources_1/bd
```

```tcl
# Option C: Script the block design for recreation in the variant project
write_bd_tcl -force golden_bd.tcl; write_bd_tcl -hier_blks ps_hier -force golden_ps.tcl
```

**Warning:** Do NOT use `read_bd` — it creates a reference to the golden `.bd` source instead of a copy, so edits in the variant can corrupt the golden source.
**경고:** `read_bd`를 사용하지 마세요. 복사본 대신 골든 `.bd` 소스에 대한 참조를 생성하므로 변형을 편집하면 골든 소스가 손상될 수 있습니다.

### Step 4: Preserve NoC solution and address aperture compatibility
### 4단계: NoC 솔루션을 보존하고 애퍼처 호환성 문제 해결

Import the golden `.ncr` file into the variant flow.
골든 `.ncr` 파일을 변형 흐름으로 가져옵니다.
Prefer setting the implementation run property:
구현 실행 속성을 설정하는 것이 좋습니다.

```tcl
set_property NOC_SOLUTION_FILE <path_to_golden>/impl_1/<design>.ncr [get_runs impl_1]
```

Behind the scenes, this calls `read_noc_solution` before `place_design`.
뒤에서는 `place_design` 전에 `read_noc_solution`을 호출합니다.

If manually calling Tcl, use the syntax form supported by your Vivado release:
Tcl을 수동으로 호출하는 경우 Vivado 릴리스에서 지원하는 구문 형식을 사용하세요.

```tcl
read_noc_solution <path_to_golden>/impl_1/<design>.ncr
```

```tcl
read_noc_solution -file <path_to_golden>/impl_1/<design>.ncr
```

Then explain the aperture rules:
그런 다음 조리개 규칙을 설명하십시오.
- The golden design defines the **largest** aperture needed for each PL slave interface.
- 황금색 디자인은 각 PL 슬레이브 인터페이스에 필요한 **가장 큰** 조리개를 정의합니다.
- Variant designs may use a **subset** of that golden aperture.
- 변형 디자인에서는 해당 황금 조리개의 **하위 집합**을 사용할 수 있습니다.
- Variant designs may **not exceed** the golden aperture.
- 다양한 디자인은 골드 조리개를 **초과**할 수 없습니다.
- Set apertures explicitly on INI ports so automated address assignment does not overwrite them.
- 자동 주소 할당으로 인해 덮어쓰지 않도록 INI 포트에 명시적으로 조리개를 설정합니다.

### Step 5: Run `pr_verify -segcfg_only` after implementation
### 5단계: 구현 후 `pr_verify -segcfg_only` 실행

Use routed checkpoints only:
라우팅된 체크포인트만 사용하십시오.

```tcl
pr_verify -segcfg_only -initial <golden_project>/impl_1/<golden>_routed.dcp -additional <variant_project>/impl_1/<variant>_routed.dcp
```

When `pr_verify` fails, direct the user to `/segcfg-design-check` for detailed DRC and `pr_verify` failure guidance, and use the aperture rules above to diagnose superset/subset mismatches.
'pr_verify'가 실패하면 자세한 DRC 및 'pr_verify' 실패 안내를 위해 사용자를 '/segcfg-design-check'로 안내하고 위의 조리개 규칙을 사용하여 상위 집합/하위 집합 불일치를 진단합니다.

### Step 6: Check UID compatibility and boot partition tile consistency
### 6단계: UID 호환성 및 부팅 파티션 타일 일관성 확인

Explain and verify the runtime compatibility rules:
런타임 호환성 규칙을 설명하고 확인합니다.
- `boot.pdi` keeps the PS domain fixed while `pld.pdi` changes.
- `boot.pdi`는 `pld.pdi`가 변경되는 동안 PS 도메인을 고정된 상태로 유지합니다.
- The boot image `unique_id` is derived from the PS domain construction.
- 부팅 이미지 `unique_id`는 PS 도메인 구성에서 파생됩니다.
- The PLD image `parent_unique_id` must match the boot image `unique_id`.
- PLD 이미지 'parent_unique_id'는 부팅 이미지 'unique_id'와 일치해야 합니다.
- Any PS-domain change (CIPS, NoC, DDR, boot-path tiles) can invalidate reload compatibility.
- PS 도메인 변경(CIPS, NoC, DDR, 부팅 경로 타일)으로 인해 다시 로드 호환성이 무효화될 수 있습니다.

Use `bootgen` to inspect both PDIs:
두 PDI를 모두 검사하려면 `bootgen`을 사용하세요.

```bash
bootgen -arch versal -read <boot.pdi>
bootgen -arch versal -read <pld.pdi>
```

Also compare `SegConfig_BootTiles.tcl` from the golden and variant implementation `hd_visual` directories.
또한 골든 및 변형 구현 `hd_visual` 디렉터리의 `SegConfig_BootTiles.tcl`을 비교해 보세요.
Differences indicate a boot-partition mismatch that can lead to UID incompatibility.
차이점은 UID 비호환성을 초래할 수 있는 부팅 파티션 불일치를 나타냅니다.

---

## Error Handling
## 오류 처리

| Error<br>오류 | Symptom<br>증상 | Action<br>액션 |
|---|---|---|
| No design open<br>열린 디자인 없음 | `ERROR: No current design`<br>`오류: 현재 디자인이 없습니다` | Use the Step 1 workspace auto-detect Tcl to `open_checkpoint` or `open_run impl_1` before running validation commands.<br>유효성 검사 명령을 실행하기 전에 1단계 작업 공간을 사용하여 Tcl을 'open_checkpoint' 또는 'open_run impl_1'로 자동 감지하세요. |
| Missing `.ncr` file<br>'.ncr' 파일 누락 | `read_noc_solution` or `NOC_SOLUTION_FILE` points to a missing file<br>`read_noc_solution` 또는 `NOC_SOLUTION_FILE`이 누락된 파일을 가리킵니다. | Re-run the golden design through `route_design`, then run `write_noc_solution` and use the generated absolute `.ncr` path.<br>`route_design`을 통해 골든 디자인을 다시 실행한 다음 `write_noc_solution`을 실행하고 생성된 절대 `.ncr` 경로를 사용합니다. |
| `pr_verify` aperture mismatch<br>'pr_verify' 조리개 불일치 | `pr_verify -segcfg_only` reports address or connectivity incompatibility<br>'pr_verify -segcfg_only'는 주소 또는 연결 비호환성을 보고합니다. | Confirm the golden design is the superset, ensure variants only shrink apertures, and review `references/address-aperture.md`.<br>골든 디자인이 상위 세트인지 확인하고 변형이 조리개만 축소되었는지 확인하고 'references/address-aperture.md'를 검토하세요. |
| PLM rejects `pld.pdi`<br>PLM이 'pld.pdi'를 거부합니다. | `parent_unique_id` does not match `boot.pdi unique_id`<br>`parent_unique_id`가 `boot.pdi Unique_id`와 일치하지 않습니다. | Run `bootgen -arch versal -read` on both PDIs and compare IDs; then compare `SegConfig_BootTiles.tcl` to identify PS-domain drift.<br>두 PDI 모두에서 'bootgen -arch Versal -read'를 실행하고 ID를 비교하세요. 그런 다음 `SegConfig_BootTiles.tcl`을 비교하여 PS 도메인 드리프트를 식별합니다. |
| `read_bd` was used in a variant flow<br>변형 흐름에서 'read_bd'가 사용되었습니다. | Variant edits unexpectedly affect the golden block design<br>변형 편집이 골든 블록 디자인에 예기치 않게 영향을 미침 | Recreate the variant with `save_project_as`, `import_files`, or `write_bd_tcl`; never use `read_bd` for PL reload variants.<br>`save_project_as`, `import_files` 또는 `write_bd_tcl`을 사용하여 변형을 다시 만듭니다. PL 다시 로드 변형에는 `read_bd`를 사용하지 마세요. |

---

## Validation
## 검증

Success criteria for a correct PL reload flow:
올바른 PL 다시 로드 흐름의 성공 기준:
- ✓ Golden design completed implementation and generated the `.ncr` NoC solution file
- ✓ Golden design 구현 완료 및 `.ncr` NoC 솔루션 파일 생성
- ✓ Variant project was created with `save_project_as`, `import_files`, or `write_bd_tcl` — never `read_bd`
- ✓ `save_project_as`, `import_files` 또는 `write_bd_tcl`을 사용하여 변형 프로젝트가 생성되었습니다. 절대로 `read_bd`가 아닙니다.
- ✓ `pr_verify -segcfg_only` passes between the routed golden and variant checkpoints
- ✓ `pr_verify -segcfg_only`는 라우팅된 골든 체크포인트와 변형 체크포인트 사이를 통과합니다.
- ✓ `bootgen -arch versal -read` shows the variant `parent_unique_id` matches the boot image `unique_id`
- ✓ `bootgen -arch Versal -read`는 변형 `parent_unique_id`가 부팅 이미지 `unique_id`와 일치함을 보여줍니다.
- ✓ Golden and variant `SegConfig_BootTiles.tcl` files match for the boot partition
- ✓ Golden 및 변형 `SegConfig_BootTiles.tcl` 파일이 부팅 파티션과 일치합니다.

---

## Key Restrictions
## 주요 제한사항

- Mixed IO banks are supported but should be avoided because IO becomes active before PL loads and cannot change on reload.
- 혼합 IO 뱅크가 지원되지만 IO는 PL 로드 전에 활성화되고 다시 로드 시 변경할 수 없으므로 피해야 합니다.
- DFX within the PL is supported with Segmented Configuration starting in Vivado 2026.1; for DFX + Segmented Configuration scenarios, use `/segcfg-dfx`.
- PL 내의 DFX는 Vivado 2026.1부터 세그먼트 구성으로 지원됩니다. DFX + 세그먼트 구성 시나리오의 경우 '/segcfg-dfx'를 사용하세요.
- PL Reload support is limited to designs compiled in the same Vivado release.
- PL Reload 지원은 동일한 Vivado 릴리스에서 컴파일된 디자인으로 제한됩니다.
- For Gen 2 devices, avoid designs where both DDRMC5 ports connect to the PL domain.
- Gen 2 장치의 경우 두 DDRMC5 포트가 모두 PL 도메인에 연결되는 설계를 피하십시오.

---

## References
## 참고자료

- **UG1387**: Versal Adaptive SoC Hardware, IP, and Platform Development Methodology
- **UG1387**: Versal Adaptive SoC 하드웨어, IP 및 플랫폼 개발 방법론
- **UG949**: UltraFast Design Methodology Guide
- **UG949**: UltraFast 설계 방법론 가이드
- **UG835**: Vivado Design Suite Tcl Command Reference Guide
- **UG835**: Vivado Design Suite Tcl 명령 참조 가이드
- **UG909**: Vivado Design Suite User Guide: Dynamic Function eXchange
- **UG909**: Vivado Design Suite 사용자 가이드: 동적 기능 교환
- **UG1283**: Bootgen User Guide
- **UG1283**: Bootgen 사용자 가이드
