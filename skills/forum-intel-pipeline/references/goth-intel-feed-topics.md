# Goth Intel Feed — forum-topics digest bot (2026-09-10)

Форум-группа с топиками, куда бот постит развёрнутые дайджесты из forum_harvest данных.

## Инфраструктура

- Группа: **Goth Intel Feed**, channel_id `3927191915` (forum=True), создана через сессию R3fIex (`C:\Users\User\Desktop\CLEAN_ALIVE_SESSIONS\R3fIex_809951394.session`)
- Топики: Обсуждение(1), Трафик и темки(3), Кейсы и схемы(4), AI инструменты(5), Форум-интел(6), Инструменты и тулзы(7). Конфиг: `C:\Users\User\tmp\tg_setup\forum_config.json`
- Бот: @Hhhhhhhyyyyyyybot (8913247320), админ с manage_topics
- Пайплайн: `C:\Users\User\tmp\tg_setup\feed_bot\pipeline4.py` (WORKING), state4.json дедуп по md5(url)
- Данные: `C:\Users\User\tmp\forum_harvest\data\*.json` (листинги форумов, content = сырой HTML)
- Стиль-эталон: чат **Private Kylo** (-1003921209236, forum=True) — bold-заголовки, жирные цифры, абзацы, источники курсивом

## КРИТИЧНОЕ предпочтение Влада (2026-09-10)

**Посты-ссылки ЗАПРЕЩЕНЫ.** Первая версия постила `📌 *[заголовок](url)*` — реакция: «Максимально хуевые посты даун зачем ссылки удали все и делай посты развернутые в markdown формате и красиво». Потом дайджест с обрезками тел — снова «Хуевые посты».

Рабочий формат (pipeline4): **полный текст статьи** 1.3–4K символов:
```
💰 *Кейс: 55 470€ и ROI 113% с Facebook на Бразилию*

<абзацы статьи целиком, цифры в *bold*>

_📡 affy_
```
- Никаких URL в посте (disable_web_page_preview=True + не вставлять ссылки)
- Заголовок из `<title>` статьи (очищенный от « · AFFY»-хвостов), не из листинга
- Деньги/ROI/проценты — жирным через MONEY_RE.sub

## Архитектура pipeline4 (двухстадийная — ЕДИНСТВЕННАЯ рабочая)

1. **Stage 1 — из листингов**: `extract_urls()` тянет (url, title) из JSON-LD (`"url":"..."..."name":"..."`) и anchors. Листинги НЕ содержат тела статей — sequential h→p парсинг листинга даёт 0 результатов (проверено на 33 файлах).
2. **Stage 2 — фетч статей**: httpx GET каждого URL (кэш в `feed_bot/articles/*.html`), `extract_body()` = title из `<title>` + все `<p>` 80+ символов.
3. **Скоринг**: MONEY_RE (валюта/ROI/проценты/цифры) в заголовке ×5 и теле ×2, «кейс|схема|гайд|разбор» +8. Порог score≥12.
4. **Роутинг по топикам**: keyword-count по title+body[:800], порог ≥3.
5. **Постинг**: Bot API sendMessage, parse_mode=Markdown (legacy), message_thread_id=topic_id. Cap 4 поста/топик/прогон, sleep 2.5s между постами.

## Pitfalls

- **Telegram 429 при постинге**: читать `parameters.retry_after`, sleep+retry. При 1 посте/2.5s флуд редкий.
- **legacy Markdown parse errors**: экранировать `_*`[]` в тексте статей (`re.sub(r'([_*`\[\]])', r'\\\1', t)`), fallback на plain-text resend при 400.
- **entities + parse_mode конфликт**: НЕ передавать оба. Только parse_mode=Markdown.
- **md5-дедуп по URL**, не по title — листинги пересекаются между батчами.
- **Батч-селектор**: брать не самый свежий timestamp, а самый БОГАТЫЙ (Counter по ts → max по count) — свежие прогоны могут содержать 2 файла browser-check заглушек.
- **Browser-check заглушки** (fb-killa и др.): `if len(raw) < 15000 or 'Browser check' in raw[:2000]: skip`.
- **affy доминирует** пул кейсов; AI/Форум-интел топики пустеют — нужны другие источники или keyword-фильтр в harvester-конфиге.

## Telethon 1.42+ для forum-групп

- `CreateChannelRequest(title, megagroup=True, forum=True)` → форум-группа
- `CreateForumTopicRequest` / `EditForumTopicRequest` — в `telethon.tl.functions.messages` (НЕ channels), параметр `peer=` (не channel=)
- General topic всегда id=1
- Бот-админ: `InviteToChannelRequest` + `EditAdminRequest(admin_rights=ChatAdminRights(manage_topics=True, post_messages=True, ...))`
- Удаление: `channels.DeleteMessagesRequest(channel=ent, id=[...])` — без revoke-параметра
- `client.start()` ВИСНЕТ в unattended-скриптах → только `connect()` + `is_user_authorized()`

## Очистка при редизайне формата

```python
ids = [m.id async for m in c.iter_messages(ent) if m.sender_id == BOT_ID]
await c(DeleteMessagesRequest(channel=ent, id=ids))
# + удалить state*.json чтобы хеши не блокировали репост
```
