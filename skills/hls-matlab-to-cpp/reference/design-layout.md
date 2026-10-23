<!-- ko-translation: enabled -->
---
description: HLS design workspace manager — creates the canonical directory tree for a design at each pipeline stage. Called by /matlab-to-cpp and /architect.
argument-hint: design_name=<name> stage=<golden|sample_based|frame_based|rearchitect_v1|rearchitect_v2|show>
---
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Design Layout
# 디자인 레이아웃

Manages the canonical directory workspace for an HLS design as it progresses through the
HLS 설계가 진행되는 동안 표준 디렉터리 작업 공간을 관리합니다.
conversion pipeline.
변환 파이프라인.
Called by `/matlab-to-cpp` and `/hls-architect` — never run manually
`/matlab-to-cpp` 및 `/hls-architect`에 의해 호출됨 - 수동으로 실행하지 않음
unless inspecting or recovering a workspace.
작업 공간을 검사하거나 복구하지 않는 한.

---

## Canonical Tree
## 정식 트리

```
design_name/
├── golden/               ← MATLAB source files + matlab_input.bin + matlab_golden.bin
├── sample_based/         ← refactor_1: plain C++ kernel + testbench
└── frame_based/          ← refactor_2: frame-based C++ kernel + testbench (OUTPUT from matlab-to-cpp)
```

**Note:** The `rearchitect/` directory is created by `/hls-architect` after matlab-to-cpp completes.
**참고:** `rearchitect/` 디렉터리는 matlab-to-cpp가 완료된 후 `/hls-architect`에 의해 생성됩니다.
See `/hls-architect/reference/design-layout.md` for that structure.
해당 구조는 `/hls-architect/reference/design-layout.md`를 참조하세요.

`design_name` = MATLAB script name, snake_case, no `.m` extension.
`design_name` = MATLAB 스크립트 이름, snake_case, `.m` 확장자 없음.

---

## Stage Definitions
## 단계 정의

| `stage` arg<br>`단계` 인수 | Directories created<br>생성된 디렉터리 | Extra files written<br>추가 파일이 작성됨 |
|---|---|---|
| `golden`<br>'황금' | `design_name/golden/`<br>`디자인_이름/황금/` | none (caller copies .m + .bin files in)<br>없음(발신자가 .m + .bin 파일을 복사함) |
| `sample_based`<br>`샘플 기반` | `design_name/sample_based/`<br>`디자인_이름/샘플_기반/` | none (caller writes kernel.cpp + testbench.cpp)<br>없음(호출자가 kernel.cpp + testbench.cpp를 작성함) |
| `frame_based`<br>`프레임 기반` | `design_name/frame_based/`<br>`디자인_이름/프레임_기반/` | none (caller writes kernel.cpp + testbench.cpp)<br>없음(호출자가 kernel.cpp + testbench.cpp를 작성함) |
| `show`<br>'쇼' | (nothing created)<br>(아무것도 생성되지 않음) | prints expected tree<br>예상 트리를 인쇄합니다. |

---

## Behavior
## 행동

### For any stage (except `show`)
### 모든 무대(`show` 제외)

1. Receive `design_name` and `stage` from the calling skill's context
1. 호출 스킬의 컨텍스트에서 `design_name` 및 `stage`를 수신합니다.
2. Look up the stage in the table above
2. 위 표에서 무대를 찾아보세요
3. Create directories with parents:
3. 부모와 함께 디렉터리를 만듭니다.
   ```bash
   mkdir -p <path>
   ```

4. Print confirmation — example for `frame_based`:
4. 인쇄 확인 — `frame_based`의 예:
   ```
   [design-layout] Created design_name/frame_based/
     ← write kernel.cpp and testbench.cpp here
   ```

5. Return the root path of the created stage directory to the calling skill.
5. 생성된 스테이지 디렉터리의 루트 경로를 호출 스킬에 반환합니다.

### For `stage=show`
### `무대=쇼`의 경우

Print the full expected tree without creating anything:
아무것도 생성하지 않고 전체 예상 트리를 인쇄합니다.

```
[design-layout] Workspace for design_name (matlab-to-cpp stages):
design_name/
├── golden/          ← MATLAB sources + golden bins
├── sample_based/    ← Refactor 1: line-by-line C++ port
└── frame_based/     ← Refactor 2: frame-based C++ (OUTPUT → handed to /hls-architect)
```

---

## Callers
## 발신자

| Skill<br>스킬 | When it calls<br>그것이 부를 때 | Stage<br>무대 |
|---|---|---|
| `/matlab-to-cpp`<br>`/matlab-to-cpp` | Before Step 0<br>0단계 전 | `show` (orientation)<br>'쇼'(방향) |
| `/matlab-to-cpp`<br>`/matlab-to-cpp` | End of Step 0<br>0단계 종료 | `golden`<br>'황금' |
| `/matlab-to-cpp`<br>`/matlab-to-cpp` | End of Step 2<br>2단계 종료 | `sample_based`<br>`샘플 기반` |
| `/matlab-to-cpp`<br>`/matlab-to-cpp` | End of Step 3<br>3단계 종료 | `frame_based`<br>`프레임 기반` |

---

## Rules
## 규칙

- **Never create directories outside `design_name/`**
- **`design_name/` 외부에 디렉터리를 만들지 마세요**
- **Never overwrite existing directories** — if the directory already exists, print a warning and skip creation; do not delete existing content
- **기존 디렉터리를 덮어쓰지 마세요** — 디렉터리가 이미 존재하는 경우 경고를 인쇄하고 생성을 건너뜁니다. 기존 콘텐츠를 삭제하지 마세요
