---
name: ecommerce-takeover
description: "Use this skill to connect Claude Code to e-commerce store backends via official APIs, enabling automated store management. Trigger whenever the user mentions connecting to Shopee/TikTok Shop/Etsy/Lazada API, automating store operations, bulk price changes, creating discount campaigns, batch product listing, monthly profit reports, OAuth authorization for e-commerce platforms, or any request to take over, manage, or automate an online shop. Also trigger for Chinese phrases like 接管店铺, 接入API, 自动改价, 批量上新, 店铺自动化."
---

# E-Commerce Store Takeover

You are helping a seller connect Claude Code to their e-commerce store via the platform's official API, then build automation scripts for their specific needs.

The goal is: seller fills in a spreadsheet → you execute via API. Human decides, AI does.

## Non-negotiable safety rules (read first)

These apply to every platform and every script you generate. They exist because a single wrong cell in a spreadsheet can overwrite dozens of live prices with no undo.

1. **Snapshot before any write.** Before any bulk price/stock/listing change, the script must first dump the *current* live values to a timestamped file (`snapshots/<feature>-<YYYYMMDD-HHMMSS>.json`). This is the rollback source.
2. **Preview + explicit confirmation before pushing.** Every write script runs in two phases: it prints a diff table (`item / variant / old → new`) and the row count, then requires the user to type `yes` to proceed. Support a `--dry-run` flag that stops after the preview.
3. **Audit log.** Append every write (what changed, old value, new value, API response, timestamp) to `logs/changes-<YYYYMMDD>.log`. Never silently mutate the store.
4. **Rollback path.** Generate a `rollback.py` that reads a snapshot file and restores the previous values. Mention it to the user when delivering write scripts.
5. **Respect rate limits.** Add `time.sleep()` / backoff in every loop that calls the API. Platform limits are in each reference file. On a `429`/rate-limit error, back off and retry, don't hammer.
6. **Never paste the App Secret / tokens into chat or git.** Write them only to local files; `tokens.json`, `config.py`, `snapshots/`, and `logs/` all go in `.gitignore`.

## Phase 1 — Understand their setup

Ask these together in one message (adapt based on what they've already told you):

1. **Platform**: Shopee / TikTok Shop / Etsy / Lazada / Other?
2. **Developer credentials**: Do they have an App Key/Partner ID and App Secret/Partner Key yet? If not, walk them through registration first.
3. **What to automate**: Which operations matter most?
   - Discount campaigns / bulk price changes
   - Batch product listing (new arrivals)
   - Monthly profit & payout reports
4. **Tech comfort**: Can they run Python in a terminal? (This determines how much to explain.)

If they've already shared credentials or described their setup, extract what you can and only ask about gaps.

## Phase 2 — Platform registration (if needed)

If they don't have credentials yet, guide them through registration before writing any code. Load the relevant reference file:

- Shopee → read `references/shopee.md` → section: Registration & Go Live
- TikTok Shop → read `references/tiktok.md` → section: Registration
- Etsy → read `references/etsy.md` → section: Registration

Key point across all platforms: for self-use tools, always choose **"Self-developed"** / In-house / equivalent. Approval is faster and requirements are simpler.

**Go Live form honesty note:** describe the tool truthfully as a personal self-use automation tool. Don't fabricate a public product URL or fake UI screenshots — strict reviewers reject those. A terminal screenshot plus an honest "local self-use tool, operates only on my own shop" description is enough for self-developed apps.

## Phase 3 — Generate the script set

Once credentials are confirmed, generate a complete local script set. Create files in a `~/ecommerce/<platform>/` directory.

Foundation files (always generate):

```
~/ecommerce/<platform>/
├── config.py        # credentials (user fills in)
├── client.py        # signed request wrapper: signing + rate limit + retry/backoff
├── auth.py          # OAuth flow + token management
├── rollback.py      # restore values from a snapshot file
├── main.py          # menu entry point
├── snapshots/       # auto-created; pre-change state dumps
├── logs/            # auto-created; audit logs
└── .gitignore       # config.py, tokens.json, snapshots/, logs/
```

### `config.py` — credentials (user fills in)
Field names vary per platform — read the reference file. Generic shape:
```python
APP_KEY = ""          # App Key / Partner ID / Client ID
APP_SECRET = ""       # App Secret / Partner Key  (PKCE platforms like Etsy have none)
REDIRECT_URI = ""     # exact value registered in the dashboard
SHOP_ID = 0           # filled in after OAuth
# TikTok Shop only:
SERVICE_ID = ""       # from app dashboard — drives the authorize URL
SHOP_CIPHER = ""      # filled in after OAuth (per-shop, required on most calls)
```

### `client.py` — one place for signing, rate limiting, retry
All feature scripts call through this wrapper so signing and limits are never duplicated. Load the platform's signing logic from its reference file. The wrapper must:
- sign each request per the platform spec (read the reference file — signing differs by platform)
- sleep between calls to respect the platform rate limit
- on `429` / rate-limit / transient errors, back off (e.g. 1s, 2s, 4s) and retry a few times
- auto-refresh the access token on an "expired" error, then retry once

### `auth.py` — OAuth flow + token management
The auth flow shape is similar across platforms, but the *authorize URL and token endpoints differ — always read the reference file*, don't assume:
- Build the authorization URL the way the reference file specifies (some platforms sign it, some use a `service_id`, some use PKCE).
- Open browser for user to authorize.
- Capture callback URL (user pastes it back — don't rely on local server catching it).
- Exchange code for access_token + refresh_token.
- Save tokens to `tokens.json`; auto-refresh when expired.

**Critical auth gotchas (apply to all platforms):**
- Auth codes are single-use and expire in minutes — user must paste the callback URL immediately.
- Merchant/business accounts may return `merchant_id`/`shop_cipher` instead of a raw `shop_id` — handle both.
- Save tokens to `tokens.json`; add it to `.gitignore`.

### `main.py` — menu entry point
```python
MENU = {
    "1": ("Export product template", export_products),
    "2": ("Create discount campaign (preview→confirm)", create_discount),
    "3": ("Bulk upload new products (preview→confirm)", bulk_upload),
    "4": ("Monthly profit report", profit_report),
    "5": ("Rollback from snapshot", rollback),
    "6": ("Refresh token", refresh_token),
}
```
Only generate the menu items for features the user actually asked for (but always keep rollback + refresh token).

Then generate the feature-specific scripts. Read the platform reference file for endpoint details — **endpoints and API versions change, so trust the reference file over memory, and if a call 404s, suspect an outdated path/version.**

## Phase 4 — Walk through OAuth

After generating scripts, guide the user through first-time authorization:

```bash
cd ~/ecommerce/<platform>
pip3 install requests openpyxl
python3 auth.py
```

Expected flow:
1. Browser opens to authorization page
2. User logs in with seller account and approves
3. Browser redirects to localhost (page may error — that's fine)
4. User copies the full URL from address bar and pastes it back
5. Script exchanges code for tokens and saves `tokens.json`

If they get a `merchant_id` / `shop_cipher` / multiple shops in the callback, that's a multi-shop business account. Use the shop list from the token response to let them pick their target shop, and save the chosen shop identifier to `config.py`.

## Phase 5 — Test the connection

After auth succeeds, immediately verify by fetching the product list (read-only, safe):

```bash
python3 main.py
# Select: Export product template
```

If it returns product data, everything is working. If it errors, diagnose against the reference file's error table:
- auth/token expired → refresh or re-run auth.py
- invalid signature → re-check signing logic in the reference file
- not found / 404 → wrong shop identifier OR outdated endpoint path/version

## Discount Campaign / bulk price workflow (most common request)

The full flow — note the mandatory snapshot + preview + confirm steps:

1. **Export**: Script fetches all products + variants → writes Excel template.
   - Group same-size variants into one row (different colors merged).
   - Sort by size descending for easy editing.
2. **User fills**: `discounted_price` column — leave blank to skip that product.
3. **Snapshot**: Before any write, dump current prices to `snapshots/`.
4. **Preview**: Print the `old → new` diff table + row count. Stop here if `--dry-run`.
5. **Confirm**: Require the user to type `yes`.
6. **Create/apply**: Read Excel → create the campaign / push prices → log every change.

When building the export script, parse variant names carefully:
- Shopee: model_name is often `"Color,Size"` or `"Color / Size"` — split by comma or " / ".
- Size detection: match `XS|S|M|L|XL|XXL|XXXL` or dimension patterns like `\d+\s*[*x]\s*\d+\s*cm`.
- Excel saves numbers as floats — always `int(float(value))` when parsing IDs.

When building the create-campaign script:
- Read both `.xlsx` and `.csv` (try multiple encodings: utf-8-sig, utf-8, latin-1).
- Batch item additions per the platform's per-request limit (Shopee: max 24 items/request).
- Field names are platform-specific (e.g. Shopee uses `model_promotion_price`, not `model_discounted_price`) — read the reference file.

## Profit report workflow

Pull from two sources and join them:
1. **Platform API**: order list + escrow/payout details (actual received amount after fees).
2. **Local cost table**: user-maintained spreadsheet with cost per SKU.

Output: monthly summary + per-product breakdown (revenue, cost, platform fees, profit, margin %).

The cost table must be linked to orders via item_id or SKU. Ask the user what identifier they use in their cost table before building this.

## Reference files

Load these when you need platform-specific details. **Treat them as the source of truth for endpoints and signing; APIs version over time, so prefer the file over assumptions and verify against the platform's current official docs if a call fails.**

| Platform | File | Contents |
|---|---|---|
| Shopee | `references/shopee.md` | Signing, Go Live guide, API endpoints, rate limits |
| TikTok Shop | `references/tiktok.md` | Versioned API (202309+), signing, authorize via service_id, shop_cipher |
| Etsy | `references/etsy.md` | PKCE auth, price format (cents), rate limits |

Read only the file for the user's platform — don't load all three.
