# Bridge v11 Filter & Scoring — 2026-07-19

## SMART is_ai() Filter

Source-aware filtering to prevent news articles from leaking into the feed.

```python
def is_ai(item):
    text = json.dumps(item).lower()
    source = item.get("source", "")
    
    # Skip RSS news unless abuse keywords
    if source.startswith("rss/"):
        abuse_kw = ["jailbreak", "bypass", "exploit", "api key", "free", "leak", "hack",
                     "sk-", "ghp_", "token", "reverse proxy", "uncensored", "unlimited",
                     "crack", "pirate", "warez", "nulled", "botnet", "malware", "0day",
                     "zero-day", "vulnerability", "cve", "rce", "backdoor", "phish"]
        return any(k in text for k in abuse_kw)
    
    # Always pass: github, dev_to, lobsters, v2ex, arxiv, api_keys
    if source in ("github", "github_trending", "github_search", "dev_to",
                   "lobsters", "v2ex", "api_keys", "arxiv"):
        return True
    
    # 4chan: only AI/tech threads
    if source.startswith("4chan"):
        tech_kw = ["ai", "llm", "gpt", "claude", "codex", "jailbreak", "bypass",
                    "model", "agent", "mcp", "plugin", "skill", "api", "token",
                    "python", "coding", "programming", "linux", "hack", "exploit",
                    "deepseek", "gemini", "grok", "kimi", "qwen", "openai",
                    "github", "proxy", "reverse"]
        return any(k in text for k in tech_kw)
    
    # HN: only tools/projects, not news
    if source == "hn":
        ai_kw = ["show hn", "github.com", "tool", "plugin", "mcp", "api", "jailbreak",
                 "bypass", "hack", "exploit", "reverse", "proxy", "open source",
                 "launch", "released", "cli", "self-hosted", "free", "extension",
                 "skill", "agent"]
        return any(k in text for k in ai_kw)
    
    # Fallback: basic AI keywords
    kw = ["claude", "codex", "gpt-5", "gpt-4", "openai", "chatgpt",
          "deepseek", "qwen", "gemini", "grok", "kimi", "llama",
          "jailbreak", "bypass", "mcp", "agent", "windsurf", "cursor",
          "opencode", "cline", "aider", "skill", "plugin"]
    return any(k in text for k in kw)
```

## SCORING Priority

```python
def score(item):
    s = 0
    source = item.get("source", "")
    text = json.dumps(item).lower()
    
    # MASSIVE priority for GitHub repos
    if source in ("github_trending", "github", "github_search"):
        s += 1000
        try: s += int(item.get("stars", 0) or 0) * 2
        except: pass
    
    # API keys = instant top
    if source == "api_keys": s += 2000
    if "api" in text and "key" in text: s += 500
    if "sk-" in text or "ghp_" in text: s += 500
    
    # Jailbreak/bypass
    if "jailbreak" in text or "bypass" in text: s += 400
    if "uncensored" in text or "unlimited" in text: s += 200
    
    # Tools/plugins
    if "mcp" in text and ("server" in text or "tool" in text): s += 300
    if "plugin" in text or "skill" in text: s += 200
    if "proxy" in text and "api" in text: s += 300
    if "reverse" in text and "proxy" in text: s += 300
    if "free" in text and ("api" in text or "key" in text or "access" in text): s += 300
    
    # Claude/Codex/Code tools
    if "claude" in text and "code" in text: s += 250
    if "codex" in text: s += 250
    if "opencode" in text or "windsurf" in text or "cursor" in text: s += 200
    
    # 4chan
    if source.startswith("4chan"): s += 100
    
    # HN with AI tools
    if source == "hn" and ("show hn" in text or "tool" in text or "github" in text): s += 150
    
    # RSS news — PENALIZE unless specific keywords
    if source.startswith("rss/"):
        s -= 500  # baseline penalty
        if "jailbreak" in text or "bypass" in text: s += 800
        if "api" in text and "key" in text: s += 800
        if "exploit" in text or "cve" in text: s += 500
        if "free" in text and "tool" in text: s += 400
    
    try: s += int(item.get("stars", 0) or item.get("score", 0) or 0)
    except: pass
    try: s += int(item.get("replies", 0) or 0)
    except: pass
    
    return s
```

## AI Post Generation — Anti-Empty Guard

Vlad rejected posts where AI just outputs "Ссылка". The guard:

```python
def call_ai(title, desc, url, stars):
    # ... API call ...
    if r.status_code == 200:
        content = r.json()["choices"][0]["message"]["content"]
        # ANTI-EMPTY: if content is just "Ссылка" or too short, skip
        clean = content.strip().replace('<a href="','').replace('</a>','').replace('Ссылка','').strip()
        if len(clean) < 30:
            log.warning(f"AI returned too short: {content[:100]}")
            return None  # skip this item
        if url and url not in content:
            content += f'\n\n<a href="{url}">Ссылка</a>'
        return content
    return None
```

## System Prompt

```
Ты автор Telegram-канала про AI/LLM инструменты, джейлбрейки, API-хаки и бесплатный доступ к нейросетям.

ЖЁСТКИЕ ПРАВИЛА:
1. ВСЕГДА пиши минимум 3-4 строки осмысленного текста. НИКОГДА не пиши просто "Ссылка".
2. Объясни ЧТО ЭТО, ЗАЧЕМ НУЖНО, и КАК ИСПОЛЬЗОВАТЬ.
3. Если это GitHub репа — опиши что она делает, какие модели/API даёт.
4. Если это API ключи — напиши какие модели доступны.
5. Если это джейлбрейк/обход — опиши технику.

ФОРМАТ:
<b>Жирный заголовок — суть</b>
2-4 строки текста: что это, зачем, как работает.
<a href="ССЫЛКА">Ссылка на источник</a>

СТОП-СЛОВА: "Ссылка" как единственный текст, "вот", "данный", "представляем", "стоит отметить", "Если пост был полезен"
НЕ ПЕРЕВОДИ: Claude Code, Codex, OpenCode, Windsurf, Cursor, Cline, Aider

АНТИ-ПУСТОТА: если не знаешь что писать — НЕ ПИШИ ВООБЩЕ. Лучше пропустить чем высрать "Ссылка".
```

## Current Config

- AI_URL: `http://localhost:16432/v1/chat/completions`
- AI_MODEL: `glm-5.2` (better Russian than deepseek-v4-pro)
- AI_KEY: `no-auth-needed` (localhost)
- DO_URL: `http://134.209.72.186:8000/forum_data.json`
- Posts per cycle: 10
- Cron: `a86a1f1490ab`, `*/15 * * * *`, no_agent=true