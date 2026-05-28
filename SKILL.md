---
name: huaban-scraper
description: Scrape all images from a Huaban (花瓣) board. Uses the public API to extract pin data and downloads full-resolution images from the CDN.
triggers:
  - "huaban.com/boards"
  - "huaban.com"
  - "花瓣"
  - "花瓣网"
  - "扒花瓣"
---

# Huaban (花瓣) Board Scraper

Extract ALL images from any public Huaban board via the public API.

## Workflow

### Step 1: Parse Board URL

Board URL format: `https://huaban.com/boards/{BOARD_ID}`

Extract the numeric `BOARD_ID` from the URL.

### Step 2: Get Board Info

```bash
curl -s -H "User-Agent: Mozilla/5.0" -H "Accept: application/json" \
  "https://huaban.com/v3/boards/{BOARD_ID}"
```

Parse `board.title` (board name), `board.pin_count` (image count), `board.user.username`.

### Step 3: Fetch All Pins

```bash
curl -s -H "User-Agent: Mozilla/5.0" -H "Accept: application/json" \
  "https://huaban.com/v3/boards/{BOARD_ID}/pins?limit={pin_count}"
```

Each pin contains:
- `raw_text`: the title/description
- `file.url`: full CDN download URL with auth_key
- `file.type`: MIME type (image/jpeg, image/png)
- `file.width` x `file.height`: dimensions

### Step 4: Download All Images

The `file.url` field contains the full CDN URL including auth_key. Download with browser headers:

```powershell
$headers = @{
    "User-Agent" = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36"
    "Referer" = "https://huaban.com/"
    "Accept" = "image/avif,image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8"
    "Accept-Language" = "zh-CN,zh;q=0.9"
}
Invoke-WebRequest -Uri $pin.file.url -Headers $headers -OutFile $dest
```

Download to `~/Desktop/{BoardName}_花瓣/`.

### Step 5: Notion Integration (Optional)

If the user has Notion configured:
1. Create a new database under the Notion workspace page, named after the board
2. Use the CDN URLs as external cover images
3. Add image blocks in page content

**Note:** Huaban CDN URLs contain expiring auth_keys. Images will display in Notion temporarily but may expire. For permanent storage, users should drag local files into Notion.

## Key Details

- **No DevTools needed** — Huaban has a public REST API
- **Auth key expiry** — CDN URLs have a `auth_key` parameter that expires (usually same day). Download immediately.
- **No pagination** — Single API call with `limit={total}` gets all pins
- **Per-pin naming** — Use `raw_text` as filename; fallback to "图片 {index}"

## Common Pitfalls

- **403/567 CDN error**: The auth_key in the URL has expired or the headers are insufficient. Re-fetch the API to get fresh URLs.
- **0 pins returned**: The board might be private. The API only returns public boards.
- **Downloaded file is HTML not image**: Missing the browser Accept and Referer headers — the CDN requires them.
