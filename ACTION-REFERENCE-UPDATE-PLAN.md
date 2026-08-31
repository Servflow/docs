# Action reference update + `servflow-pro` → `servflowai` rename

**Status:** proposal. No docs changed yet.
**Ground truth:** `go run ./cmd/configgen` against `servflow-pro` @ `126fdbe`
(module `git.servflow.io/servflow/servflowai`, engine pinned
`v0.1.7-0.20260825124449-a65f52607907`). 36 registered actions, 10 integrations.
**Scope:** `concepts/actions/*` plus the rename across the whole docs site.
Everything else is listed under "Out of scope" and left alone.

---

## Part 1 — Analysis: action references

### 1.1 The headline problem

The action pages document 24 actions. The binary registers 36. The two sets
overlap on 20. So:

- **4 documented actions do not exist**: `store`, `update`, `command`,
  `memory_store`, `memory_fetch` (5 sections, 4 concepts — `store`/`update`
  merged into one live action).
- **16 live actions are undocumented**: `save`, `shell`, `store_key`, `get_key`,
  `download`, `firestore`, `callworkflow`, `github_fetch`, `github_post`,
  `discord/sendmessage`, `discord/readmessages`, `discord/readguild`,
  `chromium/navigate`, `chromium/click`, `chromium/screenshot`, `chromium/body`.
- **3 documented actions have wrong fields**: `agent`, `email`,
  `binance/tradeinfo`.

### 1.2 Renamed actions — the four "does not exist" cases

| Doc page says | Live action | What changed |
|---|---|---|
| `store` + `update` (two sections) | `save` | One action. `filters` present → update; absent → insert. Same `integrationID`/`table`/`fields`/`datasourceOptions`. Output is now an object with `id`. |
| `command`, field `command` | `shell` | Field is `script` (required), plus `shell` (`sh`\|`bash`\|`zsh`, default `sh`). Output is combined stdout+stderr, and **a failing script reports through the output rather than failing the step** — the doc's `fail: response.error` examples are misleading. |
| `memory_store`, fields `key`+`contents` | `store_key`, fields `key`+`value` | Storage is **persistent**, not "cleared when the server restarts". The page's `<Note>` is wrong. Output is the stored value. |
| `memory_fetch`, field `key` | `get_key`, fields `key`+`failIfEmpty` (default `false`) | Same rename; gains `failIfEmpty`. |

### 1.3 Wrong fields on documented actions

**`agent`** (`concepts/actions/ai-agents.mdx`) — the most stale page in the set.

- `integrationID` → `providerID`. An LLM is no longer an integration: it resolves
  through a separate provider registry (`openai`, `claude`, `qwen`, `router`).
  Every example on the page names an OpenAI *integration*, which no longer wires
  an agent to a model.
- `conversationID` — **removed**. Conversation threading is owned by the run, not
  configured per action.
- `returnLastMessage` — **removed**. The output is always the agent's final reply.
- New: `accessMemory` (boolean; grants built-in `read_file`/`write_file`/
  `list_files` over the config's workspace) and `name`.
- Tool config is wrong in two places: MCP's field is `toolsList`, not `tools`;
  and a workflow tool is `workflowConfig: {id, values}` with `name`/
  `description`/`params` sitting on the **tool**, not nested under
  `workflowConfig`.
- Two tool types are undocumented: `agent` (delegate to another agent node in the
  same config) and `action` (embed one engine action as a tool, with fields
  either fixed by the author or filled by the model).

**`email`** (`communication.mdx`) — documents `serverHostname`, `serverPort`,
`username`, `password` as config fields. Live fields are `auth` (a map, required)
plus a required `name`. The four SMTP fields moved inside `auth`; the page's
examples will not load.

**`binance/tradeinfo`** — documents `infoType`, `orderID`, `futures`. Live fields
are `integrationID` + `symbol` only.

**`binance/futuresorder`** — `time_in_force` undocumented.

**`fetch`** — `datasourceOptions` and `shouldFail` undocumented. Also
`failIfEmpty` defaults to **`true`** here (it is `false` on `get_key`), which is
the kind of thing the page should state and currently does not.

**`static`** — `config` field undocumented.

### 1.4 The structural problem the pages have not caught up with

Every example on every action page uses this shape:

```yaml
actions:
  action_id:
    type: <action>
    config: {...}
    next: action.next_step
    fail: response.error
```

That shape is now **workflow-only**, and workflows are a narrower thing than the
docs imply:

- A workflow is `kind: workflow` and is reached **only by invocation** — an
  agent's workflow tool, the `callworkflow` action, or its own cron. It can never
  be reached by an HTTP request. Its body is `{cron, start, actions, conditionals,
  responses, integrations}`, and `start` is required.
- A workflow **rejects the `agent` action outright**: "workflows are pure actions,
  so put LLM steps in an agent config." So every example on `ai-agents.mdx` is not
  just stale on field names — it is in a container that refuses to hold it.
- An **agent config** (`kind: agent`) is where actions mostly live now, and there
  the shape is completely different — steps in a context group, no `next`/`fail`
  per step:

  ```yaml
  contextGroups:
    fetch:
      name: Fetch PR context
      isParallel: true
      steps:
        - id: diff
          type: github_fetch
          config: {...}
      next: agent.responder
  ```

Consequence: `concepts/actions/overview.mdx` teaches one shape as *the* shape, and
it is now the less common of two. This is the single biggest correctness issue in
the action reference, larger than any individual field.

Related: `parallel` is documented as a hand-written action. It still registers and
still works, but in an agent config the normal way to get it is
`isParallel: true` on a context group, which lowers to a `parallel` action. The
page should say so rather than presenting hand-authoring as the only route.

### 1.5 What the catalog gives us for free

`configgen` emits `action-config.json` with, per action: display name,
description, and per field the type, label, placeholder, required flag, default,
and enum values — plus an `output` block describing what the action publishes
(`none`/`value`/`object`/`dynamic`, with per-variant shapes for `github_fetch`).

None of that output metadata is in the docs today, and it is exactly what a
reference page needs: "what can I read from `{{ .step_id }}` afterwards". I
propose every action section gains a short **Output** subsection generated from
this block.

### 1.6 Out of scope but worth flagging

Found while verifying; **not** part of this change unless you say so.

- `concepts/agents.mdx` documents `servflow-pro resource agent add-config`. The
  `agent` noun no longer has that subcommand — an agent is a config kind, not a
  separate record. That whole CLI block is wrong.
- `ai-agents.mdx` positions agents as "an action inside a workflow". Product-wise
  it is now the reverse: the agent config is the top-level object and workflows
  are things it calls. Fixing that framing is a rewrite of `concepts/agents.mdx`
  and the overview, which `DOCS-UPDATE-PLAN.md` already proposes.
- No `loop` action shipped; nothing in the docs claims one. No action needed.

---

## Part 2 — Analysis: the rename

58 occurrences across 9 files. Three distinct kinds, and only the first is a
mechanical swap.

**A. The binary / CLI invocation** — `servflow-pro start …`, `servflow-pro
resource …`. Verified new name: **`servflowai`** (`.goreleaser.yaml`
`project_name: servflowai`, `README.md`). ~35 occurrences. Straight swap.

**B. Distribution identifiers** — each verified separately, because a blind
`sed` gets some of these wrong:

| Doc says | Correct | Source |
|---|---|---|
| `brew install Servflow/servflow/servflow-pro` | `Servflow/servflow/servflowai` | goreleaser `brews.name: servflowai`, tap `Servflow/homebrew-servflow` |
| `npm install -g servflow-pro` | `npm install -g servflowai` | `npm/package.json` `"name": "servflowai"` |
| `docker pull servflow/servflow-pro` | `servflow/servflowai` | `Makefile` `DOCKER_HUB_REPO` |
| `github.com/Servflow/servflow-pro` | `github.com/Servflow/servflowai` | goreleaser homepage, npm repository |
| `servflow-pro_Linux_x86_64.tar.gz` | `servflowai_Linux_x86_64.tar.gz` | goreleaser `project_name` drives the archive name |
| `--name servflow-pro` (docker run) | `--name servflowai` | cosmetic, but keep it consistent |
| `service_name = "servflow-pro"` | `service_name = "servflowai"` | `config.example.toml`, `internal/config/toml.go` default |

**C. Prose brand name** — "ServFlow Pro" appears ~14 times in sentences
("ServFlow Pro is a visual backend engine…"). This is **not** decided by the
repo: `README.md` says "ServflowAI", the dashboard title says "ServFlow.io", and
the docs elsewhere just say "ServFlow". **This needs your call** — see Open
decisions. It is the only thing blocking a clean sweep.

---

## Part 3 — The plan

### Phase 0 — Generate the reference source (prerequisite)

Add a small generator step that runs `configgen` and emits an intermediate JSON
the doc pages are written against, so the reference cannot silently drift again.
Not a build dependency for Mintlify — a checked-in snapshot plus a note on how to
refresh it.

Deliverable: `references/action-catalog.json` (snapshot) + a `make docs-catalog`
style note in the docs README.

### Phase 1 — Fix the four renamed actions

| File | Change |
|---|---|
| `concepts/actions/command.mdx` | Rename page to **Shell**, action to `shell`, field `command` → `script`, add `shell` enum + default, correct the failure semantics (a failing script does not route to `fail`), rewrite all 5 examples. |
| `concepts/actions/memory.mdx` | `memory_store` → `store_key` (`contents` → `value`), `memory_fetch` → `get_key` (+ `failIfEmpty`). Delete the "cleared on restart" note — storage is persistent. Rewrite the cache-aside pattern accordingly. |
| `concepts/actions/data-operations.mdx` | Merge the `store` and `update` sections into one `save` section explaining the filters-decide-insert-vs-update rule. Add `save`'s `id` output. Add `fetch`'s `datasourceOptions` + `shouldFail`, and state `failIfEmpty` defaults to `true`. |

### Phase 2 — Fix wrong fields on existing pages

| File | Change |
|---|---|
| `concepts/actions/ai-agents.mdx` | `integrationID` → `providerID` (+ explain providers ≠ integrations); delete `conversationID` and `returnLastMessage`; add `accessMemory` and `name`; fix MCP `tools` → `toolsList`; fix workflow-tool nesting; document the `agent` and `action` tool types; **rewrite every example as an agent config**, not a workflow action, and state plainly that a workflow rejects the `agent` action. |
| `concepts/actions/communication.mdx` | Replace the four SMTP fields with the `auth` map + required `name`. Rewrite examples. |
| `concepts/actions/binance.mdx` | `binance/tradeinfo`: drop `infoType`/`orderID`/`futures`, document `integrationID`+`symbol`. `binance/futuresorder`: add `time_in_force`. |
| `concepts/actions/transformation.mdx` | Document `static`'s `config` field. |

### Phase 3 — Add the 16 missing actions

Proposed page placement (three new pages, rest folded into existing ones):

| Action(s) | Page |
|---|---|
| `save` (done in Ph.1), `firestore` | `data-operations.mdx` |
| `store_key`, `get_key` (done in Ph.1) | `memory.mdx` (retitle → **Key-Value Storage**) |
| `shell` (done in Ph.1), `download` | `command.mdx` → retitle **System** |
| `callworkflow` | `flow-control.mdx` |
| `github_fetch`, `github_post` | **new** `concepts/actions/github.mdx` |
| `discord/sendmessage`, `discord/readmessages`, `discord/readguild` | **new** `concepts/actions/discord.mdx` |
| `chromium/navigate`, `click`, `screenshot`, `body` | **new** `concepts/actions/browser.mdx` |

Each new section carries: description (from the catalog), field table
(Field/Type/Required/Default/Description), an **Output** subsection from the
catalog's `output` block, and one worked example in the *correct* container
(agent context-group step for the integration-backed ones; workflow `actions:`
map for `callworkflow` and the pure-data ones).

### Phase 4 — Fix the structural framing

| File | Change |
|---|---|
| `concepts/actions/overview.mdx` | Document **both** containers: an agent config's `contextGroups[].steps[]` (no per-step `next`/`fail`) and a workflow's `actions:` map (with `next`/`fail`, plus required `start` and the invocation-only rule). State that `agent` is not valid in a workflow. Add an Output section explaining `{{ .step_id }}` and pointing at each action's Output block. Note that `isParallel: true` on a context group is the normal route to `parallel`. |
| `concepts/actions/available.mdx` | Regenerate the whole index from the catalog: 36 cards, correct names, three new category groups. |
| `docs.json` | Add `github`, `discord`, `browser` to the Available Actions group; keep ordering stable. |

### Phase 5 — The rename sweep

1. Mechanical pass for kind A (CLI invocations) across all 9 files.
2. Hand pass for kind B using the verified table in §Part 2 — do **not** sed
   these; the archive filename and brew formula path each need their own edit.
3. Prose pass for kind C once the brand name is decided.
4. Verify: `grep -rIn "servflow-pro\|ServFlow Pro" .` returns nothing.

### Phase 6 — Verify

- Re-run the diff script (catalog vs. `**YAML Key**` tables) — it should report
  zero doc-only fields, zero undocumented fields, zero undocumented actions.
- Check every `href` added in Phase 3 resolves to a real anchor.
- Mintlify build.

---

## Open decisions

1. **Brand name in prose.** "ServFlow Pro" → which of:
   - **"ServFlow"** — matches how the rest of the docs already read, and the
     dashboard title. My recommendation: the "Pro" distinction has no meaning in
     the docs, and "ServFlow" reads cleanest.
   - **"ServflowAI"** — matches `README.md` and the new package identity.
   - **"ServFlow AI"** — neither repo uses this spelling today.

2. **Chromium and Binance pages.** Both action families are registered and
   working, but they are narrow. Document them at the same depth as everything
   else (current plan), or demote Binance's 803-line page to a compact reference?

3. **Scope of Phase 4.** Fixing the container framing is strictly necessary for
   the action examples to be runnable, but it touches `overview.mdx`, which
   `DOCS-UPDATE-PLAN.md` also earmarks for a rewrite. Do this narrowly now, or
   defer to that larger effort?

---

## Effort

| Phase | Files | Size |
|---|---|---|
| 0 Catalog snapshot | 2 new | small |
| 1 Renamed actions | 3 | medium |
| 2 Wrong fields | 4 | medium (ai-agents is most of it) |
| 3 Missing actions | 3 new + 4 edited | large |
| 4 Framing | 3 | medium |
| 5 Rename | 9 | small, mechanical |
| 6 Verify | — | small |
