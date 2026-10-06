# Garbage Filter Blocklist (v14)

Critical pitfall: 4chan garbage posts leaked into the digest as "💥 Эксплойты/Пентест" because the `is_ai()` filter was too broad.

## GARBAGE BLOCKLIST (50+ words)

These words BLOCK any item from appearing in the digest:

```python
garbage = [
    # 4chan /b/ garbage
    "twitch gossip", "wife thread", "gf thread", "nobody general",
    "ukraine happening", "persia israel",
    
    # Slurs / offensive
    "slut", "nigger", "faggot", "tranny", "pedo",
    
    # Incel/manosphere garbage
    "incel", "femcel", "cuck", "simp", "based",
    "redpill", "bluepill", "blackpill", "sigma", "alpha", "beta",
    "looksmax", "mewing", "goon", "coom",
    
    # Social media garbage
    "discord", "egirl", "e-girl", "streamer", "twitch", "youtuber",
    "tiktok", "instagram", "snapchat", "tinder", "bumble", "hinge",
    
    # NSFW
    "porn", "hentai", "furry", "vore", "scat", "gore", "necrophilia",
    "feet", "onlyfans",
]
```

## 4chan Filter Rules

4chan posts pass ONLY if ALL of:
1. `replies > 50` (popular threads only)
2. Contains SPECIFIC AI tool names:
   ```python
   ai_specific = [
       "claude code", "codex", "jailbreak", "bypass", "mcp server",
       "deepseek", "gemini api", "openai api", "windsurf", "cursor ide",
       "opencode", "cline", "aider", "llama", "mistral", "mixtral",
       "stable diffusion", "comfyui", "automatic1111", "ollama",
       "langchain", "huggingface", "vllm", "tgi",
       "reverse proxy", "api key leak", "free api", "sk-", "ghp_",
       "uncensored", "unlimited", "prompt injection", "red team",
       "pentest", "exploit", "cve", "vulnerability", "0day",
       "ransomware", "malware", "botnet", "phish", "backdoor",
       "payload", "shellcode", "reverse engineering", "ghidra",
       "burp suite", "metasploit", "nmap", "wireshark",
       "chatgpt", "gpt-4", "gpt-5", "gpt-3", "copilot",
   ]
   ```
3. Passes garbage blocklist

## HN Filter Rules

HN posts pass ONLY if ALL of:
1. `stars > 10`
2. Contains tool/project keywords: "show hn", "github.com", "tool", "plugin", "mcp", "api", "jailbreak", "bypass", "hack", "exploit", "reverse", "proxy", "open source", "launch", "released", "cli", "self-hosted", "free", "extension", "skill", "agent", "code", "library", "framework"

## RSS Filter Rules

RSS posts pass ONLY if contains specific abuse keywords: "jailbreak", "bypass", "exploit", "api key", "free", "leak", "hack", "sk-", "ghp_", "token", "reverse proxy", "uncensored", "unlimited", "crack", "pirate", "warez", "nulled", "botnet", "malware", "0day", "zero-day", "vulnerability", "cve", "rce", "backdoor", "phish", "ransomware", "keylogger", "spyware", "rootkit", "payload", "pentest", "red team", "offensive", "defense evasion", "privilege escalation", "lateral movement", "exfiltration", "command and control", "dropper", "loader", "shellcode"

## Example: What Vlad saw (GARBAGE)

```
💥 Эксплойты / Пентест
▸ Coreverse NMS STBL THUGS FLOCK M3 SKZS OSCS Twitch Gossip
  4chan/b
▸ /vcg/: Vibe-coding General
  4chan/g
▸ /aaa/ anti-ai alliance
  4chan/g
```

These ALL failed the garbage filter tests and were REMOVED in v14.