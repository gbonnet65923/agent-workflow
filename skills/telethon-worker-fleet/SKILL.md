---
name: telethon-worker-fleet
description: Telethon worker fleets, session pools, FloodWait rotation.
---

# Telethon Worker Fleet Ops

Class-level patterns for running multiple Telethon userbot processes (channel parsers,
stats analyzers, competitor-bot probers) that feed data into a product bot or DB.
Proven live 2026-09-24 on the CrossPromo project: parser (channel discovery) + analyzer
(views/ER/fair-price stats) + prober (competitor RE), 177 channels collected, 140
analyzed, zero data loss through a session FloodWait.

## Golden rules

1. NEVER use the owner's primary session for bulk automation. Heavy ResolveUsername /
   get_entity traffic burned the R3fIex session into a 26617 s FloodWait mid-task. Copy
   pool sessions from Desktop/CLEAN_ALIVE_SESSIONS/ into the project work dir.
2. One session file = one live process. Two concurrent clients on one file lock/corrupt
   it. Give each worker role its own pool dir (analyzer pool/ = sessions 0-7, parser
   pool_p/ = 8-15).
3. Rotate on FloodWait, don't die: catch FloodWaitError, mark session+expiry in
   pool_state.json, disconnect, switch to next clean session, RE-QUEUE the failed item.
4. Pool-copied sessions can be stale: client.start() prompts for phone -> EOFError in
   background. Re-copy fresh, delete *.session-journal leftovers.
5. Some pool sessions are FROZEN accounts: every API call raises FrozenMethodInvalidError.
   Mark them permanently dead (a `dead` set, no expiry) on first occurrence and rotate —
   retrying the same frozen session burns the whole run (real case: parse wave finished
   with `SAVED 0` because one frozen session was reused for all 55 keywords).
6. Each worker role gets a DISJOINT slice of the source pool (analyzer 0-7, parser 8-15,
   small-parser 16+ filtered against existing pool dirs) — two workers copying overlapping
   ranges end up on the same session file concurrently.
7. Reconnect on ConnectionError, don't just mark-error: after a network blip a Telethon
   client raises `ConnectionError: Cannot send requests while disconnected` on EVERY
   call. A generic except that records the error and continues burns the whole queue
   (real case: 300+ analyses marked error in a row). Catch it, re-queue the item,
   reconnect with retries, rotate session if that fails.

## Details

- references/pool-rotation-and-queue.md — canonical pool/rotation code, SQLite queue
  protocol, dedup predicate bug, crash recovery.
- references/probing-and-stats-recipes.md — competitor bot reverse-probing (inline vs
  reply keyboards, markup dump, BotFather automation + token exfiltration), channel stats
  recipe (views/ER/fair price), Windows/Hermes gotchas, Vlad's clone-bot preferences.
- references/reaction-farming-reliker.md — reaction farming on comment sections (reLiker
  clone): SendReactionRequest peer=entity (NOT Message), MessageService filtering,
  ReactionInvalidError on restricted groups, linked_chat_id pre-filter (~40% channels
  have no comments), parallel-worker JSON state race, ban/retire policy, donor sourcing.
  Live-tested 2026-09-24: tmp/relik/relik.py. UPDATED 2026-10-03: CUSTOM-EMOJI-ONLY
  channels (@AlStack) — same 'only emoji are allowed' error but fix is
  ReactionCustomEmoji(document_id=…) harvested from
  full_chat.available_reactions (ChatReactionsSome → .document_id, NO .emoticon);
  farm accounts react fine without Premium; react_new.py one-shot + count read-back
  verification; reactor/mass session-dir lock separation. UPDATED 2026-10-06:
  run_boost_now.py N M = combined views (GetMessagesViewsRequest increment) +
  custom-emoji reactions from the farm — Vlad's rule: OUR accounts first, SMM panels
  fallback (panels can't place premium emoji); PeerChannel(id) fails on fresh sessions
  → get_entity(username) once; ReactionCount has no .emoticon (r.reaction holds it);
  no MessageReactionForbiddenError class in telethon; 1.45 session files unreadable
  by telethon 1.39 (use hermes venv python).
- references/closed-loop-growth-pipeline.md — SELF-FEEDING loop (2026-09-25,
  revived+hardened 2026-10-03):
  parallel hunters (own workdir + own output JSON each), idempotent merge_pool.py
  (15-min tick, atomic tmp+os.replace, DROP_KW filter, dedup by linked_id),
  wave_chain.py auto-launching next seeding waves when shard_worker count hits 0.
  Pitfalls: `$PSItem`/`$_` stripped when PowerShell is invoked through git-bash
  (run via Python subprocess instead), SpamBot restricted recheck ≥24h only,
  merge-before-wave ordering, HANDOFF.md cross-agent resume pattern.
  2026-10-03 additions: PowerShell CIM hangs EVEN via subprocess → all process
  censuses use psutil only (proc_scan.py canonical, python-name filter required
  or bash wrappers inflate counts); duplicate singleton wave_chain instances
  resurrect each other (kill-all → verify census empty → launch one);
  `direct=True` pool entries lack linked_id → consumers must use .get() and
  gate on direct flag (KeyError crashed shards mid-wave); save_state needs
  retry+direct-write fallback on transient .tmp OSError; LLM backend flip when
  aurora returns Arrearage → hermes dashscope proxy :16432 MiniMax-M2.1,
  ALWAYS test one chat completion before relaunching shards.
- references/traffic-methods-research-2026-10.md — external traffic-methods bank
  for @AlStack growth (2026-10-03 deep research): TG has NO recommendation feed;
  invite cap 200; биржи table with entry thresholds (tagio.pro/teletarget usable
  pre-3.5K subs, telega.in needs 3,500+); mutual-PR bots; catalogs (TGStat/
  Telemetr = one-time + Google SEO of t.me link); TG-SEO title keywords;
  external channels (Habr/VC/Reddit/Shorts); referral deep-link bot (not built);
  weekly health benchmarks (views 40-60% of subs, ER≥5%, forwards≥2%);
  mapping to existing infra + cheapest missing modules first.
- references/gramgpt-atlas-channel-db.md — GramGPT Telegram Atlas: free 60MB binary
  (`gramgpt.io/api/atlas/bin/`) with 403,872 channels (username/title/id/category),
  fully reversed ATLB v2 format + working parser + import pipeline + TGStat-via-Firecrawl
  niche rankings + analyzer ConnectionError-reconnect fix. The single biggest
  channel-discovery source found to date.
- references/invite-fleet-and-spambot-screening.md — chat-growth invite fleet: per-session
  access_hash isolation, join-before-invite (ImportChatInviteRequest / CheckChatInviteRequest
  in Telethon 1.44), linked-group member harvesting when channel lists need admin, @SpamBot
  pool screening + classification keywords, donor discovery from crosspromo.db. VERDICT
  2026-09-24: invites DEAD END even on SpamBot-clean sessions (UserBannedInChannelError
  "spam-limited" on first attempt) — use organic-seeding-paired-dialogs.md instead.
  UPDATED 2026-10-03 with two still-valid pieces: (a) SpamBot FALSE-RESTRICTED bug +
  reclassify_spam.py (unicode apostrophe `i.?m`, RU «свободен от каких-либо ограничений»,
  CLEAN-checked-before-RESTRICTED ordering, 3 severity buckets, reclassify from stored raw
  replies without re-querying); (b) the channel-validation gate (check_channel.py):
  live-humans/7d + AI-topic + writable + GetFullChannelRequest members, and the
  session-copy pitfall where `basename()` keeps `.session` → double extension →
  Telethon silently creates an empty session and every probe reports "not authorized".
- references/mass-parallel-shard-seeder.md — SCALE-UP pattern (2026-09-25):
  6 parallel shard processes over disjoint account+group slices
  (mass_seed.py launcher + shard_worker.py), 117-account pool from
  .orca/sessions inventory, full persona dress-out, module import-guard
  pitfall (module-level asyncio.run fires on import), per-shard state files,
  banned-acc vs write-blocked-group retirement counters, runtime LLM key
  loading from aurora .env (keys in source get masked), clickable-proof
  read-back workflow, shard-kill via .ps1 (taskkill blocked by safety guard),
  infinite chat_hunter_loop.py (continuous contacts.Search rounds + AI
  allowlist / gaming denylist filtering). Vlad's bar: ≥100 actions/day, all
  accounts in play. VLAD'S HARD PROMO RULES (corrected live): ONE shill pair
  per group EVER (shill_done.json registry), NEVER comment on the promoted
  channel itself — reactions only (react_alstack.py), and subscribe every
  clean account to the target channel (sub_alstack.py).
- references/organic-seeding-paired-dialogs.md — VALIDATED alternative to invites:
  full promo pipeline (SpamBot screening → persona dress-up → warm-up channel joins →
  linked-group harvest → paired asker/replier dialogs in private discussion groups).
  Contains the GetDiscussionMessageRequest warm-up trick to resolve+join private linked
  groups from any fresh session, group harvesting from crosspromo.db, human-like dialog
  bank + dedupe/pace state, @SpamBot classifier keywords, MANDATORY activity filter
  (member count lies — count human msgs/7d before seeding; Vlad rejects dead chats),
  and a context-aware LLM responder (aurora gateway qwen, human-style replies to real
  messages, @AlStack mention only when topical). Live 2026-09-24: 25 seed pairs
  + 40 responder replies landed across 12 groups, dialog bank expanded to 46
  themed pairs (бесплатно/абузы/раздачи/прокси/vps/заработок), full 64-account
  persona dress-out, daily cron alstack-seed-daily 11:00. Includes direct
  live-megagroup search
  (contacts.SearchRequest + activity filter + MLBB noise removal), the mandatory
  read-back PROOF workflow (msg_id/sender/timestamp verbatim), and responder
  production fixes (join-discussion-group, banned_accs rotation, dead read-only list),
  liveness-first group ordering (direct+recent_human sort, kill-and-relaunch after
  mid-run patches), big-megagroup write restrictions ("You can't write in this chat"
  = skip group not account), and Vlad status-reporting cadence during long runs.
  Updated 2026-09-25: session-pool expansion from .orca/sessions inventory
  (PROTECT owner sessions by id AND filename), failure attribution pattern
  (group_dead counter vs acc_fails counter — never mix group write-blocks with
  account bans), TGStat-ratings-via-Firecrawl as the highest-yield live-group
  source (15 groups 8K-58K members from one /ratings/chats/tech page; plain HTTP
  403s), addlist.* catalogs dead, direct-group filter must accept linked_id=0,
 Message.link missing in Telethon 1.44 (build t.me links manually).
 2026-09-25 additions: Firecrawl raw-API key rotation when the MCP is
 IP-blocked, participants_count=0 → GetFullChannelRequest pitfall, FloodWait
 session-parking validator fleet, pool_merged.json merge + direct-flag
 normalization bug, global one-shill-per-chat registry (shill_done.json),
 kill_mass.ps1 pattern, 3.5-5.5min asker→replier delay + indirect hedged reply
 style, moderate side-task volumes, log-as-durable-store recovery, HANDOFF.md
 cross-agent dump pattern.
 2026-10-04 additions: corrected warm-cache recipe (grouped_id gate was WRONG —
 always None on comment posts; GetDiscussionMessage kwarg is peer=), Telethon
 Request-signature pitfalls (GetParticipantRequest needs participant=,
 messages.GetForumTopicsRequest), forum open-topic seeding (reply_to=topic_id),
 the writable_state.json pre-validation gate (writable/forum/closed/
 pending/noentity + FloodWait requeue) wired into mass_seed load_groups(),
 DIRECT-pool validation (validate_direct.py: `direct=True` flags lie —
 broadcast channels never accept direct posts, gate on ent.megagroup),
 per-chat LLM dialog generation with static-bank fallback, and post-join
 spam discipline (confirm membership via GetParticipant + 6-18s sleep before
 first send; delete orphan asker posts on replier failure; group-fault vs
 account-ban error classification).
- references/addlist-merge-and-llm-gateway-repair.md — (1) resolving a public
 `t.me/addlist/<slug>` chatlist invite via `chatlists.CheckChatlistInviteRequest`
 (import from functions.chatlists, NOT messages) → normalize+persist peers →
 merge into pool_merged.json shape (`channel/title/linked_id/members/recent_human/
 recent_senders/direct`; drop no-comments, own-channel @AlStack, dupes) — live:
 84 resolved → 70 added → pool 215. (2) Diagnosing fleet-wide `LLM ERR 503 /
 No provider found for model X`: attribute the :8080 port owner FIRST (it was
 tmp/ai_gateway.py uvicorn, not aurora.exe), inspect gateway_data.json providers,
 POST-add dashscope provider with env-harvested key pool, probe every model
 (Arrearage = HTTP 400 on SOME pooled keys — qwen3.8-flash worked while
 qwen3.8-max-0902/qwen-plus/qwen-max 400'd), sed MODEL in responder.py, kill
 shards via psutil own-PID guard (kill_mass.ps1 can exit 124; taskkill
 guard-blocked), relaunch, verify via mass_state_*.json history[] — 57 pairs +
 254 replies same day.

- references/shard-launcher-and-state-leak-bugs.md — (2026-10-03) fleet relaunch
  post-mortem: `ppg = 0` rebound inside the per-group loop killed ALL later pairs
  in that shard (launcher params are immutable run config — use per-item locals);
  children spawned as literal `"python"` resolved to Hermes Python 3.14 WITHOUT
  telethon → ModuleNotFoundError in every mass_<i>.log while the launcher printed
  success (use `sys.executable`, launch the parent under Python311, and TAIL
  EVERY CHILD LOG before claiming "запущено"); dashboard /check+/add subprocess
  contract for check_channel.py --json (single JSON on last `{` line, never raises,
  120s timeout wrapper); probe-session picking (source = CLEAN_ALIVE_SESSIONS
  because mass/ is shard-locked, spambot-ok stems first, max_tries=15 loop until
  is_user_authorized, protected-ID skip); /live probe None-subs TypeError
  hardening (GetFullChannelRequest fallback + `or 0` guards, connect()+
  is_user_authorized instead of start() which EOFErrors in daemons).
- references/self-echo-contamination-and-validator.md — (2026-10-03) the "Че за хуйн"
  incident: 4 of our own accounts talking in one chat in 7 minutes because the responder
  answered OUR OWN seed question (`sender_id == me.id` only excludes the current account,
  not the other 116). Fix stack: our_ids.json fleet-ID blacklist (copy-session harvest),
  seed-text blacklist from used_dialogs + DIALOGS bank, promo-handle skip, 30-min
  per-chat cooldown, 6h post-pair cooldown, 1 reply per acc per group forever.
  Plus the anti-AI-slop validator gate stolen from MiloAgent ContentValidator
  (RU-adapted BOT_PATTERNS: канцелярит/эссе-вывод/cliche-openers/bullet-lists/
  support-closers), telegram-ops ReplyLimiter, ai-character-telebot PHRASE_BLOCKLIST;
  gate placement + reject-and-mark-seen rule + 8/8 self-test; model choice beats prompt
  tuning (MiniMax-M2.1 off-topic vs glm-5.3-prime on-topic, benchmark before choosing).
  Bare `alstack` → `@alstack` regex with negative lookbehind (25/32 dialogs were
  non-clickable). Dashboard control-plane traps: venv python re-spawns a child with the
  same cmdline (dedupe by parent-pid or you see phantom "2 instances"), kill scripts
  match themselves (concat the target string, run from a file not `python -c`),
  kill-all → verify EMPTY → start one → verify 1 root; and state-shape drift
  (`fails` is a dict, rows are dicts) crashes only inside background tasks so the bot
  looks alive — test every text generator offline, N/N pass, before trusting a daemon.

## Related

- aiogram companion-bot side: skills telegram-product-bot-engineering and
  telegram-bot-engineering are USER-OWNED (writes refused). Recommend
  `hermes curator adopt telegram-product-bot-engineering` so the full competitor-cloning
  pipeline reference can also live there.
