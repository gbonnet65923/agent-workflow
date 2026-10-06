# Session Pool Rotation + SQLite Queue Protocol

Canonical code proven live 2026-09-24 (CrossPromo analyzer.py / parser_ai.py).

## Pool + rotation

```python
import glob, json, os, shutil, time
POOL_DIR = 'pool'; STATE_FILE = 'pool_state.json'

def pool_sessions():
    os.makedirs(POOL_DIR, exist_ok=True)
    have = sorted(glob.glob(POOL_DIR + '/*.session'))
    if len(have) >= 8: return have
    for s in sorted(glob.glob('C:/Users/User/Desktop/CLEAN_ALIVE_SESSIONS/*.session'))[:8]:
        dst = POOL_DIR + '/' + os.path.basename(s)
        if not os.path.exists(dst): shutil.copy(s, dst)
    return sorted(glob.glob(POOL_DIR + '/*.session'))

async def make_client(state):
    now = int(time.time())
    state['flooded'] = {k: v for k, v in state.get('flooded', {}).items() if v > now}
    sessions = pool_sessions()
    for _ in range(len(sessions)):
        s = sessions[state['idx'] % len(sessions)]; state['idx'] += 1
        if os.path.basename(s) in state['flooded']: continue
        client = TelegramClient(s, API_ID, API_HASH); await client.start()
        return client, s
    wait = max(min(min(state['flooded'].values()) - now, 600), 10)
    await asyncio.sleep(wait)
    return await make_client(state)

# work loop:
try:
    result = await do_work(client, item)
except FloodWaitError as e:
    state['flooded'][os.path.basename(spath)] = int(time.time()) + int(e.seconds) + 60
    save_state(state)
    await client.disconnect()
    requeue(item)          # status back to 'queued' — never drop work
    client, spath = await make_client(state)
    continue
```

## Queue protocol (shared SQLite, WAL)

- Work table with `status IN ('queued','analyzing','done','error')`; worker claims one row:
  `SELECT ... WHERE status='queued' ORDER BY id LIMIT 1` then `UPDATE status='analyzing'`.
  Connect with `timeout=60`, `PRAGMA journal_mode=WAL` — the aiogram bot process and the
  Telethon workers share the same DB file.
- Crash recovery on startup: `UPDATE ... SET status='queued' WHERE status='analyzing'`.
- **Dedup predicate bug (real, cost a whole parser wave):**
  `WHERE username=? OR chat_id=?` with chat_id defaulting to 0 matches EVERY username-only
  row (all have chat_id=0) → nothing ever queues. Correct:
  `WHERE (username=? AND username!='') OR (chat_id=? AND chat_id!=0)`.
- Result push-back to the bot: bot process polls the results table every ~5 s
  (`status IN ('done','error')`), dedupes pushed ids in an in-memory set, sends cards.

## Keyword-wave discovery (parser)

- `contacts.SearchRequest(q=kw, limit=30)` in waves of ~10 ru/en keywords, 30–60 min cycle.
- Keep broadcast channels; skip megagroup/scam/fake flags.
- Dedup against DB (predicate above) BEFORE insert; insert catalog rows with owner=0 and
  queue one analyses row each.
- Wave 1 typically yields 30–70 new channels per 10 keywords; later waves converge.
- Size filtering (e.g. "small channels 200–10k"): search results carry no reliable
  participants_count — call `GetFullChannelRequest` per candidate and filter on
  `participants_count`. ~0.6 s pause per channel keeps the session alive.
- Niche keys beat broad keys for small channels: "нейросети для новичков",
  "промпты midjourney", "ai side hustle" surface the 200–10k tier that "chatgpt" misses.
  55 ru/en niche keys → ~120 new small channels per full pass (2026-09-24).

## Frozen/dead session classification (Rotator)

```python
except Exception as e:
    en = type(e).__name__
    if 'Frozen' in en or 'AuthKeyUnregistered' in en or 'UserDeactivated' in en or 'SessionRevoked' in en:
        self.dead.add(basename)        # PERMANENT — no expiry, never retry
    else:
        pass                            # transient — retry/rotate normally
```
Also check inside the per-keyword loop, not just at client creation: a session can freeze
mid-run (`FrozenMethodInvalidError` arrives from SearchRequest itself). On frozen error:
add to dead set, disconnect, `rot.get()` next client, `continue` to retry SAME keyword.

## Secondary source: TGStat via Firecrawl (when curl gets captcha)

- `curl tgstat.ru/tag/<topic>` returns a ~5 KB Cloudflare/captcha stub; 
  `firecrawl_scrape(url, formats=["markdown"], onlyMainContent=true)` returns the FULL
  tag ranking: channel name, description, exact subscriber count, `tgstat.ru/channel/@username`
  links. Parse the markdown into (username, title, subs) tuples and import the same way as
  search results (dedup vs DB, queue analyses).
- Useful tag URLs: tgstat.ru/tag/artificial_intelligence, and any /tag/<slug> from the
  TGStat tag catalog (tgstat.ru/tag/tg-catalogs lists catalogs, not channels).
- Yield 2026-09-24: 57 small AI channels (200–10k) from one tag page, 52 new after dedup.
- Combined pipeline proof: TG search 120 + TGStat 57 + prior DB 85 → 257 unique small
  channels, all run through the analyzer (views/ER/fair price), exported CSV+JSON,
  225/257 quality-alive (ER≥5%, views≥100).
