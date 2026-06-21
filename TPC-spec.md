# Conducting Trees on wash + SDT — Specification v0.1

**Status:** Draft
**Date:** 2026-06-19
**Extends:** `runtime.md` (the wash runtime spec) and `sdt-spec.md` (the Sequential Directory Tree format, v1.1).

> This spec is normative only where it cites a DECISION from the source log. Where the log underdetermines a point, the text carries an explicit `> OPEN:` callout instead of a guess. Base-spec changes are stated as deltas in §3 and never assumed silently.

---

## 1. Problem Statement

A **tree** is a directory of ordinary files grown by POSTing to wash over a Sequential Directory Tree. Each node holds some text and optional metadata; children of a node are either a single deeper line (one child) or a fan of explored alternatives (≥2 children). Such a tree can hold a conversation, a command history, or any mixture — the format privileges none of these. This spec defines how wash conducts a turn against an SDT tree, how nodes are named and addressed, and how HTTP carries the cost, determinism, and freshness of the command that produced a node.

The wash runtime already maps URLs to a project directory and resolves commands against a search path (runtime.md §6, §10). SDT already defines a static, ordinal-named directory layout that a reader can classify by inspection (sdt-spec.md §3). This spec is the thin layer that (a) fixes a node's on-disk file model over SDT, (b) defines a tree-producing command (TPC) and the conventions by which wash turns its output into an HTTP redirect, and (c) maps per-node cost/freshness onto HTTP cache semantics.

## 2. Goals, Non-goals, Terminology

### 2.1 Goals

1. A node is a directory of ordinary files, readable and writable by ordinary tools (log A).
2. Naming a node names three things: the node, its path to the root, and the subtree under it — with one addressing mechanism, not three (invariant 2; log E).
3. Keep *succession* and *alternatives* structurally distinguishable, with *continuation* a reader-level gloss layered on top (invariant 3; log B).
4. One POST conducts one turn: it writes the input node and the response subtree beneath it (log C).
5. Treat every command identically — a model call, `wc`, and an Eliza script are all just commands; cost, determinism, and freshness are per-command metadata, never a built-in distinction (invariant 6; log F, G).
6. Carry cost/determinism/freshness on HTTP itself: cache-control, `ETag`, `Last-Modified` (invariant 7; log D).
7. Stay Git-friendly: plain text, line-oriented, present-state-classifiable (inherited from both base specs).

### 2.2 Non-goals

1. **Budgets.** Out of scope; not modeled here (invariant 8).
2. **Addressable request/response perspective.** Pairing is never stored; request/response survive only as informal description, never as a stored type or an addressable view (invariant 4; log B/C).
3. **Cross-tree naming.** Out of scope (log E).
4. **Multi-writer coordination.** Out of scope; the contract is single-writer (log H).
5. **A template-vs-behavior distinction.** There is none; parameterization is extrinsic (log E).
6. **Replay.** There is no re-run verb and the vocabulary is never "replayable" (invariant 7; log D).

### 2.3 Terminology

- **Tree** — an SDT tree conducted by wash under this spec. Always "tree," never privileging "conversation."
- **Node** — an SDT node (a directory, sdt-spec.md §1) whose covered files are exactly `a` (text) and optionally `b` (metadata); see §4.
- **Succession** — the parent→child structural relation: depth in the SDT directory sequence. One child = a chain step (log B).
- **Alternatives** — sibling children of one node: breadth in the SDT directory sequence, ≥2 children, = explored branches (log B).
- **Continuation** — a *reader-level* gloss over a succession run whose nodes share an `author` (log B). Not structural.
- **Turn** — an input node plus the response subtree written beneath it by one POST (log C).
- **TPC (tree-producing command)** — a wash command that writes node files into the tree and emits a `Created Node <path>` manifest on stderr (log C, G).
- **Adapter** — a wash command that consumes a TPC's manifest and synthesizes the HTTP 303 + cache headers; the node-producer is HTTP-ignorant (log C).
- **Reviewable / re-doable / not-reproducible** — the vocabulary for a node's command character (invariant 7; §6). Never "replayable."

---

## 3. Base-spec deltas (extensions required)

The log forced out exactly six extensions (log G). Five are tree-specific; one is discipline, not a new mechanism. Each is stated as a delta: target base section, current behavior, required change, scope.

### 3.1 Delta 1 — the `a`/`b` node model over SDT

- **Target:** sdt-spec.md §3 (covered files), §4 (the `.0` sidecar).
- **Currently:** SDT covered files are bare ordinals with no assigned meaning ("Path components carry no semantic meaning," sdt-spec.md Abstract). Per-node metadata in SDT lives in the reserved `.0` sidecar (sdt-spec.md §4).
- **Required change:** This layer assigns meaning to the first two covered files of a node: `a` is the node's **text**, `b` is the node's **metadata** (line-oriented `key SP value`). The node's covered-file sequence is exactly `a`, `b` — never `c` or beyond (log A). All metadata, including `author`, is optional; missing metadata is uninterpreted, not an error (log A).
- **Scope:** Tree-specific. "`a` is text, `b` is metadata" is this layer's convention, not SDT's (log A).

> **Conflict flagged (log invariant 1 vs. sdt-spec.md §4).** Invariant 1 states there is **no provenance in an SDT `.0` sidecar** — the node's own directory is the only home for node metadata, and that metadata lives in `b`. SDT §4, however, reserves `.0` for recursive-rollup *statistics* (`total_files`, `total_bytes`, etc.), which are subtree bookkeeping, not node provenance. These do not actually collide: `b` carries this layer's per-node metadata; `.0` (if present at all) carries SDT's subtree stats and nothing about the node's content. The delta the base spec must absorb is only that **this layer never uses `.0` to carry node metadata** and never introduces a provenance sidecar; `b` is the sole home. SDT's optional `.0` stats remain permitted and untouched.

### 3.2 Delta 2 — the `Created Node` stderr manifest over wash

- **Target:** runtime.md §15.4 (stderr).
- **Currently:** runtime.md §15.4 says stderr is not merged into the response body by default, and a stage's stderr is merged into stdout only via a `/&` prefix or `stderr merge` metadata.
- **Required change:** A TPC writes node files to disk and emits one `Created Node <path>` **marker line** per node on **stderr**. The adapter reads these marker lines as a manifest; the **last** such line names the redirect leaf (log C). This is a deliberate, tree-specific overload of the §15.4 stderr channel.
- **Scope:** Tree-specific, and deliberately constrained: a TPC **MUST NOT** be composed with stderr-merge or `/&` (runtime.md §8.8, §15.4), because merging stderr into the body would corrupt the manifest channel (log G). The runtime does **not** sandbox or enforce the marker-line convention; it is convention only (log C).

### 3.3 Delta 3 — the adapter command

- **Target:** runtime.md §12.5 (command output; commands may return full HTTP responses) and §9.3 (POST).
- **Currently:** wash lets a command emit a full HTTP response including redirects and caching headers (runtime.md §12.5), but defines no standard command for turning a node-producer's side effects into a PRG redirect.
- **Required change:** Define an **adapter** command that consumes a TPC's stderr manifest, materializes nothing itself, and synthesizes a `303 See Other` to the leaf plus the cache headers of §6. The node-producer stays HTTP-ignorant (log C).
- **Scope:** Tree-specific.

### 3.4 Delta 4 — per-node freshness → cache-control, `no-cache` default

- **Target:** runtime.md §9.6–§9.7 (refresh/generated results) and §12.5 (caching headers).
- **Currently:** wash leaves caching to the command and requires no caching contract (runtime.md §9.7).
- **Required change:** A node's freshness is declared in the leaf's `b` metadata; the adapter reads it and sets `Cache-Control` accordingly (`no-cache` vs `immutable`). The **default is `no-cache`** — the safest and most ergonomic choice for a single developer on a single machine (log D).
- **Scope:** Tree-specific.

### 3.5 Delta 5 — `ETag` / `Last-Modified` mapping

- **Target:** runtime.md §12.5 (caching headers).
- **Currently:** wash permits but does not specify ETag/Last-Modified.
- **Required change:** `ETag` = a hash of the node's `a` file **only**. `Last-Modified` = the `b`.`created` value if present, else the filesystem mtime — advisory, not an identity validator (log D). See §6.2.
- **Scope:** Tree-specific.

### 3.6 Delta 6 — TPC POST-only / `mutates true`

- **Target:** runtime.md §9.5, §13.2 (method discipline; `mutates`).
- **Currently:** wash already requires a mutating command to declare a non-GET method and `mutates true`, and forbids `mutates true` together with GET (runtime.md §9.5, §13.2).
- **Required change:** **None** beyond honoring the existing rule. A TPC mutates the tree, so it is POST-only with `mutates true`. This is discipline within the base contract, not a new mechanism (log G).
- **Scope:** Tree-specific use of a general base rule.

---

## 4. The node model

### 4.1 A node is a directory of ordinary files

A node is an SDT node (sdt-spec.md §1): a directory whose covered entries are classified by present state. This layer fixes its covered **files** to be exactly:

- `a` — the node's **text** (the sole text payload of the node).
- `b` — the node's **metadata**, OPTIONAL, line-oriented `key SP value` (one entry per line).

The covered-file sequence of a node MUST be exactly `a`, or `a` then `b`; a covered file at `c` or beyond is outside this layer's node model (log A). A reader encountering a covered file `c+` in a node MAY treat the directory as a non-conformant node for this layer while it still classifies as a conformant SDT node.

All metadata is OPTIONAL, including `author`. Interpretation of metadata is reader-specific. Missing metadata is **uninterpreted, not an error** (log A). A node with only `a` and no `b` is a complete, conformant node.

> **OPEN:** the `b` metadata key set. The log fixes only three keys by use — `author` (§5), `created` (§6.2), and a freshness declaration (§6.1) — and states the grammar is line-oriented `key SP value`. It does not enumerate a closed key set, name the freshness key, or say how unknown keys are treated beyond "missing metadata is uninterpreted." Candidate resolutions: (a) an open key space where unknown keys are ignored; (b) a fixed v1 key list with reserved-prefix extension. Deferred for a later pass.

### 4.2 Subtree naming excludes node files

Covered **directories** of a node are its children (§7). Covered **files** `a`/`b` are the node's own content, not children. This matches SDT cleanly: files and directories are separate covered sequences (sdt-spec.md §3.5), so `a`/`b` never collide with child directories `0`, `1`, `A`, …

---

## 5. Structural relations

### 5.1 Two structural relations, one reader-level gloss

The tree carries **two** structural relations on disk and **one** relation that is read off them (invariant 3; log B):

- **Succession** — depth. A node with exactly **one** child directory continues the line by one step.
- **Alternatives** — breadth. A node with **≥2** child directories fans into explored branches.
- **Continuation** — a reader-level gloss over a succession run whose consecutive nodes share an `author` value in `b`. Continuation is **not** structural; it is computed from optional `author` metadata and is metadata-dependent (log B).

Succession and alternatives are distinguished purely by **child count in the SDT directory sequence** and are **metadata-independent**. The two layouts MUST stay distinguishable:

- `a → b → c` — a **chain**: depth. Each node has one child directory.
- `a → {b, c}` — a **fan**: breadth. One node has two child directories.

This distinction is exactly depth-vs-breadth in the SDT directory tree (sdt-spec.md §3) and survives with no metadata present (log B).

### 5.2 Pairing is never stored

Storage is bare succession. Request/response bracketing is a reader's interpretation, never a stored type and never an addressable view (invariant 4). This spec defines no node field, no directory convention, and no URL form that marks a node as "request" or "response." A reader MAY bracket nodes into turns for display; the tree does not record the bracketing (log B/C).

---

## 6. Freshness, review, redo

The character of the command that produced a node — its cost, determinism, and freshness — rides on HTTP, using the vocabulary **reviewable / re-doable / not-reproducible**, never "replayable" (invariant 7; log D).

### 6.1 Freshness → `Cache-Control`

Per-node freshness is declared in the **leaf's** `b` metadata. The adapter reads it and sets `Cache-Control`:

- A node whose output is fixed and re-fetchable identically → `immutable`.
- A node whose output should be re-derived rather than trusted from cache → `no-cache`.

The **default, when the leaf declares no freshness, is `no-cache`** — the safest and most ergonomic default for a single developer on a single machine, because it never silently serves a stale derivation (log D).

> **OPEN:** the exact `b` key and value vocabulary for the freshness declaration. The log fixes the default (`no-cache`) and the two target `Cache-Control` values (`no-cache`, `immutable`) but not the metadata key name or its permitted values. Candidate resolutions: (a) a single `freshness` key with values mapping to cache-control; (b) a `cache-control` key written through verbatim. Deferred.

### 6.2 `ETag` and `Last-Modified`

- **`ETag`** = a hash of the node's `a` file **only** (not `b`). Two nodes with identical text have identical ETags regardless of metadata (log D).
- **`Last-Modified`** = `b`.`created` if present, else the filesystem mtime of the node. It is **advisory**, not an identity validator: it is not used to decide content identity, and it degrades under Git for nodes that have no `b` (Git does not preserve mtime), so a `b`-less node loses a stable `Last-Modified` across a clone (log D).

> **OPEN:** the `ETag` hash algorithm and serialized form. The log fixes the input (the bytes of `a`, nothing else) but not the digest function or the header encoding (e.g. strong vs. weak ETag, hex vs. base64). Deferred.

### 6.3 Re-doing a node

There is **no re-run verb**. Re-doing is just POSTing again to the same node, which writes a **new sibling branch** — a new alternative under that node's parent branch point (§7.2). "Change one thing" means **change what you POST**; the tree records the new attempt as another alternative rather than mutating the old one (log D). This is why the tree is append-structured: a redo fans out, it does not overwrite.

---

## 7. Conducting a turn

A turn = an **input node plus the response subtree written beneath it** by one POST (log C). The request-target's only job is to name the branch point B (log B).

### 7.1 The two POST shapes

- **Body-POST** (`POST /<tpc>/<path>` with a request body): the body becomes a **new user node**, written as the next sibling under the URL-targeted node, and the response subtree is written beneath that new node (log C).
- **Existing-node-POST** (`POST /<tpc>/<path>/a`): the existing node addressed by `…/a` is used as the branch point directly; no duplicate user node is written (log C).

In both shapes the new node(s) are written at the **next dense sibling ordinal** under the branch point (log B). **Append is the leaf special case** (the branch point is a leaf, so the new line just extends it); **branch is the general case** (the branch point already has children, so the new node becomes an additional alternative) (log B).

### 7.2 One POST, zero-or-many response nodes

One POST normally writes **two** nodes — the user text and the command response — but **may write zero or many** response nodes (invariant 5; log C). A reader MUST NOT assume exactly two. The response subtree beneath the input node is whatever the TPC chose to materialize.

### 7.3 What the TPC does

The TPC writes node files into the tree and emits a `Created Node <path>` marker line on stderr per node (§3.2). The **last** such marker line is the redirect leaf. The runtime does not sandbox the TPC; the marker-line manifest is convention only (log C).

### 7.4 The three invocation modes

A TPC may be invoked in three modes (log C):

1. **TPC direct over HTTP** — non-PRG, **POST-only**, where stdout is treated as approximately the last node's content. This mode is **ergonomic-only**.
2. **Generic adapter** — the adapter (§3.3) consumes the manifest and produces a PRG redirect.
3. **Custom per-TPC wrapper** — a bespoke wrapper around one TPC that produces a PRG redirect.

Modes 2 and 3 are **PRG**: they serve the leaf's `a` **from disk** and **disregard the TPC's stdout** (log C). Mode 1 does not redirect and reads stdout directly.

A **nondeterministic** TPC **MUST** be invoked in mode 2 or 3. Mode 1 is unsafe for it because a browser refresh in mode 1 silently recomputes the command, producing a different result with no record (log C).

### 7.5 Read-after-write

In modes 2 and 3 the flow is **materialize then read**: the TPC writes node files, then the adapter (or wrapper) reads the leaf's `a` back **from the filesystem**, which is the source of truth (log C). The HTTP response body is the on-disk `a`, not the TPC's stdout.

### 7.6 The full PRG shape

The full request→redirect→view shape for modes 2/3:

```
POST /<tpc>/<path>          (or POST /<tpc>/<path>/a)
  → TPC writes node dirs, emits "Created Node <path>" lines on stderr
  → adapter reads manifest, last line = leaf
  → 303 See Other, Location: /<leaf-path>/a
     + Cache-Control (§6.1), ETag (§6.2), Last-Modified (§6.2)
GET /<leaf-path>/a
  → serves the leaf's a from disk (ordinary wash file serving, runtime.md §6.1)
```

---

## 8. Naming and addressing

### 8.1 The core names a path

Core addressing names a **path** to a node directory, with `/…/a` and `/…/b` reaching the node's two files (log E). A name resolves to a node; the node's **path to the root** and its **transitive subtree** are the other two things the name denotes (invariant 2), but they are **command-computed views**, not separate addressing mechanisms (log E). There is one address — the path — and two projections of it.

> **OPEN:** the command names and output contracts for the path-to-root view and the subtree view. The log establishes that both are command-computed views over a node path (not core addressing) but does not name the commands or fix their output. Candidate resolutions: bespoke wash commands (e.g. a `root` view and a `subtree` view) defined in a later pass. Deferred.

### 8.2 Bare node-directory GET

A GET of a bare node directory (no `/a` or `/b` suffix) receives **ordinary wash directory treatment** (runtime.md §6.5): index file if present, else a directory listing, else 404. There is **no SDT-specific treatment** of a bare node-directory GET (log E).

### 8.3 No template-vs-behavior distinction

There is no template-vs-behavior distinction in the tree. Parameterization is **extrinsic** — done by an outside tool such as `sed` today, or a future custom substitution command. The nameable reusable unit is a **path/command**, consistent with the base spec's position that reusable URL fragments are modeled as commands (runtime.md §14.1; invariant 6 / "the LLM is not special"; log E).

> **OPEN:** a custom parameter-substitution command better than `sed`. The log explicitly **defers** this — it is *not* a non-goal, but an intended future tool whose contract is unspecified. Carried forward as a Future Extension (§13), not resolved here.

### 8.4 Cross-tree naming

Out of scope (log E; §2.2).

---

## 9. The LLM is not special

There is no prompt-vs-script notion anywhere in the runtime. A model call is a command that shells out; an Eliza command is equally first-class. `wc` and a GPT-calling script are treated **identically** (invariant 6; log F).

Cost, nondeterminism, and freshness are **per-command metadata properties** carried on HTTP (§6), not built-in attributes of a privileged "LLM" path. A node is **one shared node, read by many**: reads never copy it, and only a TPC's POST writes — and what it writes is **its response, not a copy of the input** (log F). A reader fanning out from a node does not duplicate it.

---

## 10. Failure and boundaries

### 10.1 TPC nonzero exit

If a TPC exits nonzero, the adapter returns an **error and no redirect**. Any **partial writes remain** on disk (log H). This is consistent with wash's default mapping of nonzero command exit to a client error (runtime.md §15.3), except that the adapter, owning the HTTP response, surfaces the error and withholds the 303.

> **OPEN:** the exact HTTP status the adapter returns on TPC nonzero exit. The log fixes "error, no redirect, partial writes remain" but not the status code. wash's default for nonzero exit is `400` (runtime.md §15.3); whether the adapter reuses `400` or distinguishes a TPC failure (e.g. `500`) is unspecified. Deferred.

### 10.2 Crashed TPC

A crashed TPC leaves a **conformant SDT tree with a short subtree** — there is **no corruption class**. Recovery is to **POST again**; there is **no rollback, repair, or cleanup** (log H). This follows directly from SDT's present-state semantics (sdt-spec.md §4.1, Appendix B.3): a partially written subtree still classifies, it is merely shorter than intended.

### 10.3 Single-writer; internal parallelism allowed

Multi-writer coordination is **out of scope**. The single-writer contract is **independent POSTs** (log H). One TPC **MAY** parallelize internally — that is still one POST and one writer. A TPC that races **itself** is a TPC bug, not a supported mode. Automated conducting (invariant 6) is fine **under a single serial driver** (log H).

### 10.4 Density is intended but unenforced

Dense sibling-ordinal creation (§7.1) is **intended but unenforced**, preserving SDT's posture that density is a property a reader checks, not a rule imposed on a writer (sdt-spec.md G6, Appendix B.1). TPCs **SHOULD** use SDT tools to allocate the next dense ordinal. Gaps are **tolerated** and are **invisible to path access** — a present node at a sparse ordinal is reached by its path regardless of gaps below it (log H).

---

## 11. Examples

All examples show real SDT directories and ordinals. Recall: covered child directories use bijective base-36 (`0`,`1`,…,`9`,`A`,…), and a letter-bearing directory is stored `_`-prefixed on disk but bare in URLs (sdt-spec.md §3.7, §6.4).

### 11.1 A two-node turn, end to end

Tree before — a single root-level node `0` with text only:

```
root/
  0/
    a            ← "Hello."
```

Request (body-POST, generic adapter):

```
POST /tree/0
Content-Type: text/plain

What is 2+2?
```

The TPC writes the user node as the next dense sibling under `0`'s branch point, then the response beneath it, emitting markers on stderr:

```
Created Node 0/0          (stderr)
Created Node 0/0/0        (stderr)   ← last line = redirect leaf
```

Tree after:

```
root/
  0/
    a            ← "Hello."
    0/
      a          ← "What is 2+2?"
      0/
        a        ← "4"
        b        ← created 2026-06-19T14:00:00Z
```

Adapter response:

```
303 See Other
Location: /tree-view/0/0/0/a
Cache-Control: no-cache
ETag: "<hash of 0/0/0/a>"
Last-Modified: Thu, 19 Jun 2026 14:00:00 GMT
```

Follow-up GET serves the leaf's `a` from disk (ordinary wash, runtime.md §6.1):

```
GET /tree-view/0/0/0/a   →   200 OK,  body: "4"
```

### 11.2 Continuation vs. alternatives, side by side

**Continuation** — a chain (depth), one child per node, here glossed as continuation because all three share `author user`:

```
root/
  0/
    a    ← "part one"
    b    ← author user
    0/
      a  ← "part two"
      b  ← author user
      0/
        a  ← "part three"
        b  ← author user
```

This is `0 → 0/0 → 0/0/0`: succession depth 3, one child each. The shared `author` makes a reader call it a continuation; structurally it is plain succession (§5.1).

**Alternatives** — a fan (breadth), one node with three child directories:

```
root/
  0/
    a    ← "prompt"
    0/
      a  ← "answer, attempt 1"
    1/
      a  ← "answer, attempt 2"
    2/
      a  ← "answer, attempt 3"
```

This is `0 → {0, 1, 2}`: three explored alternatives, distinguished from the chain purely by child count (§5.1), with no metadata required. A redo (§6.3) POSTed to node `0` would write child `3` here — another alternative.

### 11.3 A nondeterministic-command node

A node produced by a model call (a nondeterministic TPC, so invoked in mode 2/3, §7.4). Its leaf `b` declares `no-cache` freshness; the directory holds only ordinary files:

```
root/
  0/
    a    ← "draw me a haiku about gulls"
    0/
      a  ← "<the generated text>"
      b  ← author model
           created 2026-06-19T14:05:00Z
           freshness no-cache
```

(`freshness no-cache` shown per the §6.1 OPEN; the key name is not yet fixed.)

The GET of the leaf carries:

```
GET /tree-view/0/0/a
200 OK
Cache-Control: no-cache
ETag: "<hash of 0/0/a — text only, not b>"
Last-Modified: Thu, 19 Jun 2026 14:05:00 GMT
```

`no-cache` ensures a refresh re-validates rather than silently serving a stale generation; because the TPC ran in mode 2/3, the body is the on-disk `a`, and re-deriving means POSTing again (a new alternative, §6.3), never a transparent recompute.

---

## 12. Resolved Design Decisions

Compressed from the log; each cross-referenced to its section.

- Node = directory of ordinary files; `a` = text, `b` = metadata (`key SP value`); covered files are exactly `a`,`b`, never `c`+ (§4.1; delta §3.1).
- All metadata optional, including `author`; missing metadata uninterpreted, not an error (§4.1).
- Two structural relations — succession (depth, one child) and alternatives (breadth, ≥2); continuation is a reader-level `author` gloss on succession, not structural (§5.1).
- Chain vs. fan must stay distinguishable as depth vs. breadth, metadata-independent (§5.1).
- A turn = input node + response subtree beneath it; body-POST writes a new user node, existing-node-POST (`…/a`) reuses the existing node, no duplicate (§7.1).
- POST writes at the next dense sibling ordinal; append = leaf special case, branch = general case; request-target only names the branch point (§7.1).
- One POST normally writes two nodes but may write zero or many (§7.2; invariant 5).
- TPC writes node files + `Created Node <path>` stderr markers; last marker = redirect leaf; runtime does not sandbox (§7.3; delta §3.2).
- Adapter consumes manifest, synthesizes 303 + cache headers; node-producer HTTP-ignorant (§7.3; delta §3.3).
- Three invocation modes; modes 2/3 are PRG, serve leaf `a` from disk, disregard stdout; mode 1 ergonomic-only; nondeterministic TPC MUST be mode 2/3 (§7.4).
- Read-after-write: modes 2/3 materialize then read; filesystem is source of truth (§7.5).
- Freshness declared in leaf `b`; default `no-cache` (§6.1; delta §3.4).
- `ETag` = hash of `a` only; `Last-Modified` = `b`.`created` else fs mtime, advisory, dies under Git when `b`-less (§6.2; delta §3.5).
- Redo = POST again → new sibling branch; no re-run verb; "change one thing" = change what you POST (§6.3).
- Core names a path (`/…/a`, `/…/b`); path-to-root and subtree are command-computed views (§8.1); bare node-dir GET = ordinary wash §6.5 (§8.2).
- No template-vs-behavior distinction; parameterization extrinsic; reusable unit = path/command (§8.3).
- The LLM is not special; one shared node read by many; reads never copy; only TPC POST writes its response, not a copy of input (§9; invariant 6; log F).
- TPC is POST-only with `mutates true` — discipline within the base contract, not a new mechanism (§3.6).
- TPC nonzero exit → error, no redirect, partial writes remain (§10.1); crashed TPC → conformant short subtree, recovery = POST again, no rollback (§10.2).
- Single-writer = independent POSTs; one TPC may parallelize internally; automated conducting OK under a single serial driver (§10.3).
- Dense creation intended-but-unenforced; gaps tolerated and path-access-invisible (§10.4).

## 13. Minimal Viable Implementation

1. The `a`/`b` node model over an SDT tree (§4).
2. At least one TPC that writes node dirs and emits `Created Node <path>` stderr markers (§7.3).
3. The generic adapter: manifest → 303 to leaf `/…/a`, serving `a` from disk (§3.3, §7.4 mode 2).
4. `Cache-Control` from leaf `b` freshness, default `no-cache` (§6.1).
5. `ETag` from `a`, `Last-Modified` from `b`.created/mtime (§6.2).
6. Body-POST and existing-node-POST shapes writing at the next dense sibling ordinal (§7.1).
7. Bare node-dir GET via ordinary wash directory behavior (§8.2).
8. Single serial driver for any automated conducting (§10.3).

## 14. Non-goals (restated)

Budgets (invariant 8); addressable request/response perspective / stored pairing (invariant 4, §5.2); cross-tree naming (§8.4); multi-writer coordination (§10.3); template-vs-behavior distinction (§8.3); a replay verb or "replayable" vocabulary (§6).

## 15. Future Extensions

1. A custom parameter-substitution command, better than `sed` (deferred in the log, **not** a non-goal; §8.3).
2. Named command/output contracts for the path-to-root and subtree views (§8.1 OPEN).
3. A fixed `b` metadata key set and freshness vocabulary (§4.1, §6.1 OPENs).
4. A pinned `ETag` algorithm and header form (§6.2 OPEN).
5. A distinguished HTTP status for TPC failure vs. ordinary bad request (§10.1 OPEN).

---

# Coverage report

### (a) Log DECISIONs not expressed in the spec

None omitted. Every DECISION in sections A–H maps to a home:

- A → §4.1, delta §3.1.
- B → §5, §7.1.
- C → §7, deltas §3.2/§3.3.
- D → §6.
- E → §8.
- F → §9.
- G → §3 (all six deltas) and §3.6.
- H → §10.

One log item is carried as a Future Extension rather than a settled decision, per the log's own framing: the OPEN custom parameter-substitution command (deferred, explicitly *not* a non-goal) → §8.3 OPEN + §15.1.

### (b) `> OPEN:` callouts emitted

1. §4.1 — the `b` metadata key set (closed list vs. open space; unknown-key handling).
2. §6.1 — the freshness `b` key name and value vocabulary.
3. §6.2 — the `ETag` hash algorithm and header form (strong/weak, hex/base64).
4. §8.1 — command names and output contracts for the path-to-root and subtree views.
5. §8.3 — the custom parameter-substitution command (carried from the log's sole OPEN; deferred, not a non-goal).
6. §10.1 — the HTTP status the adapter returns on TPC nonzero exit.

### (c) Base-spec deltas

1. §3.1 — `a`/`b` node model over SDT §3/§4 (tree-specific). Includes the flagged invariant-1-vs-SDT-§4 conflict: this layer never uses `.0` for node metadata; `b` is the sole home; SDT's `.0` stats remain permitted.
2. §3.2 — `Created Node` stderr manifest over wash §15.4 (tree-specific; TPCs MUST NOT be composed with stderr-merge/`/&`).
3. §3.3 — the adapter command over wash §12.5/§9.3 (tree-specific).
4. §3.4 — per-node freshness → `Cache-Control`, `no-cache` default, over wash §9.6–§9.7/§12.5 (tree-specific).
5. §3.5 — `ETag`/`Last-Modified` mapping over wash §12.5 (tree-specific).
6. §3.6 — TPC POST-only/`mutates true` over wash §9.5/§13.2 (discipline, not an extension; tree-specific use of a general rule).