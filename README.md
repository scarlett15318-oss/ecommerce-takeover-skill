# ecommerce-takeover

A [Claude Code](https://claude.com/claude-code) skill that connects Claude to your e-commerce store backend via the platform's **official API**, then generates local automation scripts tailored to your needs.

The model is simple: **you fill in a spreadsheet → the script executes via API. Human decides, AI does.**

Supported platforms: **Shopee**, **TikTok Shop**, **Etsy** (Lazada and others via the same pattern).

## What it does

- Walks you through developer registration and OAuth for your platform.
- Generates a complete local script set under `~/ecommerce/<platform>/` (signed request client, auth/token management, feature scripts, rollback).
- Covers the common workflows:
  - **Discount campaigns / bulk price changes**
  - **Batch product listing** (new arrivals)
  - **Monthly profit & payout reports**

## Safety first

Every generated write script follows non-negotiable rules, because one wrong spreadsheet cell can overwrite dozens of live prices with no undo:

1. **Snapshot before any write** — current live values are dumped to a timestamped file first (the rollback source).
2. **Preview + explicit confirmation** — every write prints an `old → new` diff table and requires you to type `yes`; supports `--dry-run`.
3. **Audit log** — every change (old/new value, API response, timestamp) is appended to `logs/`.
4. **Rollback path** — a `rollback.py` restores previous values from any snapshot.
5. **Respect rate limits** — backoff and retry on `429`, never hammer the API.
6. **Secrets stay local** — App secrets and tokens are never pasted into chat or committed; `config.py`, `tokens.json`, `snapshots/`, and `logs/` are gitignored.

## Installation

Install as a personal skill for Claude Code:

```bash
git clone https://github.com/scarlett15318-oss/ecommerce-takeover-skill.git \
  ~/.claude/skills/ecommerce-takeover
```

Then in Claude Code, trigger it by describing what you want, e.g. *"help me connect to my Shopee store API and set up bulk price changes"* (Chinese phrases like *接管店铺 / 接入API / 自动改价 / 批量上新* also trigger it).

## Repo contents

| File | Purpose |
|---|---|
| `SKILL.md` | The skill definition and workflow Claude follows |
| `references/shopee.md` | Shopee signing, Go Live guide, endpoints, rate limits |
| `references/tiktok.md` | TikTok Shop versioned API, signing, `service_id` / `shop_cipher` |
| `references/etsy.md` | Etsy PKCE auth, price format, rate limits |

> ⚠️ Platform APIs version over time. The reference files are the source of truth for endpoints and signing, but always verify against the platform's current official docs if a call fails.
