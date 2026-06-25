# TikTok Shop Reference

**Developer platform**: partner.tiktokshop.com
**API base URL**: `https://open-api.tiktokglobalshop.com`
**Authorize page**: `https://services.tiktokshop.com/open/authorize`
**Token endpoints host**: `https://auth.tiktok-shops.com`
**Token TTL**: Access token ~7 days, Refresh token ~30 days (read `expires_in` from the response, don't hardcode)

> ⚠️ **Version note (important):** TikTok Shop's current API is the **version-dated** API — paths look like `/product/202309/...`, `/order/202309/...`, `/promotion/202309/...`. The older un-versioned paths (`/api/products/search`, `/api/promotions`, …) are deprecated and will 404. Always use the dated paths below. If a new dated version (e.g. `202405`) has shipped, prefer it and verify against the official docs. Most calls require a `shop_cipher` query param and the access token in the **`x-tts-access-token` header**, not in the query string.

---

## Registration

1. Register at partner.tiktokshop.com
2. Create App → Self-developed / In-house
3. Required scopes (Authorized scope):
   - `Product` (read + write)
   - `Order Information` (read)
   - `Promotion` (read + write)
   - `Finance` (read)
4. Set the app's **Redirect URL** to `http://localhost:8080/callback`
5. Note three things from the app page: **App Key**, **App Secret**, and the **Service ID** (drives the authorize URL).

Review: 1–3 business days.

---

## Signing

Current TikTok Shop signing (HMAC-SHA256) signs the request path + sorted query params + body, wrapped by the app secret on **both** ends. Exclude `sign` and `access_token` from the signed params.

```python
import hmac, hashlib, time, json

def sign_request(app_secret, path, query_params: dict, body: dict | None = None):
    """Returns (signature, timestamp). `path` is the URL path only, e.g. '/product/202309/products/search'."""
    timestamp = str(int(time.time()))
    params = dict(query_params)
    params["timestamp"] = timestamp           # timestamp is part of the signed query
    # 1. sort params (exclude sign + access_token), concat as key+value
    sorted_params = "".join(
        f"{k}{params[k]}" for k in sorted(params)
        if k not in ("sign", "access_token")
    )
    # 2. base = path + sorted params, then body (if any)
    base = path + sorted_params
    if body is not None:
        base += json.dumps(body, separators=(",", ":"))
    # 3. wrap with secret on both ends, HMAC-SHA256 with the secret
    to_sign = app_secret + base + app_secret
    signature = hmac.new(app_secret.encode(), to_sign.encode(), hashlib.sha256).hexdigest()
    return signature, timestamp
```

Every authenticated request includes query params `app_key`, `timestamp`, `sign`, `shop_cipher`, and the header `x-tts-access-token: <access_token>` (plus `Content-Type: application/json` for POST/PUT).

> If you get `40001 invalid signature`: most common causes are (a) signing a path that includes the query string, (b) body JSON not serialized identically to what's sent, (c) forgetting `timestamp` in the signed params, or (d) not wrapping with the secret on both ends. Match the exact serialized body bytes you send.

---

## OAuth Flow

### Generate authorize URL (no manual signature — uses service_id)
```python
import urllib.parse

def get_auth_url(service_id, state="random"):
    base = "https://services.tiktokshop.com/open/authorize"
    params = {"service_id": service_id, "state": state}
    return f"{base}?" + urllib.parse.urlencode(params)
```
The seller approves on this page; TikTok redirects to your Redirect URL with `?code=...&state=...` (the `app_key` is bound to the service_id, so you don't build a signed authorize URL anymore).

### Exchange code for token
```python
import requests

def get_access_token(app_key, app_secret, auth_code):
    url = "https://auth.tiktok-shops.com/api/v2/token/get"
    params = {
        "app_key": app_key,
        "app_secret": app_secret,
        "auth_code": auth_code,
        "grant_type": "authorized_code",
    }
    # token endpoints take params in the query string; no x-tts-access-token header yet
    return requests.get(url, params=params).json()
```
The response contains `access_token`, `refresh_token`, `access_token_expire_in`, and a list of authorized shops. Each shop has a **`shop_cipher`** (and `shop_id`, `shop_name`, `region`). Save the chosen shop's `shop_cipher` to `config.py`.

### Get authorized shops (confirm shop_cipher)
```
GET /authorization/202309/shops      # signed; returns shop_id, shop_cipher, region, name
```
Use this if the token response doesn't list shops, or to let a multi-shop account pick.

### Refresh token
```python
def refresh_access_token(app_key, app_secret, refresh_token):
    url = "https://auth.tiktok-shops.com/api/v2/token/refresh"
    params = {
        "app_key": app_key,
        "app_secret": app_secret,
        "refresh_token": refresh_token,
        "grant_type": "refresh_token",
    }
    return requests.get(url, params=params).json()
```

---

## Key API Endpoints (versioned)

All paths below are signed and require `shop_cipher` + `x-tts-access-token` header.

### Products
| Action | Method | Path |
|---|---|---|
| Search products | POST | `/product/202309/products/search` |
| Get product detail | GET | `/product/202309/products/{product_id}` |
| Update price | POST | `/product/202309/products/{product_id}/prices/update` |
| Update stock | POST | `/product/202309/products/{product_id}/inventory/update` |
| Create product | POST | `/product/202309/products` |
| Upload image | POST | `/product/202309/images/upload` |

### Promotions
| Action | Method | Path |
|---|---|---|
| Create activity | POST | `/promotion/202309/activities` |
| Get activity | GET | `/promotion/202309/activities/{activity_id}` |
| Update activity products | POST | `/promotion/202309/activities/{activity_id}/products` |
| Search activities | POST | `/promotion/202309/activities/search` |

Promotions must be submitted ahead of their start time (schedule at least ~1 hour out to be safe).

### Orders & Finance
| Action | Method | Path |
|---|---|---|
| Search orders | POST | `/order/202309/orders/search` |
| Get order detail | GET | `/order/202309/orders/{order_id}` |
| Statement transactions | GET | `/finance/202309/statements` |
| Statement detail (payout) | GET | `/finance/202309/statements/{statement_id}/statement_transactions` |

---

## Rate limits

TikTok Shop enforces per-app QPS limits (varies by endpoint, commonly single-digit to low-tens QPS). In any loop:
- add `time.sleep(0.2)` between calls as a baseline,
- on a rate-limit error, back off (1s → 2s → 4s) and retry,
- paginate with the `page_token` / `next_page_token` the search endpoints return.

---

## Regional Notes

- **Southeast Asia** (MY, TH, VN, PH, SG, ID), **US**, **UK/EU** all use the same `open-api.tiktokglobalshop.com` base, but each shop is scoped by its own `shop_cipher` and `region`.
- US shops have separate merchant onboarding; UK/EU may add compliance fields on listings.
- Always pass the target shop's `shop_cipher`; a token can authorize multiple shops.

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `40001` | Invalid signature | Re-check signing: path only, sorted params incl. `timestamp`, exact body bytes, secret on both ends |
| `40002` / token expired | Access token expired | Refresh access token, retry once |
| `40101` | Insufficient scope | Add the required scope in app settings, re-authorize |
| `404` / not found | Outdated/un-versioned path, or wrong `shop_cipher` | Use the `/...202309/...` paths above; verify `shop_cipher` |
| promotion start too soon | Start time not far enough ahead | Schedule further out (~1h+) |
| `429` / rate limited | Too many requests | Back off and retry; add `sleep` in loops |
