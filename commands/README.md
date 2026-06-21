# TPC commands

Reference scaffolds for the three pieces of [TPC-spec.md](../TPC-spec.md), one
per file, best-fit language per component (spec-faithful "ordinary tools"):

| Command  | Language        | Spec role |
|----------|-----------------|-----------|
| `tpc`    | bash            | Tree-Producing Command skeleton — writes the input node + response subtree, emits the `Created Node <path>` stderr manifest (§3.2, §7, §10.4). |
| `adapter`| Python (stdlib) | Generic adapter — consumes the manifest, reads the leaf from disk, synthesizes `303` + cache headers (§3.3, §6, §7.4–7.6). |
| `subst`  | Python (stdlib) | Extrinsic parameter substitution, "better than `sed`" (§8.3, §15.1). |

No third-party dependencies; no compiled toolchain.

## Quick end-to-end (matches §11.1)

```sh
mkdir -p root/0 && printf 'Hello.' > root/0/a

# body-POST through the adapter, with `rev` standing in as "the command" (§9)
printf 'What is 2+2?' | TPC_CMD=rev \
  ./adapter --tree-root ./root --view-prefix /tree-view -- ./tpc ./root 0
# → 303 See Other, Location: /tree-view/0/0/0/a, Cache-Control: no-cache, ETag, Last-Modified
```

## Assumptions to reconcile with the base specs

`runtime.md` and `sdt-spec.md` are referenced by the spec but not present in
this repo. Where their contract is needed, these scaffolds make an explicit,
centralized assumption (so it is one edit to align):

- **Command invocation** (`tpc`/`adapter` headers): path via argv, POST body on
  stdin, full HTTP response on stdout (wash §12.5). Confirm against `runtime.md`.
- **SDT ordinals** (`tpc`: `ordinal_name`/`disk_name`; `adapter`: `url_to_disk`):
  base-36 names `0..9,A..Z,10,…`; letter-bearing names `_`-prefixed on disk,
  bare in URLs; "dense" next index == child count. Confirm against `sdt-spec.md`
  §3.7/§6.4 (esp. whether ordinals are *truly* bijective base-36).

## Spec OPENs resolved here (all easy to change)

- **§6.1 freshness key** → key `freshness`, values `immutable` | `no-cache`
  (`adapter`: `FRESHNESS_KEY`; `tpc`: `TPC_FRESHNESS`).
- **§6.2 ETag** → strong ETag, sha256 of `a` bytes, lowercase hex
  (`adapter`: `compute_etag`).
- **§10.1 failure status** → default `400` (wash's nonzero default), overridable
  via `adapter --error-status`.
- **§15.1 subst contract** → a v0 proposal (`{{NAME}}` placeholders, literal
  values, strict-by-default, `--escape none|json|shell`). See `subst` header.

## Notes

- `tpc` MUST NOT be composed with stderr-merge or `/&` — that would corrupt the
  manifest channel (§3.2).
- A nondeterministic TPC MUST run via the adapter (mode 2) or a custom wrapper
  (mode 3), never direct mode 1 (§7.4).
- `TPC_CMD` is the content-producing command ("the LLM is not special", §9):
  defaults to `cat` (an echo TPC); swap in a model call, `wc`, Eliza, etc.
