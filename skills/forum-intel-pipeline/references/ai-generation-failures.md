# AI Generation Failures

ALL AI generation attempts for the forum digest FAILED. Manual no-AI formatting is the standard.

## Failed attempts

### deepseek-v4-pro (dashscope-rotator :16432)
- **Prompt >2000 chars**: Returns empty content, only reasoning_content
- **Prompt 500-800 chars**: Returns content but often "Ссылка" as only text
- **Prompt <500 chars**: Works, but too short for digest
- **Verdict**: UNUSABLE for digest generation

### glm-5.2 (dashscope-rotator :16432)
- **Prompt >700 chars**: Returns empty content
- **Prompt <500 chars**: Works, returns good Russian text
- **Prompt 526 chars with 5 items**: Works, 1072 chars output
- **Prompt 821 chars with 8 items**: EMPTY content
- **Verdict**: Usable for short prompts only. Too fragile for production.

### Why AI generation failed
1. Russian-language generation is weak on both models
2. Reasoning models (deepseek) put everything in reasoning_content, not content
3. Vlad rejected posts as "кал" — "Ссылка" as only text, too generic
4. Manual format is faster (no API latency) and more reliable

## Manual No-AI Digest Format (CURRENT)

```python
digest = ""
for cat, cat_items in cats.items():
    digest += f"<b>{cat}</b>\\n"
    for item in cat_items[:5]:
        title = item.get("title")[:130]
        desc = item.get("description")[:250]
        url = item.get("url")
        # Strip HTML, escape entities
        desc = re.sub(r'<[^>]+>', '', desc)
        desc = desc.replace('&', '&amp;')
        digest += f"▸ <b>{title}</b>\\n  {desc}\\n"
        digest += f"  <a href='{url}'>🔗</a>\\n\\n"
```

## Do NOT attempt to switch back to AI generation
Vlad explicitly rejected AI posts. Manual format is the standard.