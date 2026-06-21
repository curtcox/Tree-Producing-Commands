# TPC commands

Reference scaffolds for the three pieces of [TPC-spec.md](../TPC-spec.md), one
per file, best-fit language per component (spec-faithful "ordinary tools"):

| Command  | Language        | Spec role |
|----------|-----------------|-----------|
| `tpc`    | bash            | Tree-Producing Command skeleton — writes the input node + response subtree, emits the `Created Node <path>` stderr manifest (§3.2, §7, §10.4). |
| `adapter`| Python (stdlib) | Generic adapter — consumes the manifest, reads the leaf from disk, synthesizes `303` + cache headers (§3.3, §6, §7.4–7.6). |
| `subst`  | Python (stdlib) | Extrinsic parameter substitution, "better than `sed`" (§8.3, §15.1). |

No third-party dependencies; no compiled toolchain.

`env/meta/{tpc,adapter,subst}` are the wash command-metadata files
(pipeline_parsing.md §5; runtime.md §7.3). Copy them under the served tree
root's `env/meta/` directory.

## Quick end-to-end (matches §11.1)

```sh
mkdir -p root/0 && printf 'Hello.' > root/0/a

# wash-native shape: cwd = tree root, branch path as argv, TPC from $TPC,
# with `rev` standing in as "the command" (§9)
( cd root && printf 'What is 2+2?' | TPC=../tpc TPC_CMD=rev \
    ../adapter --view-prefix /tree-view 0 )
# → 303 See Other, Location: /tree-view/0/0/0/a, Cache-Control: no-cache, ETag, Last-Modified

# explicit/testing shape also works:
printf 'What is 2+2?' | TPC_CMD=rev \
  ./adapter --tree-root ./root --view-prefix /tree-view -- ./tpc --root ./root 0
```

## Contracts (confirmed against the base specs)

- **Command invocation** (runtime §10.2/§10.6/§12.3/§12.4, pipeline_parsing §5):
  cwd = configured tree root; URL suffix → positional argv (`arity *`); POST
  body → stdin (`input stdin`); a command may emit a full HTTP response on
  stdout (§12.5). Root overridable via `--root`/`--tree-root`/`$WASH_ROOT`.
- **SDT ordinals** (sdt-spec §3.2/§3.3/§3.7/§6.4): covered child dirs use
  **bijective** base-36 over `0..9A..Z` (1-based: `0,1,…,9,A,…,Z,00,01,…`,
  `divmod(n-1,36)`); next dense sibling = ordinal `child_count + 1`; a name
  containing any `A..Z` is stored `_`-prefixed on disk, bare in URLs.
  Centralized in `tpc`:`ordinal_name`/`disk_name`/`url_to_disk` and
  `adapter`:`url_to_disk`.
- **Metadata** (pipeline_parsing §5): line-oriented `field value`. `tpc` and
  `adapter` are `methods POST` / `mutates true` / `stderr discard` (never
  `merge`, per §3.2); `subst` is `mutates false`, GET-able.

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
