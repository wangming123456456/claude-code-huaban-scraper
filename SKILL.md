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

### Step 3: Fetch All Pins (with Pagination)

API caps at **100 pins per request**. Loop with `max` parameter until no more pins:

```python
import json, urllib.request
H = {"User-Agent": "Mozilla/5.0", "Accept": "application/json"}
all_pins = []
last_id = None
while True:
    url = f"https://huaban.com/v3/boards/{BOARD_ID}/pins?limit=100"
    if last_id: url += f"&max={last_id}"
    req = urllib.request.Request(url, headers=H)
    with urllib.request.urlopen(req, timeout=30) as resp:
        data = json.loads(resp.read())
    pins = data["pins"]
    if not pins: break
    all_pins.extend(pins)
    last_id = pins[-1]["pin_id"]
    if len(pins) < 100: break
```

Each pin contains:
- `raw_text`: the title/description
- `file.url`: full CDN download URL with auth_key
- `file.type`: MIME type (image/jpeg, image/png, image/webp)
- `file.width` x `file.height`: dimensions

### Step 4: Download All Images (with auto-compress)

Use Python `urllib` (NOT PowerShell — encoding issues on Windows). Key rules:

**Filename rules:**
- Prefer `raw_text` as filename
- Sanitize: replace `\n \r \t` with space, strip `\/:*?"<>|`
- If raw_text is empty, use `"img_{pin_id}"`
- If filename already exists in folder, prepend `{pin_id}_` to avoid overwrite

**Size auto-compress:**
- After download, check file size
- If `>= 5MB` AND not already `.jpg` → convert to JPEG quality 85 via PIL, remove original

```python
from PIL import Image
MAX_SIZE_MB = 5

# ... download to dest ...
size_mb = os.path.getsize(dest) / (1024 * 1024)
if size_mb >= MAX_SIZE_MB and not dest.endswith('.jpg'):
    img = Image.open(dest)
    new_dest = dest.rsplit('.', 1)[0] + '.jpg'
    img.convert('RGB').save(new_dest, 'JPEG', quality=85)
    os.remove(dest)
```

Download to `d:/VS/花瓣素材/{BoardName}/`.

### Step 5: Notion Integration

1. Create database under "花瓣" page (ID: `399edf71-069e-80e5-a42a-f732f466d0c7`)
2. User opens database → switches to Gallery view → drags local folder in manually
3. Do NOT create pages via API — let the user drag files themselves

## Key Details

- **No DevTools needed** — Huaban has a public REST API
- **Pagination required** — API caps at 100 per call; paginate with `&max={last_pin_id}`
- **Auth key expiry** — CDN URLs expire same day. Download immediately after fetching.
- **pin_count ≠ actual** — Board pin_count includes repins. API only returns unique original pins.
- **Use `py` on Windows** — not `python` directly
- **long filenames** — If all pins have the same title (e.g. GameUI screenshots), rename to short numbered format after download

## Common Pitfalls

- **Only 100 pins returned**: Need pagination with `&max=` parameter
- **Filename has newlines**: raw_text can contain `\n`, must sanitize before using as filename
- **All files overwritten to 1 file**: All pins share same raw_text → use `{pin_id}_` prefix for dedup, or rename to numbered format after download
- **403/567 CDN error**: Auth key expired. Re-fetch API for fresh URLs.
- **0 pins returned**: Board might be private, or all pins are repins from deleted sources.
- **Chinese garbled in terminal**: Use Python, not PowerShell. GBK encoding issue.
