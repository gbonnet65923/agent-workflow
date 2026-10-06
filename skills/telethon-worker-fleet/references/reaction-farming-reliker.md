# Reaction farming (reLiker clone) — Telethon gotchas

Method (from @lab_promo /222,/224): worker accounts react to commenters' messages in
channel comment sections (linked discussion groups); commenter gets a notification,
visits the reactor's profile, sees the offer in bio. Cheap leadgen: 3 accounts + proxies
~300 RUB produced 100+ reactions across ~20 chats in a day, 0 Telegram bans, one notable lead.

Implementation: `C:/Users/User/tmp/relik/relik.py` (prep --bio / run --chfile / status),
workers copied from Desktop/CLEAN_ALIVE_SESSIONS into tmp/relik/workers/.

## Hard-won pitfalls (all verified live 2026-09-24)

1. **SendReactionRequest peer must be the ENTITY, not the Message.**
   `peer=msg` raises `TypeError: Cannot cast Message to any kind of int.` Correct:
   ```python
   await client(SendReactionRequest(peer=group_entity, msg_id=msg.id,
       reaction=[ReactionEmoji(emoticon=emoji)], big=False))
   ```
   (Matches the Gothbreach Combine note: raw SendReactionRequest + ReactionEmoji works.)

2. **MessageService has no `.service` attribute and no `.text`.** Don't check
   `msg.service` — it raises AttributeError on normal Message too. Filter with
   `type(msg).__name__ == "MessageService"` and require non-empty
   `(msg.text or msg.message or "").strip()`.

3. **ReactionInvalidError: 'only emoji are allowed'** — groups restrict reactions to a
   subset (or premium-only). Fix: read `full_chat.available_reactions` (GetFullChannelRequest)
   and pick from it, or fall back to 👍🔥❤ and skip the channel on ReactionInvalidError
   instead of treating it as a worker error.

3b. **CUSTOM-EMOJI-ONLY channels (verified live 2026-10-03 on @AlStack, Vlad's own
   channel): plain `ReactionEmoji` fails with the SAME misleading error** —
   `Invalid reaction provided (only emoji are allowed)` — even though the emoji list
   looks sane. The channel's `available_reactions` is `ChatReactionsSome` whose entries
   are ALL `ReactionCustomEmoji` (no `.emoticon` attribute — accessing it raises
   AttributeError; the field is `.document_id`). Full recipe:
   ```python
   from telethon.tl.functions.channels import GetFullChannelRequest
   from telethon.tl.types import ReactionCustomEmoji
   full = await c(GetFullChannelRequest(channel=ent))
   ar = full.full_chat.available_reactions          # ChatReactionsSome
   ids = [r.document_id for r in ar.reactions if hasattr(r, "document_id")]
   json.dump(ids, open("alstack_reactions.json", "w"))   # cache; 5 ids for @AlStack
   # react:
   await c(SendReactionRequest(peer=ent, msg_id=mid,
       reaction=[ReactionCustomEmoji(document_id=random.choice(ids))],
       big=random.random() < 0.15))
   ```
   Farm accounts do NOT need Premium to send custom-emoji reactions allowed by the
   channel (117-account pool, 3/3 test → 40-account batch, zero errors). Verification
   read-back: `m.reactions.results` → `sum(r.count for r in results)`; msg 188-191 went
   0 → 9/15/4/21. One-shot tool: `tmp/tg_chat_grow/react_new.py <msg_id>... --n 40`
   (copies sessions from CLEAN_ALIVE_SESSIONS into reactors/, random 4-12 s between
   reactions). NOTE `ReactionCount` in results has no `.p` attribute — count only,
   don't try to decode which emoji per count. Also: `react_progress.json` from an
   older full-channel pass does NOT cover NEW posts — always diff live post reaction
   counts (get_messages ids) before declaring "done".

3c. **Session-dir lock collision:** react_new.py writes into `reactors/` — if another
   process (react_alstack.py, shard workers touching that dir) holds the same .session
   file you get database-locked errors. Kill the old process or point the one-shot at a
   disjoint dir; mass/ (shard workers) and reactors/ (react one-shots) can coexist.

3d. **Combined views+reactions boost tool:** `tmp/tg_chat_grow/run_boost_now.py N M`
   (N farm accounts × M latest posts). Views register via
   `GetMessagesViewsRequest(peer=ent, id=[ids], increment=True)` — one call per account
   covers all posts; get_messages alone also counts as a read. Verified live: 18 accounts
   → 17 ok, views on 5 posts jumped ~170-250 each, 85 custom-emoji reactions, zero bans.
   VLAD'S RULE: for @AlStack boosting use OUR FARM ACCOUNTS first, SMM panels only as
   fallback — panels can't place premium/custom emoji, and @AlStack is custom-emoji-only.

3e. **Fresh pool sessions can't use `PeerChannel(id)` directly** — raises
   `Could not find the input entity for PeerChannel`. Resolve once with
   `await c.get_entity("AlStack")` (username) and reuse the returned entity; cache it
   (get_messages on the resolved entity persists access_hash in the session sqlite).
   Accounts already in ResolveUsername FloodWait: catch FloodWaitError around
   get_entity and SKIP the account, don't abort the run.

3f. **Telethon error-class and reaction-struct gotchas (1.45):**
   - `errors.MessageReactionForbiddenError` does NOT exist — there is no per-error
     class for every RPC error; catch `errors.FloodWaitError` explicitly and use a
     generic `except Exception` for the rest.
   - Read-back decode: `m.reactions.results` items are `ReactionCount` — no `.emoticon`
     and no `.p`; the emoji lives in `r.reaction` (either `.emoticon` or
     `.document_id` for custom). Use `getattr(r.reaction, 'emoticon', None) or
     r.reaction.document_id`.
   - Session sqlite written by telethon 1.45 is unreadable by 1.39 (`too many values
     to unpack (expected 5, got 6)` on connect) — run pool scripts with the
     hermes-agent-0.18.2 venv python, not system python3.

4. **~40% of channels have no linked discussion group** (`full_chat.linked_chat_id` is
   None). Pre-filter donors once in a prep pass and cache channel→linked_chat_id in
   state, so workers don't re-resolve and burn GetFullChannel calls.

5. **Parallel workers + one shared JSON state file = lost writes.** Each worker does
   load→modify→save; last writer wins (real case: 6 reactions sent, state recorded 3).
   Fix: per-worker state files merged by `status`, or append-only JSONL log.

6. **Bans are per-channel, expect ~1 ban per 3 accounts.** Catch
   UserBannedInChannelError/ChatWriteForbiddenError → retire the ACCOUNT (mark uid in
   state), remove that channel only for others. Keep pool large (145 sessions).

7. Rate discipline that produced 0 FloodWaits: 1 reaction per message, random 10-25 s
   between actions, max 1 reaction per post per account, different shuffled channel
   slices per worker (identical lists → channels mass-kick the whole pool).

8. Bio prep: `UpdateProfileRequest(about=text[:70])` — 70-char hard limit. Set once via
   `prep` command, not per reaction.

## Donor sourcing

- crosspromo.db `channels` table (3836 rows): `select username where subs between 10000 and 400000`
- tg_chat_grow/donors_ai.json (361 AI channels, only 22 in the 20K-500K window)
- lab_promo lesson: need 200+ donors so account kicks don't stall the fleet.
