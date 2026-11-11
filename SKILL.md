---
name: pr-explainer
description: >-
  Trace a code path end-to-end (e.g. API endpoint → … → DB/terminal) and produce
  two paired learning artifacts: a self-contained HTML flow diagram and a set of
  Cursor/VS Code bookmarks whose labels match the HTML steps 1:1. Use this
  whenever the user wants to *understand how something works through the codebase*
  — "trace this flow", "walk me through what happens when X", "이 흐름 시각화해줘",
  "코드 흐름 그려줘", "explain the request lifecycle and let me jump to each step",
  "make a diagram of how an order/request/job flows" — and also when they ask to
  *explain a PR* ("PR#123 를 설명해", "이 PR 이 뭘 바꾸는지 보여줘"), which runs the
  same pipeline in PR mode (before/after, per-commit, change badges) — for
  understanding the PR, not reviewing it. Trigger even if they only ask for
  "a diagram of the flow" or "help
  me follow this path", since the HTML+bookmark pair is what makes a flow
  actually stick. The page teaches shape before motion: when the flow touches a
  real data structure (tree, index, sort key, counter pair), it explains that
  structure with computed figures FIRST, then walks the timeline. Pairs with the
  `cursor-bookmark` skill for the bookmark-writing mechanics.
---

# PR Explainer

Turn "how does X work end-to-end?" (or "explain PR #N") into artifacts that
reinforce each other. **The goal is comprehension, not a review verdict** —
even in PR mode, the page explains what the PR does and why, not whether to
approve it. Flag a factual mismatch between a commit message and the code
(review callouts) because that is itself something the reader needs to
understand correctly; do not add an approval checklist or a "tests pass /
should this merge" table.

1. **A 3-tab HTML page** — 매우 쉬운 설명 / **자료구조** / 시나리오 시간순 (see
   "The three tabs"). Color-coded step cards carry `file:line` chips and verbatim
   code excerpts. The page **embeds the flow's bookmark JSON** behind a
   "북마크 JSON" toggle with one-click copy, so the HTML alone is the durable
   archive: open it later, paste the JSON into `.vscode/bookmarks.json`, and the
   jump points come back without re-tracing.
2. **Cursor bookmarks** — one per step of the 시나리오 tab, labeled
   `[<flow-tag> N] …`, so the tab and the editor line up 1:1.
3. **Figures** (`<flow-name>-*.svg` + `.png`) — the 자료구조 tab's structure
   figures and the 시나리오 tab's players map, all **computed SVG → PNG**
   (never mermaid), inlined into the HTML so it opens offline.

(A 4th "모듈별 설명" tab used to exist. The user deleted it: the scenario cards
already carry file:line and code, so regrouping them by module only repeated
them. Do not build it unless asked.)

**Model preference**: run on the most capable model available. Don't delegate the
tracing or the writing to smaller models. In an unattended run the caller picks
the model.

The payoff (the user said this directly): the picture explains the *shape*,
synchronized bookmarks let them walk the *real code* in order, and the embedded
JSON makes **one HTML per flow the archive** — `.vscode/bookmarks.json` only
ever holds the flow being studied right now.

**Build everything with a small generator in the scratchpad** (write it fresh
per flow; the structure is what carries over):
`steps.py` — the single source `(n, phase, actor, file, line, label)` →
`bm.py` — writes `.vscode/bookmarks.json` and prints each step's real line →
`figs.py` — an `Svg` class that computes coordinates + `save()` (SVG, then
`rsvg-convert -z 2` PNG) →
`plain.py` — the plain-words text of each scenario card, `n → (제목, [설명
줄], 예시)`, kept apart from `steps.py` so wording edits never touch lines →
`gen.py` — tab bodies, a `code(path, (start, end), "⋯ skipped", …)` excerpt
helper that reads the source, page assembly, and the checks (excerpt match,
excerpt boundary, forbidden terms, plain-words lint). Keep `style.css` /
`app.js` beside it.
A term fix or a moved line is then one edit + `bm.py && gen.py`, and the
bookmarks, HTML and figures change together.

**Visual format = the dark walkthrough style in `assets/style.css`.** Copy it as
the page's `<style>` and keep its classes: `.phase` (`.pnum`, `h2`, `.ptag`),
`.step.loop|seal|wire|persist|db|scan` (`.head`, `.n`, `.t`, `.loc`), `.actor`,
`.gloss`, `.invariant`, `.warn`, `.final`. Add classes; never redesign. If the
repo already has walkthrough pages, match their `<style>` instead, so pages look
alike. (A fresh light design shipped once and drew "포멧이 너무 다른데" from a
reader who expected the familiar format; it had to be rebuilt.)

---

## Step −1 — Fix the code snapshot (always; PR mode especially)

Every `file:line` is only as good as the code it was read from.

- **PR mode**: compare `gh pr view N --json headRefOid` with the local HEAD. If
  they differ (rebased, new commits — common), **ask** whether to reset the
  local branch or use a separate worktree; the user's default has been a
  worktree: `git worktree add ../<repo>-pr<N> origin/<head-branch>`. Trace,
  quote and bookmark from THAT tree. Bookmarks go into its `.vscode/` (create
  `settings.json` with `"bookmarks.saveBookmarksInProject": true` too).
- **Branch mode** ("지금 변경작업의 내역을 설명해", no PR yet): base = the
  branch point (`git merge-base`), head = current HEAD. It runs like PR mode
  (before/after, commit table, change badges); ask whether to explain the head
  before or after pending review fixes.
- Record the snapshot in the HTML header: head commit, base commit, and the
  directory the line numbers refer to.

**Ask the scope question too, in the same AskUserQuestion call**: a
multi-select of **자료구조** and **시나리오 시간순** — "이번엔 어떤 탭까지
만들까요?" — with both pre-marked as the recommended choice. 매우 쉬운 설명 is
never optional; it is always built. Skipping a tab changes downstream
behavior:
- Skip **시나리오** → no bookmarks at all (bookmarks are keyed to its step
  numbers) and no players map; say so in the answer, don't build an empty
  bookmark panel.
- Skip **자료구조** but keep 시나리오 → scenario cards that would have said
  "자료구조 3장 참조" instead explain the structure inline, briefly, right
  there (a reader has nowhere else to look).
- The tab bar only lists the tabs actually built; with one tab, skip the tab
  bar entirely.

Ask the file-location question (below) in the same AskUserQuestion call.

**Unattended runs (CI, no way to ask).** When nobody can answer — no question
tool, or the caller says the run is unattended — ask nothing: build all three
tabs, trace the tree that is already checked out, write where the caller says,
skip the checkpoint in "Generation order", and don't write
`.vscode/bookmarks.json` (the embedded bookmark JSON in the HTML stays).

## Step 0 — 자료구조부터 (모양 먼저, 움직임은 나중)

Before tracing, ask: do the flow's load-bearing nouns have **structure** — a
tree, index, sort key, queue, counter pair, bit layout, ring buffer? If any do,
**build the 자료구조 tab first.** A reader cannot follow motion over a shape
they don't hold. A 17-step timeline is hard not because it is long but because
step 1 already assumes `slot` and "node sums". A glossary line does not install
a data structure — a picture does.

If there is no structure (a plain call chain, a config value propagating), skip
the tab and say why in one line.

## Step 1 — Trace against real code (never from memory)

Walk the path with `grep`/`Read`. For each step capture the call site,
`file:line` (1-based), what happens, and why. Group steps into **phases = time
segments** of one request (`T0 접수`, `T1 저장`, …), not architecture layers.
Tag each step with an actor and note **whether that actor is a real
thread/task or a struct another thread drives**. Note invariants (ordering,
atomicity, "X durable before Y"). Nested sub-flows become `5a`, `5b`.

### 트레이싱 개선 (속도)

트레이싱이 느린 건 "다음에 뭘 읽어야 할지 하나씩 확인하며 가야 한다"는 것 자체
때문이 아니다. 대부분은 **안 읽어도 됐을 걸 읽거나, 같이 보냈어도 됐을 조회를
따로따로 보내거나, 틀린 스냅샷을 읽어서 나중에 다시 읽는** 데서 생긴다. 셋 다
"코드를 덜 이해하자"가 아니라 "같은 이해에 드는 도구 호출을 줄이자"다.

1. **스냅샷부터 고정하고 트레이싱을 시작한다.** Step −1 이 이미 시키는 일이지만
   여기서도 다시 건다 — 트레이싱 전에 건너뛰면 트레이싱 자체가 헛수고가 될 수
   있어서다. PR head 와 로컬 워크트리의 커밋이 같은지 먼저 확인하고, 다르면
   워크트리를 그 자리에서 새로 만들거나 갱신한 **다음에** 코드를 읽기 시작한다.
   (실제로 겪음: 워크트리가 리베이스 전 커밋에 머물러 있는 걸 트레이싱 중간에
   발견해서, 지우고 새로 받은 뒤 다시 읽어야 했다.)
2. **파일 전체를 Read 하지 않는다 — grep 으로 줄 번호부터 찾고, 그 근처만 읽는다.**
   "이 함수가 어디 있지?"는 `grep -n "fn 함수이름"`으로 먼저 답한 뒤, 그 줄 번호를
   `offset`/`limit` 으로 좁혀 읽는다. 파일 하나를 통째로 읽으면 대부분은 지금
   트레이싱과 무관한 내용이고, 그 파일을 또 봐야 할 때마다 매번 전체를 다시 훑게
   된다.
3. **서로 의존하지 않는 다음 조회는 한 메시지에 같이 보낸다.** "A 를 봐야 B 의 어디를
   볼지 안다"면 순서대로 할 수밖에 없지만, 이번 단계에서 어차피 둘 다 봐야 하고
   서로 내용이 영향을 안 준다면(두 구조체의 필드 정의, 서로 다른 두 테스트 함수 등)
   같은 메시지에 여러 Read/Grep 호출을 같이 보낸다. 순서대로 하나씩 보내는 것보다
   왕복이 줄어든다.

Things to establish while tracing — each one was missed once and cost a
correction round:

- **Which process holds what.** For every load-bearing type, grep which
  binaries/crates/packages construct or use it (API server, background worker,
  a separate verifier process, …). Write the "concept × process" table from that
  grep (자료구조 탭 0장). If the same code runs in two processes (e.g. a worker
  that replays the server's core logic), that fact goes first.
- **Ownership.** Before drawing the players map, read the struct fields: if A
  holds B as a field, B is drawn **inside** A. ("compute goes to A, apply goes
  to B" was once drawn as two siblings — but B was a field of A.)
- **Free functions vs methods.** A step that calls a free function (e.g.
  `orders::validate`) is not "a call into struct X" — keep it separate in edge
  labels.
- **PR mode — check commit-message claims against the code.** When a message
  says "without this, X would break because Y", trace Y in the current base.
  (Once: "the journal would ride along with the next command's rows" was false
  because a base-level leftover sweep already existed; the real fix was
  atomicity. The code was right; the message was not. That is a review point.)
- **What a check compares against.** For every "검증/대조" step, record the
  exact source read — table, row, field (`blocks` 400행 `state_root`) — and
  where that value came from. Never name a different copy of the same value
  (DB row vs on-chain vs memory). ("체인에 올린 값과 비교" was written once; the
  code read the DB row and never queried the chain. A reader took the page at
  its word, and it had to be corrected.)
- **Observable output.** Collect the exact `println!`/`log` strings each step
  emits and the DB statements it runs, in **code order** (a snapshot-log line
  was first written after the block line it actually precedes). They feed the
  "먼저 결과부터" section.

Accuracy is the whole point — a wrong `file:line` sends a bookmark to the wrong
code and erodes trust. Verify every line.

**Write bookmarks as soon as the trace is verified** (don't wait for prose):
fill `steps.py`, run `bm.py`, which writes the file and prints each step's real
source line for a quick eyeball.

**Say what you're doing.** Tracing is long and silent. Post a one-line status
when tracing starts and whenever a deliverable lands (bookmarks, figures, each
tab).

## Step 2 — Build the HTML

### The three tabs

A tab bar at the top; each tab is a `<section class="tab-section">` toggled by a
few lines of JS (remember the last tab in localStorage, wrapped in try/catch).
The bookmark panel and "코드 모두 열기/닫기" sit OUTSIDE the tabs. 매우 쉬운
설명 always exists; 자료구조 and 시나리오 시간순 exist only if the Step −1
scope question selected them (both, by default) — see the skip rules there.
A build with only one tab omits the tab bar.

1. **매우 쉬운 설명** (default) — analogy-driven, 1–2 minutes, almost no
   identifiers, no `file:line` chips. Metaphor lives HERE only.
   - **Box headings state the claim, not the jargon.** A heading built by
     quoting two terms against each other ("스냅샷"이 아니라 "현재 상태 표"인
     이유) reads as awkward translation. Say what it IS in one plain clause
     ("매 블록 바로 반영되는 최신 상태를 쓰는 이유") and let the bullets do
     the comparing.
   - PR mode adds: a **before/after box pair** ("이 PR 전 / 이 PR 후"), a
     "바뀜 / 그대로 / 거부" table, and a **커밋 표** titled
     "주요 커밋 (이 순서대로 읽는 걸 추천)" — the order is a reading order,
     not just a list. Each row's description is a **one-line Korean
     summary** written for the reader, never the commit's own (often
     English) headline pasted verbatim.
2. **자료구조** — how the data is shaped and why. No numbering (numbers belong
   to the scenario tab). Sections, in order:
   0. **어느 프로세스에 있나** — the concept × process table from Step 1, ✅/✗
      per cell with the code that proves it (`Worker.cache: Index`,
      `Index::rebuild_from`, "server/ 에 사용처 없음"). Mandatory
      whenever more than one process is involved.
   1. **층 구조** — what sits inside what (root → subtree → leaf), one figure
      plus a "층 / key / 리프 내용 / 이번 연산 때" table.
   2. **키(주소) 레이아웃** — bit bar figure plus a worked value.
      - If **two structures of the same kind** exist (two trees, two leaf
        kinds), open with a **comparison table** (경로 / 개수 / key / 내용 /
        하는 일 / 이번 연산에서 바뀌는 것) and quote the code that joins them.
   3. **카운터·시퀀스마다 독립된 절** — see "Counters get their own section".
   4. **장난감 인스턴스** — 8–16 cells with real numbers. Mention real scale in
      a number only ("실제 2⁴⁸").
   5. **연산 하나의 before / after** — the same figure twice, change by color.
   6. **왜 이 모양인가 + 반례** — see "유도와 반례".
   7. **조정값마다 "왜 이 값인가"** — every tunable constant the flow depends on
      (`KEEP = 3`, `every = 100`, chunk size): where it is used, a "작으면 /
      크면" table of concrete situations (1장만 둘 때 vs 3장 둘 때), the
      `file:line`, and whether a CLI flag can change it. ("왜 최근 3개만
      남기는거야?" was asked.)
   - **Every section title carries a location badge**: `API 서버` / `워커` /
     `양쪽` (whatever the processes are called here).
   - Each section **opens with the question it answers**, never a type name.
3. **시나리오 시간순** — ONE concrete request chronologically. **Owns the step
   numbering; bookmarks match it 1:1.** It does not re-explain structure — it
   points ("자료구조 3장 참조").
   - Use **the same toy scene** as the 자료구조 tab, so cards carry the same
     values (`seq 4`, `(price 510, slot 4)`). Seed the scene from a real
     test and name the test — the reader can check it.
   - **Open with "먼저 결과부터: 실제로 무슨 일이 일어나나"** — before the
     players map. Structure and `file:line` alone left the user at "대강만
     알것같지.. 구체적으로 뭐하는지는 모르겠네". The section shows one run as
     an operator would see it:
     ① the log lines, verbatim from the code's print/log strings (hashes as
       short examples `0x3f07…`);
     ② **before / after** of the same run (`.pairgrid`: 이 작업 전 / 후) with
       counts and time ("436줄 → 36줄");
     ③ the DB reads/writes in order — 묻는 것 / 돌아오는 것 / 판단;
     ④ memory state per moment — 시점 / field / value;
     ⑤ the log of each failure path (rejected, fell back).
     Then a heading "흐름: 누가 무엇을 하나" and the rest below.
   - Then the **players map** (below), then a small glossary of the 3–5
     load-bearing nouns, each with a "정의 코드" `<details>`.
   - **Every phase title carries the location badge** too.
   - PR mode: steps that changed get a `변경` / `변경 · 커밋N` badge; old code
     the flow passes through is labeled "(기존)", so new and base code are
     never confused.
   - A branching decision function (e.g. `should_retry`) gets an **input
     combination × verdict × when it happens** table.
   - A flow that **validates** something (restore, verify, admit) gets a
     **검사 순서** table: 단계 / # / 확인 / 무엇과 대조 / 걸리는 경우, each row
     with `file:line`; then one paragraph on which check carries the weight,
     why it suffices, and what it cannot catch plus what catches that later.
     ("400이 맞는지는 어떻게 검사해?" was asked.)
   - Branch/PR mode: steps a review round changed get a `최종 리뷰 I1`-style
     badge, next to the `변경` badges.
   - End with a final-state table (what exists where), then a **추천 읽기
     순서** table — 순서 / 파일 / 볼 것 / 북마크 — bottom-up: pure functions →
     storage → orchestrator → CLI → the one test that holds the whole flow.
     ("코드만 이해하면되겠다" was the user's next step every time.)

### Players map (scenario tab)

Not a sequence diagram — a swimlane of one request was judged "too much". The
picture shows WHO the players are and what passes between them; the cards carry
the time order. Computed, like every figure.

- **One box per player**, grouped by process (dashed group frames: server ·
  DB · worker · external service …). Label = name + `⚙` (real thread/task) or `[구조체]`
  (struct driven by another thread) + one line of role. 5–15 boxes.
- **Ownership = containment**: a struct held as a field is a box *inside* its
  owner's box (container with its own title). Never siblings.
- **Edges = handoffs**: solid = call/write, dashed = read/poll/reply, thick =
  the writes that matter and the one edge that closes the loop. Each label
  starts with the step numbers `[5–7]` (same as cards and `[<tag> N]`). Group
  only steps that reach the **same target**; a free function gets its own box
  (dashed border) and its own `[N]`. No ①②③ anywhere.
- Channels/queues are boxes too; periodic actors carry their interval
  ("poll_interval_ms 마다"). Start at the true origin.
- **Legend = a bordered box**, one item per line, vertical dividers between
  groups (box notation | line style | line color), and **real line samples**
  (color, width, dash, arrowhead). A one-line run-on legend was unreadable.
- **Multiple phases → selectable phase filter.** When the scenario has 2+
  phases, tag each phase-specific edge/label and each phase-specific line of a
  surrounding note box with `data-phase="N"` in the SVG. Add a row of phase
  pills above the figure (plain JS, no library): clicking one dims
  (`opacity:.15`) every `[data-phase]` element whose phase doesn't match and
  restores the rest; a reset/"전체" pill (default) shows everything at full
  opacity. Structural player boxes stay at full opacity always — only the
  phase-specific edges and note lines dim. Without this a reader cannot tell
  which part of one static picture belongs to which phase of the cards below
  it ("플레이어맵이랑 phase 설명이 대응이 잘 안 되는 것 같아").
- **Check the numbering before shipping.** Walk every edge label and note-box
  line in the map and confirm its step range matches the actual cards (same
  numbers, same phase) — a label drafted early and never rechecked after the
  cards were finalized is a common source of drift.

### 그림은 계산해서 그린다 (mermaid 금지, 오프라인 자립)

Every figure comes from a Python generator that computes coordinates, writes
SVG, and converts with `rsvg-convert -z 2` to PNG. Mermaid's auto-layout is
uncontrollable (2400×272 strips, edges across the page) and cannot express
figures where *position is meaning* (trees, bit layouts, before/after).

- One function → many figures: `grid(orders, marks)` drew the toy, before,
  after and the counterexample. Before/after differ only by color.
- Output `<flow-name>-<figure>.svg` + `.png`; inline the SVG in
  `<div class="fig">`; link both under it. No CDN scripts.
- **Read the PNG yourself** before moving on. Fix overflow/overlap in the
  generator, re-render.
  - Give **every edge label explicit coordinates**; midpoint auto-placement
    overlapped other edges every time.
  - Route around boxes; a straight line through a box reads as a connection.
  - **No color emoji in figures** — rsvg-convert renders 📦 as a blank blob.
    `⚙` works. Use text tags like `[구조체]` instead.
- Dark page → dark figures (background, lines, text matching the page).
- **Click to enlarge** (user request): `.fig` is `cursor:zoom-in`; a
  library-free lightbox clones the SVG at its viewBox width with −/＋/100%/fit/
  PNG/close, Esc/±, click-outside closes.
- **Re-render on request** by editing the generator, never the SVG/PNG.

### 모든 탭에 공통

**Long prose is bullets, not paragraphs — in all three tabs.** Any
explanation over ~2 sentences ships as a short bullet list (2–4 items, one
idea per item): 쉬운설명 `.easy` box bodies, 자료구조 절 아래의 분석 문단,
시나리오 카드 `.d` 본문. "Scenario cards are plain words" below is this same
rule's fullest instance (title/who-line/bullets/예:); apply the same
skimmability to the other two tabs even without that full card shape.

**Check as you write, not after the page is done.** The figure-overlap check
below already works this way — `save()` flags a figure's own overlaps the
moment that one figure is rendered, before the next figure is even started.
Give the other checks the same shape: verify each excerpt, each `file:line`
reference, each card's wording **the moment it is written**, not in one scan
after the whole tab (or the whole page) is finished. A mismatch caught one
card late costs nothing — fix it and move on. Caught at the end of the tab, it
costs re-reading that card to remember what it was supposed to say, across
however many other mismatches turned up in the same sweep (one timing run:
76s and 17 fixes, all found in one end-of-tab pass). Same total amount of
checking either way — the cost is in how far the fix is from the write.

**Code excerpts.** `<details class="code"><summary>코드 — path:START–END</summary><pre>…</pre></details>`.
The goal is to understand the code **without opening the editor**.
- Verbatim, read by the script from the source tree of Step −1 (never typed).
  Prefix each line with its number (`1386│ `). Trim with `⋯ (what was skipped)`
  lines; never rename. 5–15 lines; one step may show two excerpts.
- **Mechanical check right after writing each excerpt** (not after the tab is
  finished): parse it and compare each `NNN│` line to the real line. Ship only
  at zero mismatches.
- **Boundary check**, same moment (text-equal is not enough): warn when an
  excerpt's first line starts with `)` `}` `);` or its last line ends with
  `(` — it was cut mid-statement. (`(624, 625)` started at `);` and passed the
  text check.)
- **`file:line` in prose, card chips and figure text** go through one helper
  **the moment each reference is written**: it checks the line exists and
  prints the **shortest path suffix that is unique in the repo**
  (`git ls-files`): `api/handlers.rs`, `core/src/lib.rs` — never a bare
  `lib.rs` / `mod.rs` / `handlers.rs`. (Basename-only card chips once produced 12
  ambiguous references, all found in one end-of-page sweep.) Still re-run it
  as one final scan of the whole page before shipping — a safety net for
  anything added out of order — but it should find 0 by then, not discover
  the 12.
- **Figure label overlap**: the `Svg` class records each text's approximate box
  (Hangul ≈ 1.0 em, other ≈ 0.6 em wide) and `save()` prints overlapping pairs.
  Fix until none remain, then still look at the PNG. (This is the pattern the
  other checks above now follow too.)
- **Template leftovers**: `{{` / `}}` — doubled braces written in a string that
  is not an f-string print literally (`Order{{id 5…}}` shipped once).
  Rare enough to check once, as a final scan of the page (minus
  `<style>`/`<script>`/excerpts) before shipping; must be 0.
- Chrome (self-contained JS): 코드 모두 열기/닫기, 경로복사 per excerpt, click a
  card's `file:line` chip to copy. **No highlighter / search box** — it was
  tried and removed: it was unclear what it was for and added nothing. Syntax colouring inside excerpts (keyword, type, string,
  comment, number; a small tokenizer in `code()`) is a different thing and is
  welcome.

**Scenario cards are plain words.** The first card version (function name +
a dense one-paragraph summary) drew "코드별 설명이 눈에 잘 들어오지 않아 …
말을 쉽게하고, lint도 적용해야해". Card shape, from `plain.py`:
- **title** = what it does, in plain Korean ("블록 번호가 100 의 배수인가?"),
  not the function name;
- a **who line** under it: actor badge + `function`;
- **1–3 bullets**, one short sentence each;
- an **예: line** with the toy value ("400 → 저장, 401 → 건너뜀");
- the code excerpts, unchanged.
The generator **lints each card in `plain.py` right after writing it** — not
after the whole file is drafted: a line over 70 characters, more than 3
bullets, a jargon word without an explanation in parentheses (SMT, BTreeMap,
멱등 키, outbox, …), and the forbidden terms. Ship at 0; a final lint
over the whole file before shipping is the safety net, not the first catch.

**Counters get their own section.** Any counter/sequence that appears in a key,
a sort order or a commitment (`seq`, `nonce`, `next_*`) gets a dedicated
자료구조 section — a glossary line was not enough ("이 seq 는 뭘
말하는거야?"):
- what it is in one sentence;
- a properties table — who assigns it / when (the one place it is bumped) /
  range / shared or per-side / what 0 means / where it is stored (memory, DB
  column, chain leaf) — each row with `file:line`;
- a **"왜 다른 값으로는 안 되나"** table: what was used before (`created_at`,
  `order_id`, …) and the case where it breaks. The code's doc comments usually
  record this history — quote them;
- a numeric walk: arrival → number received → counter after.

**Worked examples with numbers.** Evolving state (block numbers, queues,
deadlines, nonces) needs numbers, not more prose: `1000 ─슬롯1─ 1500`. Pick
small readable values (price 500·510, size 2·3·4, 8 cells); real scale only as
a mention. Values must agree everywhere; when there are more than ~5 or there
is sorting/aggregation, let the script compute them. One event changing several
values → a table.

**유도와 반례 — for every load-bearing choice.** (1) a derivation paragraph:
which requirement forces the choice; (2) a **counterexample figure on the same
toy**, with a red ✗ where it breaks ("나중에 온 주문이 먼저 체결된다"). If you
cannot draw the counterexample, the choice isn't load-bearing — skip it.

**Terminology — run this before writing any prose.**
1. **Read the project's term table** and build a **forbidden → preferred**
   list from it. The table lives in the project's agent instructions:
   `CLAUDE.md` and/or `AGENTS.md` (read both if both exist), section
   `## pr-explainer` → `### 용어`:
   ```markdown
   ## pr-explainer

   ### 용어
   | 쓰지 말 것 | 쓸 말 | 이유 |
   |---|---|---|
   | 쉬는 주문 | 걸려 있는 주문 | resting order 의 업계 표현 |
   ```
   No section → no project terms; the rules below still apply. (Once, a
   forbidden word was used 15 times despite a recorded rule — read the table
   before writing, not after.)
   **When the user corrects a term** ("그 말 이상해", "X 말고 Y"), fix the page
   and then propose the row for that table: show the exact line and the file,
   and edit `CLAUDE.md` / `AGENTS.md` only after they agree (create the section
   if it is missing). Recording it in the project — not in private memory — is
   what lets teammates and CI runs use the same words. In an unattended run,
   only read the table.
2. **No literal translation of domain jargon.** Use the Korean the trade uses;
   if none fits, keep the English in parentheses. (resting order ≠ "쉬는
   주문" → "걸려 있는 주문", rests again → "다시 걸린다".)
3. **No coined nouns.** A term the walkthrough invents ("칸", "봉투", "자리
   예약") cannot be grepped and collides with real structures ("칸" was read as
   the order-tree leaf). Use the code's own name with an owner prefix
   ("오더북 리프", "주문 리프"). If a shorthand truly earns its place, flag it
   "코드에 없는 표현" and point at the exact code.
4. **Introduce every internal noun before use**: what it is, **where it lives**
   (`Struct.field`, table, memory vs DB, which process), who fills/consumes it.
   Phase titles pair plain words with the term.
5. Metaphor only in the 쉬운설명 tab; elsewhere state operations literally.
6. The generator **scans each section for forbidden terms the moment it is
   written** (excluding code excerpts, which are verbatim) — bookmark labels
   and figure text included, not just prose. A final scan of the whole output
   before shipping must report 0, but that is the safety net; catching a term
   mid-write is what actually keeps the count near 0 by the time that scan
   runs.

### Generation order + checkpoint

1. Shell (bookmark panel; tab bar only if 2+ tabs are in scope) with the
   쉬운설명 tab → save → `open`.
2. If 자료구조 is in scope: build it → save.
3. **Stop and ask, only if 시나리오 is also in scope and another tab was just
   built** — "여기까지 이해되시면 시간순으로 넘어갑니다. 더 풀어야 할 곳
   있나요?" Keep it short, and include a preview of what the next tab will
   flag (e.g. the review point found), so the user can just say "계속". Skip
   this ask when 시나리오 is the only tab left to build after 쉬운설명 alone,
   or when it is the only tab in scope at all (nothing to check against yet).
4. Then the 시나리오 tab, if in scope. The user reloads to see it.

**When the user says it's hard, go down, not on**: back to the 자료구조 tab and
a new figure with smaller values and fewer cells — not another paragraph.

**Questions fold into the page by default.** When the user asks "X 가 뭐야?" /
"이건 어디 이야기야?", answer in chat with code evidence **and update the HTML
section it belongs to** in the same turn, saying where it went. (Every "HTML 에도
넣을까요?" was answered "응".) Likewise a mistake the question exposes (wrong
ownership in a figure, wrong term) is fixed in the generator and re-rendered.

**Where to keep the file**: ask (AskUserQuestion, together with the snapshot
question; in an unattended run, use the path the caller gave). Recommend `docs/flows/<flow-name>.html` in-repo; offer the session
scratchpad as volatile. Then `open` it. If it went to `/tmp` or the scratchpad,
offer once at the end to move the HTML + figures into `docs/flows/`.

## Step 3 — Matching bookmarks (two copies)

**HTML step `N` ⇔ bookmark `[<flow-tag> N] …`.** Use a per-flow tag (`[login N]`,
`[retry N]`). Build from `steps.py`, write to:

1. **`.vscode/bookmarks.json`** of the code tree from Step −1 (the worktree in PR
   mode) — mechanics per the **`cursor-bookmark`** skill (0-based lines,
   merge-vs-clobber, `saveBookmarksInProject`, Reload Window).
2. **Embedded in the HTML panel** — a complete standalone bookmarks.json holding
   ONLY this flow (replacement semantics: paste over, one flow active).

When a label changes (term fixes), re-run `bm.py` + `gen.py` so both copies
change together. Verify each bookmark against its source line and tell the user
to Reload Window (in the folder the bookmarks were written to).

Label shape:
```
[login 1] router.post — 로그인 요청을 받는다
[retry 3] should_retry — [변경·커밋3] 다시 시도할 오류인지 판단
```

## Conventions

- **모양 먼저, 움직임 나중.** Structure tab before timeline tab, built first.
- **어디 이야기인지 먼저.** Process table at the top of 자료구조, location badges
  on every section and phase.
- **제목은 질문으로.** "같은 가격이면 누가 먼저 체결되나?" ✓ / "`BookHeader`
  구조체" ✗.
- **Identical numbering** between scenario tab, players map and bookmarks; no
  other tab or figure invents numbers.
- **Real `file:line` everywhere**, from the snapshot of Step −1.
- **Phase = mental chunk**: 3–5 phases, not 19 flat steps.
- **결과 먼저, 구조와 흐름은 그다음.** The scenario tab opens with what an
  operator sees (logs, DB rows, memory); cards explain how it got there.
- **Call out the non-obvious** with invariant (green), warning (red) and, in PR
  mode, review (orange) callouts.
- **One artifact without the other is half the value.** Bookmarks ride along
  with 시나리오 automatically (see Step −1's scope question) — never build one
  without the other.
- **The HTML is the archive; bookmarks.json is the working copy.**

## Notes

- Composes with `cursor-bookmark` for the bookmark file; if unavailable, write
  `.vscode/bookmarks.json` directly (0-based lines, merge, reload).
- Very large flows: anchor links per phase, bookmark only load-bearing steps.
