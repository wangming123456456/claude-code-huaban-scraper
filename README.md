# Claude Code Huaban (花瓣) Board Scraper

A Claude Code skill that scrapes all images from any public Huaban board. Uses the public REST API.

## What It Does
- Takes a Huaban board URL
- Extracts all image data via the public API
- Downloads full-resolution images to a local folder
- Optionally imports into Notion as a new database

## Installation
```bash
git clone https://github.com/wangming123456456/claude-code-huaban-scraper.git ~/.claude/skills/huaban-scraper
```

## Usage
Just paste a Huaban board URL to Claude Code.

## How It Works
1. Board info endpoint: `/v3/boards/{BOARD_ID}`
2. Pins endpoint: `/v3/boards/{BOARD_ID}/pins?limit={count}`
3. Each pin contains a CDN download URL
4. Downloads with browser headers to bypass restrictions

## Requirements
- Windows 10/11
- Claude Code
