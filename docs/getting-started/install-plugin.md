<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Install as a plugin
# 플러그인으로 설치

This repository is an [Agent Plugins 1.0](https://agent-plugins.org/specification) package: `plugin.json` at the root, with every skill under `skills/`.
이 저장소는 [에이전트 플러그인 1.0](https://agent-plugins.org/사양) 패키지입니다. 루트에는 `plugin.json`이 있고 모든 기술은 `skills/` 아래에 있습니다.
A local clone of it is the plugin, and every client below installs from that clone.
로컬 복제본은 플러그인이며 아래의 모든 클라이언트는 해당 복제본에서 설치됩니다.

Find your client in [step 2](#2-configure-your-client) and follow that one section.
[2단계](#2-configure-your-client)에서 클라이언트를 찾고 해당 섹션을 따르세요.

> **Choose one install method.** If you install the plugin, do not also install the skills with `npx skills add` or by copying them into `~/.claude/skills/`.
> **설치 방법 중 하나를 선택하세요.** 플러그인을 설치하는 경우 `npx Skills add`를 사용하거나 `~/.claude/skills/`에 복사하여 스킬도 설치하지 마세요.
> Every skill would then reach the model twice.
> 그러면 모든 기술이 모델에 두 번 도달하게 됩니다.

The plugin contains skills only.
플러그인에는 스킬만 포함되어 있습니다.
The Vivado MCP Server or the Vivado AI Extension from the [Prerequisites](../../README.md#prerequisites) is still installed separately.
Vivado MCP 서버 또는 [전제 조건](../../README.md#prerequisites)의 Vivado AI Extension은 여전히 ​​별도로 설치됩니다.

## 1.
## Clone the repository
## 저장소 복제

```bash
git clone https://github.com/Xilinx/ross-ai-assistant /path/to/ross-ai-assistant
```

The clone is the **plugin root** below.
복제본은 아래의 **플러그인 루트**입니다.
Keep it intact: skills read their own `scripts/` and `reference/` files by relative path.
그대로 유지: 스킬은 상대 경로를 통해 자체 `scripts/` 및 `reference/` 파일을 읽습니다.
To stay on a specific release, check out its tag, for example `git -C /path/to/ross-ai-assistant checkout <tag>`.
특정 릴리스를 유지하려면 해당 태그(예: `git -C /path/to/ross-ai-assistant checkout <tag>`)를 확인하세요.

Cursor is the exception: it loads local plugins only from one fixed directory, so clone there instead.
커서는 예외입니다. 하나의 고정 디렉터리에서만 로컬 플러그인을 로드하므로 대신 거기에 복제하세요.
See [Cursor](#cursor).
[커서](#cursor)를 참조하세요.

## 2.
## Configure your client
## 클라이언트 구성

### Cursor
### 커서

Cursor has no "install from disk" action.
커서에는 "디스크에서 설치" 작업이 없습니다.
It scans `~/.cursor/plugins/local/`, so cloning there *is* the install.
`~/.cursor/plugins/local/`을 검색하므로 거기에 복제하는 것이 *설치*입니다.

1. Clone so that `plugin.json` lands **exactly one level** below `local/`:
1. `plugin.json`이 `local/`보다 **정확히 한 수준** 아래에 위치하도록 복제합니다.

    ```bash
    git clone https://github.com/Xilinx/ross-ai-assistant ~/.cursor/plugins/local/ross-ai-assistant
    ```

    ```powershell
    git clone https://github.com/Xilinx/ross-ai-assistant "$env:USERPROFILE\.cursor\plugins\local\ross-ai-assistant"
    ```

The result must be `~/.cursor/plugins/local/ross-ai-assistant/plugin.json`, with no extra level in between.
결과는 `~/.cursor/plugins/local/ross-ai-assistant/plugin.json`이어야 하며, 사이에 추가 레벨이 없어야 합니다.

2. Enable **Cursor Settings → Rules, Skills, Subagents → Include third-party Plugins, Skills, and other configs**.
2. **커서 설정 → 규칙, 기술, 하위 에이전트 → 타사 플러그인, 기술 및 기타 구성 포함**을 활성화합니다.
2. Local plugins do not load without it.
2. 이 플러그인이 없으면 로컬 플러그인이 로드되지 않습니다.

3. Restart Cursor.
3. 커서를 다시 시작하십시오.

Clone or copy, do not link: Cursor rejects a symlink or junction whose target lies outside `local/`.
복제 또는 복사, 연결 안 함: 커서는 대상이 'local/' 외부에 있는 심볼릭 링크나 접합을 거부합니다.
If you also use Claude Code, see [Several clients on one machine](#several-clients-on-one-machine) first.
Claude Code도 사용하는 경우 먼저 [한 시스템의 여러 클라이언트](#several-clients-on-one-machine)를 참조하세요.

### VS Code / GitHub Copilot
### VS 코드/GitHub 부조종사

VS Code reads the clone where it lies.
VS Code는 복제본이 있는 위치에서 복제본을 읽습니다.

1. Open **Preferences: Open User Settings (JSON)**.
1. **기본 설정: 사용자 설정(JSON) 열기**를 엽니다.

2. Register the plugin root:
2. 플러그인 루트를 등록합니다:

    ```json
    {
      "chat.plugins.enabled": true,
      "chat.pluginLocations": {
        "/path/to/ross-ai-assistant": true
      }
    }
    ```

On Windows the key is a JSON string, so escape the separators: `"C:\\Users\\you\\ross-ai-assistant": true`.
Windows에서 키는 JSON 문자열이므로 구분 기호를 이스케이프 처리합니다: `"C:\\Users\\you\\ross-ai-assistant": true`.

3. Reload the window.
3. 창을 다시 로드합니다.

`chat.plugins.enabled` is required.
`chat.plugins.enabled`가 필요합니다.
Without it VS Code ignores plugins and gives no error.
그것이 없으면 VS Code는 플러그인을 무시하고 오류를 발생시키지 않습니다.

### Claude Code
### 클로드 코드

Claude Code installs only through a *marketplace*, so the repository ships a single-entry one that points at itself:
Claude Code는 *마켓플레이스*를 통해서만 설치되므로 저장소는 자신을 가리키는 단일 항목을 제공합니다.

```shell
/plugin marketplace add /path/to/ross-ai-assistant
/plugin install amd-ross-agentic-ai-assistant@amd-ross-agentic-ai-assistant
```

The repeated name is expected: `/plugin install` takes `plugin@marketplace`, and here both are this repository.
반복되는 이름이 예상됩니다. `/plugin install`은 `plugin@marketplace`를 사용하며 여기에 둘 다 이 저장소가 있습니다.

Claude Code copies the plugin into `~/.claude/plugins/cache/`, so it survives moving the clone.
Claude Code는 플러그인을 `~/.claude/plugins/cache/`에 복사하므로 복제본을 이동해도 유지됩니다.
To try it for one session without installing:
설치하지 않고 한 세션 동안 사용해 보려면:

```bash
claude --plugin-dir /path/to/ross-ai-assistant
```

### Codex
### 코덱스

Codex also installs through a marketplace, shipped at `.agents/plugins/marketplace.json`:
Codex는 `.agents/plugins/marketplace.json`에 제공되는 마켓플레이스를 통해서도 설치됩니다.

```bash
codex plugin marketplace add /path/to/ross-ai-assistant
codex plugin add amd-ross-agentic-ai-assistant@amd-ross-agentic-ai-assistant
```

Codex copies the plugin into `~/.codex/plugins/cache/`.
Codex는 플러그인을 `~/.codex/plugins/cache/`에 복사합니다.
Browse what is installed with `/plugins`.
`/plugins`로 설치된 것을 찾아보세요.

### Kiro
### 키로

Kiro calls Agent Plugins **powers** and reads the same manifest.
Kiro는 Agent Plugins **powers**를 호출하고 동일한 매니페스트를 읽습니다.

1. Open the **Powers** panel → **Add Custom Power**
1. **Powers** 패널 열기 → **Custom Power 추가**
2. Choose **Import power from a folder**
2. **폴더에서 전원 가져오기**를 선택하세요.
3. Select the plugin root
3. 플러그인 루트를 선택하세요
4. Click **Install**
4. **설치**를 클릭하세요.

Powers activate on context, keyed off the `keywords` in `plugin.json`, such as `vivado`, `vitis-hls`, `chipscope`, `versal` and `timing-closure`.
전원은 'vivado', 'vitis-hls', 'chipscope', 'versal' 및 'timing-closure'와 같은 'plugin.json'의 '키워드'에 맞춰 컨텍스트에 따라 활성화됩니다.
Mentioning one in a prompt brings the skills into play.
프롬프트에서 하나를 언급하면 ​​기술이 작동됩니다.

### Devin CLI
### 데빈 CLI

```bash
devin plugins install --local /path/to/ross-ai-assistant
```

A `--local` install links to the clone, so changes to it apply from the next session.
`--local` 설치는 복제본에 연결되므로 변경 사항은 다음 세션부터 적용됩니다.
Devin lists the skills as `/amd-ross-agentic-ai-assistant:<skill>`.
Devin은 기술을 `/amd-ross-agentic-ai-assistant:<skill>`로 나열합니다.

### Antigravity
### 반중력

Antigravity uses its own plugin manifest, and its schema does not allow the extra fields in this repository's `plugin.json`.
Antigravity는 자체 플러그인 매니페스트를 사용하며 해당 스키마는 이 저장소의 `plugin.json`에 추가 필드를 허용하지 않습니다.
Instead of the clone from step 1, paste this prompt into Antigravity's agent.
1단계의 복제 대신 이 프롬프트를 Antigravity 에이전트에 붙여넣습니다.
It clones the repository into Antigravity's plugin directory and writes a minimal manifest:
Antigravity의 플러그인 디렉터리에 저장소를 복제하고 최소한의 매니페스트를 작성합니다.

```text
Install the AMD Ross Agentic AI Assistant skills as an Antigravity plugin.

1. Set DIR to ~/.gemini/config/plugins/amd-ross-agentic-ai-assistant
   (on Windows, %USERPROFILE%\.gemini\config\plugins\amd-ross-agentic-ai-assistant).
2. If DIR does not exist, create its parent directories and clone
   https://github.com/Xilinx/ross-ai-assistant into it. If DIR already exists,
   update it in place instead: git -C DIR fetch origin, then
   git -C DIR reset --hard origin/main.
3. Overwrite DIR/plugin.json so it contains exactly this and nothing else:
   {
     "name": "amd-ross-agentic-ai-assistant",
     "description": "Agent Skills for Developing Products with AMD Embedded Technologies"
   }
   The repository ships a cross-client manifest with extra fields that
   Antigravity's plugin schema does not allow.
4. Check that DIR/plugin.json exists and count the DIR/skills/*/SKILL.md
   files. Report the count. If it is zero, stop and say the install failed.
5. Tell me to reload Antigravity, and that re-running this prompt is how to
   update to a later release.
```

Re-run the prompt whenever you want the latest release.
최신 릴리스를 원할 때마다 프롬프트를 다시 실행하세요.
To pin a specific release instead, tell the agent to check out that tag in step 2.
대신 특정 릴리스를 고정하려면 상담사에게 2단계에서 해당 태그를 확인하라고 지시하세요.
If you use the Antigravity CLI rather than the IDE, clone the repository anywhere and run `agy plugin install <path to the clone>` after step 3.
IDE가 아닌 Antigravity CLI를 사용하는 경우, 아무데나 저장소를 복제하고 3단계 이후 `agy 플러그인 설치 <복제 경로>`를 실행합니다.

## 3.
## Verify
## 확인

| Client<br>클라이언트 | Where the skills appear<br>스킬이 나타나는 곳 |
|---|---|
| Cursor<br>커서 | **Customize → Skills**, under "Agent Decides"<br>**사용자 지정 → 기술**, "에이전트 결정" 아래 |
| VS Code<br>VS 코드 | **Chat: Configure Skills**, and the plugin under **Agent Plugins - Installed**<br>**채팅: 기술 구성** 및 **에이전트 플러그인 - 설치됨** 아래의 플러그인 |
| Claude Code<br>클로드 코드 | `claude plugin list`, or `/plugin`<br>`claude 플러그인 목록` 또는 `/plugin` |
| Codex<br>코덱스 | `/plugins`<br>`/플러그인` |
| Kiro<br>키로 | the **Powers** panel<br>**Powers** 패널 |
| Devin CLI<br>데빈 CLI | `devin plugins list`<br>`devin 플러그인 목록` |
| Antigravity<br>반중력 | **Customizations** → plugins and skills<br>**맞춤 설정** → 플러그인 및 기술 |

Then ask your agent something a skill owns, for example *"check whether this loop nest can be pipelined in HLS"*, or work through the [pre-flight checklist](preflight-checklist.md).
그런 다음 에이전트에게 기술이 소유한 항목을 물어보세요(예: *"이 루프 중첩이 HLS에서 파이프라인화될 수 있는지 확인"*)하거나 [실행 전 체크리스트](preflight-checklist.md)를 통해 작업하세요.

## Several clients on one machine
## 한 컴퓨터에 여러 클라이언트

Each client keeps its own plugin state, with one exception: **Cursor also loads Claude Code's installed plugins**.
각 클라이언트는 한 가지 예외를 제외하고 자체 플러그인 상태를 유지합니다. **커서는 Claude Code의 설치된 플러그인도 로드합니다**.
With both, install once, in Claude Code.
둘 다 Claude Code에서 한 번만 설치하면 됩니다.
A second copy under `~/.cursor/plugins/local/` causes [Every skill appears twice](#every-skill-appears-twice).
`~/.cursor/plugins/local/` 아래의 두 번째 복사본은 [모든 스킬이 두 번 나타남](#every-skill-appears-twice)을 발생시킵니다.
Cursor's **Plugins** panel then stays empty, but the skills still appear under **Skills**.
커서의 **플러그인** 패널은 비어 있지만 기술은 여전히 ​​**스킬** 아래에 표시됩니다.

Any other combination installs into each client separately, and one clone can serve several of them.
다른 조합은 각 클라이언트에 개별적으로 설치되며 하나의 복제본이 여러 클라이언트에 서비스를 제공할 수 있습니다.
Keep every client on the same release.
모든 클라이언트를 동일한 릴리스로 유지하십시오.

## Updating to a new release
## 새 릴리스로 업데이트 중

Pull the clone, or check out the new release tag:
클론을 가져오거나 새 릴리스 태그를 확인하세요.

```bash
git -C /path/to/ross-ai-assistant pull
```

Then, per client:
그런 다음 클라이언트별로 다음을 수행합니다.

- **Cursor**: pull the clone under `~/.cursor/plugins/local/`, then restart Cursor.
- **커서**: `~/.cursor/plugins/local/` 아래에 복제본을 가져온 다음 커서를 다시 시작하세요.
- **VS Code**: reads the clone in place; reload the window.
- **VS Code**: 복제본을 제자리에서 읽습니다. 창을 다시로드하십시오.
- **Claude Code** and **Codex**: both copied the plugin into their own cache, so pulling the clone alone changes nothing.
- **Claude Code** 및 **Codex**: 둘 다 플러그인을 자체 캐시에 복사했으므로 복제본을 가져오는 것만으로는 아무것도 변경되지 않습니다.
- Remove the plugin and install it again.
- 플러그인을 제거하고 다시 설치하십시오.
- **Kiro**: re-import the folder.
- **Kiro**: 폴더를 다시 가져옵니다.
- **Devin CLI**: start a new session.
- **Devin CLI**: 새 세션을 시작합니다.
- **Antigravity**: re-run the install prompt.
- **반중력**: 설치 프롬프트를 다시 실행하세요.

## Troubleshooting
## 문제 해결

### Cursor: the plugin does not appear
### 커서: 플러그인이 나타나지 않습니다

In order of likelihood:
가능성 순으로:

- **Wrong depth.** `plugin.json` must sit exactly one level below `local/`.
- **깊이가 잘못되었습니다.** `plugin.json`은 `local/`보다 정확히 한 수준 아래에 있어야 합니다.
- A clone into a folder you created first yields `local/ross-ai-assistant/ross-ai-assistant/`, which is not found.
- 처음 만든 폴더에 복제하면 `local/ross-ai-assistant/ross-ai-assistant/`가 생성되지만 찾을 수 없습니다.
- **Linked from outside.** A symlink under `local/` is accepted only when its target resolves inside `local/`.
- **외부에서 링크됨.** `local/` 아래의 심볼릭 링크는 대상이 `local/` 내부에서 확인되는 경우에만 허용됩니다.
- **Third-party plugins disabled.** See step 2 of [Cursor](#cursor).
- **타사 플러그인이 비활성화되었습니다.** [커서](#cursor)의 2단계를 참조하세요.
- **Blocked by policy.** On Cursor Teams and Enterprise, **Allow Local Plugin Imports** (Dashboard → Settings → Security & Identity → Marketplace and Plugins) is off by default.
- **정책에 의해 차단됨.** Cursor Teams 및 Enterprise에서는 **로컬 플러그인 가져오기 허용**(대시보드 → 설정 → 보안 및 ID → 마켓플레이스 및 플러그인)이 기본적으로 꺼져 있습니다.

### Every skill appears twice
### 모든 스킬이 두 번 나타납니다.

Two copies are loaded: the plugin and an `npx skills add` install, or a Cursor install alongside a Claude Code one, which Cursor also reads.
두 개의 복사본이 로드됩니다: 플러그인과 'npx 기술 추가' 설치 또는 Cursor가 읽는 Claude Code와 함께 설치되는 Cursor.
Remove one.
하나를 제거하십시오.

This matters beyond the clutter.
이것은 혼란을 넘어 중요합니다.
In Claude Code the copy that is not the plugin takes the bare command name, so `/hls-optimize` runs that copy, and the plugin's copy answers only to `/amd-ross-agentic-ai-assistant:hls-optimize`.
Claude Code에서 플러그인이 아닌 사본은 기본 명령 이름을 사용하므로 `/hls-optimize`는 해당 사본을 실행하고 플러그인 사본은 `/amd-ross-agentic-ai-assistant:hls-optimize`에만 응답합니다.
Because the skills call each other by bare name, the other copy keeps winning even after the plugin updates.
스킬들이 서로 이름만 부르기 때문에 플러그인 업데이트 후에도 다른 카피가 계속 승리합니다.

### Skills missing
### 기술 누락

A skill is skipped without warning if its directory name does not match the `name` in its `SKILL.md` frontmatter.
스킬의 디렉토리 이름이 'SKILL.md' 앞부분의 '이름'과 일치하지 않으면 경고 없이 스킬을 건너뜁니다.
Discovery is also non-recursive: only immediate children of `skills/` count.
발견은 또한 비재귀적입니다. `skills/`의 직계 하위 항목만 계산됩니다.
