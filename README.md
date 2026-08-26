# app-warehouse

**A dormant WMS edge surface.** This repository holds the Cloudflare Worker that
was to serve `warehouse.etzhayyim.com` — a warehouse-management layer exposing
four XRPC methods (`registerSku`, `putaway`, `pick`, `getInventory`) as a thin
edge in front of business logic hosted elsewhere. It was extracted verbatim from
`etzhayyim/root:60-apps/etzhayyim-project-warehouse`.

**Every hostname it speaks to no longer exists in DNS** — the two routes it
claims, and both of the upstreams its two dispatchers proxy to. The
`etzhayyim.com` zone itself is healthy, so this is four missing labels, not an
outage.

It is not, however, rotted. **It still builds clean from a cold checkout**
(measured 2026-08-17: `npm install` + `npm run build` in `svelte/`, 92 packages,
no errors, a 4,335-byte Worker). The distinction matters: this is dormant code
with its live dependencies gone, not code that has decayed past use.

> **Reading this repo to learn how the warehouse surface works?** Read
> [Two dispatchers, one deploys](#two-dispatchers-only-one-deploys) first. The
> file that looks like the implementation is not the file that ships.

Every claim below is reproducible from
[`docs/operator-quickstart.md`](docs/operator-quickstart.md), measured
2026-08-17 — **except where marked below**: this section, the two-dispatchers
table, and the provenance section were overtaken by the frontend migration
described next.

## Frontend migrated to ClojureScript (2026-08-26)

The `svelte/` directory (SvelteKit) is gone. The frontend is now
ClojureScript — reagent + re-frame + `jp-go-dds` (デジタル庁デザインシステム) —
at [`cljs/`](../cljs). This was a **frontend-only** migration; the backend
Worker/XRPC logic was moved, not rewritten:

| Then | Now |
|---|---|
| `svelte/src/routes/+page.svelte` (the status page below documents) | [`cljs/src/warehouse/app.cljs`](../cljs/src/warehouse/app.cljs) — same seven facts + own path, faithfully ported |
| `svelte/src/routes/xrpc/[...path]/+server.ts` (**the file that deployed**, per the table below) | [`src/xrpc-dispatcher.ts`](../src/xrpc-dispatcher.ts) — moved byte-for-byte, only a provenance header comment added |
| `wrangler.jsonc` `main: svelte/.svelte-kit/cloudflare/_worker.js` | `main` dropped entirely |
| `wrangler.jsonc` `assets.directory: ./svelte/.svelte-kit/cloudflare/client` | `assets.directory: ./cljs/public` |

**This changes the "two dispatchers, only one deploys" story below to "two
dispatchers, neither deploys."** Neither `src/app.ts` nor `src/xrpc-dispatcher.ts`
calls `env.ASSETS.fetch`, so `main` was dropped rather than repointed at either
— putting either Worker in front of the static assets with no `env.ASSETS.fetch`
call would mean nothing serves the frontend. Both backend files are now
equally orphaned source, same status `src/app.ts` already had. This is
**unverified**: `wrangler deploy`/`wrangler dev` were not run.

Everything from here to "Where the warehouse domain lives" is the
**2026-08-17, pre-migration snapshot** — kept for its investigative value
(the DNS/provenance measurements it reports are still accurate as historical
record), but any path under `svelte/` it names no longer exists.

## What changed underneath it

| What this repo assumes | What is true now |
|---|---|
| `wrangler.jsonc` routes `warehouse.etzhayyim.com/*` and `w4r3h0u5.etzhayyim.com/*` | Both are **NXDOMAIN**. No record at all, not a stale route. |
| `src/app.ts` proxies to `https://dispatcher.etzhayyim.com` | **NXDOMAIN.** |
| `svelte/src/routes/xrpc/[...path]/+server.ts` proxies to `https://mcp.etzhayyim.com/xrpc/com.etzhayyim.mcp.message` | **NXDOMAIN.** |
| `kotodama.jsonld` declares the actor `did:web:warehouse.etzhayyim.com` | A `did:web` DID resolves over HTTPS at that host. The host does not resolve, so **the DID is unresolvable**. |
| `migration.edn` extracted from `etzhayyim/root:60-apps/etzhayyim-project-warehouse` | `etzhayyim/root@main:60-apps` now contains exactly one entry, `etzhayyim-project-organism`. **The source path is gone from the default branch.** |

The zone is served by Cloudflare (`vivienne`/`everton.ns.cloudflare.com`) and the
apex answers `200`. Nothing is broken at etzhayyim — these four labels were
removed or never created.

## Two dispatchers, only one deploys

This repository contains **two independent XRPC implementations that disagree
with each other**, and the more readable one is dead code.

| | `src/app.ts` | `svelte/src/routes/xrpc/[...path]/+server.ts` |
|---|---|---|
| Referenced by anything? | **No.** `git grep src/app` matches nothing. | Yes — compiled into the deploy artifact. |
| Deployed? | **Never.** | **Yes.** `wrangler.jsonc` `main` is `svelte/.svelte-kit/cloudflare/_worker.js`. |
| Upstream | `dispatcher.etzhayyim.com` | `mcp.etzhayyim.com` |
| Wire format | plain JSON body | **JSON-RPC 2.0** `tools/call` envelope |
| Auth | `x-internal-secret` header | none |
| Method filter | only `com.etzhayyim.apps.warehouse.*`, else `404` | **any** NSID is forwarded |
| Health endpoint | `/health`, `/_app/meta` | none |

`src/app.ts` is the file a reader opens: it carries the explanatory header
comment, the `ACTOR_DID`, and the four-method list. It is also the file that has
never run. It typechecks clean when pointed at directly, so it is orphaned
rather than broken — but **nothing in the repository references it**, and the
behaviour it describes (secret-authenticated, prefix-filtered, health-checked)
is not the behaviour that would deploy. (Whether it compiles at all depends on
which TypeScript you point at it — see below.)

Treat `src/app.ts` as a design sketch, and the SvelteKit route as the contract.

## The declared check does not check anything

`package.json` exposes one script, and it cannot run:

```
$ npm run typecheck        # tsc --noEmit
tsc: The TypeScript Compiler - Version <whatever resolves>
        (usage text)
EXIT=1
```

There is no `tsconfig.json` at the repository root, so `tsc` prints usage and
exits non-zero without examining a single file. **`src/app.ts` is typechecked by
nothing.** The SvelteKit half has its own working `svelte/tsconfig.json` and its
own `check` script, which is the one that means something.

### …and the two halves disagree about TypeScript

The two `package.json` files declare **different TypeScript majors** — root
`^6.0.2`, `svelte/` `^5.9.3` — and the orphaned dispatcher only compiles under
one of them:

| Compiler | `src/app.ts` |
|---|---|
| `6.0.3` (root's declared range) | clean, exit `0` |
| `5.9.3` (svelte's declared range) | **exit `2`** — `TS2339: Property 'entries' does not exist on type 'URLSearchParams'` |

So "does `src/app.ts` typecheck?" has no answer until you say which compiler.
There are also **no lockfiles anywhere** (`package-lock.json`, `pnpm-lock.yaml`,
`yarn.lock` all absent from the tracked tree), so which one you get depends on
where you run it from and what the registry serves that day. The 2026-08-17
build is evidence the code is sound, not evidence the build is reproducible.

## What is actually in here

Twelve tracked files, 13,282 bytes, extracted verbatim from
`etzhayyim/root@c9e7df4b:60-apps/etzhayyim-project-warehouse`, **as of
2026-08-17, before the 2026-08-26 frontend migration**. The table below is
that original snapshot; the "Now" column says where each moved.

| Path (2026-08-17) | Role | Now (2026-08-26) |
|---|---|---|
| `src/app.ts` | Thin-edge dispatcher, 4 methods. **Unreferenced — see above.** | Unchanged, still unreferenced. |
| `svelte/src/routes/xrpc/[...path]/+server.ts` | The handler that actually deployed. | Moved verbatim to `src/xrpc-dispatcher.ts` — no longer deployed either (`wrangler.jsonc` `main` was dropped). |
| `svelte/src/routes/+page.svelte` | Placeholder status page. Its embedded metadata says `routeCount: 0`, `routes: []`. | Ported to `cljs/src/warehouse/app.cljs` (reagent + re-frame + jp-go-dds). |
| `wrangler.jsonc` | Worker config, routes, and the app's `vars` (capabilities, display name). | Same file, `main`/`assets.directory`/`APP_FRAMEWORK` updated — see "Frontend migrated" above. |
| `kotodama.jsonld` | Actor descriptor — DID, system prompt, capabilities, governance `raci: responsible`. | Unchanged. |
| `NOTICE` | Apache-2.0 + etzhayyim Charter Rider v3.1. | Unchanged. |

`README.edn` and `migration.edn` are machine-readable records added by the
extraction tooling; they are the two entries in `migration.edn`'s
`:allowed-additions` and are not part of the twelve.

Two loose ends worth knowing: `NOTICE` points at `CHARTER-RIDER.md`, which **is
not in this repository**; and `wrangler.jsonc` sets
`assets.not_found_handling: "none"` while declaring an `ASSETS` binding, so the
static surface has no SPA fallback.

### Provenance is intact

Verified byte-for-byte on 2026-08-17, **before** the 2026-08-26 frontend
migration added `cljs/` and moved the XRPC handler to `src/xrpc-dispatcher.ts`.
`migration.edn` declares `:tree f19958a84d…`, `:tracked-files 12`,
`:bytes 13282`; the real upstream tree sha for that path at that revision is
`f19958a84d2822f0a320fc137485bcf0b0c50206`, and the twelve files (as they
stood on 2026-08-17) totalled exactly 13,282 bytes. The one subsequent commit
(`db2531e`, "record cloud-itonami ownership") touched only `README.edn` and
`migration.edn` — the two permitted additions. **Nothing drifted before the
frontend migration.**

That said: `migration.edn`'s `:allowed-additions` was never an enforced
allowlist — no checker in this repository pins `svelte/` (or anything else)
by hash against it (confirmed again while doing the 2026-08-26 migration; see
"What to do with it" for the standing options this repo has always had, one
of which — re-home or retire — would have changed the tree regardless). The
twelve 2026-08-17 files are still individually traceable to the upstream
commit above; the frontend migration adds new files on top rather than
altering any of those twelve at the byte level (`src/app.ts`, `wrangler.jsonc`'s
non-frontend fields, `kotodama.jsonld`, `NOTICE`, `README.edn`, `migration.edn`
are untouched; `svelte/src/routes/xrpc/[...path]/+server.ts` moved with only a
provenance comment added; `svelte/src/routes/+page.svelte` and the rest of
`svelte/` were replaced by the `cljs/` port).

`etzhayyim/com-etzhayyim-app-warehouse` is a **GitHub redirect to this
repository**, not a second copy. This repo is registered in `manifest/west.yml`
as `app-warehouse`, pinned at `3eac6d18`.

## Where the warehouse domain lives

The reusable domain logic exists, and it is not here:
[`kotoba-lang/soko`](https://github.com/kotoba-lang/soko) (倉庫) is the shared
warehouse / logistics library — multi-location stock, putaway/reserve/transfer,
pick-list and fulfillment planning, shipment statechart. Portable `.cljc`, zero
host effects. It covers all four methods this Worker exposes.

`soko` is a **library, not a deployment**. There is no live WMS actor anywhere in
the fleet: `warehouse.itonami.cloud` and `wms.itonami.cloud` are both NXDOMAIN
(the `itonami.cloud` apex answers `200`). So unlike some sibling extractions,
this repository's responsibility has **not** been re-homed onto a running
service — the domain model was built, the surface was not.

## What to do with it

This is an owner decision. The options, and what each costs:

1. **Re-home it onto `itonami.cloud`.** The code builds, `soko` already holds the
   domain logic, and no competing WMS surface exists to duplicate. This is the
   cheapest of the three in engineering terms; it costs a hostname, a dispatcher
   upstream, and a decision about which of the two dispatchers survives.
2. **Retire it.** Defensible: the source path is gone upstream, all four hosts
   are gone, and nothing depends on it. Costs one west entry and stops it
   surfacing in "least mature repository" rankings, where it scores low for the
   honest reason that it is dormant.
3. **Leave it.** Costs nothing, but keeps a repository that reads — to anyone who
   opens `src/app.ts` first — as a live, secret-authenticated WMS actor.

Until one is chosen, this file is the correction: **nothing here is reachable,
and the file that looks like the implementation is not the one that ships.**
