# Etsy Reference

**Developer platform**: etsy.com/developers  
**Base URL**: `https://api.etsy.com/v3`  
**Auth URL**: `https://www.etsy.com/oauth/connect`  
**Token TTL**: Access token 1 hour, Refresh token 90 days  
**Rate limit**: 10 requests/second — always add `time.sleep(0.1)` in loops

---

## Registration

1. Register at etsy.com/developers
2. Create a New App — fill in name and use description
3. Set Redirect URI to `http://localhost:3000/callback`
4. Note **Keystring** (= Client ID) — there is no separate "App Secret" for PKCE
5. Etsy approval is near-instant for basic scopes

Required scopes:
```
listings_r listings_w transactions_r shops_r
```

---

## OAuth Flow (PKCE — no server-side secret needed)

Etsy uses OAuth 2.0 with PKCE, which is designed for local/installed apps. No App Secret is needed in the token exchange.

### Generate PKCE values + auth URL
```python
import secrets, hashlib, base64, urllib.parse

CLIENT_ID = "your_keystring"
REDIRECT_URI = "http://localhost:3000/callback"
SCOPES = ["listings_r", "listings_w", "transactions_r", "shops_r"]

def generate_pkce():
    verifier = secrets.token_urlsafe(64)
    challenge = base64.urlsafe_b64encode(
        hashlib.sha256(verifier.encode()).digest()
    ).rstrip(b"=").decode()
    return verifier, challenge

def get_auth_url():
    verifier, challenge = generate_pkce()
    state = secrets.token_urlsafe(16)
    params = {
        "response_type": "code",
        "redirect_uri": REDIRECT_URI,
        "scope": " ".join(SCOPES),
        "client_id": CLIENT_ID,
        "state": state,
        "code_challenge": challenge,
        "code_challenge_method": "S256",
    }
    url = "https://www.etsy.com/oauth/connect?" + urllib.parse.urlencode(params)
    return url, verifier  # save verifier to use in token exchange
```

### Exchange code for token
```python
def get_access_token(code, verifier):
    resp = requests.post(
        "https://api.etsy.com/v3/public/oauth/token",
        data={
            "grant_type": "authorization_code",
            "client_id": CLIENT_ID,
            "redirect_uri": REDIRECT_URI,
            "code": code,
            "code_verifier": verifier,  # from generate_pkce()
        }
    )
    return resp.json()
```

### Refresh token
```python
def refresh_access_token(refresh_token):
    resp = requests.post(
        "https://api.etsy.com/v3/public/oauth/token",
        data={
            "grant_type": "refresh_token",
            "client_id": CLIENT_ID,
            "refresh_token": refresh_token,
        }
    )
    return resp.json()
```

Save the `verifier` alongside tokens — you need it for the token exchange step.

---

## API Requests

Etsy uses standard Bearer token auth — no custom signing needed:

```python
def etsy_get(path, access_token, params=None):
    headers = {
        "Authorization": f"Bearer {access_token}",
        "x-api-key": CLIENT_ID,
    }
    resp = requests.get(f"https://api.etsy.com/v3{path}", headers=headers, params=params)
    time.sleep(0.1)  # respect 10 req/s rate limit
    return resp.json()
```

---

## Key API Endpoints

### Shop & Listings
| Action | Method | Path |
|---|---|---|
| Get shop info | GET | `/application/shops/{shop_id}` |
| List all listings | GET | `/application/shops/{shop_id}/listings` |
| Get listing detail | GET | `/application/listings/{listing_id}` |
| Update listing price | PATCH | `/application/listings/{listing_id}` |
| Update listing inventory (variants) | PUT | `/application/listings/{listing_id}/inventory` |
| Create listing | POST | `/application/shops/{shop_id}/listings` |

### Orders
| Action | Method | Path |
|---|---|---|
| List orders (receipts) | GET | `/application/shops/{shop_id}/receipts` |
| Get order detail | GET | `/application/shops/{shop_id}/receipts/{receipt_id}` |
| List transactions | GET | `/application/shops/{shop_id}/transactions` |

---

## Price Format

**Etsy stores prices in cents (integer).** Always multiply by 100 when writing, divide by 100 when reading.

```python
# Writing a price of $19.99
price_payload = {
    "amount": 1999,       # cents
    "divisor": 100,
    "currency_code": "USD"
}

# Reading a price
price_dollars = listing["price"]["amount"] / listing["price"]["divisor"]
```

### Update listing price
```python
def update_price(listing_id, price_usd, access_token):
    headers = {"Authorization": f"Bearer {access_token}", "x-api-key": CLIENT_ID}
    payload = {
        "price": {
            "amount": int(price_usd * 100),
            "divisor": 100,
            "currency_code": "USD"
        }
    }
    resp = requests.patch(
        f"https://api.etsy.com/v3/application/listings/{listing_id}",
        json=payload, headers=headers
    )
    time.sleep(0.1)
    return resp.json()
```

### Update variant prices (inventory)
Etsy doesn't have "discount campaigns" — price changes are direct. For variants, use the inventory endpoint:

```python
def update_inventory(listing_id, offerings, access_token):
    # offerings: list of {"sku": "...", "price": {"amount": cents, ...}, "quantity": N}
    headers = {"Authorization": f"Bearer {access_token}", "x-api-key": CLIENT_ID}
    resp = requests.put(
        f"https://api.etsy.com/v3/application/listings/{listing_id}/inventory",
        json={"products": offerings}, headers=headers
    )
    time.sleep(0.1)
    return resp.json()
```

---

## Getting Shop ID

The shop_id isn't shown prominently on Etsy — fetch it from the token response or the API:

```python
def get_my_shop(access_token):
    return etsy_get("/application/users/me/shops", access_token)
```

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Token expired | Refresh access token |
| `403 Forbidden` | Missing scope | Add scope during authorization |
| `429 Too Many Requests` | Rate limit hit | Add `time.sleep(0.1)` between calls |
| `400 Bad Request` on price | Wrong price format | Check amount is integer cents, not dollars |
