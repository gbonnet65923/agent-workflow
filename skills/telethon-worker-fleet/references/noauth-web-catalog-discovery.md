# No-Auth Web Catalog Discovery (SSR + hidden JSON APIs)

Live-validated 2026-10-03, two phases, 205 NEW AI channels found with ZERO Telegram
sessions (no FloodWait risk, no account pool needed). Complements
gramgpt-atlas-channel-db.md (binary dump) — this is the *web-search* discovery lane.
Workdir: `C:/Users/User/tmp/tg_api_recon/` (all scripts re-runnable).

## The lane in one paragraph

1. Source candidate catalog domains from awesome-telegram-osint lists (git clone zip,
   regex `https?://([a-z0-9.-]+)` minus t.me/github/socials minus KNOWN set).
2. Mass-probe each candidate: GET / (liveness), GET `/search?q=chatgpt` (status/len),
   parse `<form action+name>` and any `/api/`|`.json` URLs in HTML.
3. Harvest usernames from SSR results with the AI query set.
4. Resolve titles via t.me preview pages (ThreadPool, slow retry for rate-limited tail).
5. AI-regex filter + NSFW/anime drop + dedupe vs tg_chat_grow pool.
6. Emit poolfmt `{channel,title,source}` for merge_pool.

## WORKING endpoints (verified 2026-10-03)

| Site | Method | Notes |
|---|---|---|
| **tgdr.io** | `GET /api/entry?skip=N` (JSON) | THE prize. Full catalog ~364 entries: `username, title, desc, type, members, category, telegram_id, likes/dislikes, is_active`. Pagination only (no search params — tested query/q/search/title/username/category all ignored). Found via `_next/static/.../pages/_app.js` bundle grep for `/api/`. Ideal for session-free liveness/member checks. Saved: tgdr_all.json. |
| **tg-me.com** | `/search.php?u=QUERY` + SSR listing `/telegram-group/<query>/<page>` | Biggest phase-1 source (373 usernames). |
| **hottg.com** | `/search.php?u=QUERY`; detail links = `https://www.hottg.com/<username>/index.html` | 338 usernames. Regex MUST be `/([A-Za-z0-9_+]{3,})/index\.html` — the first attempt (`/.../com.<name>`) matched nothing on a 200/116KB page. |
| **tgram.io** | SSR `/search?query=` | 77 usernames (phase 2). |
| **lyzem.com** | SSR `/search?query=` | No JSON API despite looking like an app; only `/api/0/profile` exists. 35. |
| **telegram-group.com** | `?q=` | 18. |
| **channelgram.com** | `?query=` | 12. |
| **uztelegram.com** | `?q=` | 11. |
| **catalog-telegram.ru** | `?q=` | 2. |
| **telegramcatalog.com** | SSR works but results carry NO t.me links | usernames only from sidebar. |

## DEAD / not-for-search (don't waste cycles)

- combot.org — top-group pagination ONLY, no search.
- sssoou.com — JS-render, empty body to curl.
- xtea.io — search runs through Google CSE `006249643689853114236:a3iibfpwexa`; CSE 403 outside the browser context.
- telegram-store.com — search dead (404); `/catalog/?q=` returns fixed sidebar names regardless of q (proven: q=chatgpt → hamster_kombat junk).
- tgram.ru (404), all-catalog.ru (404), telegramindex.com (DNS dead), intelx.io (key-walled).
- DLE/SPA sites with no server-side search: tgrm.su, add-groups, telegram-sliv, tgbox, telegram-region.
- Sites CAN flip status between sessions — re-probe before harvesting (probe_all.py is cheap).

## Title resolution without Telethon

- `https://t.me/<username>` preview page → `<title>` / og:title / meta. ThreadPool(8-16) is OK for ~750 names but t.me rate-limits the tail: phase 1 left 446/746 unresolved in one pass; a slow second pass (sleep 1-2s, sequential) recovered 673/746.
- Pre-seed from existing titled pools (tg_chat_grow/*.json, gramgpt atlas) to cut t.me traffic.
- Never use TG sessions for bulk resolve — that's exactly how R3fIex got a 26617s FloodWait (see pool-rotation-and-queue.md).

## Username extraction regex

`t\.me/([A-Za-z][A-Za-z0-9_]{3,31})` then EXCLUDE: share, proxy, addlist, iv, s, c, joinchat, telegram, addstickers, addemoji, addtheme. Invite links (`t.me/+...`) are NOT stable usernames — drop them from poolfmt (merge_pool keys on username).

## AI filter (two-stage)

STRONG pass: title/desc/username matches `(?i)\b(ai|artificial|gpt|chatgpt|llm|neural|нейросет|gemini|claude|midjourney|prompt|machine learning|deep learning|openai|copilot|agent)\b`. NSFW/anime drop BEFORE the AI regex (phase-1 noise: 18+ NSFW names passed the naive filter). Reuse tg_chat_grow/pool_filter.py semantics (HARD_DROP → STRONG_AI → SOFT_DROP → WEAK_AI) when merging into the growth pool.

## Pool dedupe pitfall

tg_chat_grow pools use INCONSISTENT keys across files: `username` / `channel` / `channel_id` / `linked_id` / `name`. Glob ALL `*.json` in the dir and normalize every key (lowercase, strip leading `@`) or you'll re-discover hundreds of known channels (pool = 2266 entries across files).

## Windows/Hermes gotchas hit this session

- Python (native) can't open MSYS paths `/c/Users/...` — always `C:/Users/...` in scripts run from git-bash.
- terminal(background=true) rejects `notify true`/`notify True` as bare words in some call shapes → just redirect `> job.log 2>&1; echo DONE >> job.log` and tail the log.
- For SPA sites: grep `_next`/webpack bundles for `/api/` strings BEFORE giving up — that's how tgdr.io's real API was found.
- Probe script that checks status+len+forms for 40 domains in ThreadPool takes ~30s and saves hours of blind harvest attempts.

## Yield summary (for expectation setting)

- Phase 1 (tg-me + hottg + lyzem, 25 queries): 746 usernames → 207 AI → **190 new** vs pool.
- Phase 2 (6 new SSR sites + tgdr.io, 18 queries): 120 usernames + 364 catalog → **15 new**.
- Diminishing returns are real: the second wave of catalogs yielded ~8% of the first. The high-value move after exhausting SSR sites is tgdr.io-style hidden APIs, not more catalogs.

## Files

`extract_candidates.py` (awesome-list → domains), `probe_all.py` (mass liveness/search probe), `harvest.py`/`harvest2.py`/`harvest3.py` (per-site parsers), `resolve_filter.py` (t.me titles + AI filter), `finalize.py` (slow retry + noise drop), `merge_final.py` (tgdr merge + pool dedupe + poolfmt). Outputs: `ai_channels_final.json`, `ai_channels_new*.json`, `ai_channels_new*_poolfmt.json`, `tgdr_all.json`, `probe_candidates.json`.
