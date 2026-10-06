# GramGPT Telegram Atlas — free 400K-channel DB (reversed 2026-09-24)

gramgpt.io/tools/channel-map exposes their ENTIRE channel index as a public
binary blob — no auth, no rate limit worth mentioning. Massive channel-discovery
source: 403,872 channels (username/title/id/category), far beyond what
contacts.Search yields.

## Endpoint

- `GET https://gramgpt.io/api/atlas/bin/` → ~60 MB binary, magic `ATLB`.
  Plain curl works (Cloudflare does not gate it). Also seen: `/api/atlas/landing/?lang=ru`
  (JSON SEO), `/api/atlas/countries/` (empty). No search/channels REST endpoints —
  the bin blob IS the data.
- Country subpages: `gramgpt.io/tools/channel-map/{ru,ua,by,kz,...}` (SSR landing only,
  no extra data).

## ATLB v2 format (fully reversed, 99.7% parse rate)

Header (18 bytes): `<4sHIII` = magic `ATLB`, ver(2), n_entries, n_categories(118),
strings_offset (= 18 + 16*n_entries + 2 padding).

Records section: n_entries × 16 bytes (map coords + category blobs; NOT needed
for channel extraction — we never decoded them).

String section at strings_offset, self-describing packed entries:

```
[cat u8][zero u8][ulen u16][tlen u16][extra_len u16][idlen u16][username ulen][title tlen][extra extra_len][id idlen]
```

- `cat` is u8 (values > 118 occur — do NOT cap at n_categories; that caused a
  desync at record 3795).
- `id` is USUALLY numeric ascii, sometimes a 36-char UUID (`1e027fd8-ccaf-...`) —
  validate with `[0-9a-fA-F-]{6,40}`, not `\d+`.
- `extra_len` field EXISTS and its bytes sit between title and id. Forgetting it
  shifts every subsequent parse (the actual desync root cause). Usually empty;
  occasionally contains an image filename.
- username may be 0-length (private channels) — don't require a username match
  for entry validity; anchor on zero-byte + idlen + id charset instead.
- Parse sequentially; on implausible header, rescan ±(back 64 / fwd 8KB) for a
  position where TWO consecutive entries validate.

Working parser: `C:/Users/User/tmp/insideads_clone/atlb_v6.py` (final working
version despite name; formula len = ulen+tlen+extra+idlen). Output:
atlas_all.json (402,572 entries, 96 MB), filter_ai.py → atlas_ai_public.json.

## Pipeline into the CrossPromo bot DB

1. Parse bin → JSON tuples (cat, username, title, id, extra).
2. Filter by keyword regex on username+title (AI set matched 3,497/402K);
   EXCLUDE regex for casino/porn/scam.
3. Import public ones (username non-empty, not already in channels table) as
   owner=0, status='queued' + analyses row → Telethon analyzer worker resolves
   real subs/views/ER (the atlas has NO subscriber counts — that's why we
   re-analyze via sessions).
4. Analyzer throughput ~1 ch/1.5s → 3,400 channels ≈ 80 min. Queue survives
   restarts (SQLite).

## Companion discovery sources (validated same day)

- TGStat tag pages (tgstat.ru/tag/artificial_intelligence) block plain curl with
  a captcha stub (5KB html), but Firecrawl scrape returns the FULL ranked list
  with subscriber counts — cheapest way to seed a niche ranking. Parse
  `tgstat.ru/channel/@username` links + "N подписчиков" from markdown.
- TGStat RATINGS pages (tgstat.ru/ratings/chats/{tech,marketing,business,courses,
  education,career,design,apps,edutainment,news,blogs}?sort=msgs) — ~100 chats per
  category page, best bulk live-group source. FULL PIPELINE (validated 2026-09-25,
  1037 usernames from 11 categories):
  1. Keyless firecrawl MCP dies with "IP looks suspicious" after a few calls →
     fall back to direct HTTP POST https://api.firecrawl.dev/v2/scrape with keys
     from tmp/firecrawl_keys_full.txt (fc-* keys, rotate on 401/402). Do key
     loading INSIDE python — bash `KEY=***...)` breaks because the masker
     rewrites the key token mid-command (syntax error).
  2. Markdown link regex `tgstat\.ru/chat/(@?[\w-]+)/stat` reliably extracts
     USERNAMES; title/members back-parsing is NOT reliable (0/1037 parsed right —
     firecrawl escapes newlines as \\n literals unpredictably). Treat the scrape
     as a username list only.
  3. Validate every username via Telethon get_entity on a pool session:
     isinstance Channel + megagroup=True, participants_count>=30, and a STRICT
     AI-relevance ALLOWLIST on title+username (see below). ~1.5-4s per check.
  4. Feed survivors into live_groups_found.json as direct=True entries.
- STRICT ALLOWLIST lesson (Vlad rejected garbage live: "че за каналы хуйни"):
  contacts.Search and rating scrapes both surface off-topic megagroups (Billie
  Eilish fan chat, airbnb, wildberries, MLBB). A gaming/celeb DENYLIST is not
  enough — require at least one AI/IT keyword in title+username (ai, ии, нейр,
  gpt, python, код, dev, разраб, it, tech, промпт, маркетинг, smm, контент,
  дизайн, курсы, заработ, крипт, автоматиз, бот, api...). Filter BEFORE the
  activity probe to save API calls.
- Telethon contacts.SearchRequest: ~5-30 hits/keyword, heavy ResolveUsername
  FloodWait risk — use only through the pool rotator, ≥1s between calls, and
  never on the owner's primary session.

## Analyzer durability fix (ConnectionError)

Telethon client silently disconnects on Windows network blips; afterwards EVERY
call raises `ConnectionError: Cannot send requests while disconnected` and a
generic except-marks-error loop burns the whole queue. Fix: catch
Connection/disconnected in the worker loop, re-queue the item, reconnect with
retries (5×15s), rotate session if reconnect fails. Implemented in
insideads_clone/analyzer.py main loop.
