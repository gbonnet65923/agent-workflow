# Discourse Forum Discovery & Scraping (2026-07-22)

## 45+ Forums Discovered via Small-Forums-List + Manual Probing

Full list with Discourse API status, platform, and access method.

### Discourse API (curl-accessible, no auth needed for public data)

```
nodeloc.com              → 337 AI abuse topics, JS-only auth
v2ex.com                 → Custom API, 26 AI topics
community.openai.com     → Jailbreak, bypass, Codex
discuss.huggingface.co   → 88 uncensored/jailbreak topics
community.fly.io         → LLM API abuse
forums.developer.nvidia.com → GPU/AI
forums.docker.com        → AI containers
discuss.python.org       → LLM/API
forum.openwrt.org        → Networking
techpowerup.com/forums   → Hardware/AI
forum.level1techs.com    → VPS/AI
community.ui.com         → Networking
forum.mikrotik.com       → Networking
discourse.ubuntu.com     → Linux
forum.manjaro.org        → Linux
forum.endeavouros.com    → Linux
forums.opensuse.org      → Linux
```

### Cloudflare Blocked (FlareSolverr or browser required)

```
linux.do                 → 🔥 #1 AI abuse forum, CF challenge
nodeseek.com             → VPS hosting, CF challenge
lowendtalk.com           → VPS, CF challenge
webhostingtalk.com       → Hosting business, CF challenge
namepros.com             → Domains, CF challenge
xss.is                   → Russian hacking, CF challenge
lolz.live                → Russian marketplace, CF challenge
zelenka.guru             → Russian marketplace, CF challenge
community.cloudflare.com → CF challenge
forums.linode.com        → 403 forbidden
```

### Other Platforms (Discuz!/XenForo/custom)

```
hostloc.com              → Discuz! Chinese VPS/proxy community
cn.naixi.net             → Discuz! Chinese MJJ community
dalao.net                → Custom, domains/VPS
right.com.cn             → Discuz! 恩山, routers/OpenWrt
lowendspirit.com         → Custom, VPS budget
4pda.to                  → Custom, Russian mobile (403)
forum.ru-board.com       → Custom, Russian IT/hacking
leak.sx                  → Custom, data leaks
raidforums.com           → Custom, data leaks
lobste.rs                → Custom, invite-only programming
tildes.net               → Custom, invite-only tech
```

## Curl-Based Discourse Scraping Pattern

All Discourse forums expose public JSON API at standard endpoints. No auth required for reading.

```bash
# Latest topics (30 per page)
curl -s 'https://forum.com/latest.json'

# Topic content (full post stream)
curl -s 'https://forum.com/t/{topic_id}.json'

# Search (URL-encoded query)
curl -s 'https://forum.com/search.json?q=KEYWORD'

# User profile check (404 = user doesn't exist)
curl -s -o /dev/null -w '%{http_code}' 'https://forum.com/u/{username}'
```

## NodeLoc Abuse Keywords (high-yield for scraping)

These Chinese keywords return 50+ topics each:
- 免费 API, 白嫖,  халява,  абуз,  обход, 焚诀
- Codex, Grok, Claude, GPT, K12, DeepSeek
- 注册机, 接码, SMS, верификация
- 企业,  корпоративный,  триал, trial
- 中转,  прокси, API proxy

## Discovery Source

- **small-forums-list**: `github.com/Amiyadesi/small-forums-list` — 49+ niche forums, categorized
- **Online page**: `https://forums.cc.cd/`
- **Categories**: Tech/AI, VPS, Security/Reverse, ACG, Hardware, Adult/Grey, Old Web