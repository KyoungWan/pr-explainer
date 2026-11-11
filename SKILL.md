---
name: pr-explainer
description: >-
  Trace a code path end-to-end (e.g. API endpoint → … → DB/terminal) and produce
  two paired learning artifacts: a self-contained HTML flow page and a set of
  Cursor/VS Code bookmarks whose labels match the page's steps 1:1. Use this
  whenever the user wants to *understand how something works through the codebase*
  — "trace this flow", "walk me through what happens when X", "explain the
  request lifecycle and let me jump to each step", "make a diagram of how an
  order/request/job flows", "이 흐름 시각화해줘", "코드 흐름 그려줘" — and also when
  they ask to *explain a PR* ("explain PR #123", "PR#123 를 설명해", "what does
  this PR change?"), which runs the same pipeline in PR mode (before/after,
  per-commit, change badges) — for understanding the PR, not reviewing it.
  Trigger even if they only ask for "a diagram of the flow" or "help me follow
  this path", since the HTML+bookmark pair is what makes a flow actually stick.
  The page teaches shape before motion: when the flow touches a real data
  structure (tree, index, sort key, counter pair), it explains that structure
  with computed figures FIRST, then walks the timeline.
---

# PR Explainer

Turn "how does X work end-to-end?" (or "explain PR #N") into artifacts that
reinforce each other. **The goal is comprehension, not a review verdict.** Even
in PR mode the page explains what the PR does and why, not whether to approve
it.
- Flag a factual mismatch between a commit message and the code (a review
  callout): the reader needs to understand that correctly too.
- Do not add an approval checklist or a "tests pass / should this merge" table.

What gets built:

1. **A 3-tab HTML page.** The tabs are **Plain explanation** (`tab.easy`) /
   **Data structures** (`tab.structure`) / **Scenario, step by step**
   (`tab.scenario`); see "The three tabs".
   - Color-coded step cards carry `file:line` chips and verbatim code excerpts.
   - The page **embeds the flow's bookmark JSON** behind a toggle
     (`ui.bookmarks_json`) with one-click copy (`ui.copy`). So the HTML alone is the
     durable archive: open it later, paste the JSON into
     `.vscode/bookmarks.json`, and the jump points come back without
     re-tracing.
2. **Editor bookmarks**: one per step of the scenario tab, labeled
   `[<flow-tag> N] …`, so the tab and the editor line up 1:1.
3. **Figures** (`<flow-name>-*.svg` + `.png`): the structure figures and the
   scenario tab's players map, all **computed SVG → PNG** (never mermaid),
   inlined into the HTML so it opens offline.

Do not build a 4th "by module" tab unless asked. It existed once and was
deleted: the scenario cards already carry `file:line` and code, so regrouping
them by module only repeated them.

**Model preference**: run on the most capable model available. Don't delegate
the tracing or the writing to smaller models. In an unattended run the caller
picks the model.

The payoff: the picture explains the *shape*, synchronized bookmarks let the
reader walk the *real code* in order, and the embedded JSON makes **one HTML per
flow the archive**. `.vscode/bookmarks.json` only ever holds the flow being
studied right now.

**Build everything with a small generator in a scratch directory.** Write it
fresh per flow; the structure is what carries over:
- `steps.py`: the single source `(n, phase, actor, file, line, label)`
- `bm.py`: writes `.vscode/bookmarks.json` and prints each step's real line
- `figs.py`: an `Svg` class that computes coordinates + `save()` (SVG, then
  `rsvg-convert -z 2` PNG)
- `plain.py`: the plain-words text of each scenario card,
  `n → (title, [lines], example)`, kept apart from `steps.py` so wording edits
  never touch line numbers
- `gen.py`: tab bodies, a `code(path, (start, end), "⋯ skipped", …)` excerpt
  helper that reads the source, page assembly, and the checks (excerpt match,
  excerpt boundary, forbidden terms, plain-words lint)
- `style.css` / `app.js` beside it

A term fix or a moved line is then one edit + `bm.py && gen.py`, and the
bookmarks, HTML and figures change together.

**Visual format = the dark walkthrough style in `assets/style.css`.**
- Copy it as the page's `<style>` and keep its classes: `.phase` (`.pnum`,
  `h2`, `.ptag`), `.step.loop|seal|wire|persist|db|scan` (`.head`, `.n`, `.t`,
  `.loc`), `.actor`, `.gloss`, `.invariant`, `.warn`, `.final`.
- Add classes; never redesign.
- If the repo already has walkthrough pages, or its agent instructions name a
  style source, match that `<style>` instead, so pages look alike.
- (A fresh light design shipped once. The reader, who expected the familiar
  format, said it looked nothing like it, and it had to be rebuilt.)

---

## Output language

- **Write the page in the language of the user's request, in the same register.**
  A request in Korean gets a Korean page; a request in English gets an English
  page. Chat replies about the page follow the same language.
- **Fixed strings come from `assets/strings.md`.** That covers tab names, section
  titles, table headers, badges, buttons and questions. Use the column for the
  output language. For a language without a column, translate the `en` column
  once and reuse that translation everywhere.
- **Never translate code.** Identifiers, code excerpts, log strings, file paths
  and commit hashes stay verbatim.
- **Unattended runs**: the caller names the language. If it doesn't, use the
  language of the project's agent instructions (`CLAUDE.md` / `AGENTS.md`), and
  English if those don't settle it.

---

## Step −1 — Fix the code snapshot (always; PR mode especially)

Every `file:line` is only as good as the code it was read from.

- **PR mode**: compare `gh pr view N --json headRefOid` with the local HEAD.
  - If they differ (rebased, new commits — common), **ask** whether to reset
    the local branch or use a separate worktree. A worktree is the usual
    choice: `git worktree add ../<repo>-pr<N> origin/<head-branch>`.
  - Trace, quote and bookmark from THAT tree.
  - Bookmarks go into its `.vscode/` (create `settings.json` with
    `"bookmarks.saveBookmarksInProject": true` too).
- **Branch mode** ("explain what I've changed so far", no PR yet): base = the
  branch point (`git merge-base`), head = current HEAD. It runs like PR mode
  (before/after, commit table, change badges). Ask whether to explain the head
  before or after pending review fixes.
- Record the snapshot in the HTML header (`ui.snapshot`): head commit, base
  commit, and the directory the line numbers refer to.

**Ask the scope question in the same AskUserQuestion call** (`ask.scope`): a
multi-select of the structure tab and the scenario tab, with both pre-marked as
the recommended choice. The plain-explanation tab is never optional; it is
always built. Skipping a tab changes downstream behavior:
- Skip **scenario** → no bookmarks at all (they are keyed to its step numbers)
  and no players map. Say so in the answer; don't build an empty bookmark panel.
- Skip **structure** but keep scenario → cards that would have pointed to it
  (`st.see_section`) explain the structure inline, briefly, right there. The
  reader has nowhere else to look.
- The tab bar lists only the tabs actually built. With one tab, skip the tab bar
  entirely.

Ask the file-location question (below) in the same AskUserQuestion call.

**Unattended runs (CI, no way to ask).** When nobody can answer (no question
tool, or the caller says the run is unattended), ask nothing:
- build all three tabs;
- trace the tree that is already checked out;
- write where the caller says, in the language the caller names;
- skip the checkpoint in "Generation order";
- don't write `.vscode/bookmarks.json` (the embedded bookmark JSON in the HTML
  stays).

## Step 0 — Structure first (shape before motion)

Before tracing, ask: do the flow's load-bearing nouns have **structure** — a
tree, index, sort key, queue, counter pair, bit layout, ring buffer? If any do,
**build the structure tab first.**
- A reader cannot follow motion over a shape they don't hold.
- A 17-step timeline is hard not because it is long but because step 1 already
  assumes `slot` and "node sums".
- A glossary line does not install a data structure — a picture does.

If there is no structure (a plain call chain, a config value propagating), skip
the tab and say why in one line.

## Step 1 — Trace against real code (never from memory)

Walk the path with `grep`/`Read`. For each step capture the call site,
`file:line` (1-based), what happens, and why.
- Group steps into **phases = time segments** of one request (`T0 received`,
  `T1 stored`, …), not architecture layers.
- Tag each step with an actor and note **whether that actor is a real
  thread/task or a struct another thread drives**.
- Note invariants (ordering, atomicity, "X durable before Y").
- Nested sub-flows become `5a`, `5b`.

### Faster tracing

Tracing is rarely slow because you must find out what to read next, one look at
a time. It is slow because of **reading what didn't need reading, sending
lookups one by one that could have gone together, and reading the wrong
snapshot and then reading it all again**. All three rules below aim at fewer
tool calls for the same understanding, not at understanding less.

1. **Fix the snapshot before tracing starts.** Step −1 already asks for this;
   it is repeated here because skipping it can waste the whole trace.
   - Check that the PR head and the local worktree are on the same commit.
   - If not, create or update the worktree **first**, then start reading.
   - (This happened: a worktree was still on the pre-rebase commit, found
     mid-trace, and everything had to be read again.)
2. **Don't read whole files: grep for the line, then read around it.**
   - Answer "where is this function?" with `grep -n` first.
   - Then read with `offset`/`limit` around that line.
   - A whole file is mostly unrelated to the trace, and every later visit
     re-scans all of it.
3. **Send independent lookups together, in one message.**
   - If you need A to know where to look in B, go in order.
   - If both are needed now and neither changes the other (two structs' field
     lists, two different test functions), send the Read/Grep calls together.
     Fewer round trips.

Things to establish while tracing — each one was missed once and cost a
correction round:

- **Which process holds what.** For every load-bearing type, grep which
  binaries/crates/packages construct or use it (API server, background worker,
  a separate verifier process, …).
  - Write the "concept × process" table from that grep (structure tab, §0).
  - If the same code runs in two processes (e.g. a worker that replays the
    server's core logic), that fact goes first.
- **Ownership.** Before drawing the players map, read the struct fields. If A
  holds B as a field, B is drawn **inside** A. ("Compute goes to A, apply goes
  to B" was once drawn as two siblings, but B was a field of A.)
- **Free functions vs methods.** A step that calls a free function (e.g.
  `orders::validate`) is not "a call into struct X". Keep it separate in edge
  labels.
- **PR mode: check commit-message claims against the code.** When a message
  says "without this, X would break because Y", trace Y in the current base.
  (Once, "the journal would ride along with the next command's rows" was false,
  because a leftover sweep already existed in the base. The real fix was
  atomicity. The code was right; the message was not. That is a review point.)
- **What a check compares against.** For every verify/compare step, record the
  exact source read: table, row, field (`blocks` row 400, `state_root`), and
  where that value came from.
  - Never name a different copy of the same value (DB row vs on-chain vs
    memory).
  - (A page once said "compared with the value on chain"; the code read the DB
    row and never queried the chain. A reader took the page at its word, and it
    had to be corrected.)
- **Observable output.** Collect the exact `println!`/`log` strings each step
  emits and the DB statements it runs, in **code order**. (A snapshot-log line
  was once written after the block line it actually precedes.) They feed the
  "result first" section.

Accuracy is the whole point. A wrong `file:line` sends a bookmark to the wrong
code and erodes trust. Verify every line.

**Write bookmarks as soon as the trace is verified** (don't wait for prose):
fill `steps.py`, run `bm.py`, which writes the file and prints each step's real
source line for a quick eyeball.

**Say what you're doing.** Tracing is long and silent. Post a one-line status
when tracing starts and whenever a deliverable lands (bookmarks, figures, each
tab).

## Step 2 — Build the HTML

### The three tabs

- A tab bar at the top. Each tab is a `<section class="tab-section">` toggled by
  a few lines of JS; remember the last tab in localStorage, wrapped in
  try/catch.
- The bookmark panel and the expand/collapse-all buttons (`ui.code_open_all`,
  `ui.code_close_all`) sit OUTSIDE the tabs.
- The plain-explanation tab always exists. The other two exist only if the
  Step −1 scope question selected them (both, by default); see the skip rules
  there. A build with only one tab omits the tab bar.

1. **Plain explanation** (`tab.easy`, the default tab): analogy-driven, 1–2
   minutes to read, almost no identifiers, no `file:line` chips. Metaphor lives
   HERE only.
   - **Box headings state the claim, not the jargon.** A heading that quotes
     two terms against each other ("why it's a 'live table', not a
     'snapshot'") reads like an awkward translation. Say what it IS in one
     plain clause ("why every block reads the latest state directly") and let
     the bullets do the comparing.
   - PR mode adds:
     - a **before/after box pair** (`easy.before_pr` / `easy.after_pr`);
     - a changed / unchanged / rejected table (`easy.change_cols`);
     - a **commit table** titled `easy.commits_title`, with columns
       `easy.commits_cols`. The order is a reading order, not just a list.
       Each row has a short hash (8 characters) and a **one-line summary in
       the output language**, written for the reader. Never paste the commit's
       own headline verbatim.
2. **Data structures** (`tab.structure`): how the data is shaped and why. No
   numbering; numbers belong to the scenario tab. Sections, in order:
   0. **Which process holds it?** (`st.processes`): the concept × process table
      from Step 1, ✅/✗ per cell with the code that proves it
      (`Worker.cache: Index`, `Index::rebuild_from`, "no use under server/").
      Mandatory whenever more than one process is involved.
   1. **What sits inside what** (`st.layers`): root → subtree → leaf, one figure
      plus a `st.layers_cols` table.
   2. **Key (address) layout** (`st.key_layout`): a bit-bar figure plus a
      worked value.
      - If **two structures of the same kind** exist (two trees, two leaf
        kinds), open with a **comparison table** (`st.compare_cols`) and quote
        the code that joins them.
   3. **One section per counter or sequence**: see "Counters get their own
      section".
   4. **A small example** (`st.toy`): 8–16 cells with real numbers. Mention the
      real scale as a number only ("2⁴⁸ in production").
   5. **One operation, before and after** (`st.before_after`): the same figure
      twice, change by color.
   6. **Why this shape, and a counterexample** (`st.why_shape`): see
      "Derivation and counterexample".
   7. **Why this value?** (`st.why_value`) for every tunable constant the flow
      depends on (`KEEP = 3`, `every = 100`, chunk size):
      - where it is used;
      - a `st.why_value_cols` table of concrete situations (keeping 1 vs 3);
      - the `file:line`, and whether a CLI flag can change it.
      - ("Why keep only the last 3?" was asked.)
   - **Every section title carries a location badge** naming the process
     (`API server` / `worker` / `badge.both`, whatever the processes are called
     here; the project's agent instructions may name them).
   - Each section **opens with the question it answers**, never a type name.
3. **Scenario, step by step** (`tab.scenario`): ONE concrete request in time
   order. **It owns the step numbering; bookmarks match it 1:1.** It does not
   re-explain structure; it points (`st.see_section`).
   - Use **the same toy scene** as the structure tab, so cards carry the same
     values (`seq 4`, `(price 510, slot 4)`). Seed the scene from a real test
     and name the test, so the reader can check it.
   - **Open with "result first"** (`sc.result_first`), before the players map.
     Structure and `file:line` alone once left the reader with "I get the
     outline, but not what it actually does". The section shows one run as an
     operator would see it:
     1. the log lines, verbatim from the code's print/log strings (hashes as
        short examples, `0x3f07…`);
     2. **before / after** of the same run (`.pairgrid`, `sc.run_before_after`)
        with counts and time ("436 lines → 36 lines");
     3. the DB reads/writes in order (`sc.db_cols`);
     4. memory state per moment (`sc.memory_cols`);
     5. the log of each failure path (rejected, fell back).
     Then a heading `sc.flow_heading` and the rest below.
   - Then the **players map** (below), then a small glossary of the 3–5
     load-bearing nouns, each with a `ui.definition` `<details>`.
   - **Every phase title carries the location badge** too.
   - PR mode: steps that changed get a `badge.changed` / `badge.changed_commit`
     badge. Old code the flow passes through is labeled `badge.existing`, so
     new and base code are never confused.
   - A branching decision function (e.g. `should_retry`) gets a
     `sc.decision_cols` table.
   - A flow that **validates** something (restore, verify, admit) gets a
     `sc.checks_title` table (`sc.checks_cols`), each row with `file:line`.
     Then one short block on which check carries the weight, why it suffices,
     what it cannot catch, and what catches that later. ("How is row 400
     checked?" was asked.)
   - Branch/PR mode: steps a review round changed get a `badge.review` badge
     next to the change badges.
   - End with a final-state table (`sc.final_state`: what exists where), then a
     `sc.reading_order` table (`sc.reading_order_cols`), bottom-up: pure
     functions → storage → orchestrator → CLI → the one test that holds the
     whole flow. (Reading the code itself was the reader's next step every
     time.)

### Players map (scenario tab)

Not a sequence diagram: a swimlane of one request was judged "too much". The
picture shows WHO the players are and what passes between them; the cards carry
the time order. Computed, like every figure.

- **One box per player**, grouped by process (dashed group frames: server · DB ·
  worker · external service …). Label = name + `⚙` (real thread/task) or
  `fig.struct` (a struct driven by another thread) + one line of role.
  5–15 boxes.
- **Ownership = containment**: a struct held as a field is a box *inside* its
  owner's box (a container with its own title). Never siblings.
- **Edges = handoffs**: solid = call/write, dashed = read/poll/reply, thick = the
  writes that matter and the one edge that closes the loop.
  - Each label starts with the step numbers `[5–7]` (same as cards and
    `[<tag> N]`).
  - Group only steps that reach the **same target**. A free function gets its
    own box (dashed border) and its own `[N]`.
  - No ①②③ anywhere.
- Channels/queues are boxes too. Periodic actors carry their interval
  (`fig.every`). Start at the true origin.
- **Legend = a bordered box**, one item per line, vertical dividers between
  groups (box notation | line style | line color), and **real line samples**
  (color, width, dash, arrowhead). A one-line run-on legend was unreadable.
- **Multiple phases → a selectable phase filter.** When the scenario has 2+
  phases:
  - tag each phase-specific edge/label, and each phase-specific line of a
    surrounding note box, with `data-phase="N"` in the SVG;
  - add a row of phase pills above the figure (plain JS, no library). Clicking
    one dims (`opacity:.15`) every `[data-phase]` element whose phase doesn't
    match and restores the rest. A reset pill (`ui.phase_all`, the default)
    shows everything at full opacity;
  - structural player boxes always stay at full opacity; only the
    phase-specific edges and note lines dim.
  - Without this a reader cannot tell which part of one static picture belongs
    to which phase of the cards below it.
- **Check the numbering before shipping.** Walk every edge label and note-box
  line in the map and confirm its step range matches the actual cards (same
  numbers, same phase). A label drafted early and never rechecked after the
  cards were finalized is a common source of drift.

### Figures are computed (no mermaid, works offline)

Every figure comes from a Python generator that computes coordinates, writes
SVG, and converts it with `rsvg-convert -z 2` to PNG. Mermaid's auto-layout is
uncontrollable (2400×272 strips, edges across the page) and cannot express
figures where *position is meaning* (trees, bit layouts, before/after).

- One function → many figures: `grid(orders, marks)` drew the toy, before,
  after and the counterexample. Before/after differ only by color.
- Output `<flow-name>-<figure>.svg` + `.png`. Inline the SVG in
  `<div class="fig">`; link both files under it. No CDN scripts.
- **Read the PNG yourself** before moving on. Fix overflow/overlap in the
  generator and re-render.
  - Give **every edge label explicit coordinates**; midpoint auto-placement
    overlapped other edges every time.
  - Route around boxes; a straight line through a box reads as a connection.
  - **No color emoji in figures**: rsvg-convert renders 📦 as a blank blob. `⚙`
    works. Use text tags like `fig.struct` instead.
- Dark page → dark figures (background, lines, text matching the page).
- **Click to enlarge**: `.fig` is `cursor:zoom-in`. A library-free lightbox
  clones the SVG at its viewBox width with −/＋/100%/fit/PNG/close, Esc/±, and
  closes on a click outside.
- **Re-render on request** by editing the generator, never the SVG/PNG.

### Rules for every tab

**Long prose is bullets, not paragraphs, in all three tabs.**
- Any explanation over ~2 sentences ships as a short bullet list (2–4 items, one
  idea per item).
- That covers the `.easy` box bodies, the analysis under each structure
  section, and the scenario cards' `.d` bodies.
- "Scenario cards are plain words" below is the fullest instance of this rule
  (title / who line / bullets / example). Apply the same skimmability to the
  other two tabs even without that full card shape.

**Check as you write, not after the page is done.**
- The figure-overlap check already works this way: `save()` flags a figure's
  own overlaps the moment that figure is rendered, before the next one starts.
- Give the other checks the same shape: verify each excerpt, each `file:line`
  reference and each card's wording **the moment it is written**, not in one
  scan after the whole tab (or page) is finished.
- A mismatch caught one card late costs nothing. Caught at the end of the tab,
  it costs re-reading every flagged card to remember what it was supposed to
  say. (One timing run: 76 s and 17 fixes, all found in one end-of-tab pass.)
- The total amount of checking is the same either way; the cost is in how far
  the fix is from the write.

**Code excerpts.**
`<details class="code"><summary>{ui.code_summary}</summary><pre>…</pre></details>`.
The goal is to understand the code **without opening the editor**.
- Verbatim, read by the script from the source tree of Step −1 (never typed).
  Prefix each line with its number (`1386│ `). Trim with `⋯ (what was skipped)`
  lines; never rename. 5–15 lines; one step may show two excerpts.
- **Mechanical check right after writing each excerpt** (not after the tab is
  finished): parse it and compare each `NNN│` line to the real line. Ship only
  at zero mismatches.
- **Boundary check**, at the same moment (equal text is not enough): warn when
  an excerpt's first line starts with `)` `}` `);` or its last line ends with
  `(`. It was cut mid-statement. (`(624, 625)` started at `);` and passed the
  text check.)
- **`file:line` in prose, card chips and figure text** go through one helper
  **the moment each reference is written**.
  - It checks that the line exists and prints the **shortest path suffix that
    is unique in the repo** (`git ls-files`): `api/handlers.rs`,
    `core/src/lib.rs` — never a bare `lib.rs` / `mod.rs` / `handlers.rs`.
  - (Basename-only card chips once produced 12 ambiguous references, all found
    in one end-of-page sweep.)
  - Still re-run it as one final scan of the whole page before shipping, as a
    safety net for anything added out of order. It should find 0 by then, not
    discover the 12.
- **Figure label overlap**: the `Svg` class records each text's approximate box
  (CJK characters ≈ 1.0 em wide, others ≈ 0.6 em) and `save()` prints
  overlapping pairs. Fix until none remain, then still look at the PNG. (This is
  the pattern the other checks above follow.)
- **Template leftovers**: `{{` / `}}`, i.e. doubled braces written in a string
  that is not an f-string, print literally (`Order{{id 5…}}` shipped once). Also
  any unfilled `{placeholder}` from `assets/strings.md`. Rare enough to check
  once, as a final scan of the page (minus `<style>`/`<script>`/excerpts) before
  shipping; must be 0.
- Page chrome (self-contained JS): expand/collapse all, a copy-path button per
  excerpt (`ui.copy_path`), click a card's `file:line` chip to copy.
  - **No highlighter / search box.** It was tried and removed: it was unclear
    what it was for and added nothing.
  - Syntax coloring inside excerpts (keyword, type, string, comment, number; a
    small tokenizer in `code()`) is a different thing and is welcome.

**Scenario cards are plain words.** The first card version (function name + a
dense one-paragraph summary) drew "I can't take in what each piece of code does
— say it simply, and lint it." Card shape, from `plain.py`:
- **title** = what it does, in plain words ("Is the block number a multiple of
  100?"), not the function name;
- a **who line** under it: actor badge + `function`;
- **1–3 bullets**, one short sentence each;
- an **example line** (`sc.example`) with the toy value ("400 → saved, 401 →
  skipped");
- the code excerpts, unchanged.

The generator **lints each card in `plain.py` right after writing it**, not
after the whole file is drafted:
- a line over ~70 characters (CJK) or ~110 characters (Latin scripts);
- more than 3 bullets;
- a jargon word without an explanation in parentheses (SMT, BTreeMap,
  idempotency key, outbox, …);
- the forbidden terms.

Ship at 0. A final lint over the whole file before shipping is the safety net,
not the first catch.

**Counters get their own section.** Any counter/sequence that appears in a key,
a sort order or a commitment (`seq`, `nonce`, `next_*`) gets a dedicated
structure section. A glossary line was not enough ("what is this seq?"). The
section has:
- what it is, in one sentence;
- a properties table (`st.counter_props_cols`): who assigns it / when (the one
  place it is bumped) / range / shared or per-side / what 0 means / where it is
  stored (memory, DB column, chain leaf), each row with `file:line`;
- a **"why not another value?"** table (`st.why_not_other`): what was used
  before (`created_at`, `order_id`, …) and the case where it breaks. The code's
  doc comments usually record this history; quote them;
- a numeric walk: arrival → number received → counter after.

**Worked examples with numbers.**
- Evolving state (block numbers, queues, deadlines, nonces) needs numbers, not
  more prose: `1000 ─slot 1─ 1500`.
- Pick small readable values (price 500·510, size 2·3·4, 8 cells); mention real
  scale only as a number.
- Values must agree everywhere. When there are more than ~5, or there is
  sorting/aggregation, let the script compute them.
- One event changing several values → a table.

**Derivation and counterexample, for every load-bearing choice.**
1. A short derivation: which requirement forces the choice.
2. A **counterexample figure on the same toy**, with a red ✗ where it breaks
   ("a later order fills first").

If you cannot draw the counterexample, the choice isn't load-bearing; skip it.

**Terminology: run this before writing any prose.**
1. **Read the project's term table** and build a **forbidden → preferred** list
   from it.
   - The table lives in the project's agent instructions: `CLAUDE.md` and/or
     `AGENTS.md` (read both if both exist), section `## pr-explainer` →
     `### Terms` (or the heading in another language, `terms.heading`, e.g.
     `### 용어`), with columns `terms.cols`:
     ```markdown
     ## pr-explainer

     ### Terms
     | Avoid | Use | Why |
     |---|---|---|
     | sleeping order | resting order | the trade's own term |
     ```
   - The same section may also name the processes for location badges, a style
     source, or where pages are kept. Follow it.
   - No section → no project terms; the rules below still apply.
   - (Once, a forbidden word was used 15 times despite a recorded rule. Read the
     table before writing, not after.)
   - **When the user corrects a term** ("that word is off", "say Y, not X"):
     fix the page, then propose the row for that table. Show the exact line and
     the file, and edit `CLAUDE.md` / `AGENTS.md` only after they agree; create
     the section if it is missing. Recording it in the project, not in private
     memory, is what lets teammates and CI runs use the same words. In an
     unattended run, only read the table.
2. **No literal translation of domain jargon.** Use the term the trade uses in
   the output language. If none fits, keep the original term in parentheses.
   (In Korean, "resting order" is "걸려 있는 주문", not the literal "쉬는 주문".)
3. **No coined nouns.**
   - A term the walkthrough invents ("cell", "envelope", "seat reservation")
     cannot be grepped and collides with real structures. ("Cell" was once read
     as a different tree's leaf.)
   - Use the code's own name with an owner prefix ("order-book leaf", "order
     leaf").
   - If a shorthand truly earns its place, flag it `term.not_in_code` and point
     at the exact code.
4. **Introduce every internal noun before use**: what it is, **where it lives**
   (`Struct.field`, table, memory vs DB, which process), and who fills/consumes
   it. Phase titles pair plain words with the term.
5. Metaphor only in the plain-explanation tab; elsewhere state operations
   literally.
6. The generator **scans each section for forbidden terms the moment it is
   written** (excluding code excerpts, which are verbatim), including bookmark
   labels and figure text, not just prose. A final scan of the whole output
   before shipping must report 0. That scan is the safety net; catching a term
   mid-write is what keeps the count near 0 by then.

### Generation order + checkpoint

1. The shell (bookmark panel; tab bar only if 2+ tabs are in scope) with the
   plain-explanation tab → save → `open`.
2. If the structure tab is in scope: build it → save.
3. **Stop and ask (`ask.checkpoint`), only if the scenario tab is also in scope
   and another tab was just built.**
   - Keep it short, and include a preview of what the next tab will flag (e.g.
     the review point found), so the user can just say "go on".
   - Skip this ask when the scenario is the only tab left after the plain
     explanation alone, or the only tab in scope (nothing to check against yet).
4. Then the scenario tab, if in scope. The user reloads to see it.

**When the user says it's hard, go down, not on**: back to the structure tab and
a new figure with smaller values and fewer cells, not another paragraph.

**Questions fold into the page by default.** When the user asks "what is X?" /
"which process is this about?", answer in chat with code evidence **and update
the HTML section it belongs to** in the same turn, saying where it went. (Every
"should I add this to the page too?" was answered "yes".) Likewise, a mistake
the question exposes (wrong ownership in a figure, wrong term) is fixed in the
generator and re-rendered.

**Where to keep the file**: ask (AskUserQuestion, together with the snapshot
question). In an unattended run, use the path the caller gave.
- Recommend `docs/flows/<flow-name>.html` in-repo, unless the project's agent
  instructions name another place.
- Offer a scratch directory as the volatile option.
- Then `open` it. If it went to `/tmp` or a scratch directory, offer once at the
  end to move the HTML + figures into the repo.

## Step 3 — Matching bookmarks (two copies)

**HTML step `N` ⇔ bookmark `[<flow-tag> N] …`.** Use a per-flow tag
(`[login N]`, `[retry N]`). Build from `steps.py`, and write to:

1. **`.vscode/bookmarks.json`** of the code tree from Step −1 (the worktree in
   PR mode): 0-based lines, merge with existing entries rather than clobbering,
   `"bookmarks.saveBookmarksInProject": true`, then Reload Window.
2. **Embedded in the HTML panel**: a complete standalone bookmarks.json holding
   ONLY this flow (replacement semantics: paste over, one flow active).

When a label changes (term fixes), re-run `bm.py` + `gen.py` so both copies
change together. Verify each bookmark against its source line and tell the user
to Reload Window (in the folder the bookmarks were written to).

Label shape (the text after the identifier is in the output language):
```
[login 1] router.post — receives the login request
[retry 3] should_retry — [Changed · commit 3] decides whether the error is retryable
```

## Conventions

- **Shape first, motion second.** Structure tab before timeline tab, built first.
- **Say where it happens first.** Process table at the top of the structure tab,
  location badges on every section and phase.
- **Titles are questions.** "Who fills first at the same price?" ✓ /
  "The `BookHeader` struct" ✗.
- **Identical numbering** between the scenario tab, players map and bookmarks;
  no other tab or figure invents numbers.
- **Real `file:line` everywhere**, from the snapshot of Step −1.
- **Phase = mental chunk**: 3–5 phases, not 19 flat steps.
- **Result first, then structure and flow.** The scenario tab opens with what an
  operator sees (logs, DB rows, memory); the cards explain how it got there.
- **Call out the non-obvious** with invariant (green, `callout.invariant`),
  warning (red, `callout.warning`) and, in PR mode, review (orange,
  `callout.review`) callouts.
- **One artifact without the other is half the value.** Bookmarks ride along
  with the scenario tab automatically (see Step −1's scope question); never
  build one without the other.
- **The HTML is the archive; bookmarks.json is the working copy.**

## Notes

- Bookmark files follow the VS Code / Cursor "Bookmarks" extension format
  (alefragnani): 0-based lines, merge, reload.
- Very large flows: anchor links per phase; bookmark only load-bearing steps.
