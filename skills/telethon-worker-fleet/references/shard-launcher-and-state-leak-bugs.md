# Shard Launcher & State-Leak Bugs (2026-10-03, tg_chat_grow campaign relaunch)

Two bugs that made a "successfully launched" fleet do almost nothing, plus the
dashboard-side check/add contract that came out of the same session.

## 1. Config-param leak inside the per-item loop (shard_worker.py)

Symptom: earlier runs produced far fewer seed pairs than expected (~58 pairs
across 4 shards over 240 groups) with no errors — most groups got `[pair] ...
already shilled, skip` or silently 0 pairs.

Root cause:

```python
for gi, g in enumerate(groups):
    gk = g["channel"]
    if gk in shill_registry():
        ppg = 0            # <-- mutates the RUN-LEVEL param
    while pairs_done < min(ppg, 1):   # now 0 for EVERY later group too
```

Once the first already-shilled group set `ppg = 0`, every subsequent group in
that shard got zero pairs. The shard still ran replies and exited "DONE".

Fix: compute a per-item local, never rebind launcher-provided config inside the
loop:

```python
ppg_g = 0 if gk in shill_registry() else min(ppg, 1)
while pairs_done < ppg_g:
```

Rule: `--pairs-per-group` / `--replies-per-group` style params are immutable
run config. Any per-item skip must be a per-item variable.

## 2. Child-process interpreter resolution (mass_seed.py)

Symptom: launcher prints `all shards launched` with pids, but every
`mass_<i>.log` contains:

```
ModuleNotFoundError: No module named 'telethon'
```

Root cause: the launcher spawned children as `["python", "shard_worker.py", ...]`.
On this machine, git-bash `python` on PATH = Hermes tools Python 3.14.7 WITHOUT
telethon. The parent was also being run under that interpreter, so `python`
resolved to it for children too.

Fixes (both applied):
- Children: use `sys.executable` in the Popen cmd, never the literal `"python"`.
- Parent: launch the fleet under Python 3.11 which has telethon+aiogram:
  `P311="C:/Users/User/AppData/Local/Programs/Python/Python311/python.exe"`
  `env -u PYTHONPATH "$P311" mass_seed.py --shards 4 ...`

Interpreter map on this box (verify with `-c "import telethon"` before trusting):
- `C:/Users/User/AppData/Local/hermes/tools/python-3.14.7*/python.exe` (PATH
  `python` in git-bash) — NO telethon.
- hermes-agent venv python — has telethon + aiogram (dashboard bot runs here).
- Python311 (`AppData/Local/Programs/Python/Python311/python.exe`) — has
  telethon + aiogram; canonical for mass_seed/shard_worker/check_channel.

Rule: **launcher output ≠ child success.** After any fleet launch, tail the
first lines of EVERY child log before reporting "запущено". A green launcher
print with 4 dead children is the default failure mode here.

## 3. Dashboard /check + /add subprocess contract (check_channel.py)

- `check_channel.py <target> [--json]` prints exactly one JSON object on the
  LAST line starting with `{` (parse reversed splitlines, first `{`-prefixed
  line wins). On any failure it still prints JSON with `ok:false, why:...` —
  never raises to the caller.
- Dashboard invokes it via `asyncio.create_subprocess_exec(sys.executable,
  "check_channel.py", target, "--json", cwd=BASE, env=PYTHONPATH-stripped)`
  with a 120 s timeout → on timeout returns `{"ok": false, "why": "timeout"}`.
- `add_to_pool()` writes pool_merged.json entries with `src: "dash-add"`,
  dedupes by channel name, and only runs when `ok:true` (activity ≥4 live
  humans/7d + AI-topic + writable gate lives in check_channel.py).
- Session picking for probes: copy from `Desktop/CLEAN_ALIVE_SESSIONS` (the
  live shards hold locks on mass/), skip journal/wal-locked files and protected
  IDs (7448683285, 809951394), PRIORITIZE stems listed in
  spambot_classified.json `ok` (guaranteed authed), max_tries=15, loop until
  one reports `is_user_authorized()`. 6 random tries can all be dead.
- Related pitfall already logged in invite-fleet-and-spambot-screening.md:
  `os.path.basename(f)` keeps `.session` → double extension → Telethon silently
  creates an EMPTY session → every probe says "not authorized". Strip the
  suffix before building the destination name.

## 4. /live probe hardening (dashboard_bot.py)

`ent.participants_count` is None for uncached channels → `{data['subs']:,}`
raises `TypeError: unsupported format string passed to NoneType` inside the
callback handler (bot looks alive; only the log shows
`Cause exception while process update`). Fix: `subs = data.get("subs") or 0`,
`GetFullChannelRequest` fallback for the real count, `data.get("posts") or []`,
and `c.connect()` + explicit `is_user_authorized()` check instead of
`c.start()` (start() prompts for phone → EOFError in a daemon).
