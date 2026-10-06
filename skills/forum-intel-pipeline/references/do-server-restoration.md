# DO Server Restoration Runbook

## Symptoms

- `curl http://188.166.189.92:8000/forum_data.json` times out
- Bridge cron fails with connection errors
- SSH works but no Python process on port 8000

## Restoration Steps

### 1. Kill stale processes
```bash
ssh -i ~/.ssh/do_access -o ControlMaster=no root@188.166.189.92
fuser -k 8000/tcp
```

### 2. Check if forum_scraper.py exists
```bash
ls /root/forum_scraper/forum_scraper.py
```
If missing, upload from local backup:
```bash
scp -i ~/.ssh/do_access -o ControlMaster=no \
  C:/Users/User/Desktop/forum_scraper.py \
  root@188.166.189.92:/root/forum_scraper/forum_scraper.py
```

### 3. Build cache
```bash
ssh ... root@188.166.189.92 'python3 /root/forum_scraper/build_cache.py'
```
Should output: `Cache built: N items`

### 4. Start serve_cache.py
```bash
ssh ... root@188.166.189.92 'cd /root/forum_scraper && nohup python3 -u serve_cache.py > http_server.log 2>&1 &'
```

### 5. Verify
```bash
# Local
ssh ... root@188.166.189.92 'timeout 5 curl -s http://localhost:8000/forum_data.json | python3 -c "import sys,json; print(len(json.load(sys.stdin)))"'
# External
curl -s -o /dev/null -w '%{http_code}' http://188.166.189.92:8000/forum_data.json
# Expected: 200
```

### 6. Restore crons if missing
```bash
ssh ... root@188.166.189.92 '(crontab -l 2>/dev/null; echo "*/15 * * * * cd /root && python3 forum_scraper.py --once >> /root/forum_scraper/scraper.log 2>&1") | crontab -'
ssh ... root@188.166.189.92 '(crontab -l 2>/dev/null; echo "*/15 * * * * cd /root/forum_scraper && python3 build_cache.py >> /root/forum_scraper/cache.log 2>&1") | crontab -'
```

## Why the old --serve mode failed

`forum_scraper.py --serve 8000` parsed ALL JSON files on every request. With 182 files accumulated, it hung indefinitely. The fix: separate `build_cache.py` (cron, generates forum_data.json once) + `serve_cache.py` (daemon, reads and serves the pre-built file).

## SSH pitfalls

- Stale ControlMaster: `rm -f ~/.ssh/cm-*` before connecting
- Always use `-o ControlMaster=no` for one-off commands
- Key: `~/.ssh/do_access` (not id_ed25519 or id_rsa)