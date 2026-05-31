# Setup Guide — Credentials & Wiring

This guide covers everything you need to configure before the automation can run: Etsy API credentials, Bambu Lab printer access, and the product mapping file.

---

## 1. Etsy API Credentials

### 1.1 Create an Etsy App

1. Go to https://www.etsy.com/developers/register
2. Sign in with your Etsy seller account
3. Click **"Create a new app"**
4. Fill in:
   - **App name:** Anything (e.g., "Print Farm Automation")
   - **Description:** "Automates order processing for my 3D print shop"
   - **Website:** Your personal site or leave blank
   - **OAuth Redirect URI:** `http://localhost:8080/callback` (you'll use this for the initial auth flow)
5. Submit — you'll get:
   - **Keystring** (this is your `ETSY_API_KEY`)
   - **Shared Secret** (this is your `ETSY_SHARED_SECRET`)

### 1.2 Get a Refresh Token

The Etsy API uses OAuth 2.0 with PKCE. You need to do a one-time authorization to get a refresh token.

**Method: Use the Etsy MCP server's one-time setup**

```bash
# Install the Etsy MCP server (it has a credential setup flow)
pip install etsy-mcp

# Run the setup — it opens a browser for you to authorize
etsy-mcp setup \
  --api-key YOUR_KEYSTRING \
  --shared-secret YOUR_SHARED_SECRET \
  --redirect-uri http://localhost:8080/callback
```

This will:
1. Open your browser to Etsy's authorization page
2. You click "Allow Access"
3. It captures the redirect and prints your refresh token

**Manual method (if the above doesn't work):**

1. Build this URL in your browser:
   ```
   https://www.etsy.com/oauth/connect?response_type=code&redirect_uri=http://localhost:8080/callback&scope=listings_r%20transactions_r&client_id=YOUR_KEYSTRING&code_challenge=YOUR_CHALLENGE&code_challenge_method=S256&state=abc123
   ```
   (PKCE challenge generation is complex — use a library like `etsy-oauth` or the MCP server)

2. After authorizing, you'll be redirected to `http://localhost:8080/callback?code=XXXXXX`
3. Exchange the `code` for a refresh token via POST:
   ```bash
   curl -X POST https://api.etsy.com/v3/public/oauth/token \
     -d "grant_type=authorization_code" \
     -d "client_id=YOUR_KEYSTRING" \
     -d "code=XXXXXX" \
     -d "code_verifier=YOUR_VERIFIER" \
     -d "redirect_uri=http://localhost:8080/callback"
   ```
4. The response contains `refresh_token` — save this.

### 1.3 Get Your Shop ID

```bash
# Once you have a valid access token, find your shop ID:
curl -H "x-api-key: YOUR_KEYSTRING" \
     -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
     "https://openapi.etsy.com/v3/application/shops?shop_name=YOUR_SHOP_NAME"
```

The response includes `shop_id`. Save this as `ETSY_SHOP_ID`.

### 1.4 Fill in `.env`

```bash
ETSY_API_KEY=your_keystring_here
ETSY_SHARED_SECRET=your_shared_secret_here
ETSY_REFRESH_TOKEN=your_refresh_token_here
ETSY_SHOP_ID=12345678
```

---

## 2. Bambu Lab P1S Printer Setup

### 2.1 Enable LAN Mode

1. On the printer's touchscreen: **Settings → Network → LAN Only Mode**
2. The printer will display its **IP address** and **Access Code**
3. Note both down

### 2.2 Find Printer Serial Number

1. On the printer's touchscreen: **Settings → Device Info**
2. Note the **Serial Number** (e.g., `03WXXXXXXXXXXXXX`)

### 2.3 Test FTPS Connectivity

```bash
# Test you can connect to the printer via FTPS
# (the printer uses implicit FTPS on port 990)
curl -k --ftp-ssl "ftps://bblp:YOUR_ACCESS_CODE@192.168.1.100:990/" --list-only
```

If this lists files, your printer is reachable.

### 2.4 Fill in `.env`

```bash
PRINTER_HOST=192.168.1.100
PRINTER_ACCESS_CODE=your_access_code
PRINTER_SERIAL=03WXXXXXXXXXXXXX
```

---

## 3. Product Mapping (`products.yaml`)

This file tells the system which GCODE file to print for each Etsy listing + variation combination.

### 3.1 Folder Structure

Organize your pre-sliced GCODE files like this:

```
/data/products/              ← base_path in products.yaml
├── iron_man_helmet/
│   ├── helmet_circumference_22inch.gcode.3mf
│   ├── helmet_circumference_23inch.gcode.3mf
│   └── helmet_circumference_24inch.gcode.3mf
├── mjolnir_hammer/
│   ├── full_size_standard.gcode.3mf
│   └── mini_desk_version.gcode.3mf
└── dragon_skull_decor/
    └── dragon_skull_one_size.gcode.3mf
```

**Important:** Bambu Lab P1S accepts `.gcode.3mf` files (3MF with embedded GCODE). Export from Bambu Studio as "Bambu Studio 3MF" with GCODE embedded.

### 3.2 How Etsy Variations Map to Files

When a customer buys an Iron Man Helmet and selects "Size: 22 inch" from the dropdown:

1. The Etsy API returns the transaction with `variations: [{"formatted_name": "Size", "formatted_value": "22 inch"}]`
2. The system looks up `listing_id: 2001` in `products.yaml`
3. It finds `property: Size` — so it looks for the value of the "Size" property
4. It matches `"22 inch"` → `helmet_circumference_22inch.gcode.3mf`
5. Full path: `/data/products/iron_man_helmet/helmet_circumference_22inch.gcode.3mf`

### 3.3 Example `products.yaml`

```yaml
# Base path where all product folders live
base_path: /data/products

# Map Etsy listing IDs to product folders and variation files
listings:
  # Iron Man Helmet — variations: Size dropdown
  2001:
    folder: iron_man_helmet
    property: Size                              # The Etsy property name
    files:
      "22 inch": helmet_circumference_22inch.gcode.3mf
      "23 inch": helmet_circumference_23inch.gcode.3mf
      "24 inch": helmet_circumference_24inch.gcode.3mf

  # Mjolnir Hammer — variations: Version dropdown
  2002:
    folder: mjolnir_hammer
    property: Version
    files:
      "Standard": full_size_standard.gcode.3mf
      "Mini Desk": mini_desk_version.gcode.3mf

  # Dragon Skull Decor — NO variations (one size fits all)
  2003:
    folder: dragon_skull_decor
    property: null                              # null = no variations
    files:
      default: dragon_skull_one_size.gcode.3mf

  # Cosplay Helmet — variations: Color AND Size
  # NOTE: Etsy combines multi-variation selections like "Color: Red, Size: 22 inch"
  # The system matches on the FIRST property listed here. For multi-property items,
  # you need one listing config per combination OR name files accordingly.
  2004:
    folder: samurai_helmet
    property: Size                              # The system matches on this property
    files:
      "22 inch": samurai_22inch.gcode.3mf
      "23 inch": samurai_23inch.gcode.3mf
      # If you have color variations too, handle them in the filename:
      # "22 inch (Red)": samurai_22inch_red.gcode.3mf
      # But Etsy's API returns property values separately, not combined.
      # For multi-property items, you may need to extend the mapper.
```

### 3.4 Finding Your Etsy Listing IDs

```bash
# Once you have Etsy API working, list your active listings:
# (The CLI will have a command for this in a future version)
# For now, you can find listing IDs in the Etsy URL:
# https://www.etsy.com/listing/1234567890/your-item-name
# The number after /listing/ is your listing_id
```

### 3.5 Slicing Tips for Bambu Lab P1S

- **Export format:** In Bambu Studio → Export → "Export plate sliced file" → saves as `.gcode.3mf`
- **Name files clearly:** Include size/variation in the filename
- **Organize by product:** One folder per listing
- **Test prints:** Always test-print each variation before listing it for sale
- **AMS (multi-filament):** If using AMS, the filament assignments are embedded in the 3MF. The dispatcher supports `use_ams=True` (though automatic filament mapping by order isn't implemented in v1)

---

## 4. Environment Variables (Complete `.env`)

```bash
# Etsy API v3
ETSY_API_KEY=your_keystring_here
ETSY_SHARED_SECRET=your_shared_secret_here
ETSY_REFRESH_TOKEN=your_refresh_token_here
ETSY_SHOP_ID=12345678

# Polling (how often to check for new orders)
POLL_INTERVAL_SECONDS=300     # 5 minutes

# Bambu Lab P1S Printer
PRINTER_HOST=192.168.1.100
PRINTER_ACCESS_CODE=your_access_code
PRINTER_SERIAL=03WXXXXXXXXXXXXX

# Paths
PRODUCTS_YAML_PATH=./products.yaml
DATABASE_URL=sqlite:///etsy_store.db
```

---

## 5. Verifying Everything Works

### 5.1 Test Etsy Connection

```bash
python -m etsy_store poll --once
```

Expected: "Polling for new orders..." — no errors (even if no new orders).

### 5.2 Test Printer Connection

```bash
python -c "
from etsy_store.dispatcher import PrinterDispatcher
d = PrinterDispatcher('192.168.1.100', 'YOUR_CODE', 'YOUR_SERIAL')
print('Printer dispatcher initialized successfully')
"
```

### 5.3 Test Product Mapper

```bash
python -c "
from etsy_store.mapper import ProductMapper
m = ProductMapper('./products.yaml')
path = m.resolve(2001, {'Size': '22 inch'})
print(f'Resolved: {path}')
"
```

### 5.4 Manual End-to-End Test

1. Place a test GCODE file in your product folder
2. Create a test order manually in the database (or use Etsy's sandbox)
3. Run `python -m etsy_store poll --once`
4. Check `python -m etsy_store queue` and `python -m etsy_store metrics`

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| `EtsyError: Token refresh failed: HTTP 401` | Refresh token expired | Re-run the OAuth flow to get a new refresh token |
| `EtsyError: HTTP 403` | API key doesn't have `transactions_r` scope | Re-authorize with the correct scopes |
| `FileNotFoundError: Product mapping file not found` | `products.yaml` path wrong | Check `PRODUCTS_YAML_PATH` in `.env` |
| `ConnectionRefusedError` on printer FTPS | Printer not in LAN mode or wrong IP | Enable LAN mode on printer, verify IP |
| `ftplib.error_perm: 530 Login incorrect` | Wrong access code | Get access code from printer settings |
| Test `test_mapper_raises_on_unknown_listing` fails | Listing ID not in YAML | Add the listing to `products.yaml` |
