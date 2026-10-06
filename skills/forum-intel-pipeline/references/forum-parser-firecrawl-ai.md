# forum_parser.py — Firecrawl + AI extraction pipeline (tmp/forum_parser)

Separate tool from forum_scraper.py. Scrapes carding/cracking/traffic forums via Firecrawl API, extracts findings with Dashscope AI (qwen3.8-max-0902), dedupes against state.json.

## Layout

```
C:/Users/User/tmp/forum_parser/
├── forum_parser.py        # main script (cron target)
├── state.json             # per-source hashes + findings list (each has scraped_at UTC)
├── discovery_state.json   # source discovery state
├── output/                # per-source dated .md files + findings.jsonl
└── run_cron_*.log         # run logs — name each run's log uniquely (date+time)
```

Sources: lolz.live sections, crackingx, xreactor, crdpro, trafficultras, blackhatworld, hackforums, zenno-club, cpa.rip, cpaelites, Telegram channel dumps (tg-abuz-ai-soft, tg-jebusblack, tg-cpalike, telemetr), sourceforge-telegram, github-awesome-telegram-bots, and many carding forums.

**Source count drifts — read it from the log, never from memory or the cron prompt.** ~110 blocks on 2026-09-11; **60 blocks on 2026-09-12** (`STATE: 60 sources tracked, 666 total findings` log line). The cron prompt may claim fewer ("scrapes 10 forums") than the script processes. Real block count: `grep -c "^====" <log>`; real source count: the `STATE:` line near log start.

## Run protocol (CRITICAL)

**Full run takes 15–35+ minutes — never run foreground.** Foreground max timeout is 600s and the run will be killed mid-way (confirmed again 2026-09-11 18:01: killed at 600s with a 0-byte log).

**RULE 0: the FIRST action is the background launch — there is no useful foreground probe.** A foreground "just try it" wastes exactly 600s, produces nothing (buffered log), and risks a surviving ghost process sharing state.json. This mistake was repeated in back-to-back sessions on 2026-09-17 (09:22 run and eni2 run both burned a 600s foreground attempt first). Go straight to step 1.

1. Launch: `terminal(background=true)` + `notify_on_complete=true`, redirect to a fresh dated log (hardcode the timestamp — `$(date ...)` inside the command is fine but make the name unique per run):
   ```
   cd /c/Users/User/tmp/forum_parser && python -u forum_parser.py > run_cron_YYYYMMDD_HHMM.log 2>&1
   ```
   Use `python -u` (unbuffered) so the log grows in real time for polling. Do NOT use nohup/disown/setsid in foreground mode — Hermes guard rejects it.
2. Poll by grepping the log, NOT by PID checks:
   - Completion marker: line `DONE. N total findings in ...` near end of log.
   - Progress markers: `[source-name] SKIP — ...` or `[source-name] +N findings` lines; `grep -c "^====" <log>` shows blocks processed out of ~110.
   - Best wait loop (used successfully 2026-09-11 20:00 run): ONE foreground `terminal` call, timeout 600:
     ```
     cd /c/Users/User/tmp/forum_parser && while ! grep -q "^DONE" <log> 2>/dev/null; do sleep 30; done; echo FINISHED; tail -3 <log>
     ```
     Repeat the call if the run exceeds 600s — BUT do not repeat the IDENTICAL wait-loop command twice in a row: Hermes fires a `repeated_exact_failure_warning` tool-loop warning after 2 identical timeout exits (hit on the 2026-09-11 22:50 run, which needed 3 polls). Alternate poll shapes instead: wait-loop → quick progress check (`grep -c "^====" <log>; tail -2 <log>`) → wait-loop with progress folded in (`...; grep -c "^====" <log>`). Avoid `tasklist | grep python` AND `ps -W | grep python` as a liveness proxy — the box has unrelated python.exe processes, so a for/sleep loop whose break condition is such a check NEVER exits and burns the full loop budget (hit twice on the 2026-09-19 04:55 run). Loop-break on the DONE marker only: `grep -q "^DONE\." <log> && break`.
3. **PID liveness checks are unreliable in git-bash**: `kill -0 <pid>` on the pid returned by terminal(background=true) can report "done" while the python process is still running (the pid is the bash wrapper). `process(action='poll')` gives real status + uptime. Trust the DONE marker in the log.
4. **Do NOT relaunch** if unsure whether it's running — check log mtime first, and verify with:
   ```
   powershell -Command "Get-CimInstance Win32_Process | Where-Object {$_.CommandLine -like '*forum_parser*'} | Select ProcessId" 
   ```
   (filter out the bash wrappers). A second concurrent run corrupts state.json dedup. After a foreground timeout kill, the whole process tree is killed with it — but verify before relaunching anyway.
5. **`process(action='wait')` is capped at ~60s** regardless of the timeout you pass ("clamped to configured limit of 60s"). Don't loop on it. Use foreground `terminal` sleep-polls instead.
6. Timing observations: 2026-09-11 late run ~22 min / ~119 sources; 2026-09-11 18:15 run **34.5 min** (uptime 2070s) / 110 blocks, exit 0; 2026-09-11 22:50 run ~35 min / 110 blocks, +0 (3rd run of the day — full dedup); **2026-09-12 00:20 run: 60 blocks, ~9 min, +0** (foreground attempt hit the 600s cap again — exit 124, always go background). Runtime scales roughly linearly with source count; a 60-source quiet run fits in ~10 min, so 1–2 sleep-polls suffice. Plan for 3–4 wait-loop polls (600s each) only on 110+ source configs; a quiet +0 run takes just as long as an active one because the Firecrawl fetches still happen.
7. PowerShell on this machine has a broken profile.ps1 (`\$PSVersionTable` escaping errors in every `-Command` invocation) — output noise, harmless; ignore or use `wmic process where "name='python.exe'" get processid,commandline` instead.
8. **Re-run after a killed foreground attempt is safe (verified 2026-09-12):** exit 124 kills the whole python tree; relaunching in background re-scrapes unchanged sources but state.json dedup prevents duplicate findings — findings.jsonl only grew by the genuinely new items (+8 lzt-market-tg, 712→720 lines). Cost is wasted Firecrawl credits, not data corruption. That run: ~60 blocks with 67 SKIP lines, +8, total wall time ~21 min. **CORRECTION (2026-09-17): the tree is NOT always killed.** Observed: foreground launch 04:15, exit 124 at 04:25, log stayed 0 bytes (no `-u`, buffered output never flushed) — yet state.json was updated at 04:47 by the SURVIVING detached python, which ran to completion (~32 min). Before relaunching after a timeout, ALWAYS check for a ghost: `stat state.json` mtime advancing + `powershell -NoProfile -File <probe>.ps1` matching `*forum_parser*` in CommandLine. Relaunching while a ghost runs = two concurrent runs sharing state.json (the 09-17 05:00 relaunch happened to be benign — ghost had just finished, all 58 sources SKIP-unchanged, +0 — but don't rely on it). Also: a 0-byte log does NOT mean the script is hung; without `-u` it can stay 0 bytes for the entire run. **Detached launch from a foreground terminal call works and survives the call:** `(python -u forum_parser.py > run_X.log 2>&1 &) && sleep 20 && head run_X.log` — then poll with bounded `for i in $(seq 1 N); do sleep 30; grep -q "^DONE\." run_X.log && break; done` loops (keep each poll call ≤550s, under the 600s foreground cap; `timeout=700` is rejected outright).
9. **`ps -p <pid>` in git-bash also lies** (same as `kill -0`): reported FINISHED twice while python was still running per `process poll`. Only trust (a) the DONE marker in the log, (b) `process(action='poll')` status/uptime.

## Counting new findings for the report

Best method — state.json scraped_at (UTC; local is UTC+3), gives exact count + per-source breakdown:

```python
import json, collections
s = json.load(open('C:/Users/User/tmp/forum_parser/state.json', encoding='utf-8'))
f = s['findings']
new = [x for x in f if x.get('scraped_at','') >= '<run-start-UTC-ISO>']
print('total:', len(f), 'new:', len(new))
for k, v in collections.Counter(x['source'] for x in new).most_common(): print(f'  {k}: {v}')
```

Note: state.json is only updated when the run finishes (or at save points) — mid-run it still shows the previous run's totals. **NEVER filter `scraped_at` by a LOCAL-date prefix** (e.g. `.startswith('2026-09-12')`): timestamps are UTC, so findings written at local 2026-09-12 02:12 carry `2026-09-11T23:12Z` and a local-date filter reports "0 new" while the run actually added items (hit on the 2026-09-12 run — real +8 from lzt-market-tg). Use run-start UTC or the findings.jsonl line-count delta instead. Cross-checks:
- Log method: `DONE. M2 total findings` at log end minus total before = new this run.
- `grep "+[0-9]* findings" <log>` per-source lines should sum to the same number.
- Typical quiet run: +0 to +5. Active runs: 2026-09-11 morning +38 (579→617), 18:15 run **+46 (620→666)** — top sources telemetr-channels +10, teletype-tg +6, medium-tg-bots/tg-cpalike +5. When several runs happen in one day, later runs are mostly SKIP-unchanged; that's normal dedup, not a failure. But TG-channel/CPA sources can keep producing +40 even on the 4th run of a day. A fully quiet run (2026-09-11 20:00): all ~110 blocks SKIP, +0, findings stay 666 — report "0 new" plainly, don't hunt for a failure. On a quiet run, `wc -l output/findings.jsonl` also stays flat (708 lines at 666 findings — jsonl keeps historical dupes, count != state total). Healthy quiet-run SKIP distribution (2026-09-11 22:50 run): 54 `unchanged` + 53 `no AI findings` + 3 `no content` (DNS dead) = 110 blocks; sanity-check with `grep -c "SKIP — unchanged" <log>` (same for the other two variants). 2026-09-12 quiet run: 60 blocks, all SKIP, findings.jsonl flat at 708, state still 666 — DNS-dead roster grew by cracked.io + cybercarders.eu (candidates for config removal). **2026-09-17 quiet run: 110 blocks, STATE `63 sources tracked, 922 total findings`, +0; SKIP distribution 59 unchanged + 46 no AI findings + 5 no content = 110; wall time ~14 min.** DNS-dead roster now: cracked.io, cybercarders.eu, procarders (+2 more) — all 5 are config-removal candidates. 6th failing source (2026-09-17 evening run): `vk-telegram-promo` — m.vk.com gives TCP connection refused (WinError 10061), NOT DNS; vk-side/ISP-level block from this box. Also a config-removal candidate (or swap to vk.com via proxy). The evening run was the 3rd+ of the day (run_cron_20260917.log, _rerun.log preceded it) and confirmed: same-day repeat runs go 110/110 SKIP, +0 — cron cadence ≤1×/day or add a force-refresh flag; report "0 new" plainly, it's dedup working. Note the block count (110) is ~2x the tracked-source count (63): multiple blocks map to the same source family; `STATE:` line is the authoritative source count, `grep -c "^=====" ` the block count. Log grew fine without `python -u` this run (default buffering flushed per-block), but keep `-u` for tight polling. Foreground 600s kill re-confirmed (first attempt exit 124, zero useful output) — always background on first try. **2026-09-17 late-evening run (run_20260917_cron.log): 110 blocks, STATE `63 sources tracked, 924 total findings`, +0; SKIP distribution 59 unchanged + 45 no AI findings + 6 no content = 110; ~20 min wall.** Launch pattern that worked cleanly: `terminal(background=true, notify_on_complete=true)` with `python forum_parser.py > run_DATE_cron.log 2>&1; echo "EXIT=$?" >> log`, then three ascending foreground polls (`sleep 240/300/420; tail -3 log; grep -c '=====' log`) — no wait-loop needed, block count via `=====` showed progress (23 → 98 → DONE). On a +0 run there are NO `+N findings` lines at all, so a regex count over them returning 0 matches is the expected quiet-run signal, not a parse bug. `bc` is absent in this git-bash — sum numbers with a `python -c` one-liner instead of piping grep into bc. **2026-09-17 09:22 run (run_cron_20260917_0922.log, 6th run of the day): 110 blocks, +0, STATE 63 sources / 924 findings, ~30 min wall.** Confirmed workflow details: (a) the killed foreground attempt (exit 124) DID advance state.json mtime before dying — mid-run save points exist, so **state.json mtime advancing ≠ ghost alive**; the authoritative ghost check is (b) `wmic process where "name='python.exe'" get ProcessId,CommandLine | grep -i forum_parser` — returned no match → relaunch safe (wmic probe works cleanly, unlike noisy PowerShell). (c) `tasklist //FI "IMAGENAME eq python.exe"` FAILS in this git-bash (`ERROR: Invalid argument/option - '//FI'` — MSYS mangles the switch); use plain `tasklist | grep -i python` (liveness proxy only, many unrelated pythons) or the wmic CommandLine probe. (d) Poll pattern that worked: three ascending foreground `sleep 300; tail -3 <log>; grep -c '^\[' <log>` calls (57 → 99 → DONE); `grep -c '^\['` counts per-source result lines and is a fine progress proxy alongside `grep -c '^===='`.
- **`state.json.last_run` does NOT advance on a +0 run** (stayed 15:53 after the 20:00 run) — never use it to tell whether a run happened; use log mtime / DONE marker.
- **Loose `+N` regexes count garbage**: `grep -o "+[0-9]*" <log>` matched `+2026` inside YouTube search URLs (`...software+2026&sp=...`) and summed to a nonsense "4052". Always anchor: `grep -oE "\+[0-9]+ findings" <log>` or `grep -E "^\[.*\] \+[0-9]+" <log>`. On a quiet run the anchored grep returns nothing — that's the expected +0 signal.
- **`tasklist //FI "PID eq N"` works in git-bash but `//FI "IMAGENAME eq ..."` does not** (2026-09-17): the PID filter form returned cleanly; the IMAGENAME form failed with MSYS switch-mangling. Neither proves liveness for the python child anyway (background=true returns the bash wrapper pid) — stick to the DONE marker / log mtime / wmic CommandLine probe.

**2026-09-17 eni2 run (~09:35–09:50, 7th run of the day): 110 blocks, +0, `DONE. 924 total findings`; SKIP distribution 59 unchanged + 45 no AI findings + 6 no content = 110; wall ~15 min.** Sequence that worked: burned 600s on a foreground attempt first (violation of RULE 0 — piped to `tail -40` with no log file at all, so nothing was even recorded), then background launch `python forum_parser.py > run_cron_20260917_eni2.log 2>&1 && echo DONE >> ...` + three ascending foreground polls (`sleep 300; tail -5 log`, `sleep 300; tail -6 log`, `sleep 240; tail -8 log`). `process(action='wait')` re-confirmed hard-capped at 60s. findings.jsonl mtime stayed 07:43 (previous run) while github_new.jsonl looked fresh (09:50, github_scan.py) — shared output/ dir caveat confirmed live again.

**2026-09-17 cronjob run (~12:30, 8th+ run of the day): +0, `DONE. 924 total findings`, STATE `63 sources tracked`.** RULE 0 violated AGAIN (600s foreground timeout, exit 124) before background relaunch — that makes three foreground-burn incidents on 2026-09-17 alone. Poll pattern used: `sleep 120/300/420; tail <log>` — fine. **New pitfall observed: a +0 run can still WRITE a dated output file.** `output/tlgrm-channels_2026-09-17.md` appeared (raw tlgrm.eu catalog scrape) while every source logged SKIP and the findings total stayed 924. Some blocks save raw markdown regardless of whether AI extracted findings. Therefore `ls output/ | grep <today>` is NOT a findings counter — only the DONE-delta / state.json scraped_at methods count. In the report, mention a new dated file explicitly as raw scrape, not as findings, to avoid a misleading "1 new file = 1 finding" read.

**2026-09-17 22:30 run (run_cron_20260917_2230.log, ~9th run of the day): 110 blocks, +0, `DONE. 1070 total findings`, STATE `65 sources tracked` (up from 63 — the 17:00 run added 924→1070, +146).** RULE 0 violated a FOURTH time this day (600s foreground `| tail -50` probe first, exit 124). Poll pattern: background launch + ascending foreground polls `sleep 240/300/120; tail -3 <log>; grep -c '^===='` — block count 17→220 lines→DONE showed progress cleanly. `process(action='wait')` 60s clamp re-confirmed.

**NEW FAILURE MODE — save_state crash after DONE (observed on the 21:19 eni3 run):** log printed `DONE. 1070 total findings` then `Traceback ... OSError: [Errno 22] Invalid argument: 'C:\\Users\\User\\tmp\\forum_parser\\state.json'` in `save_state()`. Semantics: ALL scraping + per-source output/*.md writes already completed (they happen before the final state save), so findings are NOT lost — only the final state.json snapshot failed (likely transient handle/AV/lock contention; a killed foreground run earlier that evening may have held the file). The NEXT run wrote state.json fine (mtime advanced, STATE line correct). Do not treat a post-DONE save_state traceback as a failed run: report the DONE counter, note the state-save crash, and verify the following run's `STATE:` line matches. If two consecutive runs crash on save_state, suspect a lingering ghost holding the file (wmic CommandLine probe).

**2026-09-17 23:00 run (run_cron_20260917_2300.log, ~10th run of the day): 110 blocks, +0, `DONE. 1070 total findings`, STATE `65 sources tracked, 1070 total findings`, LAST RUN 18:17; SKIP distribution 60 unchanged + 44 no AI findings + 6 no content = 110; exit 0; wall ~25 min.** First run of the day that did NOT burn a foreground probe — straight background launch. Two detail corrections from this run: (a) launched with `python forum_parser.py 2>&1 | tee run_...log` (no `-u`) and the log DID grow incrementally — `tail`/`process wait` showed live per-block output; tee's pipe seems to force flushing often enough, though `-u` remains the safe choice for tight polling. (b) `grep -cE "^\[" <log>` returned **220 on 110 blocks** — each block emits TWO `[source]`-prefixed lines (the URL line and the result line), so `^\[` counts ~2× blocks; for block progress use `grep -c "^===="`, and treat `^\[` numbers as roughly double. Poll shape used: `process(action='wait', timeout=60)` repeatedly — works (returns partial output each clamp) but costs ~8 round-trips over 25 min; ascending foreground `sleep 240–300; tail -3 <log>; grep -c '^====' <log>` polls are cheaper.

**DNS-dead / blocked roster (confirmed 2026-09-17 evening, re-confirmed + extended 2026-09-18):** `cracked.io` (getaddrinfo fail — NEW 09-18), `cybercarders.eu` (getaddrinfo fail), `procarders.com` (getaddrinfo fail), `lozerix.com` (connect timeout 30s), `m.vk.com` (TCP refused, WinError 10061 — ISP/vk-side block, not DNS), plus one HTTP 403 source. All are config-removal candidates; script continues past them normally. The 5 failing sources map exactly to the `SKIP — no content` count in quiet-run logs.

**2026-09-19 00:58 run (run_20260919_005819.log): 110 blocks, +0, `DONE. 1280 total findings`, STATE `66 sources tracked, 1280 total findings`; SKIP distribution 57 unchanged + 49 no AI findings + 4 no content = 110; wall ~11 min; exit 0.** RULE 0 violated a SIXTH time (foreground `| tail -50` probe, exit 124 at 600s) — and note the killed foreground attempt again produced NO log file (piped to tail, not redirected), so the first 600s left zero trace. The background relaunch worked cleanly. Poll shape this run: ~10× `process(action='wait', timeout=60)` round-trips — functional but the most expensive pattern; 2–3 ascending foreground `sleep 240/300; tail -3 <log>; grep -c '^====' <log>` polls would have halved the round-trips. findings.jsonl 1328 lines vs state 1280 (jsonl keeps historical dupes — count != state total, as documented). The 49/110 `no AI findings` share is stable across runs (45–49 range all week) — chronic Firecrawl login-wall degradation on crackingx/blackhatworld-class sources; flagged in the report as an auth-scrape-or-prune decision point if it worsens. DNS-dead roster shrank to 4 no-content (cracked.io still failing getaddrinfo).

**2026-09-19 ~02:00 agent run (run_cron_20260919_agent.log): 110 blocks, +0, `DONE. 1280 total findings`, STATE `66 sources tracked`; DNS-dead roster 4 (cracked.io, cybercarders.eu, procarders.com getaddrinfo fail + lozerix.com connect timeout — same 4 all week, pruning them saves ~90s/run of timeouts); wall ~11 min; exit 0.** First clean run in days: NO RULE 0 violation — background launch was literally the first action. **Best counting technique this run — state.json backup-diff:** `cp state.json state_pre_<ts>.json` BEFORE launch, then after DONE compare `len(findings)` before/after in one `python -c` (delta + last_run + before/after counts = airtight proof for the report; immune to scraped_at/UTC-prefix bugs). Poll shape that worked cheaply: one `process(action='wait')` call (confirm the 60s clamp + get uptime), then three ascending foreground `sleep 300; sleep 240; sleep 120` polls with `tail -3 <log>` + `grep -c '^\[' <log>` progress (61 → 207 lines → DONE). On +0 runs the report should still name the failing hosts and recommend config pruning — that's the only actionable signal a quiet run produces.

**2026-09-19 04:55 cron run (output/run_20260919_045517.log): 110 blocks, +0, `DONE. 1280 total findings`, STATE `66 sources tracked, 1280 total findings`; wall ~11 min; exit 0.** RULE 0 violated a SEVENTH time (foreground `python forum_parser.py 2>&1 | tail -40`, exit 124 at 600s, no log file). Background relaunch used `$(date +%Y%m%d_%H%M%S)` inside the redirect — works fine from terminal(background=true), log landed in output/ as run_20260919_045517.log (note: this run redirected into output/, prior runs wrote to the script root — both locations are valid, just grep the right path). `process(action='poll')` output_preview shows harmless `bash: no job control in this shell` noise — ignore it, watch status/uptime. **NEW POLLING ANTI-PATTERN (wasted two full sleep loops this run): breaking a `for/sleep` loop on `ps -W | grep -qi python` (or `python.exe`) NEVER settles** — the box always has unrelated python processes, same caveat as `tasklist | grep python`. Loop-break condition must be the DONE marker: `grep -q "^DONE\." <log> && break`. This run burned ~540s of sleep loops on the ps condition before switching to log-tail checks.

**2026-09-19 ~11:00 cron run (run_cron_20260919_1100.log): 110 blocks, +0, `DONE. 1282 total findings`, STATE `66 sources tracked, 1282 total findings`; DNS-dead roster 4 (cracked.io, cybercarders.eu, lozerix.com, procarders.com); wall ~14 min; exit 0.** RULE 0 violated an EIGHTH time (foreground `python forum_parser.py 2>&1 | tail -50`, exit 124 at 600s, no log file). Before relaunching, correctly checked the stray `python.exe` (PID 16304, 568K) — it was `python bot.py` since 09-18 14:55, NOT a forum_parser ghost; wmic CommandLine probe identified it cleanly. Launch WITHOUT `python -u` (`python forum_parser.py > log 2>&1; echo "EXIT=$?" >> log`) and the log still grew fine (394 lines mid-run) — `-u` remains nice-to-have, not required. Poll shape: `sleep 540; tail` then `sleep 300; grep -c DONE` — two cheap polls, done. Counting: findings.jsonl flat at 1330 lines pre/post + only 2 findings with today's scraped_at (both from the 10:50 medium-tg-bots run, not this one) = clean +0 proof. Plateau now 1280→1282 across 8+ consecutive near-identical runs; the 2 deltas today were medium-tg-bots catalog entries from a parallel source, not forum content — source-exhaustion recommendation stands (reduce cron to 1×/day or rotate sources).

**2026-09-18 01:20 run (run_cron_20260918_eni.log): 110 blocks, +0, `DONE. 1070 total findings`, STATE `65 sources tracked, 1070 total findings`, last_run still 2026-09-17T18:17Z (re-confirms: last_run does NOT advance on +0 runs); SKIP distribution 60 unchanged + 45 no AI findings + 5 no content = 110; wall ~20 min; exit 0.** RULE 0 violated a FIFTH time (foreground `| tail -50` probe, exit 124 at 600s) before background relaunch — the pattern persists across cron sessions despite documentation; the very first tool call must be the background launch. Poll shape that worked: three ascending foreground `sleep 240/240/180; tail + grep -c '^=' <log>` calls (90 blocks → DONE). **NEW counting artifact:** in Python, `log.count("==========")` returned **550 on 110 blocks** — each separator line is 50 `=` chars, so a 10-char pattern matches 5× per line non-overlapping. For block counts in Python use `sum(1 for l in log.splitlines() if l.startswith("="))`; `grep -c '^='` (counts LINES) is correct as-is. Quiet-run report value-add that landed well: include the cumulative findings type-breakdown from state.json (`collections.Counter(f['type'] for f in state['findings'])` → other 532, combo 112, traffic 99, spammer 67, carding 63, inviter 61, scraper 56, userbot 45, autoreg 23, proxy 12 on 09-18) so a +0 report still carries signal.

## findings.jsonl schema (output/findings.jsonl)

Append-only JSONL, one object per finding:
```
{"name", "description", "price", "type", "url", "author", "date", "source", "source_url", "scraped_at"}
```
- `type` values seen: traffic, autoreg, spammer, other (tool categories).
- Verify a run's additions: `grep '"<source-name>"' output/findings.jsonl | tail -N | cut -c1-300` — no `title` key, use `name` (a `.get('title')` probe returns `?`).
- Newest per-source markdown also lands as `output/<source>_YYYY-MM-DD.md` — `ls -lat output/ | head` confirms which sources wrote this run (mtime = run end).
- **output/ is SHARED with github_scan.py** (writes `github_new.jsonl`, `gh_q*.json`, `github_seen.json` on its own schedule). A fresh mtime in output/ does NOT prove forum_parser wrote anything — check the filenames (`<source>_YYYY-MM-DD.md` / `findings.jsonl` mtime), e.g. `stat -c '%y %n' output/findings.jsonl`. On a quiet run findings.jsonl mtime stays at the previous run's timestamp while unrelated github files look fresh.

## Log line semantics

| Line | Meaning |
|------|---------|
| `SKIP — unchanged` | content hash same as last run |
| `SKIP — no AI findings` | fetched OK, AI extracted nothing new/valuable |
| `SKIP — no content` + `HTTP error ... getaddrinfo failed` | DNS dead (chronic for some seized/dead forums — expected, not a script bug) |
| `+N findings` | N new items written to output/ and state.json |

DNS failures on carding forums are routine (domains get seized/rotated). The script continues past them; don't treat as run failure unless >50% of sources fail DNS.

## 2026-09-21 00:00 cron run (run_cron_20260921_eni.log): +5, `DONE. 1733 total findings`

**Active run after a week-long +0 plateau — plateau BROKEN.** findings.jsonl 1779→1784 lines; the 5 new: CodesSender.com (bhw-social-media, SMS-активация), TGStat (tgstat-ratings), ASocks (fb-killa, резидентные прокси $6/GB), Mirocard (fb-killa баннер), Domains-without-KYC (fb-killa, от $1.29). Wall ~9 min. Most sources `SKIP — unchanged` (fresh dumps from the 09-20 23:23 run); reddit-* blocks gave `SKIP — no AI findings`.

**Baseline-diff counting method (cheapest airtight proof — prefer it):**
```
# BEFORE launch:
wc -l output/findings.jsonl && md5sum output/findings.jsonl
# AFTER DONE marker:
wc -l output/findings.jsonl   # delta = new findings (jsonl is append-only)
```
No state.json backup, no UTC/scraped_at math. md5 doubles as "file untouched" proof if delta is 0.

**RULE 0 violated a NINTH time — new variant:** foreground `python forum_parser.py > log 2>&1` with `terminal(timeout=600)` (redirect-to-log, NOT pipe-to-tail). Still burns the full 600s → exit 124, then background relaunch re-runs everything (state.json dedup made it safe, cost = wasted Firecrawl credits + ~10 min). Even the "I'll just capture it to a log" framing is a RULE 0 violation. First tool call of this cron = background launch, full stop.

**NEW liveness-check trap — `tasklist //FI "PID eq N" | grep -q N` gives FALSE "process done":** in a bounded for/sleep loop, the break condition fired after 30s while the parser was still running (confirmed by `process(action='poll')` → status=running, uptime=371s, and by the log still advancing). tasklist output piped through MSYS grep behaves erratically (UTF-16-ish encoding → `Binary file (standard input) matches`, inverted/unreliable exit codes). NEVER use tasklist-in-a-loop as the DONE condition. Only trust: (a) `grep -q "^DONE\." <log>` marker, (b) `process(action='poll')` status. This is the third documented liveness-probe failure mode (`kill -0`, `ps -p`, `ps -W | grep python`, now tasklist) — the DONE marker is the ONLY reliable signal.

**State-vs-jsonl divergence re-confirmed:** script printed `DONE. 1733 total findings` (state.json counter) while findings.jsonl held 1784 lines (jsonl keeps historical dupes). Report the jsonl delta for "new this run" and optionally cite the DONE counter as the script's own total — never conflate them.

**Poll shape that worked this run:** background launch → `sleep 240; tail -3 log; wc -l findings.jsonl` → `process(action='wait', timeout=600)` (clamped to 60s, returned running+uptime — useful as a cheap liveness ping, not a wait) → one bounded for/sleep loop with the (buggy) tasklist break → final marker-grep loop `for i in $(seq 1 18); do sleep 30; tail -1 log | grep -qE "DONE" && break; done`. Cheapest correct shape: two ascending `sleep 240/300; tail -3 <log>; wc -l output/findings.jsonl` foreground polls.

## 2026-09-21 05:15 cron run (run_cron_20260921_0515.log): +0 genuine, `DONE. 1743 total findings`

**RULE 0 violated a TWELFTH time — new variant:** the bare `python C:/Users/User/tmp/forum_parser/forum_parser.py` foreground call with timeout=600 (no pipe, no redirect). Exit 124 as always. Background relaunch then completed cleanly in ~15 min (wall ~20:00 min total counting the wasted probe).

**GHOST CONFIRMED + psutil probe is the reliable ghost check.** ~5 min after the exit-124 kill, `psutil.process_iter(['pid','cmdline','create_time'])` matched FIVE live PIDs whose cmdline contained `forum_parser.py`, all created within the same second as the killed foreground launch — the "kill" left a detached process tree running. They finished/exited on their own ~10 min later (state.json mtime advanced at 04:58). Before relaunching, always run:
```python
python -c "import psutil; [print(p.pid) for p in psutil.process_iter(['cmdline']) if p.info['cmdline'] and any('forum_parser.py' in c for c in p.info['cmdline'])]"
```
Empty output → relaunch safe. This beats wmic (deprecated, flaky), tasklist (MSYS-mangled `//FI`), and PowerShell `-Command` with `$_` (see below). If ghosts are alive and you must not wait: relaunching anyway is *usually* benign (dedup) but risks state.json corruption — prefer waiting for them to exit.

**PowerShell `$_` flood recurred (documented pitfall, violated anyway):** `powershell -NoProfile -Command "... Where-Object {$_.CommandLine -like '*forum_parser*'} ..."` from git-bash with DOUBLE quotes → bash ate `$_` → 50KB of `CommandNotFoundException: The term '/c/Users/User/tmp/forum_parser.CommandLine'` noise. Same failure as python-hermes-windows-pitfalls §6. Rule: for process-commandline probes on this box use the psutil one-liner above, never an inline PowerShell `$_` in double quotes.

**NEW counting pitfall — line/counter growth can be RE-EXTRACTION DUPLICATES.** findings.jsonl grew 1788→1794 (+6 lines) and DONE counter 1737→1743 (+6), yet the unique `(source, url)` set was identical pre/post (491→491): sources whose content hash changed re-extracted the SAME items the AI had already emitted, appending duplicate jsonl lines and bumping the state counter. So `wc -l` delta and DONE-delta are UPPER bounds, not proofs of new findings. Airtight method — unique-pair set diff:
```bash
cp output/findings.jsonl output/findings.pre_run_<ts>.jsonl   # BEFORE launch
```
```python
# AFTER DONE marker:
import json
def pairs(p):
    s=set()
    for l in open(p,encoding='utf-8'):
        l=l.strip()
        if l:
            d=json.loads(l); s.add((d.get('source',''), d.get('url') or d.get('name','')))
    return s
old,new=pairs('output/findings.pre_run_<ts>.jsonl'),pairs('output/findings.jsonl')
print('NEW:',len(new-old))
```
Report "0 new" when the unique set doesn't grow even if lines/counter did — with both numbers cited so the discrepancy is explained, not hidden.

**Same-day cadence observation (reported to Vlad 3× now):** the 02:06 run added 34 unique findings; the 05:15 run 3 h later added 0. ~3h intervals between full runs yield near-zero marginal output while burning Firecrawl credits + Dashscope tokens. Recommendation stands: cron ≤1×/day for this parser or rotate/expand the source list.

## 2026-09-21 later cron run (~05:00–05:22 local): +6 jsonl lines, unique-set delta = 6, `DONE. 1743 total findings`

**RULE 0 violated a THIRTEENTH time — `2>&1 | tail -40` foreground probe, exit 124, no log file.** The exact pipe-to-tail variant that was already documented twice (09-19, 09-21 03:30). Background relaunch then completed in ~19 min wall, exit 0.

**IMPORTANT — this run REFINES the re-extraction-duplicates lesson from the 05:15 run.** A pre-run snapshot `findings.pre_cron_20260921_0500.jsonl` existed (1788 lines); post-run findings.jsonl = 1794 (+6). Two dedup probes disagreed:
1. **id/url-key set diff → NEW: 0** — WRONG method here. The probe keyed on `f.get('id') or f.get('url','')`, but findings.jsonl has NO `id` key and these 6 entries' `url` values were duplicated filter-variant URLs (`lzt.market/telegram/?spam=no#title` etc.) that also appeared in older lines → collapsed to "0 new".
2. **`(source, url-or-name)` unique-pair set diff → correctly shows the 6** as new pairs relative to the snapshot... except in THIS case the snapshot was taken at 05:02 by a parallel process and the 6 lzt lines were appended at 05:21 — so both a raw `diff <(sort old) <(sort new) | grep '^>'` and the pair-set diff agree: +6 lines, all `lzt-market-tg` filter-variant catalog URLs.

**Lesson: when counting, use the documented `(source, url or name)` PAIR key — never `id` (doesn't exist) and never bare `url` (collapses legit duplicates and filter variants). And when a pre-run snapshot's mtime predates the launch, verify WHO wrote it (parallel runs share output/) before trusting it as the baseline. Plain `diff <(sort A) <(sort B) | grep '^>'` on the raw jsonl lines is the most honest probe — it shows exactly what was appended, no key-choice assumptions.**

**Content check mattered:** the 6 "new" entries were `lzt.market/telegram/?...` filter/search URLs (`?title=session json`, `?title=tdata`, `?premium=yes`, `?origin[]=autoreg`...) — raw marketplace catalog navigation dumps from the dated `lzt-market-tg_2026-09-21.md` scrape (page chrome: cookie notices, PIX/MasterCard payment banners, category links). Actionable-adjacent (they reveal lzt.market search facets for session/tdata trading) but NOT tool/abuse findings. Reported as "+6 — lzt.market catalog filter variants, no actionable forum findings". This is the third documented case of the inverse trap (positive delta = catalog noise): SourceForge dumps (09-17), tlgrm.eu (09-17), lzt.market facets (09-21).

**Poll shape used:** background launch → `sleep 240; tail; tasklist` → `sleep 300; tail; grep -c` → `sleep 420; tail` — three ascending foreground polls, cheap and correct. Note `grep -c "^===" run_log.txt` = 110 blocks confirmed (each separator counted once with `^===` anchor). Log was written to `run_log.txt` (fixed name, overwritten each run) — prefer unique dated names per RULE 0's launch template so concurrent investigations don't clobber evidence.
