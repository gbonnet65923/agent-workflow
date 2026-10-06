# Paste-Site Archives as Intel Sources (verified 2026-09-25)

Task origin: Vlad asked for paster.sh-style forums (public paste archives with leaks/combos/configs) and then abuse forums. All statuses below are curl-verified from the local machine on 2026-09-25.

## Tier 1 — public archive/recent feed (direct paster.sh analogues)

| Site | Feed URL | Status | Notes |
|------|----------|--------|-------|
| paster.sh | /archive?page=N (334 pages, 6664 pastes) | 200 | Plain HTML, Firecrawl or curl both work; refreshes hourly; monitor keys: combo, checker, config, proxy, dump, smtp, stealer |
| rentry.co | /discover | 200 | Markdown pastes, no captcha; JS-rendered feed — curl returns empty link list, needs browser/Firecrawl |
| pastebin.com | /archive | 200 | Classic; per-language archive tabs |
| pastelink.net | /archive | 200 | Archive of all public pastes |
| paste.jp | /recent | 200 | Recent feed |
| nullpaste.org | /recent | 200 | Fresh 2026 anon paste service with recent feed |
| pastey.gg | /recent | 200 | Recent + public pastes |
| pastebin.fi | /archive | 200 | Finnish clone with archive |
| leaked.lol | / | 200 | Name says it all; no <title> via curl — JS-rendered |
| modernpaste.com | /archive | 200 | SPA, archive loads via JS |
| controlc.com | / | 200 | /recent.php is 404 — no working public feed found |
| psbdmp.ws | /api/search/<query> | 000 local | Pastebin-dump indexer with search API; Cloudflare-blocked from local IP, works via proxy — primary paste-OSINT aggregator |

## Tier 2 — anon paste services (no feed, but paste sources; 200 unless noted)

vpaste.net, termbin.com (`cat file | nc termbin.com 9999`), paste.rs (`curl --data-binary @file https://paste.rs`), nekobin.com (open API), ctxt.io, xi.pe, jaw.gg, zigbin.io, fragbin.com, hastebin.dev, snippet.host, paste.myst.rs, n0paste.tk, 235523.xyz, pastecn.com, pastecode.io, dpaste.com/.org/.de, temp.sh, privatebin.net, bin.disroot.org, p.defau.lt (robots ALL — fully Google-indexed), defuse.ca/pastebin.htm, privsen.com, lock.pub (encrypted), cl1p.net (000 via curl second pass — flaky).

## Dead / blocked (do not retry without proxy change)

- hastebin.com → redirects to toptal (pastes no longer public)
- ghostbin.com 403, privnote.com 403 (bot-blocked)
- paste.io, microbin.eu, paste.se, dpaste.io, ix.io, bpa.st, paste.debian.net, paste.ubuntu.com, snopyta.org, paste.voidnet.tech → DNS/000
- justpaste.it → 200 but /archive returns 451 for RU geo (needs non-RU proxy)
- sprunge.us 404, 0bin.net 404, paste.sh /archive 404, pastecode.io /recent 404

## Discovery sources (self-updating)

- github.com/lorien/awesome-pastebins — maintained list (updated 2026-09-06)
- wiki.archiveteam.org/index.php/Paste_hosting — feature table (expire, captcha, recent-feed flags)

## Verification recipe

```bash
# batch liveness:
for s in <urls>; do curl -s -o /dev/null -w "%{http_code}" -L --max-time 10 -A "Mozilla/5.0" "$s"; done
# feed presence check: grep -iocE "archive|recent|latest|discover|public" on first 3KB
```

Pitfall: curl HTTP 200 with size 0 often means gzip response — always add `--compressed` or decode `Content-Encoding: gzip` manually when reading bodies (bit us: nodeloc/v2ex JSON looked empty until gzip handling added).

## Monitoring pool recommendation (9 entry points)

paster.sh + rentry.co/discover + pastebin.com/archive + pastelink.net/archive + nullpaste.org/recent + paste.jp/recent + pastey.gg/recent + leaked.lol + psbdmp.ws (via proxy). Candidate addition to forum_parser cron (job 60eea17a0221) — not yet wired as of 2026-09-25.

## Telegram delivery

Posts built in AlStack catalog style (🩸 sections, [Name](url) links, "Репост 🔁", footer 👻 иишко) and sent via `tmp/tg_prem_send.py --text-file <file>` (HTML parse_mode + premium emoji map, chat 7448683285 thread 343536). Proven msg IDs: 77163, 77164, 77190, 77235.
