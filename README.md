# pr-explainer

An agent skill (Claude Code, Codex, and other runtimes that read `SKILL.md`) that turns
"explain this PR" or "how does this flow work?" into **one self-contained HTML page**
you can read to understand the change.

* **3 tabs**: a very plain explanation, the data structures involved, and one concrete scenario in time order
* **Real code**: every step carries a verified `file:line` and a verbatim excerpt read from the source
* **Computed figures**: SVG drawn from computed coordinates (no mermaid), inlined so the page works offline
* **Editor bookmarks**: one per scenario step, matching the page's step numbers 1:1 (VS Code / Cursor Bookmarks extension)

The goal is **comprehension, not review**. The page explains what changed and why. It does not decide whether to merge.

## Install

Clone into your runtime's skills directory, or clone anywhere and symlink:

```bash
git clone <this repo> ~/src/pr-explainer
ln -s ~/src/pr-explainer ~/.claude/skills/pr-explainer   # Claude Code
ln -s ~/src/pr-explainer ~/.agents/skills/pr-explainer   # Codex and other runtimes
```

Requirements: `git`, `python3`, `rsvg-convert` (librsvg) for SVG → PNG. `gh` is optional, for PR mode.

## Use

* "PR #123 를 설명해" / "explain PR #123"
* "이 흐름 시각화해줘" / "walk me through what happens when an order is placed"

## Project terms (optional)

To make the page use your project's vocabulary, add a term table to `CLAUDE.md` or `AGENTS.md`:

```markdown
## pr-explainer

### 용어
| 쓰지 말 것 | 쓸 말 | 이유 |
|---|---|---|
| 쉬는 주문 | 걸려 있는 주문 | resting order 의 업계 표현 |
```

When you correct a term during a session, the skill proposes a new row for this table and adds it only after you agree.

## Unattended runs (CI)

When nobody can answer questions, the skill asks nothing. It builds all three tabs from the checked-out tree and writes to the path the caller gives.

## License

MIT
