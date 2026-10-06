# Forum Scraper v4 — GitHub Trending HTML Scraping

## Why v4

v3 used GitHub API with tokens — all tokens dead (HTTP 401). Unauthenticated API = 60 req/hr, rate limit after ~15 queries. v4 pivots to GitHub Trending HTML scraping — no API, no rate limits.

## GitHub Trending HTML Scraping Pattern

```python
def gh_trending(lang="", period="daily"):
    url = f"https://github.com/trending/{lang}?since={period}"
    r = requests.get(url, headers={"User-Agent": "Mozilla/5.0"}, timeout=15)
    soup = BeautifulSoup(r.text, 'html.parser')
    for article in soup.select('article.Box-row'):
        h2 = article.select_one('h2 a')
        full_name = h2['href'].strip('/')
        desc = article.select_one('p').get_text(strip=True)
        stars = article.select_one('span.d-inline-block').get_text(strip=True)
        # ...
```

**Selectors (verified 2026-07-19):**
- `article.Box-row` — each trending repo
- `h2 a` — repo name + link (href = `/owner/repo`)
- `p` — description
- `span.d-inline-block` — stars (comma-separated, e.g. "1,234")
- `[itemprop="programmingLanguage"]` — language

**Rate limits**: NONE. This is regular web scraping, no API key needed. GitHub may block after many requests — add `time.sleep(1)` between pages.

**Languages covered**: `""` (all), `python`, `javascript`, `typescript`
**Periods**: `daily`, `weekly`

**Yield**: 77 repos per cycle (18+9+13+15+11+11)

## Deployment

Location: `/root/forum_scraper_v4.py` on eni-nyc1 (134.209.72.186)
Cron: `*/15 * * * * cd /root && python3 /root/forum_scraper_v4.py --once >> /root/forum_scraper_v4_cron.log 2>&1`
HTTP: `:8000` serves `/forum_data.json`, `/health`, `/stats`, `/scrape`
Launcher: `/root/start_scraper.sh`

## Pitfalls

- **BeautifulSoup required**: `pip3 install beautifulsoup4` on eni-nyc1
- **GitHub Trending HTML structure changes**: If selectors break, check `https://github.com/trending` in browser
- **Start script pattern**: Don't use `nohup ... &` in SSH — SSH session dies. Use `/root/start_scraper.sh` wrapper.