# Forum Scraper v3 — MEGA (2026-07-19)

## Overview

v3 is the current production scraper on eni-nyc1. 40+ sources, GitHub token rotation, API key extraction, Reddit, Lobsters.

## Deployment

**Server**: eni-nyc1 (134.209.72.186)
**SSH**: `ssh -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none root@134.209.72.186`
**Script**: `/root/forum_scraper_v3.py` (16.5 KB)
**Local dev copy**: `C:\Users\User\AppData\Local\Temp\forum_scraper_v3.py`

## GitHub Queries (30 total)

### AI jailbreak / bypass (7)
- `llm+jailbreak+bypass+2026`
- `prompt+injection+jailbreak+claude`
- `gpt+bypass+uncensored+api`
- `claude+codex+bypass+skills`
- `ai+red+team+pentest+llm`
- `jailbreak+prompt+collection+2025`
- `grok+jailbreak+bypass+2026`

### AI agents / MCP / tools (7)
- `claude+code+skills+plugin+mcp`
- `codex+plugin+mcp+server+tool`
- `ai+agent+skills+tool+automation`
- `mcp+server+tool+openai`
- `cursor+windsurf+opencode+plugin`
- `cline+codex+agent+skills`
- `copilot+agent+extension+plugin`

### Free API / keys (6)
- `free+api+key+llm+2026`
- `free+chatgpt+api+key`
- `openai+api+key+free+github`
- `free+claude+api+access`
- `reverse+proxy+openai+free`
- `llm+api+gateway+free`

### AI abuse / hacking (3)
- `hack+ai+prompt+injection+tool`
- `ai+security+vulnerability+exploit`
- `llm+red+team+attack+tool`
- `ai+malware+generator+llm`
- `deepseek+abuse+bypass`

### Telegram / bots (3)
- `telegram+bot+ai+automation+2026`
- `telegram+userbot+ai+spam`
- `telegram+scraper+ai+intel`

### Darknet / OSINT (3)
- `darknet+ai+tool+llm`
- `osint+ai+recon+tool`
- `hacking+tool+ai+2026`

## Other Sources

| Source | Limit | Method |
|--------|-------|--------|
| HN /newest | 40 | Firebase API |
| 4chan /g/ | 30 | catalog.json |
| 4chan /b/ | 30 | catalog.json |
| v2ex hot | 20 | /api/topics/hot.json |
| Reddit (13 subs) | 15 each | /.json — BLOCKED from DO |
| Lobsters | 20 | /hottest.json |
| GitHub Code Search | 6 queries × 8 | Search API — rate limited |
| Free API Key Repos | 8 repos | Raw README — key extraction |

## Reddit Subreddits (13)

ChatGPTJailbreak, ClaudeAIJailbreak, LocalLLaMA, OpenAI, ClaudeAI, hacking, netsec, ArtificialInteligence, MachineLearning, PromptEngineering, AI_Agents, MCP, LLMDevs

## GitHub Token Handling

- `GH_TOKENS` list at top of script — add real `ghp_*` tokens here
- Auto-fallback: if token returns 401, switch to unauthenticated
- Unauthenticated limit: 60 req/hr → after ~15 queries, 403 rate limit
- With 150 valid tokens: 750K req/hr → all 30 queries complete every cycle

## API Key Extraction

Scans 8 known free-API-key repos for patterns:
- `sk-[A-Za-z0-9]{20,}` (OpenAI keys)
- `ghp_[A-Za-z0-9]{36}` (GitHub tokens)
- `forge-[A-Za-z0-9]{32,}` (Echogate keys)
- `fp_live_[A-Za-z0-9_-]{30,}` (Firepass keys)
- Generic `apiKey` / `Bearer` patterns

Keys are masked to `sk-abc...xyz` format before storing.

## Deploy Procedure

```bash
# 1. Write script locally
# 2. SCP to server
scp -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none \
    /c/Users/User/AppData/Local/Temp/forum_scraper_v3.py \
    root@134.209.72.186:/root/forum_scraper_v3.py

# 3. Kill old scraper (separate SSH call to avoid killing self)
ssh -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none \
    root@134.209.72.186 "pkill -f forum_scraper"

# 4. Free port + start
ssh -i ~/.ssh/do_access -o ControlMaster=no -o ControlPath=none \
    root@134.209.72.186 "fuser -k 8000/tcp 2>/dev/null; \
    nohup python3 -u /root/forum_scraper_v3.py --serve > /root/forum_scraper_v3.log 2>&1 &"

# 5. Verify
sleep 90 && ssh ... "curl -s http://localhost:8000/health"
```

## Pitfalls

- **GitHub token 401**: all 30 queries return 0. Check `GH_TOKENS` list — token is dead.
- **GitHub unauthenticated 403**: rate limit hit. Wait 1 hour or add valid tokens.
- **Reddit 0 items**: DO IP blocked. All 13 subreddits return empty.
- **SyntaxError after patch**: python3 syntax check before deploy: `python3 -c 'import py_compile; py_compile.compile("/root/forum_scraper_v3.py", doraise=True)'`
- **SSH session killed by own pkill**: `pkill -f forum_scraper` matches the SSH command string. Use two separate SSH calls.