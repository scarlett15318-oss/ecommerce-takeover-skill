# Shopee Reference

**Developer platform**: open.shopee.com  
**Base URL (live)**: `https://partner.shopeemobile.com`  
**Base URL (test)**: `https://partner.test-stable.shopeemobile.com`  
**Token TTL**: Access token ~4 hours, Refresh token ~30 days

---

## Registration & Go Live

### Step 1 — Create app
1. Register at open.shopee.com
2. My Apps → Create App → Type: **Self-developed**
3. Note Test Partner ID and Test API Partner Key

### Step 2 — Go Live form fields

| Field | Value |
|---|---|
| Business Product URL | `http://localhost:8080` |
| Test Username / Password | Leave blank or "N/A — local tool" |
| Brief Introduction | See template below |
| Client UI screenshot | Screenshot of terminal running script |
| Test / Live Redirect URL | `http://localhost:8080` |
| APP IP Address | Run `curl https://api.ipify.org` |

**Brief Introduction template:**
```
A self-developed automation tool for managing my own Shopee store.
Personal use only. Features: promotional pricing, batch listing,
monthly profit reporting. Operates on my own shop only.
```

Review: 1–3 business days. Self-use apps have high approval rates.

After approval: update scripts to use live credentials and `BASE_URL = "https://partner.shopeemobile.com"`.

---

## Signing

Every Shopee API request needs a timestamp and HMAC-SHA256 signature.

```python
import hmac, hashlib, time

def sign(path, access_token, shop_id, partner_id, partner_key):
    timestamp = int(time.time())
    base = f"{partner_id}{path}{timestamp}{access_token}{shop_id}"
    sig = hmac.new(partner_key.encode(), base.encode(), hashlib.sha256).hexdigest()
    return timestamp, sig
```

Attach to every request URL:
```
?partner_id=X&timestamp=X&sign=X&access_token=X&shop_id=X
```

For merchant-level calls, replace `shop_id` with `merchant_id` in the base string.

---

## OAuth Flow

### Generate auth URL
```python
def get_auth_url(partner_id, partner_key, redirect_url, base_url):
    path = "/api/v2/shop/auth_partner"
    timestamp = int(time.time())
    base = f"{partner_id}{path}{timestamp}"
    sig = hmac.new(partner_key.encode(), base.encode(), hashlib.sha256).hexdigest()
    return (f"{base_url}{path}"
            f"?partner_id={partner_id}&timestamp={timestamp}"
            f"&sign={sig}&redirect={redirect_url}")
```

### Exchange code for token
```python
def get_access_token(code, shop_id=None, main_account_id=None):
    path = "/api/v2/auth/token/get"
    # sign with empty access_token and shop_id=0
    timestamp = int(time.time())
    base = f"{PARTNER_ID}{path}{timestamp}"
    sig = hmac.new(PARTNER_KEY.encode(), base.encode(), hashlib.sha256).hexdigest()
    url = f"{BASE_URL}{path}?partner_id={PARTNER_ID}&timestamp={timestamp}&sign={sig}"
    payload = {"code": code, "partner_id": PARTNER_ID}
    if shop_id:
        payload["shop_id"] = int(shop_id)
    if main_account_id:
        payload["main_account_id"] = int(main_account_id)
    return requests.post(url, json=payload).json()
```

### Refresh token
```python
def refresh_access_token(refresh_token, shop_id):
    path = "/api/v2/auth/access_token/get"
    timestamp = int(time.time())
    base = f"{PARTNER_ID}{path}{timestamp}"
    sig = hmac.new(PARTNER_KEY.encode(), base.encode(), hashlib.sha256).hexdigest()
    url = f"{BASE_URL}{path}?partner_id={PARTNER_ID}&timestamp={timestamp}&sign={sig}"
    return requests.post(url, json={
        "refresh_token": refresh_token,
        "partner_id": PARTNER_ID,
        "shop_id": int(shop_id),
    }).json()
```

### Callback handling
The callback URL contains either:
- `?code=xxx&shop_id=xxx` — direct shop authorization
- `?code=xxx&main_account_id=xxx` — business/merchant account

For merchant accounts: use the `shop_id_list` from the token response to let the user pick their shop. Don't call `get_shop_list` — it often returns errors; the token response already contains the shop IDs.

---

## Key API Endpoints

### Products
| Action | Method | Path |
|---|---|---|
| List items | GET | `/api/v2/product/get_item_list` |
| Item details | GET | `/api/v2/product/get_item_base_info` |
| Model/variant list | GET | `/api/v2/product/get_model_list` |
| Update price directly | POST | `/api/v2/product/update_price` |
| Add new item | POST | `/api/v2/product/add_item` |

Params for item list: `offset`, `page_size` (max 50), `item_status=NORMAL`  
Params for base info: `item_id_list` (comma-separated, max 50 per call)

### Discount Campaigns
| Action | Method | Path |
|---|---|---|
| Create campaign | POST | `/api/v2/discount/add_discount` |
| Add items to campaign | POST | `/api/v2/discount/add_discount_item` |
| List campaigns | GET | `/api/v2/discount/get_discount_list` |
| Delete campaign | POST | `/api/v2/discount/delete_discount` |

`add_discount` payload:
```json
{"discount_name": "string", "start_time": 1234567890, "end_time": 1234567890}
```

`add_discount_item` payload (max 24 items per request):
```json
{
  "discount_id": 123,
  "item_list": [{
    "item_id": 456,
    "purchase_limit": 0,
    "model_list": [{
      "model_id": 789,
      "model_promotion_price": 99000,
      "purchase_limit": 0
    }]
  }]
}
```
Note: field is `model_promotion_price`, not `model_discounted_price`.

### Orders & Finance
| Action | Method | Path |
|---|---|---|
| Order list | GET | `/api/v2/order/get_order_list` |
| Order detail | GET | `/api/v2/order/get_order_detail` |
| Payout detail | GET | `/api/v2/payment/get_payout_detail` |
| Escrow detail | GET | `/api/v2/payment/get_escrow_detail` |

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `error_auth` | Token expired | Re-run auth or refresh token |
| `invalid_code` | Auth code already used or expired | Re-do full OAuth, paste URL immediately |
| `error_not_found` | Wrong shop_id or endpoint | Verify shop_id; check path |
| `common.error_param` | Wrong field name in payload | Check field names carefully (e.g. `model_promotion_price`) |
| `error_permission` | Missing API scope | Check app permissions in Open Platform |
| `error.rate_limit` / 429 | Too many requests | Back off (1s→2s→4s) and retry; add `sleep` in loops |

---

## Rate limits

Shopee enforces per-partner API call limits (per-second and daily). In any loop:
- add a small `time.sleep(0.2)` between calls as a baseline,
- on a rate-limit error, back off (1s → 2s → 4s) and retry a few times,
- batch where the API allows it (item base info: ≤50 ids/call; discount items: ≤24/request) to cut call volume.

---

## Variant Parsing Notes

Model names come as plain strings like `"Màu be,90*90cm"` or `"Red / XL"`.

```python
import re
SIZE_PATTERN = re.compile(
    r'^(XXXL|XXL|XL|L|M|S|XS|\d+\s*[*x×]\s*\d+\s*cm?)$', re.IGNORECASE
)

def parse_model_name(name):
    for sep in [",", " / "]:
        parts = [p.strip() for p in name.split(sep)]
        if len(parts) >= 2:
            for i in range(len(parts)-1, -1, -1):
                if SIZE_PATTERN.match(parts[i]):
                    size = parts[i]
                    color = sep.join(parts[:i] + parts[i+1:])
                    return color, size
    return name, ""
```
