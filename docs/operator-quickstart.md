# Operator quickstart

There is no running service to operate. This repository is a dormant WMS edge
surface (see [`../README.md`](../README.md)), so its operational task is
**confirming that it is still dormant, and still revivable** — two separate
claims, both resting on measurements that go stale.

> **2026-08-26 update: the frontend was migrated from SvelteKit to
> ClojureScript** (`svelte/` → `cljs/`, reagent + re-frame + jp-go-dds; see
> `../README.md`'s "Frontend migrated" section). Checks 3–6 below were run on
> 2026-08-17, **before** that migration, and every `svelte/`-rooted path they
> name is gone. They are kept because the DNS/dormancy facts they establish
> (checks 1–2) and the historical provenance measurement (check 6, against the
> original 2026-08-17 tree) are still accurate as a record of what this repo
> was extracted from — not because those specific commands still run
> unmodified today. Check 7, added below, is the current build check.

Six checks, about two minutes. Every command below was run on 2026-08-17 from
the repository root, and the output shown is what it printed.

---

## 1. Are the four hostnames still gone?

This is the load-bearing fact. If any of them returns, the corresponding half of
this repository becomes live again.

```bash
for h in warehouse.etzhayyim.com w4r3h0u5.etzhayyim.com \
         dispatcher.etzhayyim.com mcp.etzhayyim.com; do
  printf '%-28s ' "$h"
  dig "$h" +noall +comments | grep -io 'status: [A-Z]*'
done
```

```
warehouse.etzhayyim.com      status: NXDOMAIN
w4r3h0u5.etzhayyim.com       status: NXDOMAIN
dispatcher.etzhayyim.com     status: NXDOMAIN
mcp.etzhayyim.com            status: NXDOMAIN
```

`NXDOMAIN` means the label does not exist — not that it exists and points
nowhere. The first two are the routes in `wrangler.jsonc`; the last two are the
upstreams the two dispatchers proxy to. **If any prints `NOERROR`, stop and
re-read the README before touching anything.**

## 2. Missing labels, or a dead zone?

A dead zone would mean "etzhayyim is offline", which is someone else's problem.
Check the apex:

```bash
dig +short NS etzhayyim.com | head -2
curl -sS -o /dev/null -w 'apex=%{http_code}\n' --max-time 12 https://etzhayyim.com/
```

```
vivienne.ns.cloudflare.com.
everton.ns.cloudflare.com.
apex=200
```

The zone is healthy and served by Cloudflare. Exactly the four labels this
repository needs are absent. The two nameservers come back in either order.

## 3. Which file would actually deploy?

The most consequential thing to know about this repository, and the easiest to
get wrong:

```bash
grep -E '"main"|"pattern"' wrangler.jsonc
git grep -n "src/app" -- .          # exit 1 = referenced by nothing
git ls-files 'svelte/src/routes/xrpc/*'
```

```
  "main": "svelte/.svelte-kit/cloudflare/_worker.js",
      "pattern": "w4r3h0u5.etzhayyim.com/*",
      "pattern": "warehouse.etzhayyim.com/*",
svelte/src/routes/xrpc/[...path]/+server.ts
```

`git grep` printing nothing (exit 1) is the result that matters: **`src/app.ts`
is referenced by no tracked file.** `main` is a SvelteKit build artifact, so the
deployed XRPC handler is `svelte/src/routes/xrpc/[...path]/+server.ts` — which
speaks JSON-RPC to a different upstream, with no shared secret and no NSID
filter. The two implementations disagree; only the SvelteKit one ships.

## 4. Can it still be built?

This separates "dormant" from "rotted", and it is the check that decides whether
option 1 in the README (re-home it) is cheap or expensive.

```bash
cd svelte
npm install --no-audit --no-fund
npm run build
ls -la .svelte-kit/cloudflare/_worker.js
```

```
added 92 packages
✓ built in ...            (client)
✓ built in ...            (server)
> Using @sveltejs/adapter-cloudflare
  ✔ done
4335 .svelte-kit/cloudflare/_worker.js
```

Package count and artifact size are stable; **wall-clock is not** — two runs an
hour apart on the same machine differed by 5× (this workstation runs many agents
in parallel). Compare `added 92 packages`, `✔ done`, and the 4,335-byte
artifact, not the timings.

Clean build from a cold checkout, no errors. **Before the build,
`.svelte-kit/cloudflare/_worker.js` does not exist** — it is generated, not
tracked — so a `wrangler deploy` against a fresh clone would fail on a missing
`main` until this step has run.

> Run heavy builds through the workspace resource governor rather than directly:
> `node <root>/scripts/resource-guard.mjs run build -- npm run build`.

**There are no lockfiles in the tracked tree** (`package-lock.json`,
`pnpm-lock.yaml`, `yarn.lock` are all absent — confirm with `git ls-files | grep
lock`), so this resolves current registry versions each time. A green build today
does not promise a green build next month.

> **This repository has no `.gitignore`.** After the steps above, `node_modules/`,
> `.svelte-kit/`, and a freshly generated `package-lock.json` all appear as
> *untracked*, in both this directory and the root. Do not `git add -A` here. Undo
> with:
>
> ```bash
> rm -rf node_modules .svelte-kit ../node_modules package-lock.json ../package-lock.json
> git status --short          # expect: no build artifacts listed
> ```

## 5. Does the declared check check anything?

```bash
npm run typecheck > /tmp/tc.out 2>&1; echo "EXIT=$?"; head -5 /tmp/tc.out
```

```
EXIT=1

> etzhayyim-project-warehouse@0.1.0 typecheck
> tsc --noEmit

tsc: The TypeScript Compiler - Version <varies>
```

There is no `tsconfig.json` at the root, so `tsc` prints its usage banner and
exits `1` without reading a source file. This is a check that **cannot run**, and
it is worth knowing that it fails loudly rather than passing silently — a vacuous
`EXIT=0` here would be the worse outcome.

Capture the exit code the way shown. In `zsh`, `${PIPESTATUS[0]}` is a `bash`
idiom and silently yields an empty string, which reads as "no failure".

### The answer depends on which compiler you use

Do not use `npx tsc` here — it resolves to whatever is nearest or cached, and the
two halves of this repository declare different TypeScript majors. Invoke each
explicitly:

```bash
npm install --no-audit --no-fund        # root: typescript ^6.0.2
./node_modules/.bin/tsc --version
./node_modules/.bin/tsc --noEmit --target es2022 --lib es2022,dom \
  --module esnext --moduleResolution bundler --strict src/app.ts; echo "EXIT=$?"

./svelte/node_modules/.bin/tsc --version   # svelte: typescript ^5.9.3
./svelte/node_modules/.bin/tsc --noEmit --target es2022 --lib es2022,dom \
  --module esnext --moduleResolution bundler --strict src/app.ts; echo "EXIT=$?"
```

```
Version 6.0.3
EXIT=0

Version 5.9.3
src/app.ts(48,41): error TS2339: Property 'entries' does not exist on type 'URLSearchParams'.
EXIT=2
```

`src/app.ts` is clean under the root's declared TypeScript and fails under
`svelte/`'s. It is orphaned, not broken — but **"does it typecheck" has no answer
until you name the compiler.**

## 6. Is the extraction provenance still intact?

This distinguishes "dormant because it was extracted and then stranded" from
"something was lost in transit":

```bash
git ls-files | grep -vE '^(README\.edn|migration\.edn)$' > /tmp/orig12.txt
wc -l < /tmp/orig12.txt
wc -c $(cat /tmp/orig12.txt) | tail -1
gh api "repos/etzhayyim/root/git/trees/c9e7df4b327a3ee7b5ed25fb3480db45553c33ed:60-apps" \
  --jq '.tree[]|select(.path=="etzhayyim-project-warehouse")|.sha'
```

```
      12
   13282 total
f19958a84d2822f0a320fc137485bcf0b0c50206
```

All three match `migration.edn`'s `:tracked-files 12`, `:bytes 13282`, and
`:tree f19958a84d…`. Nothing drifted. **Clean provenance is what makes
"retire it" a safe option rather than a lossy one** — the upstream original is
still addressable at that revision, even though the path is gone from `main`.

Note that `README.md` and this file are *not* in `migration.edn`'s
`:allowed-additions`. That field records what the extraction tool added at
migration time; it is not an enforced allowlist, and nothing in the workspace
verifies it. Excluding both by name, as above, keeps the byte count comparable.

## 7. Does the ClojureScript frontend build and test clean? (added 2026-08-26)

`svelte/` no longer exists — the frontend is `cljs/` (shadow-cljs + reagent +
re-frame + jp-go-dds). Two builds and a test run, from the repository root:

```bash
cd cljs
npm install --no-audit --no-fund
node <root>/scripts/resource-guard.mjs run build -- npx shadow-cljs compile app
node <root>/scripts/resource-guard.mjs run build -- npx shadow-cljs compile test
node out/tests.js
```

```
added 129 packages
[:app] Build completed. (111 files, 110 compiled, 0 warnings, 21.31s)
[:test] Build completed. (112 files, 111 compiled, 0 warnings, 10.42s)
Ran 5 tests containing 14 assertions.
0 failures, 0 errors.
```

Measured 2026-08-26. `wrangler deploy`/`wrangler dev` were **not** run against
the updated `wrangler.jsonc` — that remains unverified (see the repo README's
"Frontend migrated" section).

---

## If every check holds

Nothing to do. The repository is correctly inert, still buildable, and the
README's account of it is current.

## If a check fails

| Check | Failure | What it means |
|---|---|---|
| 1 | any `NOERROR` | A hostname is back. Find out who created it before assuming this repo should serve it. |
| 2 | apex down | An etzhayyim-wide outage, unrelated to this repository. Not yours. |
| 3 | `git grep` finds a match | Someone wired `src/app.ts` in. The two dispatchers disagree on protocol — check which upstream is intended before deploying. |
| 4 | build fails | The unpinned dependency tree drifted. Option 1 in the README just got more expensive; record the failing version before reacting. |
| 5 | `EXIT=0` on the first command | A `tsconfig.json` appeared at root. Verify it actually covers `src/`, rather than passing by checking nothing. |
| 6 | count, bytes, or sha differ | The working tree drifted from the recorded extraction. Do **not** retire — reconcile against upstream first. |
| 7 | either build or the test run fails | The `cljs/` dependency tree (or jp-go-dds itself) drifted since 2026-08-26. Record the failing output before reacting — this frontend has no lockfile pinning `reagent`/`re-frame`/`shadow-cljs` versions either. |

**Do not archive, delete, or re-point DNS on the strength of these checks
alone.** They establish that the surface is dormant and revivable, not that the
owner has decided which of the README's three options to take.
