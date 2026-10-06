# Forum Scraper V2 — eni-nyc1 Deployment (2026-07-19)

## Deployed to

eni-nyc1 (134.209.72.186), path: `/root/forum_scraper_v2.py`, 7,680 bytes.

## Sources (simplified — no FlareSolverr needed)

| Source | Method | Items/cycle |
|--------|--------|-------------|
| GitHub API | 12 search queries, `api.github.com/search/repositories` | ~87 |
| Hacker News | `hacker-news.firebaseio.com/v0/newstories.json` | 25 |
| 4chan /g/ | `a.4cdn.org/g/catalog.json` | 25 |
| v2ex | `v2ex.com/api/topics/hot.json` | 8 |

## GitHub Queries

```
claude+code+skills+plugin
codex+plugin+mcp
llm+jailbreak+bypass
ai+agent+skills+tool
free+api+key+llm
hack+ai+prompt+injection
mcp+server+tool
claude+codex+cursor+opencode
deepseek+qwen+gemini+api
gpt+openai+reverse+proxy
telegram+bot+ai+automation
jailbreak+prompt+bypass+2026
```

## Deployment

```bash
# Deploy script
scp -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none \
    forum_scraper_v2.py root@134.209.72.186:/root/

# Install deps
ssh root@134.209.72.186 "pip3 install requests"

# Kill port 8000 occupant (snowflake_proxy.py)
ssh root@134.209.72.186 "kill $(ss -tlnp 'sport = :8000' | grep -oP 'pid=\K\d+')"

# Start serving (scrapes once then serves HTTP)
ssh root@134.209.72.186 "nohup python3 -u /root/forum_scraper_v2.py --serve > /root/forum_scraper.log 2>&1 &"

# Set cron
ssh root@134.209.72.186 "echo '*/15 * * * * cd /root && python3 /root/forum_scraper_v2.py --once >> /root/forum_scraper_cron.log 2>&1' | crontab -"

# Verify
curl -s http://134.209.72.186:8000/health
curl -s http://134.209.72.186:8000/forum_data.json | python3 -c "import sys,json; print(len(json.load(sys.stdin)))"
```

## Endpoints

- `GET /forum_data.json` — all scraped items as JSON array
- `GET /health` — returns "OK"
- `GET /scrape` — triggers async scrape

## Dedup

MD5 hash of each item (sorted keys). Hash stored in `/root/forum_scraper_data/seen_hashes.json`. Cache at `/root/forum_scraper_data/forum_data.json`.

## Pitfall: Port 8000 occupied by snowflake_proxy.py

eni-nyc1 has `snowflake_proxy.py` on :8000. Must kill it before starting scraper. PID discovery: `ss -tlnp 'sport = :8000'`.