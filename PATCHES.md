# Downstream Patches — Firecrawl fork

This repository is a fork of [`firecrawl/firecrawl`](https://github.com/firecrawl/firecrawl)
that carries a small set of local patches. This file is the source of truth for what
those patches are, their status against upstream, and how each one is handled after we
refresh onto upstream `master`.

**Last refreshed onto upstream/main:** 2026-06-17 (upstream HEAD `14481332b`).

## How this fork is organised

| Ref | Meaning |
|-----|---------|
| `main` | Plain mirror of upstream `firecrawl/firecrawl` master. |
| `combined` | Deployed integration branch = `upstream/main` + the still-relevant patches below. The running stack (`/opt/firecrawl`) builds its `firecrawl-combined` image from this branch's working tree. |
| `combined-pre-refresh` | The previous integration state (Nov 2025), kept as a branch for convenience. |
| tag `archive/combined-2025-11-pre-refresh` | Immutable snapshot of the pre-refresh `combined` — preserves **every** original patch commit, including the ones we have since dropped. Nothing is ever lost. |
| `feat/anti-bot-playwright` | Anti-fingerprinting patch re-applied cleanly onto fresh upstream. **Not deployed** (the stack uses the stock `ghcr.io/firecrawl/playwright-service` image); kept ready for future use. |

To recover any original patch as authored, check out the archive tag:
`git show archive/combined-2025-11-pre-refresh` / `git log archive/combined-2025-11-pre-refresh`.

## Refresh procedure (for next time)

1. `git fetch upstream` and move `main` to `upstream/main`.
2. Branch a fresh integration branch from `upstream/main`.
3. For each **KEPT** patch below, re-apply (cherry-pick and resolve, or re-derive against the
   new architecture per the notes).
4. For each **DROPPED** patch, re-verify the bug is still fixed upstream before discarding.
5. Rename the fresh branch to `combined`, preserving the old one as `combined-pre-refresh`
   and tagging an `archive/…` snapshot.
6. Rebuild + restart `/opt/firecrawl` (see that directory's `start.sh`).

## Patch status summary

| PR | Description | Upstream status | Verdict |
|----|-------------|-----------------|---------|
| [#2379](https://github.com/firecrawl/firecrawl/pull/2379) | `DISABLE_CRAWL` / `DISABLE_MAP` env vars | open, no equivalent | **KEPT** — re-applied to `combined` |
| [#2282](https://github.com/firecrawl/firecrawl/pull/2282) | Playwright anti-fingerprinting | open (author `genius-0963`); upstream playwright service still has no stealth | **PRESERVED** — on `feat/anti-bot-playwright`, not deployed |
| [#2378](https://github.com/firecrawl/firecrawl/pull/2378) | PDF temp-file leak | open, **but fixed upstream independently** | **DROPPED** |
| [#2387](https://github.com/firecrawl/firecrawl/pull/2387) | Failed extract jobs stuck in `processing` | open, **but fixed upstream independently** | **DROPPED** |
| [#2381](https://github.com/firecrawl/firecrawl/pull/2381) | Infinite retries / disconnect-cancel / slot-leak | closed; mostly fixed upstream | **MOSTLY DROPPED** — one sub-fix deferred (see below) |

---

## KEPT

### #2379 — `DISABLE_CRAWL` / `DISABLE_MAP`
A self-host feature (not a bug fix) that lets an operator disable the `/crawl` and `/map`
endpoints (v1 + v2) via env vars, returning HTTP 403 with a uniform body. **Actively used**
by the deployment (`docker-compose.yml` sets both to `true`).

- New file `apps/api/src/lib/feature-flags.ts` (`isCrawlDisabled`, `isMapDisabled`,
  `featureDisabledBody`). `featureDisabledBody` returns `success: false as const` so the body
  satisfies the `CrawlResponse`/`MapResponse` discriminated unions.
- Router-level guard (fast 403) in `routes/v1.ts` and `routes/v2.ts`.
- Controller-level guard (defense in depth) at the top of v1/v2 `crawl.ts` and `map.ts`.
- Unit test `apps/api/src/controllers/__tests__/feature-flags.test.ts`.

**Re-apply note:** the two new files apply unchanged; the 6 controller/route files take only
import-block + guard-anchor drift (upstream commented out the payment-middleware block the
router guard used to anchor after). Guard logic is unchanged.

---

## PRESERVED (not deployed)

### #2282 — Playwright anti-fingerprinting
Adds `playwright-extra` + `puppeteer-extra-plugin-stealth`, anti-detection launch args, context
hardening (locale/timezone/permissions/headers), and `evaluateOnNewDocument` spoofing
(`navigator.webdriver`, plugins, languages, `window.chrome`, `permissions.query`) to the
open-source Playwright microservice (`apps/playwright-service-ts/api.ts`).

- Upstream's own anti-bot is **server-side fire-engine** (`chrome-cdp;stealth` / `tlsclient;stealth`
  + a proprietary stealth proxy) and **deliberately bypasses** the playwright microservice
  (`engines/index.ts`: playwright declares `stealthProxy: false`). The open-source playwright
  service still has **zero** stealth, so this patch remains the only fingerprint evasion on the
  self-hosted/playwright path.
- **Not deployed here:** the running stack uses the stock `ghcr.io/firecrawl/playwright-service:latest`
  image, so this patch is inert unless we build a custom playwright image from
  `apps/playwright-service-ts` and point compose at it.
- The original commits dragged in junk root docs (`IMPLEMENTATION_SUMMARY.md`,
  `QUICK_START_ANTI_BOT.md`) and `test-anti-detection.js` — these are **not** carried forward.
- **Compat risk to validate before deploying:** `puppeteer-extra-plugin-stealth` is bridged via
  `playwright-extra`; confirm its `evaluateOnNewDocument` shim works on upstream's current
  Playwright version.

---

## DROPPED (fixed upstream — re-verify before discarding next time)

### #2378 — PDF temp-file leak
Old code only `unlink`-ed the temp PDF on the success path, leaking the file on every error
path (non-PDF content, parse errors, aborts). Upstream rewrote
`apps/api/src/scraper/scrapeURL/engines/pdf/index.ts` to wrap all processing in `try/finally`
with `unlink(tempFilePath)` in the `finally` (same fix, same log string). Leak no longer exists.

### #2387 — Failed extract jobs stuck in `processing`
Old worker's non-success branch called `updateExtract({ error })` without `status`, leaving the
Redis record in `processing` forever. Upstream now sets `status: "failed"` in that branch
(`apps/api/src/services/extract-worker.ts`). Every terminal path now writes an explicit status.

### #2381(A) — Infinite scrape retries
Old `scrapeURL` had a `while (true)` loop that could ping-pong feature flags forever with no cap.
Upstream merged an **identical** `ScrapeRetryTracker` (their PR #2660) capping attempts via
`config.SCRAPE_MAX_ATTEMPTS`. Our file and theirs are byte-for-byte equal.

### #2381(C) — Job finalizer / concurrency-slot leak
The "leak" never actually occurred: concurrency slots are time-scored ZSET entries (self-evict
≤60s) and a pg_cron lock reaper requeues stalled `active` rows (≤75s) — both present even at our
old base and still upstream. Our retry-the-finalize wrapper was defense-in-depth, not a fix.

---

## DEFERRED (real bug, still unfixed upstream — implement as a focused branch)

### #2381(B)+(D) — Cancel an in-flight scrape on client disconnect (+ skip billing)
**This is the only genuine bug from #2381 that still exists in upstream.** Sync scrape now runs
`processJobInternal` **inline** inside `teamConcurrencySemaphore.withSemaphore`. The controller
creates an `AbortController` and wires `req.on("close", () => aborter.abort())`, but `aborter.signal`
is only consumed by the concurrency-lock *wait* (`team-semaphore.ts`). Once the scrape is actually
running, client disconnect does nothing — it runs to completion, holds a slot, and bills.

**Port plan (re-architected — do NOT re-apply the old ~1,500-line patch):**
1. Thread the controller's existing `aborter.signal` through the inline in-process `job`
   (`skipNuq: true`, so it never serialises) into `processJobInternal` → `processJob` →
   `startWebScraperPipeline`, combining it with the worker's timeout signal via
   `AbortSignal.any([...])` at `scrape-worker.ts` (the per-job abort).
2. On abort, map to a `ScrapeJobCancelledError` (the `SCRAPE_JOB_CANCELLED` serde entry already
   exists but is never thrown), skip `billScrapeJob`, and return HTTP 499 from the controller.
3. Extract (async, returns an ID) is **out of scope** — upstream's async model isn't
   disconnect-cancellable by design.
4. Add an E2E disconnect test and **verify the aborted scrape cleanly releases the
   fire-engine/browser session and the semaphore `finally`** (an aborted-but-not-cleaned scrape
   could itself leak a browser session).

Relevant upstream anchors: `controllers/v2/scrape.ts` (`req.on("close")`, ~`:220`),
`controllers/v1/scrape.ts` (~`:153`), `services/worker/team-semaphore.ts` (`withSemaphore`,
`acquireBlocking`), `services/worker/scrape-worker.ts` (per-job abort + `billScrapeJob`).
