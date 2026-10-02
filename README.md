# AliExpress MCP Server

An MCP server that lets Claude (or any MCP client) search AliExpress, read product details, check shipping, and look at your cart.

**This fork** of [justinritchie/aliexpress-mcp-server](https://github.com/justinritchie/aliexpress-mcp-server) defaults to shipping to **Israel** with prices in **USD**. You can change both ([Configuration](#configuration)). Otherwise it tracks upstream.

It is read-only by design. It does **not** add to cart, check out, or pay.

## What you can ask

| Tool | Needs login? | What it does |
|------|--------------|--------------|
| `search_products(query, min_rating, max_price, sort_by)` | No | Search AliExpress. Sort by `best_match`, `orders`, `price_asc` or `price_desc`, and filter by rating or price. |
| `get_product_details(item_id or url)` | No | Title, price and discount, rating, sold count, seller, shipping cost and delivery estimate. |
| `get_shipping_estimate(item_id)` | No | Shipping cost and delivery estimate to your country. |
| `view_cart()` | **Yes** | What's in your AliExpress cart. Needs the Chrome extension ([Cart access](#cart-access-optional-chrome-extension)). |

Example prompts:

- "Find a USB-C cable on AliExpress with at least 4.7 stars, under $5, sorted by orders."
- "What does shipping to Israel cost for https://www.aliexpress.com/item/1005007655628250.html?"
- "What's in my AliExpress cart, and what's the total?"

## Setup

You need **Python 3.10+** and `git`.

### 1. Download and install

```bash
git clone https://github.com/ohadle/aliexpress-mcp-server.git
cd aliexpress-mcp-server
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

### 2. Check that it works

```bash
.venv/bin/python smoke_test.py
```

This runs a real search, product lookup and shipping check (about 5 requests to aliexpress.com). You should see results with USD prices and shipping to IL.

### 3. Connect it to Claude

Use **absolute paths** in both cases. Run `pwd` inside the folder to get yours; below it's `/path/to/aliexpress-mcp-server`.

**Claude Desktop:** open **Settings → Developer → Edit Config**, or edit the file directly:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Add the `aliexpress` entry inside `mcpServers`:

```json
{
  "mcpServers": {
    "aliexpress": {
      "command": "/path/to/aliexpress-mcp-server/.venv/bin/python",
      "args": ["/path/to/aliexpress-mcp-server/aliexpress_mcp_server.py"]
    }
  }
}
```

Then quit and reopen Claude Desktop. On Windows the Python path is `.venv\Scripts\python.exe`.

**Claude Code:**

```bash
claude mcp add aliexpress --scope user -- /path/to/aliexpress-mcp-server/.venv/bin/python /path/to/aliexpress-mcp-server/aliexpress_mcp_server.py
```

Search, product details and shipping work now. Only follow the next section if you want cart access.

## Cart access (optional): Chrome extension

`view_cart` needs your logged-in AliExpress session. The server reads it from `~/.mcp-credentials/aliexpress.json`, which is written by the [MCP Auth Bridge](https://github.com/justinritchie/mcp-auth-bridge) Chrome extension. A Chrome extension is needed because AliExpress login cookies are httpOnly, so ordinary scripts can't read them.

The extension's installer supports **macOS and Chrome** only.

### One-time install

1. Download the extension:

   ```bash
   git clone https://github.com/justinritchie/mcp-auth-bridge.git
   ```

2. In Chrome, open `chrome://extensions`, turn on **Developer mode** (top right), click **Load unpacked**, and select the `mcp-auth-bridge` folder.
3. Copy the **ID** shown on the extension's card, a 32-letter string such as `abcdefghijklmnopabcdefghijklmnop`.
4. Register the helper that lets the extension write files, using that ID:

   ```bash
   cd mcp-auth-bridge
   ./install.sh YOUR_EXTENSION_ID
   ```

5. **Quit Chrome completely (⌘Q) and reopen it.** Chrome only picks up the helper on restart.
6. Optionally, pin the extension: click the puzzle-piece icon in Chrome's toolbar, then the pin next to **MCP Auth Bridge**.

AliExpress is already configured in the extension, so you don't need to edit anything.

### Save your session

1. Go to [aliexpress.com](https://www.aliexpress.com) in Chrome and log in.
2. Click the MCP Auth Bridge icon, then **Save AliExpress**. The card should change to **Saved**.
3. Check that the file exists:

   ```bash
   ls -l ~/.mcp-credentials/aliexpress.json
   ```

The server reads this file on every call, so you don't need to restart Claude.

**When it expires:** if `view_cart` reports that the session expired, or shows an empty cart when you have items, open aliexpress.com in Chrome and click **Save AliExpress** again.

> **Keep this file private.** `~/.mcp-credentials/aliexpress.json` holds your logged-in AliExpress session. Don't commit it or share it. To revoke it, log out of AliExpress in Chrome and delete the file.

## Configuration

All settings are optional environment variables. In Claude Desktop, add them under an `"env"` object next to `"args"`. In Claude Code, pass them with `-e` after the server name, e.g. `claude mcp add aliexpress -e ALIEXPRESS_COUNTRY=DE -e ALIEXPRESS_CURRENCY=EUR --scope user -- …`.

| Variable | Default | Meaning |
|----------|---------|---------|
| `ALIEXPRESS_COUNTRY` | `IL` | Ship-to country, as a two-letter code (`US`, `GB`, `DE`, `CA`, …). |
| `ALIEXPRESS_CURRENCY` | `USD` | Display currency (`EUR`, `GBP`, `ILS`, …). |
| `ALIEXPRESS_LOCALE` | `en_US` | Language of titles and text. |
| `ALIEXPRESS_CREDENTIALS` | `~/.mcp-credentials/aliexpress.json` | Where the Chrome extension's session file is. |

For example, to ship to Germany with prices in euros:

```json
"env": { "ALIEXPRESS_COUNTRY": "DE", "ALIEXPRESS_CURRENCY": "EUR" }
```

## Troubleshooting

- **The tools don't appear in Claude.** Check that both paths in the config are absolute and exist, and that `command` points to the `.venv` Python. Restart Claude Desktop fully. In Claude Code, run `claude mcp list`.
- **`ModuleNotFoundError: mcp`, or a FastMCP import error.** The server is running with the wrong Python. Point `command` at `.venv/bin/python`, and reinstall with `.venv/bin/pip install -r requirements.txt` (this pins `mcp<2`).
- **Search returns nothing.** Run `.venv/bin/python smoke_test.py`. If it fails too, AliExpress may have changed its page layout; please open an issue.
- **`view_cart` says no session was found.** The session file is missing or empty. Redo [Save your session](#save-your-session).
- **Clicking Save does nothing, or shows a native-host error.** The helper isn't registered for this extension ID. Rerun `./install.sh` with the ID from `chrome://extensions`, then quit and reopen Chrome.
- **Wrong country or currency.** Set `ALIEXPRESS_COUNTRY` / `ALIEXPRESS_CURRENCY`. These override your account's and IP's region.

## How it works

The initial product page is a client-side-rendered shell (`window._d_c_.isCSR = true`, `window.runParams = {}`) — real product data loads over AJAX from AliExpress's MTOP API after the JS runs. So HTML scraping doesn't work for PDP data.

Instead, this server replicates what `mtop.js` does in the browser: it makes signed calls to `mtop.aliexpress.pdp.pc.query` using the session's `_m_h5_tk` cookie as the HMAC token. The signing algorithm:

```
sign = md5(token + "&" + t_ms + "&" + "12574478" + "&" + json_payload)
```

where `token` is the prefix of `_m_h5_tk` before the underscore, and `12574478` is AliExpress's public web `appKey`. Token refresh on `FAIL_SYS_TOKEN_EXPIRED` is handled automatically.

Search still uses the simpler HTML path — no signed calls needed. `window.runParams` no longer exists on search pages, so cards are read from the embedded `"itemList":{"content":[...]}` array in the SSR HTML.

## Known limitations

- **Low-volume / brand-new listings** sometimes return empty MTOP responses (endpoint returns `SUCCESS` but an empty data block). Likely a region/visibility gate. Search results still show the listing fine.
- **Cookies expire / lag the cart.** When tool calls start returning "session expired" — or `view_cart` reports an empty cart despite having items — re-open aliexpress.com in Chrome and click **Save AliExpress** again to capture fresh session cookies. The `_m_h5_tk` token and cart state rotate together; stale cookies show an empty server-side cart.
- **Rate limiting** is the user's responsibility. The server sends realistic Chrome headers but doesn't throttle; don't hammer the search.

## Development

```bash
.venv/bin/pip install pytest
.venv/bin/python -m pytest tests     # offline, no network
.venv/bin/python smoke_test.py       # live check against aliexpress.com
```

## Architecture

Follows the pattern of [dekudeals-mcp-server](https://github.com/justinritchie/dekudeals-mcp-server) — FastMCP + httpx + BeautifulSoup — with added MTOP client code in `aliexpress_mcp_server.py`. Session cookies come from the shared credential file written by the [MCP Auth Bridge](https://github.com/justinritchie/mcp-auth-bridge) Chrome extension.

The MTOP response field paths (`PRODUCT_TITLE.text`, `PRICE.targetSkuPriceInfo.salePriceString`, `PC_RATING.rating`, `SHIPPING.originalLayoutResultList[0].bizData.displayAmount`, etc.) were reverse-engineered live from a real response in April 2026 and are documented inline in `_extract_pdp_fields()`. If AliExpress reorganizes the component layout, that function is the place to update.
