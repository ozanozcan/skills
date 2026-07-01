---
name: dispatch-plan
description: Turn a finished plan (from grill session, plan mode, any decision made on the chat, question and answers on the chat) into self-contained per-todo briefs, group them into parallel/sequential lanes, and dispatch each lane to the right subagent. Use after finishing a grill-with-docs or plan-mode session, when the user has a .cursor/plans/*.plan.md with multiple todos and wants to run them async without losing decisions or starting blind chats.
---

# Dispatch Plan

One skill, two modes, for the moment a `grill-me` + plan-mode session ends with N todos and you don't want to (a) keep building in a heavy chat, or (b) start todos in fresh chats blindly and lose decisions. the tasks can also come from /grill-me runs on agent mode, plan agent runs without /grill-me, any chat with questions asked by the agent and user made choices from the options given by the agent, any chat where some decisions are made.

| Invocation | Mode | When |
|---|---|---|
| `/dispatch-plan` | **decompose** (default) | Run **in the originating chat**, while the decisions are still live. Cheap on context. Writes briefs + index to disk. |
| `/dispatch-plan dispatch` | **dispatch** | Run from **any** chat (even fresh). Reads the index, fires subagents / emits paste-ready prompts. |

Parse the user's message: if it contains `dispatch`, run **dispatch**; otherwise run **decompose**.

**Core idea:** the plan file already persists the high-level decisions. The gap is (1) per-todo briefs carrying only the context each todo needs, (2) a grouping/order map, (3) the actual async fan-out. Decompose closes 1+2 from disk so dispatch can run anywhere.

---

## Mode: decompose (default)

Run this in the chat where the plan was created — it is the only place that holds in-chat decisions not yet written to the plan file.

### 1. Locate the plan

Use the active plan if known. Otherwise glob '/plans/*.plan.md` and pick the most recently modified; if ambiguous, ask which one. Read its frontmatter `todos` and the full body (`## Decisions locked`, workstream sections, scope notes).

### 2. Harvest live knowledge

Scan **this conversation** for any decision, constraint, file path, or "do not" that was settled during grilling but is NOT in the plan body. These would be lost in a fresh chat. Fold them into the relevant briefs in step 4. This is why decompose must run in the origin chat.

### 3. Build the lane map

Classify every todo, then group:

- **Sequential (same lane, ordered):** two todos whose file-sets overlap, or where one explicitly depends on the other's output. Shared files = shared state = never parallel.
- **Independent (separate parallel lanes):** disjoint file-sets and no dependency.
- **Final wave (alone, last):** any todo that *documents / summarizes / verifies the result of the others* (e.g. a master doc, a final test pass). It depends on everything.
- **Foreground-only (do not background):** destructive ops, anything outside the repo (home dir, global config), git history rewrites, or anything `guard-destructive.sh` would block. Mark these `needs-review`.

**Hard rule:** lanes that run in the same wave MUST have disjoint file-sets — subagents share one working tree.

### 4. Assign a subagent per lane

| Todo flavor | subagent_type | Notes |
|---|---|---|
| Multi-file code edit / refactor | `generalPurpose` | default |
| Shell / filesystem / symlinks / git / destructive | `shell` | usually foreground |
| Read-only research / "find where X is" | `explore` | |
| UI / design authoring | project's design subagent if it exists, else `generalPurpose` | a not-yet-created subagent can't build itself — bootstrap with `generalPurpose` |
| Diff review / security / complexity audit | project's review subagent / `security-review` if it exists, else `generalPurpose` | |

### 5. Write the dispatch folder

Create `.cursor/plans/<plan-stem>.dispatch/` containing `INDEX.md` plus one `NN-<todo-id>.md` brief per todo (NN = wave-ordered). Use the templates below.

Each **brief** must be runnable by a blind agent — no reference to "this chat":

```markdown
# <todo-id>: <one-line goal>

**Subagent:** <type>   **Wave:** <n>   **Background:** <yes|no>

## Goal
<what done looks like, concretely>

## Context & decisions (only what this todo needs)
- <decision + rationale, harvested from plan + chat>

## Files in scope
- <paths this lane owns; nothing outside this list>

## Depends on
- <todo-ids that must finish first, or "none">

## Do NOT
- <scope traps, anti-patterns, files to leave alone>

## Acceptance check
- <command or observable result that proves it's done, e.g. `pytest workouts/`>
```

The **INDEX** is the dispatch contract:

```markdown
# Dispatch index — <plan name>

Source plan: .cursor/plans/<plan-stem>.plan.md

## Waves
- **Wave 1 (parallel):** <lane> | <lane> | <lane(seq: a→b)> ...
- **Wave 2 (after wave 1):** <final lane>

## Lanes
| Lane | Todos (in order) | Files owned | Subagent | Background | Brief |
|---|---|---|---|---|---|
| A | ... | ... | generalPurpose | yes | 01-....md |

## Conflicts check
Confirm: no two same-wave lanes share a file. <list any risk>
```

### 6. Hand off

Tell the user it's safe to close or continue this chat; everything needed is on disk. To run: `/dispatch-plan dispatch` in any chat (or open the briefs in separate chats manually).

---

## Mode: dispatch

Reads the dispatch folder and runs it. Works from a fresh chat — it relies only on disk.

### 1. Load

Read `INDEX.md` and every brief. Re-verify the conflicts check (no same-wave file overlap). If a brief is stale vs the plan, flag it.

### 2. Run wave by wave

For each wave, in order:

- **Background fan-out:** issue all of the wave's lanes as Task subagents **in a single message** so they run concurrently. Set `subagent_type` per the index, `run_in_background: true`, and a sequential lane's brief instructs the subagent to do its steps in order internally. Pass the brief file path + its contents as the prompt; never assume inherited context.
- **Foreground / `needs-review` lanes:** do these yourself or one at a time, with the user watching — do not background destructive or out-of-repo work.
- Do not start wave N+1 until wave N's lanes report done.

### 3. Paste-ready alternative

Also surface, per lane, a copy-pasteable prompt block (the brief contents) so the user can open separate chats manually instead of/alongside background subagents. Default is to offer both: fire background subagents AND print the prompts.

### 4. Integrate

When lanes return: read each summary, check for cross-lane file conflicts, run the acceptance checks (and the repo's canonical tests, e.g. `pytest`), then handle the final-wave "documents-all" todo as the orchestrator since it synthesizes the others' results.

---

## Notes

- Decompose is read-mostly + writes small files → run it even at high context; it's the cheap insurance against losing decisions.
- Keep each lane's file ownership disjoint within a wave; that single rule is what makes parallel dispatch safe in a shared tree.
- A subagent that is itself being created by a todo cannot be the one to build it — bootstrap with `generalPurpose`.
- This skill does not edit the plan file; it reads it and writes a sibling `.dispatch/` folder.
