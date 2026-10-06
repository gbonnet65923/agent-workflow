# Ahmia.fi — Tor Hidden Service Search Engine Parsing

> Ahmia (ahmia.fi) — clearnet + onion search engine for Tor hidden services.
> Created by Juha Nurmi (2014, Google Summer of Code + Tor Project).
> Open source: github.com/ahmia/ (ahmia-site, ahmia-crawler, ahmia-index)

## CRITICAL: JS Required

Ahmia requires JavaScript to render search results. Firecrawl/web_extract return only the "no JS" fallback page. Must use browser automation (Chrome DevTools MCP or Playwright).

## Parsing Technique

### Via Chrome DevTools MCP (recommended)

```python
# Pattern: navigate → fill → submit → extract
from hermes_tools import mcp__chrome_devtools__*

# Step 1: Navigate to ahmia.fi
# Step 2: Fill search box
mcp__chrome_devtools__fill(uid="<searchbox_uid>", value="<query>")
# Step 3: Click Search button
mcp__chrome_devtools__click(uid="<button_uid>")
# Step 4: Wait for page to load, then extract
mcp__chrome_devtools__evaluate_script(function="() => { /* extract links */ }")
```

**UIDs change each page load** — always snapshot first to find current UIDs.

### Via evaluate_script (single-shot, no persistent state)

```javascript
// Set input value (React form — needs nativeInputValueSetter)
const input = document.querySelector('input[name="q"]');
const nativeInputValueSetter = Object.getOwnPropertyDescriptor(
  window.HTMLInputElement.prototype, 'value'
).set;
nativeInputValueSetter.call(input, '<query>');
input.dispatchEvent(new Event('input', { bubbles: true }));

// Submit form
document.querySelector('form').submit();
```

### Extract results

```javascript
() => {
  const links = document.querySelectorAll('a[href*="redirect_url"]');
  const seen = new Set();
  const items = [];
  links.forEach(a => {
    const t = a.textContent.trim().substring(0, 120);
    if (t && t.length > 3 && !seen.has(t)) {
      seen.add(t);
      items.push({t: t.substring(0, 100), h: a.href.substring(0, 200)});
    }
  });
  return {count: items.length, items: items.slice(0, 15)};
}
```

## Filtering Noise

Ahmia returns massive crypto/marketplace garbage. Filter these out:

```javascript
// Blocklist patterns
!t.includes('invest') && !t.includes('Bitcoin') && !t.includes('Ethereum') &&
!t.includes('crypt') && !t.includes('Cannabis') && !t.includes('counterfeit') &&
!t.includes('carded') && !t.includes('Tether') && !t.includes('Monero') &&
!t.includes('майнинг') && !t.includes('кошелек') && !t.includes('биткоин')
```

## Queries That Worked

| Query | ~Results | Quality |
|---|---|---|
| "traffic spam software" | 2051 | High — VENOMTOOLS, bulk SMS, email |
| "seo spam tools" | 753 | High — BlackHat, Spamming Tools |
| "proxy list socks5" | 699 | High — VIP72, InstantSocks, Dark0de |
| "captcha solver" | 881 | Medium — mostly captcha-protected gateways |
| "smtp mailer" | 94 | High — SpoofMe, SMTP Check Panel |
| "cpanel spam" | 600 | Medium — MoonClub, Dark Fox |
| "web shell backdoor" | 978 | High — Pathfinder RAT, ShadowTrack |
| "email spammer bulk sms sender traffic bot" | 500+ | High — VENOMTOOLS, Heart Sender |
| "forum site:forum spam traffic cpa" | 500+ | High — DarkNetArmy, BFD, CrdPro |
| "email extractor parser" | 175 | Medium — regex tools, HeLL Forum |

## Bulk Search via execute_code

For 50+ queries, use `execute_code` with `web_search` from hermes_tools:

```python
from hermes_tools import web_search
queries = ["query1", "query2", ...]
for q in queries:
    r = web_search(query=q, limit=5)
    # process results
```

This is faster than browser automation for bulk web searches, but doesn't return .onion links.

## Ahmia Onion Address

```
juhanurmihxlp77nkq76byazcldy2hlmovfu2epvl5ankdibsot4csyd.onion
```

## Saving Files

- `write_file` from hermes_tools is blocked for Desktop paths
- `mcp__filesystem__write_file` works for Desktop paths
- `mcp__filesystem__edit_file` works for appending to existing files
- Terminal heredoc (`cat > file << EOF`) fails for complex content with special chars

## Key Value

Ahmia is the best single source for discovering .onion sites related to spam/traffic/tools. Each search returns 100-2000+ results. The JS requirement and crypto noise are the main friction points.