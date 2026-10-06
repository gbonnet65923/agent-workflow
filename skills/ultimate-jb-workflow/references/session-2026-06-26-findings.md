# Session 2026-06-26 — Key Findings

## DS V4 Pro Reasoning Pitfall (DashScope)

- Model: deepseek-v4-pro via dashscope.aliyuncs.com
- **CRITICAL**: reasoning_content + content share the SAME max_tokens budget
- If max_tokens < 2000, ALL tokens go to reasoning → content is EMPTY
- Fix: max_tokens >= 2000 for DS V4 Pro
- Verify: check `usage.completion_tokens_details.reasoning_tokens` vs total
- 5 rapid requests confirmed: no rate limits, no 429

## DS Flash vs JB Prompts

- DS Flash on DashScope: 100% ASR on DIRECT requests (52/52 tests)
- JB prompts from GroveMind: REDUCE ASR to 66%
- Reason: JB prompts contain trigger words in system instructions
- Best approach for DS Flash: defensive framing ("EDR test suite")

## premium.unjail.ai Authentication

- Uses Supabase (project: bdpfsrtntrzlkefukxrx.supabase.co) + Discord OAuth
- Cookies: sb-bdpfsrtntrzlkefukxrx-auth-token.0/.1 + th_chunk_gate
- Cookie format: base64-encoded JSON containing full Supabase session
- Access token JWT lives inside the base64 cookie value
- The site is Next.js App Router — API routes require cookie-based auth, not Bearer tokens
- To scrape: Playwright with cookies pre-injected, then navigate to framework pages
- Pages need JS rendering — curl/web_extract won't work

## 201 LiteLLM Servers — ALL CLOSED

Tested 10+ supposed "open" LiteLLM proxies — all require API keys or are blocked:
- litellm.aihubmix.com — requires key
- llm.chatgot.io — requires key
- api.siliconflow.cn — requires key
- api.deepinfra.com — requires key
- api.together.xyz — requires key
- router.huggingface.co — returns HTML
- api.fireworks.ai — requires key
- api.novita.ai — requires key
- api.aimlapi.com — requires key
- api.studio.nebius.ai — requires key

## DeepSeek Web Interface

- chat.deepseek.com — FREE, unlimited, but blocked via CloudFront on datacenter IPs
- platform.deepseek.com — registration requires browser (blocks headless Playwright)
- Official API: 5M free tokens per account, no credit card

## Puter.js

- Tutorial: developer.puter.com/tutorials/free-unlimited-deepseek-api/
- OpenAI-compatible endpoint: api.puter.com/puterai/openai/v1/chat/completions
- Requires Puter auth token (user-pays model)
- Supported models: deepseek-v4-pro, deepseek-v4-flash, deepseek-v3.2, etc.
- Context: 1M tokens, max output: 384K tokens, speed: 78 tok/s