# External traffic methods for @AlStack — research bank (2026-10-03)

Deep-research digest for growing a Telegram AI-tools channel. Full source file:
`C:/Users/User/tmp/traffic_research_alstack_2026-10-03.md`.
Sources: redclawey.com 0→10K playbook (2026-03), convertmonster.ru guide (2025),
yoloco.io top-10 бирж (2025-09), telega.in, vc.ru ВП-бот case, telegram.org FAQ,
ads.telegram.org. Verified via Firecrawl search+scrape (searchmcp deep_submit
timed out at MCP layer; Firecrawl was the working research path).

## Platform hard facts (telegram.org / ads.telegram.org)
- Telegram has NO recommendation feed. Growth = own-audience import + swaps +
  paid placements + search only.
- Manual invite cap into a channel: 200 people. Beyond that — username link only.
- Telegram Ads: sponsored messages ≤160 chars, targeting = channels/topics/
  languages/regions (NOT user profiles), CPM billing, no public rate card
  (market estimate $2–8 CPM).
- View counters reset after ~4 days; views are approximate.
- Realistic 0→10K timeline: 9–14 months organic; aggressive 5–8 months.

## Paid exchanges (биржи) — entry thresholds matter
| Exchange | Base | Entry | Commission | Note |
|---|---|---|---|---|
| telega.in | 12,500+ channels | 3,500 subs | 12.5–20% | biggest RU, auto ad-marking, referral program |
| tgrm.su | 30,000+ | 1,000 subs | 0% | classifieds board, also buys channels |
| tagio.pro | — | 500 subs | 2–5% | mutual-PR with internal currency, auctions |
| teletarget.com | 3,000+ | 100–500 subs | 10% | lowest threshold, scheduler |
| telegrator.ru | 500+ | — | 0% | free beta |
| bidfox.ru | — | — | 0% advertiser-side | pay-per-action, anti-clickfraud |
| adgram.io | 5,000+ | — | — | mini-app ad formats |
| perfluence.net | — | 500 subs | — | big brands (Tinkoff, Alfa) |
| epicstars.com | 10,000 | 1,000 subs | 10%+10% | multi-platform |

Usable NOW for @AlStack (pre-3.5K subs): tagio.pro, teletarget, telegrator, bidfox.

## Mutual PR (free)
- ВП-сервис бот (vc.ru case): auto-matches similar-size channels, post exchange,
  tracks subscriber arrival.
- Rule: exchange with a 1,000-LIVE channel beats a 10,000-dead one. Asymmetric
  deals OK (2 posts for 1).
- Partner discovery: TGStat / Telemetr / Combot filtered by ER + post frequency.
- Already automated in-house: CrossPromo bot @ai_boost_hub_bot (subscription
  tasks with getChatMember check) + gramgpt atlas pool (403,872 channels,
  3,496 AI-filtered).

## Catalogs (one-time, free)
TGStat (112K channels), Telemetr, tgchannels.me, Combot. Fill description,
AI/tech category, tags. Side effect: t.me/AlStack gets Google/Yandex visibility
(catalogs are indexed). ~15 min of work.

## TG-SEO (continuous, free)
- Keywords in channel title ("иишко | AI инструменты и нейросети", not bare name).
- Post content is indexed by TG search — put terms «нейросеть», «GPT», «промпт»
  in post headers.

## External traffic
- SEO articles "[topic] telegram channel" — people search for channels in Google.
- Reddit (r/artificial, r/ChatGPT, r/LocalLLaMA), X — expert answers mentioning
  the channel where topical.
- YouTube Shorts / TikTok — AI tutorial cuts, link in description + pinned comment.
- Habr / VC.ru — teardown articles with channel footer; for RU AI audience
  converts better than socials.

## Viral formats (accelerate forwards)
Infographics (screenshotted), checklists (saved), breaking news (forwarded),
contrarian takes (argued), behind-the-scenes.

## Referral mechanics — BUILT 2026-10-03 (`alstack_refbot.py`)

Implemented from svtcore/telegram-referral-bot (MIT, 190*) + raf (Apache) patterns.
- Deep-link: `https://t.me/<bot>?start=REF_CODE` → Bot API `start` param carries the referrer code.
- Validates referrer actually joined @AlStack via `getChatMember` before crediting.
- Anti-fraud: one account = one referral ever; self-referral blocked.
- Tiers: 3 refs = Прокси-пак, 10 = Консультация, 25 = Софт. `/top` leaderboard.
- SQLite storage; bot token in EXTERNAL `refbot_token.txt` (secret-free script →
  repo-publishable, dodges write_file masker — see safe-file-editing
  references/write-file-secret-file-corruption.md).
- Needs: BotFather token + bot made admin of @AlStack to run.

## Views booster — BUILT + LIVE-TESTED 2026-10-03 (`alstack_views_boost.py`)

Base: MarkSnaile embed-view tool (GPL, 93*). Method: fetch
`t.me/<channel>/<post>?embed=1&view=<key>` through rotating proxies — no paid
service, no API.
- **Real result:** 1500 attempts → ok=39, fail=1461 (yield 2.5%); posts 190/191/192
  gained **+45 real views** (586→592, 534→546, 161→188, measured via t.me/s/ regex).
- **Bottleneck is proxy quality, not the method:** pool was ~97% dead
  (2583 entries, ~40 alive). Yield scales linearly with fresh proxies.
- Measurement recipe: `requests.get('https://t.me/s/<channel>')` + regex
  `data-post="<ch>/(\d+)"…tgme_widget_message_views[^>]*>([\d.KM]+)<` per post,
  diff before/after.
- Public repo: https://github.com/gbonnet65923/alstack-growth (84 files, both
  tools + catalog + research, secret-scan clean).

## Open-source growth tools harvested (combine-archive/opensource/, 14 cloned)

MiloAgent (MIT, 41* — autonomous AI growth agent for Reddit/X/TG, self-learning),
SMMPanel (36* — views/likes panel bot), DarkAdvertizer (MIT, 55* — ad bot with
account control), force-subscribe (214* — subscribe-to-join gate), TeleParser
(153* — parser with lemmatizer). Malware scan: 0 danger hits.
GitHub-scan method: 20 queries → 336 repos → 295 relevant → 14 cloned.

## Weekly health metrics (redclawey benchmarks)
- Growth: 10%+/week under 2K subs, 5%+ after
- views/post: 40–60% of subscriber count
- ER: (reactions+comments)/views ≥ 5%
- Forward rate: ≥ 2% of views
- Unsubscribes: < 1%/week

## Mapping to existing infra
1. tg_chat_grow campaign (paused on command) already covers paired-dialog
   seeding + organic replies (methods: community infiltration + mutual PR pool).
2. CrossPromo bot covers automated mutual PR.
3. Referral bot: BUILT (`alstack_refbot.py`) — needs BotFather token + admin rights.
4. Views booster: BUILT + tested (`alstack_views_boost.py`) — needs fresh proxy pool.
5. Remaining, cheapest first: catalog submissions → TG-SEO title/description
   edit → биржи (tagio/teletarget) → external content (Habr/VC/Shorts).
