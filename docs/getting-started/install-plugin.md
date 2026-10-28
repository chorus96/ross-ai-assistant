3. Verify

## 확인

| Client클라이언트       | Where the skills appear스킬이 나타나는 곳                                                                                                                             |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cursor커서             | **Customize → Skills**, under "Agent Decides"**사용자 지정 → 기술**, "에이전트 결정" 아래                                                               |
| VS CodeVS 코드         | **Chat: Configure Skills**, and the plugin under **Agent Plugins - Installed****채팅: 기술 구성** 및 **에이전트 플러그인 - 설치됨** 아래의 플러그인 |
| Claude Code클로드 코드 | `claude plugin list`, or `/pluginclaude 플러그인 목록` 또는 `/plugin`                                                                                           |
| Codex코덱스            | `/plugins/플러그인`                                                                                                                                                 |
| Kiro키로               | the **Powers** panel**Powers** 패널                                                                                                                       |
| Devin CLI데빈 CLI      | `devin plugins listdevin 플러그인 목록`                                                                                                                             |
| Antigravity반중력      | **Customizations** → plugins and skills**맞춤 설정** → 플러그인 및 기술                                                                                 |

Then ask your agent something a skill owns, for example *"check whether this loop nest can be pipelined in HLS"*, o

## Several clients on one machine

## 한 컴퓨터에 여러 클라이언트

Each client keeps its own plugin state, with one exception: **Cursor also loads Claude Code's installed plugins**.
각 클라이언트는 한 가지 예외를 제외하고 자체 플러그인 상태를 유지합니다. **커서는 Claude Code의 설치된 플러그인도 로드합니다**.
With both, install once, in Claude Code.
둘 다 Claude Code에서 한 번만 설치하면 됩니다.
A second copy under `~/.cursor/plugins/local/` causes [Every skill appears twice](#every-skill-appears-twice).
`~/.cursor/plugins/local/` 아래의 두 번째 복사본은 [모든 스킬이 두 번 나타남](#every-skill-appears-twice)을 발생시킵니다.
Cursor's **Plugins** panel then stays empty, but the skills still appear under **Skills**.
커서의 **플러그인** 패널은 비어 있지만 기술은 여전히 **스킬** 아래에 표시됩니다.

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
