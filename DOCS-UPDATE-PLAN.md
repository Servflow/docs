# ServFlow Docs Update Plan (v2)

**Status:** Proposal for review — no live docs changed yet.
**Grounded against:** the running server (`:8080 --dashboard`), the current CLI,
`config.example.toml`, `resource action list` / `template-functions`, and the
builder frontend source (routes, `AgentList`, `EmptyState`) on 2026-07-19.

---

## 0. The reframe: ServFlow is an agent builder

The docs position ServFlow as a "visual backend engine". The product no longer
does: the dashboard's landing flow is **create an agent → attach its first
workflow**; the sidebar groups workflow configs **under agents** (with an
"uncategorized" bucket and an assign-to-agent action); primary routes are
`/agents/:agentId` and `/workspaces/:workspaceId`.
(Verified in `builder/src/routes.tsx`, `components/EmptyState.tsx`,
`components/sidebar/AgentList.tsx`.)

**Docs consequence:** Agents are the top-level object in the introduction and
the concepts hierarchy. Workflows/configs are documented *as components of an
agent* (webhooks + tasks + workspace), with standalone workflow usage still
covered but not leading.

---

## 1. Principle audit → corrections (per user review)

| Principle | Finding | Agreed correction |
|---|---|---|
| Self-contained | Quickstarts defer to Installation | **Fine as-is.** Self-containment targets large conceptual prerequisites, not install steps. A direct link to the exact install method suffices. |
| Specific topic | `installation.mdx` mixes methods + TOML + flags | **Keep Installation as one page** — installing is a single goal. Only the *content* gets fixed (ports, first-run, TOML sample). |
| Contextualized | Only `index.mdx` explains the product | **Add context to other pages**: 1–2 sentence opener on every page — what ServFlow is (an agent builder) + where this topic fits. |
| Defined type | `running-modes.mdx` mixes concept + walkthrough | **Make "Running ServFlow" a pure concept page** that links out to walkthroughs/examples instead of inlining them. |
| Metadata | Frontmatter is title+description only | **Leave as-is.** No custom frontmatter schema. |
| Neighborhood | Next-Steps blocks are good | Keep; ensure new pages are linked from ≥2 others. |
| Consistency | Bogus code-fence info-strings (```` ```docs/quickstart.mdx#L13-L14 ````), two field-table formats | **Fix.** Plain language-tagged fences; one field-table format (Field / Type / Required / Default / Description). |

**Apply the same corrections to similar offenders** — e.g. any other page inlining
a walkthrough into a concept (parts of `references/concepts/*`), and every page
with the code-fence bug (`quickstart*.mdx`, all `concepts/actions/*.mdx`).

---

## 2. Information architecture (revised)

Single **Guides** tab structure retained (no tab-level split); type discipline
enforced per page, not per tab.

```
Getting Started
  Introduction                          [REWRITE — agent-builder positioning]
  Installation                          [FIX in place — ports, first-run, TOML]
  Quickstart
    Deploy your first API with the dashboard    [RENAME + rewrite]
    Deploy your first API declaratively         [RENAME + rewrite — resource CLI + YAML]
  Running ServFlow                      [CONVERT to concept page — modes, /dashboard
                                         mount, headless; links to walkthroughs]
  Local development                     [verify]

Concepts
  Agents                                [NEW — the main object: workflows as
                                         webhooks/tasks, workspace, agent action vs
                                         agent resource]
  Workflows & configs                   [NEW — entry → actions → conditionals →
                                         responses; config_id identity]
  Entries                               [NEW — http, trigger, entry handlers]
  Actions (overview + categories)       [REWRITE/REGENERATE — 18 missing actions]
  Composition (callworkflow)            [NEW]
  Conditionals / Responses / Dynamic content   [keep; add context openers]
  Plugins                               [NEW]

References
  Configuration (TOML)                  [FULL REWRITE — schema is wrong today]
  CLI: start                            [NEW]
  CLI: resource                         [NEW — per-noun subpages]
  CLI: plugins                          [NEW]
  Template functions                    [NEW — generated from binary]
  Management MCP API (/api/mcp)         [NEW]
  Secrets                               [verify — master_key under [sqlite]]

Tutorials
  Telegram trading bot                  [verify against current build]
```

---

## 3. Factual gaps (unchanged from v1 audit, placed in new IA)

### 🔴 Critical
1. **Dashboard URL**: `/dashboard` on `[server].port` (8080) — every `localhost:3000`
   reference is dead. One server; `--dashboard` mounts the UI, no second port.
2. **Login + first-run onboarding undocumented**: local auth
   (`[authentication] mode = "local"`), setup wizard, first `/register` bootstraps
   the account. Auth0 / `dash.servflow.io` references stale (incl. `docs.json`
   navbar button).
3. **Configuration Reference wrong**: `[dashboard]` section gone; `master_key`
   under `[sqlite]`; new `[authentication]` section; `[server]` gained keys
   (`server_prefix`).

### 🟠 Major
4. `resource` CLI (config/agent/integration/secret/workspace/action/settings,
   `--dry-run`, `--reload`) — undocumented.
5. `plugins install` (zip / GitHub URL) — undocumented.
6. First-class **agents** — now the core object, entirely undocumented.
7. Management MCP server at `/api/mcp` — undocumented.
8. **18 missing actions**: `callworkflow`, `chromium/*` (4), `discord/*` (3),
   `download`, `firestore`, `get_key`, `store_key`, `github_token`,
   `githubverify`, `save`, `shell`, `stub`.
9. Entry types beyond `http`: `trigger`, HTTP entry handlers (`github_webhook`).

### 🟡 Minor
10. `param` misused for JSON-body fields in examples (`param` = query/form;
    `body` = JSON body). Audit all examples against `template-functions`.
11. Hidden `api-reference` tab in `docs.json` points at nonexistent files.
12. Verify install channels (npm package, Homebrew tap, Docker image).
13. Re-verify `--import-configs` description.

---

## 4. Per-page content plan

Each page below states its **type** and what it will contain.

### Introduction (`index.mdx`) — Concept
Reposition: ServFlow builds and runs **AI agents** — an agent bundles workflows
(inbound webhooks + scheduled/internal tasks), a workspace, and integrations,
executed by a Go engine. "Backend engine" becomes the *how*, not the *what*.
Three build paths: dashboard, declarative (CLI/YAML), MCP. Cards → quickstarts,
Agents concept, Configuration.

### Installation (`installation.mdx`) — Task (kept as one page)
Fix in place: correct ports/URLs, correct TOML sample, add a short **First run**
section (setup wizard → first account → sign in) with a screenshot, verify
install channels. No split.

### Quickstart: Deploy your first API with the dashboard — Tutorial
Renamed. New flow matching the real UI: sign in → create **agent** → add its
first workflow → configure entry + response → save → curl. Re-shoot all five
screenshots (current ones predate login + agent-first landing).

### Quickstart: Deploy your first API declaratively — Tutorial
Renamed, rewritten around `resource` CLI: discover (`config example`, `action
describe`) → assemble JSON/YAML → `--dry-run` → `create` → `--reload` → curl.
YAML-file/headless path folded in as a variant, not a separate page.

### Running ServFlow (`running-modes.mdx`) — Concept (converted)
What runs where: one server, engine owns root, dashboard mounted at
`/dashboard`, headless mode, storage (SQLite vs file), auth modes. The
"Dashboard to Production" walkthrough moves out (linked as part of the
declarative quickstart / a deploy example). Links to walkthroughs + examples.

### Concepts: Agents — Concept (NEW, keystone page)
The main object. An agent = name + workspace + webhooks (http-entry workflows)
+ tasks (trigger workflows, cron or internal). Disambiguate **agent resource**
vs **`agent` action**. How the dashboard sidebar maps to this model. CLI
equivalents (`agent create`, `add-config --config-type webhook|task`).

### Concepts: Workflows & configs — Concept (NEW)
entry → actions → conditionals → responses; step references
(`action.x`/`response.y`); `config_id` as portable identity; where configs live
(store vs folder).

### Concepts: Entries — Concept (NEW)
`http` (path/method), `trigger` (enabled, `result`, inputs), entry handlers
(`github_webhook`) that wrap a request before the plan.

### Concepts: Composition — Concept (NEW)
`callworkflow` → trigger callee; inputs/`result` contract; `enabled: true`
requirement; no cycles; when to decompose vs inline.

### Actions catalog — Reference (regenerated)
`available.mdx` regenerated from `resource action list` (44 actions). New
category pages: Browser (chromium), Discord, Persistent storage
(`get_key`/`store_key`), Files (`download`), GitHub (`github_token`,
`githubverify`); `callworkflow` joins Flow Control; `save`/`shell` join their
categories. Field tables generated from `action describe --all`. Example sweep
fixes the `param`/`body` conflation.

### CLI reference (NEW pages) — Reference
`start` (flags incl. `--dashboard`, `--import-configs`); `resource` overview +
per-noun pages; `plugins`. Tables generated from the built binary's help + JSON
output.

### Management MCP API — Reference (NEW)
`/api/mcp`: what it is, tool families, when to use vs CLI.

### Configuration Reference — Reference (full rewrite)
Source of truth: `config.example.toml` + `resource settings view`. Documents
`[server]`, `[sqlite]`, `[authentication]`, `[tracing]`; env-var override table
retained.
**Deliberately not documented (internal/unreleased):** S3/workspace storage
(`[aws]`, `[workspaces]`), master/replica clustering (`[master]`), `[pprof]`,
`[ratelimit]`, `[action_templates]`.

---

## 5. How the 7 principles are enforced (as corrected)

1. **Self-contained** — reserved for conceptual prerequisites: concept pages
   never require reading another concept first; task pages may link to exact
   install/setup anchors.
2. **Specific topic** — one goal per page; single-goal multi-method pages
   (Installation) stay whole.
3. **Contextualized** — every page opens with 1–2 sentences: ServFlow is an
   agent builder + where this topic sits. Written per-page (no snippet
   machinery, keeps frontmatter/nav as-is).
4. **Defined type** — each page declared in this plan as Tutorial / Concept /
   Reference and structured accordingly; Running ServFlow is the model
   conversion.
5. **Metadata** — native frontmatter only (title, description, icon); nav via
   `docs.json`.
6. **Neighborhood** — keep Next-Steps CardGroups; every page links its
   cross-type neighbors (concept ↔ reference ↔ tutorial); no orphans.
7. **Consistent style** — fix bogus code-fence info-strings everywhere; one
   field-table format; imperative second-person voice; screenshots re-shot
   against the live UI.

---

## 6. Execution phases

1. **Critical fixes + reframe core** — Introduction, Installation, both
   quickstarts (renamed), Running ServFlow conversion, Configuration rewrite,
   `docs.json` (nav renames, dead api-reference entries, navbar button).
   Re-shoot screenshots.
2. **Agents + workflow concepts** — Agents (keystone), Workflows & configs,
   Entries, Composition.
3. **Reference generation** — Actions catalog + CLI + template functions,
   generated from the freshly built binary (never from memory/source).
4. **Sweep** — context openers + fence/table consistency across all surviving
   pages; example `param`/`body` audit; Plugins + MCP pages; verify Telegram
   tutorial.
