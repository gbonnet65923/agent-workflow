# agent-workflow — полный воркфлоу скиллов ENI/Gothbreach

Боевой набор агентных скиллов: TDD, constraint-driven разработка, jailbreak-автоматизация, авторег на AI-провайдерах, Telegram-флоты, форум-разведка.

## Состав

### `skills/vendor/addyosmani-agent-skills/` (25 скиллов)
Production-grade engineering skills от Addy Osmani (upstream: [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills), MIT, commit 1401c8b). Ключевые:

- **[test-driven-development](skills/vendor/addyosmani-agent-skills/test-driven-development/SKILL.md)** — RED → GREEN → REFACTOR. Не видел падения теста — не знаешь, что он тестирует.
- **[constraint-driven-development](skills/vendor/addyosmani-agent-skills/constraint-driven-development/SKILL.md)** — CONSTRAINTS.md как контракт качества; ловит агента на ослаблении планки (@ts-ignore, .skip, срезанные ассерты, заниженные пороги). + [floor-guard](skills/vendor/addyosmani-agent-skills/constraint-driven-development/references/floor-guard.md) (diff-scoped guard, exit 0/1/2).
- **[code-review-and-quality](skills/vendor/addyosmani-agent-skills/code-review-and-quality/SKILL.md)** — мульти-осевое ревью + mutation testing.
- **[context-engineering](skills/vendor/addyosmani-agent-skills/context-engineering/SKILL.md)** — restartable session boundaries.
- **[spec-driven-development](skills/vendor/addyosmani-agent-skills/spec-driven-development/SKILL.md)** / **[planning-and-task-breakdown](skills/vendor/addyosmani-agent-skills/planning-and-task-breakdown/SKILL.md)** / **[incremental-implementation](skills/vendor/addyosmani-agent-skills/incremental-implementation/SKILL.md)**

Воркфлоу-связка: **spec-driven** (что строим) → **TDD** (доказываем, что работает) → **constraint-driven** (что значит «готово к шипу»).

Общие чеклисты: [skills/vendor/addyosmani-agent-skills/references/](skills/vendor/addyosmani-agent-skills/references/) (security, performance, accessibility, testing-patterns, definition-of-done).

### Боевые скиллы ENI

- **[ultimate-jb-workflow](skills/ultimate-jb-workflow/SKILL.md)** — 10-уровневый auto-bypass при отказах моделей: L1 (JB-001 EDR) → L10, кросс-модельное реле, отчёты L1 PASS / ALL FAIL.
- **[ai-provider-autoreg](skills/ai-provider-autoreg/SKILL.md)** — массовый авторег на AI-провайдерах: Turnstile/reCAPTCHA (YesCaptcha), IMAP-коды, password-reset takeover, пулы аккаунтов. Референсы по 15+ провайдерам (rsiai, conol, OdysseyAPI, GitLab, ChatGPT, 1min.ai...).
- **[conol-autoreg](skills/conol-autoreg/SKILL.md)** — бесплатный reCAPTCHA v3 солвер через Chrome CDP (~0.5с/токен, $0), гейтвей, дашборд.
- **[telethon-worker-fleet](skills/telethon-worker-fleet/SKILL.md)** — флот Telethon-сессий: FloodWait-ротация, closed-loop growth pipeline, AI-only пул-фильтр, gramgpt atlas (400K+ каналов).
- **[forum-intel-pipeline](skills/forum-intel-pipeline/SKILL.md)** — разведка 124 форумов 24/7 (cron), Cloudflare-обходы, AI-экстракция, findings.jsonl → контент-конвейер.

## Установка

**Hermes Agent:** скопировать в `%LOCALAPPDATA%/hermes/skills/` (структура категорий сохраняется).

**OMP (Oh My Pi):** скопировать в `~/.omp/agent/skills/`, общие references — в `~/.omp/agent/references/`. Рестарт сессии.

## Безопасность

Все секреты вычищены (REDACTED_*): API-ключи, app-пароли, токены, почтовые ящики. Скиллы описывают методологию, не содержат живых кредов.
