---
name: openclaw-support
description: Clipcat - TikTok e-commerce video creation skill. Video search, product insights, viral replication, product-to-video generation, breakdown analysis, and video download via Clipcat CLI.
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "requires": { "env": ["CLIPCAT_API_KEY"] },
        "primaryEnv": "CLIPCAT_API_KEY",
      },
    "homepage": "https://clipcat.ai",
  }
---

# Clipcat CLI

Use this skill when you need TikTok e-commerce video creation through `clipcat`.

Get API key: https://clipcat.ai/workspace?modal=settings&tab=apikeys

This skill is intentionally short. Detailed flags and supported values belong to the CLI itself — always treat `clipcat -h` and `clipcat <subcommand> -h` as the primary reference.

## Installation

```bash
curl -fsSL https://clipcat.ai/cli | bash
clipcat config --api-key <your-key>
```

## What this CLI is for

`clipcat` is the local entrypoint for all Clipcat AI video generation workflows:

- Search viral TikTok videos by keyword
- Search TikTok Shop products by keyword (market intelligence)
- Get TikTok Shop product details and reviews
- Replicate viral videos with your product
- Generate product videos from images
- Generate AI images from text prompts using GPT Image 2 (with optional reference images)
- Analyze videos (script, scenes, music)
- Download TikTok/Douyin videos
- Query async task status

## Default agent workflow

1. Start with `clipcat -h` to see all commands.
2. Before using any command, run `clipcat <subcommand> -h` to see flags.
3. Default to JSON output.
4. Warn the user before running commands that consume credits.

## Choosing the right command

- `search` — find viral TikTok videos by keyword; supports `--region`, `--sort-by relevance|likes`, `--time-range any|day|week|month|quarter|half_year`, `--require-shop`
- `search_items` — search TikTok Shop products by keyword; returns market insights, competitor shops, and product intelligence; supports `--region`, `--offset`, `--page-token` for pagination
- `product_detail` — get product info by `--input <ID or URL>`; supports `--region`
- `product_comment` — get product reviews by `--input <ID or URL>`; supports `--region`, `--sort-rule`, `--filter-type`, `--filter-value`
- `replicate` — replicate a viral video with your product images (auto-detects URL type); images via `--image` (local) or `--image-url` (URL); local files and URLs can be mixed; supports `--model`, `--duration`, `--size` (only `9:16` or `16:9`), `--lang`, `--resolution`, `--character-id`
- `product_video` — generate video from product images only (no reference video); images via `--image` (local) or `--image-url` (URL); local files and URLs can be mixed; `--size` only accepts `9:16` or `16:9`
- `image` — generate an AI image from a text prompt using **GPT Image 2** model; optionally supply up to 5 reference images via `--image` (local file) or `--image-url` (URL). Use `--aspect-ratio` to pick `1:1` (default) / `16:9` / `9:16`. **Dimension hints (9:16/16:9/1:1, portrait/landscape/square, 竖版/横版/方图, banner, wallpaper) must appear in BOTH `--prompt` and `--aspect-ratio`** — `--aspect-ratio` sets canvas, the prompt hint anchors framing. Don't invent dimensions the user didn't ask for.
- `list_images` — list image generation tasks from server; supports `--status` / `--limit` / `--page` filters
- `breakdown` — analyze a video (script, scenes, music); returns cached result immediately if previously analyzed
- `download` — download TikTok/Douyin video (returns signed URL); cached results return immediately
- `user_videos` — get a TikTok user's video list with analytics (plays, likes, shares, comments, e-commerce cart data); `--unique-id` required; pass `--sec-user-id` to skip ID resolution and speed up response; supports `--max-cursor` pagination and `--sort-type 0|1`
- `query_task` — check status of a task by ID and type (`--type replicate | product | breakdown | download | image`). Omit `--task-id` to resume the latest local task.
- `list_tasks` — list recent **video-related** tasks from server (`--type` required: `replicate | product | breakdown | download`). Image tasks use `list_images`.

## replicate: URL type auto-detection

`clipcat replicate` automatically detects the URL type:

- **TikTok/Douyin link** → calls `/replicate_from_social` (costs **1 extra credit** for download)
- **Direct video URL** → calls `/replicate`

Always inform the user about the extra credit before running with a social URL.

## clipcat:// asset references

`clipcat://...` strings seen in earlier turns are stable asset references. Pass them **verbatim** to any `--image-url` / `--character-id` flag — never prepend `https://` or modify them. See subcommand `-h` for details.

## Async task rules

`replicate`, `product_video`, `image`, and `breakdown` are async. All four
**submit and return immediately** with a task ID — they never block.

Typical durations: `image` ~3 min, `breakdown` a few minutes, `product_video` /
`replicate` 10+ min. **Never try to wait synchronously inside a single tool
call** — every realistic agent harness has a tool-call timeout (commonly 60s)
that will kill the call long before the task is done. Always go submit → return
→ poll across turns.

1. Task ID is saved locally to `~/.clipcat/tasks.json` automatically.
2. Check status with `clipcat query_task --task-id <id> --type <type>`. Each
   call returns immediately with the current status. Omit `--task-id` to resume
   the latest task. Re-invoke the command across turns (suggested cadence:
   ~30s for `image`, ~1-2 min for `breakdown` / `product_video` / `replicate`)
   until `status` is `completed` or `failed`.
3. Use `clipcat list_tasks --type <replicate|product|breakdown|download>` to
   see tasks of a given type from the server.

## query_task: auto-resume

`clipcat query_task` with no flags automatically reads the latest task from `~/.clipcat/tasks.json` and resumes it. No need to remember task IDs.

## Available models

Trial channels are per-clip pricing; standard channels are per-second pricing.

| Model ID             | Duration              | Resolution        | Notes                                                             |
| -------------------- | --------------------- | ----------------- | ----------------------------------------------------------------- |
| `sora2_official_exp` | 4s, 8s, 12s           | 720p              | **Trial**, default. OpenAI Sora 2 official channel, 9:16 or 16:9  |
| `veo3.1fast`         | 8s, 16s, 24s          | 720p, 1080p       | **Trial**. Google Veo 3.1 Fast, balanced quality and cost         |
| `veo3.1pro`          | 8s, 16s, 24s          | 720p, 1080p       | **Trial**. Google Veo 3.1 Pro, high-quality variant               |
| `omini_flash`        | 10s                   | 720p, 1080p       | **Trial**. Gemini Omni Flash, Google's newest model               |
| `grok_imagine`       | 10s, 15s, 20s, 30s    | 720p              | **Trial**. 9:16 aspect ratio only, longer clips                   |
| `seedance2`          | 4-15s (any integer)   | 480p, 720p, 1080p | Standard. ByteDance Seedance 2, top quality                       |
| `seedance2_fast`     | 4-15s (any integer)   | 480p, 720p, 1080p | Standard. ByteDance Seedance 2 Fast, fast variant                 |
| `happyhorse10`       | 3-15s (any integer)   | 720p, 1080p       | Standard. Alibaba HappyHorse 1.0                                  |

Always check `clipcat replicate -h` for the current model list.

## Supported languages (`--lang`)

`en` `zh` `fr` `de` `ms` `vi` `th` `ja` `ko` `id` `fil` `es`

## Good agent behavior

- Run `clipcat -h` first if unsure which command to use.
- Show parameters to user and get confirmation before running paid commands.
- Keep record of task IDs; re-invoke `query_task` across turns to track long-running tasks.
- Preserve signed video URLs intact — they contain `X-Amz-*` params that break if truncated.
- Agents should prefer the default JSON output.
