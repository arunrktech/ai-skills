# Notebook Video Cleaner

**Turn a NotebookLM explainer into branded videos for every platform, in one command.**

NotebookLM Video Overviews are a great way to explain a topic, but each export is one landscape file that opens with a generic "Welcome to this explainer" line. This tool trims that line, adds your own branding and an AI disclosure label, and exports versions for YouTube, Shorts, Reels and LinkedIn.

![Landscape, vertical and square exports with a branded intro and thumbnail](docs/preview.png)

## Features

| | |
|---|---|
| **Smart intro trim** | Finds the "Welcome to this explainer" sentence by speech and cuts at the natural pause after it |
| **Your logo** | Places your PNG logo in any corner (top right by default), sized to the video |
| **AI disclosure label** | Optional "Made with AI" tag that helps with platform rules and the EU AI Act |
| **Multi platform export** | 16:9 YouTube, 9:16 Shorts / Reels / TikTok, 1:1 LinkedIn / Instagram feed, in one run with a blurred background fill |
| **Branded bumpers** | Adds your own intro and outro clips, any size, fitted automatically |
| **Loudness levelling** | Normalises audio toward the -14 LUFS streaming target |
| **Thumbnail** | Saves a JPG of the title card |
| **Batch mode** | Point it at a folder and process every video, with a JSON report per file |
| **Fully offline** | ffmpeg plus a bundled speech model; no API keys, no uploads |

## Install

```bash
# ffmpeg must be on PATH
pip install pillow pocketsphinx   # pocketsphinx optional, improves intro detection
```

## Quick start

```bash
# preview the cut point without rendering
python3 scripts/clean_notebook_video.py explainer.mp4 out/explainer.mp4 --dry-run

# trim the opening line and add your logo plus an AI label
python3 scripts/clean_notebook_video.py explainer.mp4 out/explainer.mp4 --logo brand/logo.png --ai-label

# full creator pack: three formats, levelled audio, thumbnail
python3 scripts/clean_notebook_video.py explainer.mp4 out/explainer.mp4 \
  --logo brand/logo.png --ai-label --format landscape,vertical,square --loudnorm --thumbnail

# add branded bumpers
python3 scripts/clean_notebook_video.py explainer.mp4 out/explainer.mp4 \
  --intro brand/intro.mp4 --outro brand/subscribe.mp4

# batch a whole folder
python3 scripts/clean_notebook_video.py exports/ cleaned/ --format landscape,vertical --ai-label
```

## All options

| Option | Purpose |
|---|---|
| `--format` | `landscape`, `vertical`, `square`; comma separated for several |
| `--logo FILE` | Your logo image (transparent PNG works best) |
| `--logo-pos` | `tr` (default), `tl`, `bl`, `br` |
| `--logo-scale` | Logo width as a share of video width (default 0.10) |
| `--ai-label [TEXT]` | Add a disclosure tag; default text "Made with AI" |
| `--label-pos` | `bl` (default), `br`, `tl`, `tr` |
| `--intro FILE` / `--outro FILE` | Branded clips to add before / after |
| `--loudnorm` | Level audio to about -14 LUFS |
| `--thumbnail` / `--thumb-at S` | Save a thumbnail from second S of the source (default 1.0) |
| `--keep-intro` | Skip the intro trim |
| `--start S` / `--end S` | Set cut points by hand |
| `--dry-run` | Print the plan only |

## How intro detection works

pocketsphinx spots the word "welcome" in the first 20 seconds, then the cut lands inside the first pause of at least 0.3s that follows. Without pocketsphinx it uses silence detection alone. Each run writes `<name>_report.json` with the cut point and the reason.

## Responsible use

This tool leaves the source tool's own branding (the NotebookLM watermark and end card) in place. Follow the terms of the service that produced your video, and tick your platform's AI disclosure option when a video could be mistaken for real footage. The `--ai-label` option makes that easy.

## Use as a Claude skill

`SKILL.md` contains the full instructions with the script inline. Add it to Claude as a skill and ask: *"Make a vertical version of this NotebookLM video with my logo and an AI label."*

## License

[MIT](../LICENSE)
