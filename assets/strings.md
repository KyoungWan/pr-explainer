# Fixed strings

Every fixed piece of page text — tab names, section titles, table headers, badges, buttons, questions — comes from this file.

* Use the column that matches the output language (see "Output language" in `SKILL.md`).
* For a language without a column, translate the `en` column once at the start of the run and reuse that translation everywhere, so the same label never appears in two wordings.
* `{name}` is a placeholder. Fill it in; never print the braces.

## Tabs

| key | en | ko |
|---|---|---|
| `tab.easy` | Plain explanation | 매우 쉬운 설명 |
| `tab.structure` | Data structures | 자료구조 |
| `tab.scenario` | Scenario, step by step | 시나리오 시간순 |

## Page chrome

| key | en | ko |
|---|---|---|
| `ui.snapshot` | Snapshot: head {head}, base {base}, line numbers refer to {dir} | 기준: head {head}, base {base}, 줄 번호 기준 {dir} |
| `ui.bookmarks_json` | Bookmark JSON | 북마크 JSON |
| `ui.copy` | Copy | 복사 |
| `ui.code_open_all` | Expand all code | 코드 모두 열기 |
| `ui.code_close_all` | Collapse all code | 코드 모두 닫기 |
| `ui.copy_path` | Copy path | 경로 복사 |
| `ui.code_summary` | Code — {path}:{start}–{end} | 코드 — {path}:{start}–{end} |
| `ui.definition` | Definition | 정의 코드 |
| `ui.phase_all` | All | 전체 |

## Plain explanation tab (PR mode)

| key | en | ko |
|---|---|---|
| `easy.before_pr` | Before this PR | 이 PR 전 |
| `easy.after_pr` | After this PR | 이 PR 후 |
| `easy.change_cols` | Changed / Unchanged / Rejected | 바뀜 / 그대로 / 거부 |
| `easy.commits_title` | Key commits (read them in this order) | 주요 커밋 (이 순서대로 읽는 걸 추천) |
| `easy.commits_cols` | # / Hash / Commit / What it does | # / 해시 / 커밋 / 한 일 |

## Data structures tab

| key | en | ko |
|---|---|---|
| `st.processes` | Which process holds it? | 어느 프로세스에 있나 |
| `st.layers` | What sits inside what | 층 구조 |
| `st.layers_cols` | Layer / Key / Leaf content / During this operation | 층 / key / 리프 내용 / 이번 연산 때 |
| `st.key_layout` | Key (address) layout | 키(주소) 레이아웃 |
| `st.compare_cols` | Path / Count / Key / Content / Role / Changes in this operation | 경로 / 개수 / key / 내용 / 하는 일 / 이번 연산에서 바뀌는 것 |
| `st.toy` | A small example | 장난감 예시 |
| `st.before_after` | One operation, before and after | 연산 하나의 전과 후 |
| `st.why_shape` | Why this shape, and a counterexample | 왜 이 모양인가 + 반례 |
| `st.why_value` | Why this value? | 왜 이 값인가 |
| `st.why_value_cols` | Situation / If smaller / If larger | 상황 / 작으면 / 크면 |
| `st.counter_props_cols` | Who assigns / When / Range / Shared or per-side / What 0 means / Where stored | 누가 주나 / 언제 주나 / 범위 / 공유 여부 / 0 의 뜻 / 저장 위치 |
| `st.why_not_other` | Why not another value? | 왜 다른 값으로는 안 되나 |
| `st.see_section` | see Data structures §{n} | 자료구조 {n}장 참조 |

## Scenario tab

| key | en | ko |
|---|---|---|
| `sc.result_first` | Result first: what actually happens | 먼저 결과부터: 실제로 무슨 일이 일어나나 |
| `sc.run_before_after` | Before this run / After this run | 이 작업 전 / 이 작업 후 |
| `sc.db_cols` | Asks / Gets back / Decides | 묻는 것 / 돌아오는 것 / 판단 |
| `sc.memory_cols` | Moment / Field / Value | 시점 / field / value |
| `sc.flow_heading` | Flow: who does what | 흐름: 누가 무엇을 하나 |
| `sc.example` | e.g. | 예: |
| `sc.decision_cols` | Inputs / Verdict / When it happens | 입력 조합 / 판정 / 언제 일어나나 |
| `sc.checks_title` | Order of checks | 검사 순서 |
| `sc.checks_cols` | Stage / # / Checks / Against / Catches | 단계 / # / 확인 / 무엇과 대조 / 걸리는 경우 |
| `sc.final_state` | Final state | 최종 상태 |
| `sc.reading_order` | Suggested reading order | 추천 읽기 순서 |
| `sc.reading_order_cols` | Order / File / Look at / Bookmark | 순서 / 파일 / 볼 것 / 북마크 |

## Badges and callouts

| key | en | ko |
|---|---|---|
| `badge.both` | Both | 양쪽 |
| `badge.changed` | Changed | 변경 |
| `badge.changed_commit` | Changed · commit {n} | 변경 · 커밋{n} |
| `badge.existing` | (existing) | (기존) |
| `badge.review` | Final review {id} | 최종 리뷰 {id} |
| `callout.invariant` | Invariant | 불변식 |
| `callout.warning` | Watch out | 주의 |
| `callout.review` | Review point | 리뷰 포인트 |
| `term.not_in_code` | not a code term | 코드에 없는 표현 |

## Figures

| key | en | ko |
|---|---|---|
| `fig.struct` | [struct] | [구조체] |
| `fig.every` | every {interval} | {interval} 마다 |

## Questions (interactive runs only)

| key | en | ko |
|---|---|---|
| `ask.scope` | Which tabs should I build this time? | 이번엔 어떤 탭까지 만들까요? |
| `ask.checkpoint` | If this part makes sense, I'll move on to the scenario. Anything to unpack first? | 여기까지 이해되시면 시간순으로 넘어갑니다. 더 풀어야 할 곳 있나요? |

## Project term table (in CLAUDE.md / AGENTS.md)

| key | en | ko |
|---|---|---|
| `terms.heading` | Terms | 용어 |
| `terms.cols` | Avoid / Use / Why | 쓰지 말 것 / 쓸 말 / 이유 |
