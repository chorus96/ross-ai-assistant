<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Learned-config cache
# 학습된 구성 캐시

`learned_params.json` is an **earned, regenerable** cache of parameter-DISCOVERY
`learned_params.json`은 매개변수-DISCOVERY의 **획득하고 재생성 가능한** 캐시입니다.
facts the skill builds as it runs (idea #3).
기술이 실행되면서 구축되는 사실(아이디어 #3)
It exists to stop re-deriving the
재파생을 막기 위해 존재합니다.
same `feature → param` mapping every run (which caused found-then-lost variance)
매 실행마다 동일한 `feature → param` 매핑(발견 후 손실된 차이 발생)
and to cut MCP calls on repeat features.
반복 기능에서 MCP 통화를 끊습니다.

## What it stores (and what it must never store)
## 저장하는 것(그리고 절대 저장해서는 안 되는 것)
Each entry is keyed `"<ip>|<normalized feature phrase>"` and holds only:
각 항목은 `"<ip>|<정규화된 기능 문구>"`로 입력되며 다음 항목만 보유합니다.
`param`, `shape`, `enabler`, `value_src` (`user`/`default`), `doc` (the citation
`param`, `shape`, `enabler`, `value_src`(`user`/`default`), `doc`(인용문
that established it), and `ip_version`.
이를 확립한 것) 및 `ip_version`.

It stores **no concrete parameter values** — there is no `value` field, and the
**구체적인 매개변수 값이 저장되지 않음** — '값' 필드가 없으며
engine (`../lib/ipcfg_cache.py`) strips any `value`/`expected*` field on write.
엔진(`../lib/ipcfg_cache.py`)은 쓰기 시 모든 `값`/`예상*` 필드를 제거합니다.
This is the blind-integrity guardrail: a benchmark run may consult and populate
이것이 블라인드 무결성 가드레일입니다. 벤치마크 실행은 참조하고 채울 수 있습니다.
the cache freely because it cannot leak an expected answer.
예상되는 답변이 유출될 수 없기 때문에 캐시를 자유롭게 사용할 수 있습니다.

## Why this is not the forbidden pre-built database
## 이것이 금지된 사전 구축 데이터베이스가 아닌 이유
- Entries are **earned at runtime** from doc search + Vivado feedback, not shipped.
- 항목은 문서 검색 + Vivado 피드백을 통해 **런타임에** 획득되며 배송되지 않습니다.
- Each entry is **version-stamped** (`ip_version`) and **regenerable** — delete the
- 각 항목에는 **버전 스탬프**(`ip_version`)가 지정되어 있으며 **재생성** 가능합니다.
file and the skill rebuilds it.
파일을 다시 작성하면 스킬이 다시 작성됩니다.
- It records *where a feature lives*, never *what value the prompt wanted*.
- *기능이 어디에 있는지* 기록하며 *프롬프트가 원하는 값*은 기록하지 않습니다.

To reset: `echo '{}' > learned_params.json` (or just delete it).
재설정하려면 `echo '{}' > learn_params.json`(또는 그냥 삭제)하세요.
