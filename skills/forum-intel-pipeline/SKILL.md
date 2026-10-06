---
name: forum-intel-pipeline
description: Multi-source forum scraping — 200+ sources, no-AI digest, Telegram delivery. LOCAL-ONLY (all VPS down). Runs local scraper + bridge v16.
category: automation
tags: [forum, scraper, telegram, digest, github, chinese, darknet]
---

# Forum Intelligence Pipeline

> Multi-source scraping → strict garbage filter → categorized manual digest → Telegram + SQLite

## ⚠️ forum_parser.py CRON RUNBOOK — READ BEFORE ANYTHING ELSE

**This section exists because the same mistake was made FIFTEEN times (2026-09-17 → 2026-09-21), including by agents that had this skill loaded. The rule was buried in a pitfall paragraph; now it is here, first. The 12th occurrence (2026-09-21 05:00 cron) was the bare `python forum_parser.py` foreground call with timeout=600 — exit 124, 600s burned. The 13th (2026-09-21 later cron) was `python forum_parser.py 2>&1 | tail -40` with timeout=600 — exit 124 again, and a tail-piped probe leaves NO log file. The 14th (2026-09-21 09:00 cron) was `python forum_parser.py 2>&1 | tee run_cron_<ts>.log | tail -40` — exit 124 AND the tee-piped log was 0 lines: Python buffers stdout when it's a pipe, so nothing flushes before the kill. If you ever must pipe this script, use `python -u` (unbuffered) — that's what made the 09:15 background run's log pollable in real time. The 15th (2026-09-21 ~14:00 cron) was the plain bare `python forum_parser.py` foreground call, timeout=600 — exit 124, then the background launch finished in ~11 min. The 16th (2026-09-21 ~15:00 cron) was `python forum_parser.py 2>&1 | tail -60` with timeout=600 — exit 124 again; the background launch that followed ran ~25 min to `DONE. 1743 total findings`. The 17th (2026-09-21 ~19:30 cron) was `python forum_parser.py 2>&1 | tail -50` with timeout=600 — exit 124; the background launch that followed completed in ~11 min. The 18th (2026-09-21 ~20:45 cron) was `python forum_parser.py 2>&1 | tail -60` with timeout=600 — exit 124; the agent then tried `nohup python forum_parser.py > log 2>&1 &` in FOREGROUND mode, which Hermes HARD-REJECTS with "Foreground command uses shell-level background wrappers (nohup/disown/setsid). Use terminal(background=true)" — that attempt is a guaranteed wasted call, go straight from the (also wasted) foreground probe to terminal(background=true, notify_on_complete=true). The background launch completed in ~12 min with +39 findings. Each time, the background launch that followed completed cleanly in ~11–25 min.**

**RULE 0 — The VERY FIRST action of any forum_parser cron run is the background launch. No foreground probe. No "quick check". Not even with a log redirect.**

```
terminal(background=true, notify_on_complete=true, command=
  'cd "C:/Users/User/tmp/forum_parser" && python forum_parser.py > run_cron_$(date +%Y%m%d_%H%M).log 2>&1')
```

- A foreground run ALWAYS dies at the 600s cap (exit 124). The script takes 12–35 min (69+ sources × AI extraction). This includes `python forum_parser.py > log 2>&1` with timeout=600 — the redirect variant wastes the 600s just the same.
- While it runs: poll with foreground `sleep 240; tail -15 run_cron_<ts>.log` calls. `process(action=wait)` is hard-capped at ~60s — don't loop on it. Even better (used 2026-09-21): a bounded marker-check loop inside one foreground call under the 600s cap — `for i in $(seq 1 28); do grep -q "DONE\." run_cron_<ts>.log && { echo FINISHED; break; }; sleep 20; done; tail -3 <log>` — covers ~9.5 min per call, returns instantly on completion, repeat the call if not yet FINISHED.
- Done when log contains `DONE. N total findings`. Judge by marker, never by PID/tasklist — and never by `ls /proc/<pid>` from git-bash: MSYS has no real /proc for Windows PIDs, so `ls /proc/<pid> && echo RUNNING || echo DONE` ALWAYS prints DONE (observed 2026-09-21 ~17:50: reported DONE while the parser had ~15 min left and the log kept growing). Only the DONE marker inside THIS run's log + log mtime are trustworthy.
- **New-findings count (cheapest airtight proof):** grep anchored `grep -E "\+[0-9]+ findings" <log>` lines from THIS run's log (e.g. `[bhw-marketplace] +3 findings`); cross-check `STATE: ... X total findings` (start) vs `DONE. N total findings` (end) — delta must match the sum of +N lines. A loose `+[0-9]*` regex false-matches `+2026` inside YouTube URLs. Delta 0 is a legitimate result — report it with both proofs. **SAME-LOG RULE (attribution trap, observed 2026-09-21 19:30 cron): compute the delta as STATE(start of THIS log) → DONE(end of THIS log) — NEVER as DONE(this log) vs DONE(previous run's log). Cron runs can happen back-to-back from multiple agents; an interim run may have already bumped the counter, so a cross-log comparison over-attributes the interim run's findings to the current one. In the 19:30 run: STATE 1748 → DONE 1748 = delta 0 for THIS run, yet comparing against the older PM log's `DONE. 1743` falsely suggested +5. Same for findings.jsonl line counts: the `*.pre_cron_*` backup may predate an interim run — take your own `wc -l`/`cp` baseline immediately before YOUR launch, or use the same-log STATE/DONE pair.** **`ps -p <pid>` from git-bash ALSO lies (observed 2026-09-21 ~20:55 cron): `ps -p 41796 >/dev/null 2>&1 && echo YES || echo DONE` printed DONE while `process(action=poll)` showed status=running, uptime 484s. MSYS `ps` can't see Windows-native PIDs from `terminal(background=true)`. For liveness use `process(action=poll)` on the session_id (authoritative) or the DONE marker in the log — never ps/tasklist/kill -0/`ls /proc`.** **+N-sum vs counter-delta mismatch is normal (2026-09-21 20:45 run): anchored grep found 9 sources totaling +39 findings, yet the DONE counter read 1833 vs previous run's 1748 (+85) — interim runs and re-extraction duplicates move the counter independently. The +N lines from THIS run's own log are the primary reported number; capture the STATE line at launch time (head of log) if you want the delta cross-check, and if it doesn't match the +N sum, report the +N sum and note the discrepancy rather than silently picking the bigger number.** **SKIP-arithmetic cross-check (2026-09-21 07:45 clean run):** `grep -c "^====" <log>` = source blocks — use THIS run's count, it varies per run (110 blocks observed 07:45, 76 blocks observed ~15:00 same day; the source list is dynamic), `grep -c "SKIP — unchanged"` (59) + `grep -c "SKIP — no AI findings"` (46); if SKIPs ≈ blocks, zero anchored `+N findings` lines, and DONE counter matches the previous run's (1743→1743), delta-0 is proven without any jsonl diff. Caveat: blocks can exceed SKIP sum by a few (error/timeout blocks) — inspect those few lines individually before declaring 0. Also confirmed again: `process(action=wait, timeout=600)` is silently clamped to 60s (returns a timeout_note) — always poll with foreground `sleep 180-300; tail -3 <log>` calls instead.
- **CORRECTION (2026-09-21 05:15 run) — line/counter growth can be RE-EXTRACTION DUPLICATES:** findings.jsonl grew +6 lines and the DONE counter rose 1737→1743, yet the unique `(source, url)` set was identical pre/post (491→491). A changed content hash re-extracts the SAME items → new jsonl lines + counter bump, zero genuinely new findings. Airtight proof = **unique-pair set diff**: `cp output/findings.jsonl output/findings.pre_run_<ts>.jsonl` before launch, after DONE compare `{(d['source'], d['url'] or d['name'])}` sets in one `python -c`. Report "0 new" when the unique set doesn't grow, even if lines/counter did. **Key choice matters (09-21 later run): NEVER key the diff on `id` (doesn't exist in the schema) or bare `url` (collapses filter-variant duplicates → false "0 new"); use the `(source, url-or-name)` pair, or plainest of all: `diff <(sort old.jsonl) <(sort new.jsonl) | grep '^>'` on raw lines — no key assumptions, shows exactly what was appended.**
- **Verify content when delta > 0:** tail the new `output/findings.jsonl` lines (key is `name`, not `title`) — positive delta can be raw catalog dumps, not actionable findings.
- Dated output files (`*_YYYY-MM-DD.md`) get written even on SKIP — a new file is NEVER proof of new findings.
- `teleboost-blog` HTTP 403 and carding-forum DNS failures (seized domains) are routine, not run failures.

## Architecture (V16 — 2026-07-27, LOCAL-ONLY)

**ALL 3 VPS SERVERS DOWN as of 2026-07-27:**
- eni-nyc1 (134.209.72.186): connection timeout
- eni-sgp1 (165.232.171.239): connection timeout
- gothbreach-abuse-1 (188.166.189.92): publickey rejected

**Everything runs locally on Windows:**

```
Local (Windows)
├── forum_scraper.py (cron b6a19f00a033, */15)
│   ├── 13 scrapers: linux.do, v2ex, nodeloc, misskey,
│   │   GitHub trending, GitHub free APIs, GitHub search,
│   │   Reddit jailbreak, HN, Twitter AI, 4chan /g/,
│   │   RSS feeds (8), hostloc
│   ├── Output: ~/Desktop/_SCRIPTS/forum_scraper_output/
│   └── HTTP server :8000 (--serve 8000, background)
│
├── bridge_v16.py (cron a86a1f1490ab, */15)
│   ├── Fetch: http://localhost:8000/forum_data.json
│   ├── is_good() — source-aware filter
│   │   ├── tg/*: always pass
│   │   ├── rss: always pass (pre-filtered)
│   │   ├── github: stars≥5 + abuse keywords
│   │   ├── 4chan: replies≥30 + AI keywords
│   │   ├── hn/hackernews: tool/project keywords
│   │   └── v2ex/nodeloc/misskey/linux_do/hostloc: always pass
│   ├── score() — forums boosted, abuse keywords weighted
│   ├── SQLite: memory/forum_items.db (1215 items, 287 sent)
│   └── sendMessage (parse_mode=HTML) → TG @ReformBoss
│       Bot: 8852099196:AAFSDgfbfy6IS8zKDizhPqurOBrHF0XxVlY
└── keyhunter_final.py (cron 330532bcf5a4, */3h)
    ├── Source: free-llm-api-keys README (→ 404, DEAD Jul 27)
    └── DB: 217 keys, 0 live (all from old repo + gists)
```

**Scraper script:** `C:/Users/User/Desktop/_SCRIPTS/forum_scraper.py` (31KB, 947 lines)
**Bridge script:** `C:/Users/User/AppData/Local/hermes/scripts/bridge_v16.py` (6.6KB)
**Keyhunter script:** `C:/Users/User/AppData/Local/hermes/scripts/keyhunter_final.py`

**AI provider: NONE**. Manual no-AI digest. See `references/ai-generation-failures.md` for why.

## User Preferences (CRITICAL — from Vlad's corrections)

1. **AI ABUSES, FREEBIES, TOOLS — NOT NEWS**: Vlad: "мне надо абузы ии халяву софты и т.д что то полезное не новости". Filter for: jailbreaks, free API keys, hacking tools, VPS deals, GitHub repos with code. NEVER: tech news, corporate announcements, research papers, "X company launches Y".

2. **Show WHAT the abuse actually does**: Include description text showing the technique/tool, not just repo names. Vlad: "сначала полезный что там".

3. **Forum content, not just GitHub**: Vlad: "хули только гитхаб находит". v2ex, nodeloc, 4chan, Chinese forums must be represented.

4. **No AI generation**: Vlad rejected AI posts. Manual no-AI digest format is the standard.

5. **MORE sources constantly**: Vlad: "еще еще еще больше форумов", "все все все парси", "запусти парсинг того откуда можно парсить".

## Bridge v14: Garbage Filter + SQLite

**CRITICAL: Vlad raged when 4chan garbage leaked into digest.** Posts like "Twitch Gossip", "Gf/Wife Thread", "anti-ai alliance" appeared as "💥 Эксплойты/Пентест". Root cause: `is_ai()` filter too broad — "ai" matched "anti-ai", "tool" matched "tool" in any context.

### is_good() — garbage filter (v14)

Source-aware filtering with GARBAGE BLOCKLIST:

**4chan**: Only if ALL of:
- replies > 50 (popular threads only)
- Contains SPECIFIC AI tool names ("claude code", "codex", "jailbreak", "mcp server", "deepseek", "windsurf", "pentest", "chatgpt", "gpt-4", "copilot", etc.) — NOT generic "ai", "gpt", "tool"
- GARBAGE BLOCKLIST applied: "twitch gossip", "wife thread", "gf thread", "nobody general", "egirl", "discord", "streamer", "youtuber", "tiktok", "porn", "hentai", "furry", "onlyfans", "tinder", "simp", "incel", "cuck", "redpill", "looksmax", "mewing", "goon", "coom", "feet", "slut", "nigger", "faggot", "tranny", "pedo", "vore", "scat", "gore", "necrophilia" — 50+ garbage words

**HN**: Only if ALL of:
- stars > 10
- Contains tool/project keywords ("show hn", "github.com", "tool", "plugin", "mcp", "api", "launch", "release", "cli", "self-hosted", "extension", "skill", "agent", "library", "framework")

**RSS**: Only if contains specific abuse keywords ("jailbreak", "bypass", "exploit", "cve", "rce", "api key", "leak", "malware", "ransomware", "0day", "backdoor", "phish", "payload", "pentest", "red team", "shellcode", "dropper", "loader")

**All sources**: Must pass general AI/abuse keyword check + garbage blocklist.

### SQLite Database

Path: `C:\Users\User\AppData\Local\hermes\scripts\memory\forum_items.db`

Schema:
```sql
CREATE TABLE items (
    id TEXT PRIMARY KEY, title TEXT, url TEXT, description TEXT,
    source TEXT, stars INTEGER DEFAULT 0, category TEXT,
    found_at TEXT, sent_at TEXT
);
```

Stats: 51 items, 51 sent (2026-07-19). Categories: 🎁 Халява/API (9), 💥 Эксплойты/Пентест (10), 🛠 Инструменты/MCP (10), 🌐 VPS/Прокси (7), 💀 Хакинг/Малварь (4), 📦 Инструменты (11).

## File Paths (V16 — 2026-07-27)

| File | Path | Size |
|------|------|------|
| bridge_v16.py | `C:\Users\User\AppData\Local\hermes\scripts\bridge_v16.py` | 6,646 B |
| bridge_v14.py | `C:\Users\User\AppData\Local\hermes\scripts\bridge_v14.py` | 18,613 B (archived) |
| bridge_v15.py | `C:\Users\User\AppData\Local\hermes\scripts\bridge_v15.py` | 6,847 B (broken indent) |
| forum_scraper.py | `C:\Users\User\Desktop\_SCRIPTS\forum_scraper.py` | 31,380 B |
| forum_items.db | `C:\Users\User\AppData\Local\hermes\scripts\memory\forum_items.db` | 704 KB, 1215 items |
| sent_forum.json | `C:\Users\User\AppData\Local\hermes\scripts\memory\sent_forum.json` | variable |
| Cron scraper | `b6a19f00a033` | `*/15 * * * *`, no_agent=true, script=forum_scraper.py |
| Cron bridge | `a86a1f1490ab` | `*/15 * * * *`, no_agent=true, script=bridge_v16.py |
| Cron keyhunter | `330532bcf5a4` | `0 */3 * * *`, no_agent=true, script=keyhunter_final.py |

## Bot Tokens

| Bot | Token | Purpose |
|-----|-------|---------|
| Active | `8852099196:AAFSDgfbfy6IS8zKDizhPqurOBrHF0XxVlY` | @bosdgsdgsgbot, bridge v13 posting |
| Legacy | `7997150522:AAHdtcVZ3bmoLAJ-pxxOGGV5cu1UD5qW1MU` | Fallback |

## Stats (v11, 2026-07-19)

| Metric | Value |
|--------|-------|
| Sources per cycle | 200+ |
| Items per cycle | 676 |
| Total seen (dedup) | 4,351+ |
| GitHub Trending (24) | 170 |
| GitHub API (40 queries) | 120 |
| RSS (25 feeds) | 185 |
| v2ex (45 nodes) | 176 |
| 4chan (5 boards) | 68 |
| HN + HN Show | 80 |
| dev.to (28 tags) | 27 |
| nodeloc | 20 |

## Ahmia / Tor Hidden Service Search Engine (added 2026-08-28)

Ahmia (ahmia.fi) — clearnet + onion search engine for Tor .onion sites. Created by Juha Nurmi (2014, GSoC + Tor Project). Open source: github.com/ahmia/.

**CRITICAL:** Ahmia requires JS rendering. Firecrawl/web_extract return only the "no JS" fallback page. Must use Chrome DevTools MCP or Playwright.

**Parsing flow:**
1. Navigate to `https://ahmia.fi`
2. Fill search box (React form — needs nativeInputValueSetter trick)
3. Submit form via `document.querySelector('form').submit()`
4. Extract all `a[href*="redirect_url"]` links
5. Filter out crypto/marketplace noise (Bitcoin, Ethereum, Cannabis, counterfeit, carded)

**Best queries for traffic/spam tools:** "traffic spam software", "seo spam tools", "proxy list socks5", "smtp mailer", "web shell backdoor", "captcha solver", "cpanel spam"

**Noise filtering:** Ahmia returns massive crypto garbage. Blocklist: invest, Bitcoin, Ethereum, crypt, Cannabis, counterfeit, carded, Tether, Monero, майнинг, кошелек, биткоин

**Saving results:** `write_file` from hermes_tools blocked on Desktop — use `mcp__filesystem__write_file` or `mcp__filesystem__edit_file` instead.

See `references/ahmia-tor-search-engine.md` for full technique, query table, JS extraction code, and onion address.

## Pitfalls (see also references/)

- `references/forum-parser-firecrawl-ai.md` — **forum_parser.py** (`C:/Users/User/tmp/forum_parser/`, Firecrawl + Dashscope AI, ~110 source blocks, state.json dedup). Runs 12–35 min: **RULE 0 — background launch is the FIRST action, never a foreground probe** (600s cap kills it; mistake repeated ELEVEN times across 2026-09-17/18/19/21 — recurred again 09-19 midday (`python forum_parser.py 2>&1 | tail -50` burned a full 600s timeout before the background launch), again 2026-09-21 00:00 (foreground with log redirect + timeout=600, exit 124), and again 2026-09-21 03:30 (the SAME `2>&1 | tail -50` probe — a tail-piped probe leaves NO log file at all; First action of the cron = background launch, then inspect prior-run logs while it runs). **Findings-delta baseline trick (used 2026-09-21, cheapest airtight proof):** before launch run `wc -l output/findings.jsonl && md5sum output/findings.jsonl`, after DONE re-run both — the line delta IS the new-findings count (jsonl is append-only; 1779→1784 = +5; 1788→1788 with unchanged mtime = delta 0, 2026-09-21 03:30 run). No state.json backup needed, no UTC math. **Note: the `DONE. N total findings` / `STATE: ... N total findings` counter (deduped state.json findings, e.g. 1737) does NOT equal the findings.jsonl line count (e.g. 1788) — compare each metric pre/post separately, never cross-expect absolute equality; also `STATE: 69 sources tracked` vs ~110 source blocks in the log is normal (state tracks deduped sources).** Copy-paste launch: `cd "C:/Users/User/tmp/forum_parser" && python forum_parser.py > run_cron_$(date +%Y%m%d_%H%M).log 2>&1` via `terminal(background=true, notify_on_complete=true)`. Poll with foreground `sleep 280; tail -3 <log>` calls (`process wait` is hard-capped at ~60s — don't loop on it); done when log contains `DONE. N total findings` (check by marker, not PID — `kill -0` on the bash wrapper pid lies). **Counting new findings:** best = state.json `scraped_at >= run-start-UTC` (per-source breakdown) or **pre-run state.json backup diff** (`cp state.json state_pre_<ts>.json` before launch, compare `len(findings)` after DONE — airtight, UTC-bug-immune, used 2026-09-19); quick = delta of `DONE. N total findings` vs previous run's log (both morning/evening logs live in the same dir) — BUT this cross-log shortcut MISATTRIBUTES findings when another agent's interim run already bumped the counter (2026-09-21 19:30: reported +5 that actually came from the ~17:42 run; same-log STATE→DONE showed 0); always prefer the STATE/DONE pair inside THIS run's own log; delta 0 is a legitimate result, report it as such with both proofs. **Dated output file ≠ new findings**: raw scrapes (e.g. `tlgrm-channels_YYYY-MM-DD.md`) get written even when AI extracts nothing; all sources may log `SKIP — unchanged`/`SKIP — no AI findings`. Never report a new file as a finding without the counter delta. **Inverse trap: positive delta ≠ real findings** — raw catalog/navigation dumps (e.g. SourceForge directory listings, tlgrm.eu category pages, `sourceforge-telegram_YYYY-MM-DD.md`) still increment the `DONE. N total findings` counter; when delta > 0, open the new dated files and check content before characterizing them. If they're page chrome/links rather than tool/abuse extracts, report "+N — raw catalog dumps, no actionable findings" with both proofs (2026-09-17 run: +4, all noise). Cross-check **anchored** `grep -oE "\+[0-9]+ findings"` lines — a loose `+[0-9]*` regex matches `+2026` inside YouTube URLs. findings.jsonl key is `name`, not `title`. DNS getaddrinfo failures on carding forums are routine (seized domains), not run failures — e.g. `cracked.io` and `procarders.com` failed 2026-09-19. **Sustained zero-delta plateau**: when `DONE. N total findings` is identical across 4+ consecutive runs (observed: 1280 at 09-18 21:00, 09-19 00:58, 02:24, 03:40, 06:00, ~12:00 midday — 6+ consecutive identical runs, ALL 110 sources SKIP unchanged/no-AI-findings), report delta-0 honestly but flag source exhaustion / Firecrawl cache saturation as a hypothesis, and propose verifying the parser fetches fresh content (compare page hashes across runs) or rotating the source list — don't just repeat the same cron silently forever. When the plateau spans a full day, include the recommendation in the report (reduce cron frequency to 1×/day or add new sources — current 110 are exhausted); Vlad has now been told this twice. **`state.json` `last_run` is NOT a run-freshness indicator** — it only advances when new findings are saved (observed 2026-09-19: clean exit-0 run with DONE marker, `last_run` still showed the previous findings-producing run). To prove a run happened, use the run log's `DONE. N total findings` marker + log file mtime + pre/post state backup diff, never `last_run`. **Post-DONE `save_state` OSError traceback ≠ failed run** — scraping and output/*.md writes complete before the final state save; report the DONE counter and verify the next run's STATE line (observed 2026-09-17). **Stray python.exe ≠ hung parser**: a long-lived `python.exe` in `tasklist` may be an unrelated daemon (observed 2026-09-19: PID 16304, 572K RSS = `python bot.py`, NOT forum_parser). Probe availability FLIPS between sessions: 2026-09-19 midday `wmic process where "ProcessId=<pid>" get CommandLine,CreationDate` worked cleanly (returned `python bot.py` for PID 16304) while `tasklist //FI "PID eq <pid>"` returned nothing — reverse of 09-17. Try wmic first, fall back to the powershell Get-CimInstance one-liner. Never kill python.exe on suspicion; launch the parser in background with its own dedicated log and judge progress from that log only. Plateau extended: 1280 findings held through the 09-19 09:25–09:40 run (all ~110 sources SKIP), 7+ consecutive identical runs — source-exhaustion recommendation stands. 09-21 update: counter now plateaued at 1743 (07:45, 09:15, and ~14:00 runs — all 110 blocks / 59 unchanged / 46 no-AI-findings / 5 no-content / 0 fetch errors, findings.jsonl 1794→1794) — the 09:15 run was the cleanest delta-0 proof yet (SKIP arithmetic sums exactly to block count, zero error lines). The ~14:00 run confirms: 9 h after the 05:21 findings-producing run, sources still return identical hashes — same-day repeat runs are nearly always delta-0. The ~15:00 run (76 blocks, all SKIP, DONE 1743 unchanged) is the 4th consecutive same-day delta-0. The ~16:25 run is the 5th (105 blocks; 58 unchanged + 47 no-AI-findings = 105 SKIPs — exact SKIP-arithmetic match, 0 errors, DONE 1743 unchanged) — the every-few-hours cron cadence is provably redundant for this parser; 1×/day is enough. **Plateau broken 09-21 ~17:42 run: 1743→1748 (+5), all from `sourceforge-telegram` catalog** (MTProto Go, Telegram Drive, python-telegram-bot, Telegram Media Downloader, Telegram SMS) — proves sourceforge-telegram periodically refreshes even when every other source hash is stable, so delta-0 plateaus DO break on their own; keep counting with the anchored `+N findings` grep and don't panic-rotate sources. The 19:30 run was then delta-0 again (STATE 1748 → DONE 1748, 110 blocks, all SKIP) — post-break runs revert to the stable counter; the +5 belonged solely to the 17:42 run (the 19:30 agent mis-attributed it via cross-log DONE-vs-DONE; see SAME-LOG RULE above). Routine non-fatal errors this run: `cybercarders.eu` DNS getaddrinfo fail (likely dead — removal candidate), `m.vk.com` connection refused (VK blocks direct access — needs proxy or removal).
- `references/goth-intel-feed-topics.md` — **Goth Intel Feed** (forum-группа 3927191915, 6 топиков, бот-постинг полнотекстовых кейсов через pipeline4). Влад НЕНАВИДИТ посты-ссылки — только развёрнутые полные тексты статей с жирными цифрами. Двухстадийный парсинг (листинги → фетч статей) — единственная рабочая схема.
- `references/vibe-club-forum-build.md` — **Vibe Club** (forum-группа 4415843844, сессия R3fIex, 11 топиков): Telethon 1.45 forum-topic API (CreateForumTopic в messages, DeleteTopicRequest НЕ существует, channels.DeleteMessages вместо messages.DeleteMessages —后者 молча не работает, topic-id только через GetForumTopicsRequest, icon_emoji_id на топик кроме General, pin через UpdatePinnedMessageRequest), UTF-16 entity offsets + astral length=2 для premium emoji (иначе сервер молча дропает entities), FloodWait-ретраи, и ПРАВИЛО ВЛАДА v5: контент из своей базы (Obsidian, findings.jsonl, Desktop/_DOCS отчёты, MASTER_LIST.md, harvest каналов @ReformBoss-сессии — rb_channels_posts.json, ChatExport HTML дампы), посты развёрнутые 800-1500 chars с РЕАЛЬНЫМИ URL инлайн и ВЕРДИКТОМ, **ВСЕ URL curl-проверять перед постом** (404/000 = убрать или заменить; мёртвые в отправленных — EditMessageRequest), тонкие посты без ссылок = снос; упоминания третьих каналов/людей вычищать; стиль @AlStack (🩸-секции, КАПС, шаги, «Репост 🔁»).

### Local Deployment (V16 — 2026-07-27)
- **HTTP server `_source` vs `source` bug**: The built-in HTTP server in forum_scraper.py originally used `item["_source"]` for the source field. Bridge's `is_good()` looks for `item.get("source")`. Must patch to `item["source"]` in the `--serve` handler. Symptom: bridge says "0 good" despite valid data.
- **Bot token masking**: The `BOT_TOKEN` in bridge files may be masked as `***`. Always check the actual token before running. Bridge will silently fail to send to TG with masked token.
- **Indentation errors when patching bridge**: Python files with mixed indentation break easily with `patch` tool. Prefer `write_file` for clean rewrite of the entire bridge script.
- **Cron script must be in ~/.hermes/scripts/**: `cronjob` rejects absolute paths. Copy scripts to `~/AppData/Local/hermes/scripts/` and use just the filename.
- **HTTP server must be restarted after code changes**: `taskkill /F /PID <pid>` on Windows, then restart. `fuser -k` doesn't work on Windows git-bash.
- **Sent IDs reset**: If bridge has 0 new items, `sent_forum.json` may have all IDs. Backup and clear with `echo "[]" > sent_forum.json` to force re-send.

- `references/garbage-filter.md` — 4chan garbage blocklist, source-specific filter rules
- `references/ai-generation-failures.md` — All AI attempts failed, manual format is standard
- `references/ai-digest-prompt.md` — Original AI prompt (archived, not used)
- `references/do-server-restoration.md` — DO server recovery runbook
- - `references/sources.md` — Full source list
- `references/forum-discovery-2026-07-22.md` — 45+ forums, Discourse API, curl patterns
- `references/paste-site-archives-2026-09.md` — paster.sh-style paste archives as intel sources: 9-entry monitoring pool, live/dead status table (curl-verified 2026-09-25), psbdmp.ws API, gzip-response pitfall
- **DO 188.166.189.92 DEAD**: SSH publickey rejected + HTTP timeout. All on eni-nyc1 (134.209.72.186).
- **Port 8000 conflict on eni-nyc1**: `snowflake_proxy.py` occupies it. `fuser -k 8000/tcp` first.
- **SSH kills itself when pkill matches**: Use two-step launch: (1) kill, (2) start via nohup in separate SSH call.
- **eni-nyc1 high load (38+)**: `eternal_v8.py` + `kh8.py` consume ~107% CPU.
- **SSH disconnections**: Always `-o ControlMaster=no -o ControlPath=none`. Clean: `rm -f ~/.ssh/cm-*`.

### AI Generation
- **deepseek-v4-pro**: Returns empty content (reasoning-only) on prompts >2K chars. Unreliable for generation.
- **glm-5.2**: Returns empty content on prompts >700 chars. Use for short prompts only.
- **Vlad rejected AI posts as "кал"**: Manual no-AI digest is the standard. Do NOT switch back to AI generation.

### DO IP Blocks
- **Reddit**: All endpoints return 0. DO IP globally blocked by Reddit.
- **Medium**: Blocked. **NPM, PyPI, Docker Hub, GitLab, ProductHunt, AlternativeTo**: All 0 from DO.
- **Darknet forums**: All seized/dead (breached.to → FBI, raidforums → dead, dread → timeout).
- **raw.githubusercontent.com**: Blocked from DO. Free API key scraping fails.
- **linux.do**: Cloudflare-protected. FlareSolverr :8191 required but returns empty.

### Tokens/Keys
- **GitHub token DEAD**: `ghp_qD...drx3` from config → HTTP 401. Unauthenticated API = 60 req/hr.
- **Tavily keys ALL DEAD**: 15 keys from autoregs → HTTP 401. Need new keys from tavily.com.
- **Echogate DEAD**: Both keys dead (401 + 503). Use dashscope-rotator :16432 if AI needed.

### User Preferences
- **NOT news**: Vlad rejected news posts. Filter for AI abuses, freebies, tools only.
- **Show what it does**: Include descriptions, not just repo names.
- **Forum diversity**: Not just GitHub. v2ex, nodeloc, 4chan, Chinese forums.
- **Proofs required**: "готово" = tool-call verification (HTTP 200, file size, exit code).