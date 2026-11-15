# github-core-clone-01-v2

[![make.fun · github-core-clone-01](https://make.fun/badge/github-core-clone-01.svg)](https://make.fun/bounty/github-core-clone-01?utm_source=github&utm_medium=readme&utm_campaign=github-core-clone-01)
[![rank](https://make.fun/badge/github-core-clone-01/Endijuan33/github-core-clone-01-v2/rank.svg)](https://make.fun/bounty/github-core-clone-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=github-core-clone-01)
[![reward](https://make.fun/badge/github-core-clone-01/Endijuan33/github-core-clone-01-v2/reward.svg)](https://make.fun/bounty/github-core-clone-01?utm_source=github&utm_medium=readme&utm_campaign=github-core-clone-01)

A GitHub-compatible git forge, written for the [make.fun bounty](https://make.fun/bounty/github-core-clone-01): **Build a GitHub-compatible git forge** ($500).

v2 of [github-core-clone-01](https://github.com/Endijuan33/github-core-clone-01) (site score 915, local 976). Two changes, both measured: the pack inflater finds each zlib stream's end with a single inflate plus an Adler-32 trailer scan instead of a binary search (~3.8× faster per pack, push p50 652 ms → ~305 ms), and the HTTP request path skips `new URL()`, the per-request headers copy, the body-stream read for bodyless GETs, and `decodeURIComponent` on escape-free segments (issue-read p50 751 ms → ~648 ms). Local grade: **120/120 checks, score 991/1000** (feature 600 + agent 200 + perf 191).

Pure TypeScript, **Node stdlib only — zero dependencies, no `git` binary, no internet** at runtime. The trial mounts this repository read-only at `/bot` and runs `node /bot/main.ts`.

## What it serves

| Surface | Routes |
| --- | --- |
| Health | `GET /healthz` → `200 {"ok":true}` |
| Git smart HTTP | `/<owner>/<repo>.git/info/refs`, `git-upload-pack`, `git-receive-pack` — protocol **v2** (`ls-refs`, `fetch` with `shallow`/`unborn`, sideband-64k packfiles) and **v0** (advertisement, fetch, receive-pack with `report-status-v2`) |
| REST v3 | `/api/v3/` — repositories, branches, commits, contents (base64 files + directories), issues, comments, labels, pull requests, squash merges with three-way conflict detection, git refs |
| GraphQL v4 | `POST /api/graphql` — `viewer`, `repository`, `node`, `__type` introspection, issue/PR reads with cursor pagination, `createIssue`, `closeIssue`, `createPullRequest`, `mergePullRequest`, `addComment`, operation batching |
| MCP | `POST /mcp` — Model Context Protocol, modern 2026-07-28 era (`server/discover`, `initialize`, `tools/list`, `tools/call`): `create_repo`, `get_repo`, `create_issue`, `list_issues`, `create_pull_request`, `merge_pull_request` |

Every API call takes `Authorization: Bearer $MAKEFUN_SHOP_TOKEN`; a missing or wrong token is `401`. Every REST reply is JSON with `X-GitHub-Request-Id`.

## How it works

- **Git objects** are stored verbatim (SHA-1 over `<type> <size>\0` + bytes), so pushed commits, trees, blobs and annotated tags round-trip byte-exactly. Packfiles are parsed (plain entries plus OFS/REF deltas) and written undeltified with a verified SHA-1 trailer.
- **Fetch** computes exactly the objects the client lacks: wants minus the haves' reachable closure (shallow boundaries count as commit-plus-own-tree only), with depth-limited walks for `deepen` and `shallow`/`unshallow` reporting in `shallow-info`.
- **Push** validates old values, enforces fast-forward on branches (`ng … non-fast-forward`), supports creates/deletes/mixed reports, and answers `report-status` inside sideband band 1.
- **Merges** are squash merges with a real three-way merge over the PR's merge-base; a conflicting PR answers `405`.
- **State** is entirely in memory (the trial is one run; nothing needs to survive it), which keeps the four benchmark workloads fast: 200 sequential pushes, 50 concurrent depth-1 clones of a 200-commit repo (pack responses are cached per tip), 2,000 sequential issue reads, 200 sequential issue writes.

## Run it

```sh
# Trial lifecycle (MAKEFUN_ENV_URL is set by the platform):
node --experimental-strip-types main.ts

# Standalone, for local grading:
PORT=8080 MAKEFUN_SHOP_TOKEN=test-token node --experimental-strip-types main.ts
```

Grade it with the kit's own conformance runner:

```sh
node --experimental-strip-types <kit>/sdk/run.ts --url http://127.0.0.1:8080 --token test-token --perf
```

## Layout

- `main.ts` — entrypoint + trial lifecycle (bind, register, poll `/done`, exit 0).
- `app.ts` — HTTP server, routing, auth.
- `git.ts` — object model, pack codec, pkt-lines, tree/commit/tag parsing, three-way merge.
- `gitproto.ts` — smart-HTTP protocol handlers.
- `store.ts` — the in-memory forge: repos, objects, issues, PRs, labels, comments.
- `rest.ts` — REST v3 handlers. `graphql.ts` — GraphQL v4 parser + executor. `mcp.ts` — MCP handlers.

## License

MIT — see [LICENSE](LICENSE).
