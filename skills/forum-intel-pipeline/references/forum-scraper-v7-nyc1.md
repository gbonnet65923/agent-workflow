# Forum Scraper v7 — 2026-07-19

## Deployment

- **Server**: eni-nyc1 (134.209.72.186)
- **SSH**: `ssh -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none root@134.209.72.186`
- **Script**: `/root/forum_scraper_v7.py` (24,640 bytes)
- **Data**: `/root/forum_scraper_data/forum_data.json`
- **Log**: `/root/forum_scraper_v7.log`
- **Cron**: `*/15 * * * * cd /root && python3 /root/forum_scraper_v7.py --once >> /root/forum_scraper_v7_cron.log 2>&1`
- **HTTP**: `:8000` → `/forum_data.json`, `/health`, `/stats`

## Sources (80+)

### GitHub Trending (HTML — NO API limits)
- 12 languages: "", python, javascript, typescript, go, rust, java, c++, csharp, ruby, swift, kotlin
- 2 periods: daily, weekly
- 196 repos per cycle
- Page: `github.com/trending/{lang}?since={period}`
- Parse: `soup.select('article.Box-row')` → `h2 a` (repo), `p` (desc), `span.d-inline-block` (stars)

### GitHub Search (HTML — requires JS, BROKEN)
- 20 queries attempted, all return 0 (JS-rendered page)
- GitHub search requires JavaScript rendering
- Only Trending page works as plain HTML

### 4chan
- Boards: g, b, pol, sci, x
- API: `a.4cdn.org/{board}/catalog.json`
- 150 threads per cycle
- Filter: only AI/tech threads in bridge

### RSS Feeds (25 sources)
- hackersnews, schneier, krebs, thehackernews, bleepingcomputer
- therecord, csoonline, darkreading, threatpost, gbhackers
- securityaffairs, latesthackingnews, hackread, cybersecinsiders
- infosecurity, nakedsecurity, zdnet_security, wired_security
- arstechnica_security, theregister_security, helpnetsecurity
- scmagazine, cybernews, securityboulevard

### Other
- HN: `hacker-news.firebaseio.com/v0/newstories.json` (50 items)
- Lobsters: `lobste.rs/hottest.json` (25 items)
- dev.to: 11 AI tags, HTML scraping (20 articles)
- ArXiv: `export.arxiv.org/api/query` XML (20 papers)
- v2ex: `v2ex.com/api/topics/hot.json` (9 items)
- Free API keys: 7 GitHub repos, raw.githubusercontent.com (BLOCKED from DO)

## Dead Sources (2026-07-19)

### Darknet Forums
- breached.to → seizedservers.com (FBI seizure)
- raidforums.com → dead
- dread.net → timeout
- exploit.in → timeout
- xss.is → timeout
- cracked.io, sinister.ly, nulled.to, leak.sx, cracking.org → all timeout

### Tavily API
- 15 keys from autoregs: ALL return HTTP 401
- Free tier: 1,000/month, sign up at tavily.com

### Reddit
- All 16 subreddits: 0 items (DO IP blocked)

### Medium
- 8 tags: 0 items (DO IP blocked)

### GitHub Search HTML
- 20 queries: 0 items (JS-rendered page, not HTML)

## Launch Procedure

```bash
# 1. Deploy
scp -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none \
    /c/Users/User/AppData/Local/Temp/forum_scraper_v7.py \
    root@134.209.72.186:/root/forum_scraper_v7.py

# 2. Kill old + start new
ssh -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none root@134.209.72.186 \
    'pkill -f "forum_scraper" 2>/dev/null; sleep 2; fuser -k 8000/tcp 2>/dev/null; sleep 2'

ssh -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none root@134.209.72.186 \
    'nohup python3 -u /root/forum_scraper_v7.py --serve > /root/forum_scraper_v7.log 2>&1 &'

# 3. Verify
ssh -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none root@134.209.72.186 \
    'sleep 120; curl -s http://localhost:8000/health; curl -s http://localhost:8000/stats'

# 4. Update cron
ssh -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none root@134.209.72.186 \
    'echo "*/15 * * * * cd /root && python3 /root/forum_scraper_v7.py --once >> /root/forum_scraper_v7_cron.log 2>&1" | crontab -'
```

## Stats (2026-07-19)
- Total seen: 1,993
- Items per cycle: 513
- GitHub Trending: 196
- 4chan: 150
- HN: 50
- RSS: 57
- Lobsters: 25
- dev.to: 20
- ArXiv: 6
- v2ex: 9