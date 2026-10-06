# Forum Intelligence Sources

## Core Forums (Chinese AI Community)

### linux.do
- URL: https://linux.do
- Type: Discourse
- Access: FlareSolverr (Cloudflare bypass)
- API: `/latest.json` (with FlareSolverr)
- Value: #1 source for Chinese AI tooling community. Claude Code, Codex, GPT, free API keys, jailbreaks.
- Telegram mirror: @linux_do_channel (26K subscribers)

### v2ex.com
- URL: https://www.v2ex.com
- Type: Custom (Python/Tornado)
- API: `/api/topics/hot.json`, `/api/replies/show.json?topic_id=X`
- Rate limit: 600 req/hour per IP
- Value: 827K members, 1.2M topics. AI, programming, VPS, free stuff.

### nodeloc.com
- URL: https://www.nodeloc.com
- Type: Discourse
- API: `/latest.json`, `/t/{id}.json`
- Auth: Account login (clayton/2t1wryyj2f0c)
- Value: VPS/ hosting, Claude Code, free API keys, K12 accounts.

### dc.hhhl.cc
- URL: https://dc.hhhl.cc
- Type: Misskey 2025.5.2
- API: `/api/notes/local-timeline`, `/api/meta`
- Auth: bobseven10@gmail.com / bobseven10@gmail.com (signin broken in 2025.x)
- Value: AI API中转 links, free keys, K12 tokens, affiliate links.

### hostloc.com
- URL: https://www.hostloc.com
- Type: Discuz! (Chinese forum)
- Method: HTML scrape
- Value: VPS/hosting deals, cross-posts with nodeloc.

## GitHub

### Trending (API)
- URL: https://api.github.com/search/repositories
- Query: `stars:>50 language:python sort:stars`
- Value: Top AI/LLM repos. Claude Code, Hermes, Anthropic Skills, LangChain.

### Free API Repos (monitored)
- guihuashaoxiang/FreeLLM-API-KeyHub — Chinese free LLM API list
- chatanywhere/GPT_API_free — ChatGPT API access
- PawanOsman/ChatGPT — ChatGPT reverse proxy
- xx025/carrot — Free ChatGPT API
- lss233/kirara-ai — AI chatbot framework
- cheahjs/free-llm-api-resources — Free LLM resources
- public-apis/public-apis — Public API collection
- open-free-llm-api/awesome-freellm-apis — 134+ free LLM APIs
- alistaitsacle/free-llm-api-keys — Daily free API keys (45 keys found!)

### GitHub Search (10 keyword queries)
1. free+llm+api+keys
2. claude+code+skills
3. ai+jailbreak
4. codex+plugin
5. openai+api+free
6. gpt+5+prompt
7. cursor+free+api
8. windsurf+api
9. copilot+free
10. mcp+server+tools

## English Tech

### Hacker News
- URL: https://hacker-news.firebaseio.com/v0/
- API: `/topstories.json`, `/item/{id}.json`
- Value: Top tech news. AI/LLM stories, Anthropic, OpenAI, Claude.

### 4chan /g/
- URL: https://a.4cdn.org/g/catalog.json
- Filter: AI keywords (ai, llm, claude, gpt, codex, api, key, jailbreak)
- Value: `/vcg/` (vibe-coding), `/lmg/` (local models), `/aicg/` (AI chatbots). Unfiltered opinions.

### Reddit (blocked from DO server)
- r/ChatGPTJailbreak — jailbreak community
- r/ClaudeAIJailbreak — Claude jailbreaks
- r/LocalLLaMA — local models
- r/ClaudeAI — Claude users
- r/OpenAI — OpenAI users
- Fix: Use FlareSolverr with session cookies

## RSS Feeds

| Feed | URL | Value |
|------|-----|-------|
| CyberSecurity News | https://cybersecuritynews.com/feed/ | AI security, jailbreaks |
| The Decoder | https://the-decoder.com/feed/ | AI news, Claude/GPT |
| AI News | https://www.artificialintelligence-news.com/feed/ | AI industry |
| Google AI Blog | https://blog.google/technology/ai/rss/ | Google AI |
| OpenAI Blog | https://openai.com/blog/rss.xml | OpenAI announcements |
| Anthropic Blog | https://www.anthropic.com/blog/rss.xml | Claude updates |

## Twitter/X (via Nitter RSS)

| Query | Nitter URL |
|-------|-----------|
| Claude Code API | https://nitter.net/search/rss?q=claude+code+api+key |
| Codex Free | https://nitter.net/search/rss?q=codex+openai+free |
| AI Jailbreak | https://nitter.net/search/rss?q=ai+jailbreak+bypass |
| LLM API | https://nitter.net/search/rss?q=llm+api+free+key |
| GPT Claude | https://nitter.net/search/rss?q=gpt+5+claude+fable |

## Additional Sources (to add)

- lobste.rs — Hacker News for security
- arxiv.org cs.AI / cs.CL — AI papers
- ProductHunt API — new AI tools
- YouTube RSS — AI channels
- Discord AI servers — via Discord API
- Telegram AI channels — via MTProto
- unjail.ai blog — TECH HAUS jailbreak community
- cybersecuritynews.com — AI jailbreak news