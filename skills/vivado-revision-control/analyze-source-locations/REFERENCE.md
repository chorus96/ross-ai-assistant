<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

---
name: analyze-source-locations
이름: 분석 소스 위치
description: Step 2 — classifies source locations as Push Button, Remote, or Mixed to decide whether to export.
설명: 2단계 — 소스 위치를 푸시 버튼, 원격 또는 혼합으로 분류하여 내보낼지 여부를 결정합니다.
---

# Analyze Source Locations (Step 2)
# 소스 위치 분석(2단계)

**Source file:** `helper-procedures/helper_scripts.tcl`
**소스 파일:** `helper-procedures/helper_scripts.tcl`
**Proc name:** `analyze_source_locations`
**프록 이름:** `analyze_source_locations`

## Procedure
## 절차

```tcl
set scenario_info [analyze_source_locations]
```

Returns a dictionary:
사전을 반환합니다:

| Key<br>열쇠 | Type<br>유형 | Description<br>설명 |
|-----|------|-------------|
| `scenario`<br>'시나리오' | string<br>문자열 | "Push Button", "Remote", or "Mixed"<br>"푸시 버튼", "원격" 또는 "혼합" |
| `local_count`<br>`local_count` | integer<br>정수 | Files under project directory<br>프로젝트 디렉토리 아래의 파일 |
| `remote_count`<br>`remote_count` | integer<br>정수 | Files outside project directory<br>프로젝트 디렉토리 외부의 파일 |
| `remote_paths`<br>`원격_경로` | list<br>목록 | Unique paths of remote source files<br>원격 소스 파일의 고유 경로 |
| `strategy`<br>'전략' | string<br>문자열 | Human-readable recommendation<br>사람이 읽을 수 있는 권장사항 |

## Decision Logic
## 결정 논리

- **Push Button** (all local) → proceed to Step 3, export everything
- **푸시버튼**(모두 로컬) → 3단계로 진행하여 모두 내보내기
- **Remote** (all external) → skip Step 3 entirely, go to Step 4
- **원격**(모두 외부) → 3단계를 완전히 건너뛰고 4단계로 이동
- **Mixed** (some local, some external) → export local files in Step 3, document remote paths
- **혼합**(일부 로컬, 일부 외부) → 3단계에서 로컬 파일 내보내기, 원격 경로 문서화

The classification compares each file's normalized path against the project
분류는 각 파일의 정규화된 경로를 프로젝트와 비교합니다.
directory (`get_property DIRECTORY [current_project]`).
디렉토리(`get_property DIRECTORY [현재_프로젝트]`).
Generated files
생성된 파일
(`IS_GENERATED == true`) are excluded from the count.
(`IS_GENERATED == true`)는 계산에서 제외됩니다.

## Edge Cases
## 엣지 케이스

- **IP from shared repositories** — counted as remote; document the IP_REPO_PATHS for build.tcl
- **공유 저장소의 IP** — 원격으로 계산됩니다. build.tcl에 대한 IP_REPO_PATHS를 문서화하십시오.
- **No source files** — returns scenario "Unknown"; check that sources are added
- **소스 파일 없음** — "알 수 없음" 시나리오를 반환합니다. 소스가 추가되었는지 확인하세요
- **Symlinked sources** — `file normalize` resolves symlinks, so symlinked files may appear as remote
- **심볼릭 링크된 소스** — `file Normalize`는 심볼릭 링크를 확인하므로 심볼릭 링크된 파일이 원격으로 나타날 수 있습니다.

## When to Use vivado_doc_search
## vivado_doc_search를 사용해야 하는 경우

If unsure how Vivado resolves source file paths (e.g., IP_REPO_PATHS behavior,
Vivado가 소스 파일 경로(예: IP_REPO_PATHS 동작,
`-no_copy_sources` flag), use `vivado_doc_search` to look up `create_project`
`-no_copy_sources` 플래그), `vivado_doc_search`를 사용하여 `create_project`를 검색하세요.
or `set_property IP_REPO_PATHS`.
또는 `set_property IP_REPO_PATHS`.
