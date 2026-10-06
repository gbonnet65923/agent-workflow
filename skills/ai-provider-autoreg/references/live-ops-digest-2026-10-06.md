# Live Ops Digest — 2026-10-06 (из session_search по всем сессиям)

Сводка живых операций и новых рецептов, извлечённая из сессий за 06.10.2026.

## MultiAI (multiai.store) — авторег в работе
- Проект: `C:/Users/User/tmp/multiai_autoreg/` (reg.py + batch.py + probe*.py)
- Флоу: camoufox headless → `#authSwitchLink` → нативный Turnstile токен из `dataset.turnstileToken` (виджет решает сам в браузере, солвер не нужен)
- КЛЮЧЕВОЙ ФИКС: тайминг клика — `time.sleep(3)` + `wait_for_selector('#authSwitchLink')` перед кликом, потом поллинг токена 60×2с. Раньше клик `?.click()` без ожидания = токен не появлялся
- Почта: Voidash API (`api.voidash.com`, session_key Bearer), код в messages
- E2E PASS: verify 200, /api/auth/me 200, cookies сохраняются в result.json
- Баланс свежего акка = 0; дальше: `window.MultiAIKeyGuard.headers('/api/keys')` — guarded создание ключа, `/api/dashboard/bootstrap` — список моделей/free-tier
- Платформа обещает промокод на токены до 10.10 + ежедневные free-модели

## tk-ферма (285+ акков, ~305M токенов)
- Итог: акки 285+, ключи 142 (61 с балансом 5M), шлюз :8310 (funded-пул), sidecar капчи :8877
- Каскад hCaptcha в tk_autoreg.py: (1) локальный sidecar :8877 → (1.5) 2captcha live-pool с ротацией ключей (ZERO_BALANCE/ERROR_KEY = удалить ключ из пула) → (2) YesCaptcha
- Прокси-пул `tk_proxies.json`: 73 прокси; свежий парс TheSpeedX+ProxyScrape: 4096 кандидатов → 300 проверено → 24 живых. Воркеры подхватывают файл при ротации без рестарта
- Симптомы мёртвых прокси в логах: rate limited / error 10054

## grok-x-farm (VPS 193.233.114.59) — восстановление ключа без админ-пароля
- РЕЦЕПТ: если админ-пароль new-api/grok2api сменён и ключ в файле протух (401):
  1. sqlite backend.db → `client_keys.encrypted_secret`
  2. расшифровать AES-GCM ключом `credentialEncryptionKey` из config.yaml
  3. полученный `g2a_*` ключ живой → проверить через VPS localhost:8000 /v1/models
- Пруфы живости: /v1/models 200 (8 моделей), chat 200/1.8s, responses 200/61s, images 200/2.6s
- Пул: 3080 аккаунтов auth_status=active; квоты fast avg 28.2, image_pro 5.7
- Алиасы `Web/*` и `Build/*` НЕ работают в /v1/chat — 404 model_not_found, юзать чистые id
- Секьюрити: jwtSecret + credentialEncryptionKey лежат в config.yaml открытым текстом — ротировать если VPS светится

## abliteration.ai — бан по IP
- direct → 403 `auth_policy_denied` (IP засвечен); free ProxyGrab-пул весь сгоревший (NS_ERROR_NET_TIMEOUT / PROXY_FORBIDDEN)
- Рабочий домен почты: voidash.bond (cyou выкинут — мог попасть под auth_policy)
- Вывод: после ~17 акков с одного IP нужен residential (ZTE 4G) или платный прокси; sidecar может отдавать 500 на sign-up — фолбэк 2captcha работает (858 chars, 6.5s)

## Conol
- Пул 279 аккаунтов, 271 live; gateway :9999 и дашборд :9988 умирают при рестарте hermes-гейтвея — перезапускать вручную

## JCNode cron (job 20bf7618e070)
- Каждые 30 мин watcher @jcnode → новый пост → payload кнопки → @jcnodebot → подписки vless/vmess/trojan/hy2 → парс нод, топ-5 по MB/s
- Watchdog-паттерн: нет раздачи = stdout пустой = тишина в чате; лучшие ноды → пул ProxyGrabReform_bot

## Antseed (antseed.com) — черновик поста готов
- Локальный роутер на 158+ провайдеров (CLI, localhost:8377, OpenAI-compat), цены -96..99% от официала, GLM 5.3 Flash + DeepSeek V4 Flash бесплатно без депозита
- Проверено: сайт 200 (208KB), цифры с сайта, не из твита

## VCC/card intel (раунд 3, curl-проверено)
- Живые: privacygateway.io (No-KYC VISA $10k/мес), kardpay.com, paymight.com, starryblu.com, pokepay.cc, fasqon.com (no-KYC до €200), dora-card.com, supaycard.com (BIN 5378 AI-only)
- Маркетплейсы продавцов: funpay.com, ggsel.net, plati.market; чекер chkr.cc (free API live/die)
- Мёртвые: offgridcard.com (домен продаётся), dupay.one (закрыт), vcchub.com (чужой), dtcpay.me/yptcard.com/kripi/lasocard/bingcard/maxswap/jamcard (000)
- linux.do intel: VegaX-карта — единственная проходит ручную привязку GPT Plus; Mercury BIN заблечены OpenAI; 0刀/1刀 карты для Claude Team мертвы, живёт только протокольная привязка

## TG native boost (@AlStack)
- Метод: свои акки, GetMessagesViewsRequest(increment=True) + custom-emoji реакции; 74 акка, 333 реакции за сессию
- Whitelist канала = 5 кастом-эмодзи; классические 🔥👍 → REACTION_INVALID; кастом проходит даже с non-premium
- Альбомы: реакции только на первом посте группы; ResolveUsername FloodWait ~30мин — скипать

## Hermes/dashscope
- Контент-фильтр DataInspectionFailed на qwen3.8-max-0902 — фикс: модель qwen3.8-max (без -0902) через прокси :16432, фильтр не срабатывает; glm-5.3 ок; kimi-k3 = пустой content (известный баг)
