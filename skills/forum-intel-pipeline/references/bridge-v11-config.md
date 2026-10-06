# Bridge V11 Configuration (2026-07-19)

Path: `C:\Users\User\AppData\Local\hermes\scripts\bridge_v11.py` (174 lines, 5,459 bytes)

## Current Config

```python
DO_URL = "http://134.209.72.186:8000/forum_data.json"  # eni-nyc1 (was DO 188.166.189.92)
BOT_TOKEN = "8852099196:AAFSDgfbfy6IS8zKDizhPqurOBrHF0XxVlY"
CHAT_ID = 7448683285  # @ReformBoss
AI_URL = "http://localhost:16432/v1/chat/completions"  # dashscope-rotator
AI_KEY = "no-auth-needed"
AI_MODEL = "deepseek-v4-pro"
```

## Cron

Job `a86a1f1490ab` (forum-bridge-v6), `*/15 * * * *`, no_agent=true, script=bridge_v11.py, deliver=telegram:7448683285.

## Dedup

`C:\Users\User\AppData\Local\hermes\scripts\memory\sent_forum.json` — 49 KB, 1813 IDs sent (934 GitHub, 729 numeric, 66 4chan, 49 HN, 35 RSS).

## AI Provider History

| Date | Provider | Status |
|------|----------|--------|
| 2026-07-19 | dashscope-rotator :16432 | ✅ Active |
| 2026-07-06 | echogate gemini-3.1-pro (forge-fe5aa4...) | ❌ 401 dead |
| 2026-07-06 | echogate (forge-81081...) | ❌ 503 all models |
| pre-07-06 | firepass | ❌ Tailscale unreachable from DO |

## Post Format

Valency Labs style: bold title, short text, 🟡 steps, link, "всех обнял ❤️". Max 500 chars. HTML parse_mode. sendRichMessage with sendMessage fallback.

## Verification

```bash
# Manual run
cd C:\Users\User\AppData\Local\hermes\scripts && python -u bridge_v11.py

# Expected: "N new AI items" → "Done. N posts"

# Check sent count
python -c "import json; print(len(json.load(open('C:/Users/User/AppData/Local/hermes/scripts/memory/sent_forum.json'))))"
```