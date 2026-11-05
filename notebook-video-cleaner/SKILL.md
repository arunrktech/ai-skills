---
name: notebook-video-cleaner
description: Repurpose NotebookLM explainer videos for every platform. Trims the generic welcome line and adds your logo, an AI disclosure label, branded intro/outro clips, vertical and square exports, levelled audio, thumbnails and batch runs.
---

# Notebook video cleaner

Use this when the user has a NotebookLM (Video Overview or explainer) video and wants it trimmed, branded, or reformatted for YouTube, Shorts, Reels, TikTok or LinkedIn. Works on one file or a whole folder.

## What it does

1. Trims the opening sentence (Welcome to this explainer, where we are unpacking...) so the video starts on real content.
2. `--logo my.png` adds the user's logo (`--logo-pos tr|tl|bl|br`, default top right, `--logo-scale 0.10`).
3. `--ai-label` adds a small "Made with AI" tag (custom text allowed, `--label-pos`, default bottom left).
4. `--format landscape,vertical,square` exports 16:9, 9:16 (1080x1920) and 1:1 (1080x1080) in one run with a blurred background fill.
5. `--loudnorm` levels audio toward about -14 LUFS.
6. `--intro clip.mp4` and `--outro clip.mp4` add branded bumpers, fitted automatically.
7. `--thumbnail` saves a JPG of the title card (`--thumb-at` picks the source second, default 1.0).
8. Folder input runs a batch and writes `batch_report.json`; every output gets a `_report.json`.

The source tool's own branding (NotebookLM watermark and end card) is left in place. Do not add options that remove or cover it; if the user asks, explain that it conflicts with Google's terms and point them to the logo placement options instead.

## Steps

1. Check tools: `which ffmpeg`, then `pip install --break-system-packages -q pocketsphinx pillow`. pocketsphinx ships its own English model, so nothing needs downloading. Without it the script falls back to silence detection.
2. Write the script below to `clean_notebook_video.py` in the working directory. Never modify the uploaded original.
3. Ask only what changes the output and was not stated: which formats, whether they have a logo or bumper clips, and whether to add the AI label. If the user is away, default to landscape with the AI label.
4. Dry run: `python3 clean_notebook_video.py INPUT.mp4 OUTPUT.mp4 --dry-run`. If start is 0.0 or above about 15s, transcribe the first 20s with pocketsphinx, pick the pause right after the welcome sentence, and pass `--start`.
5. Render with the chosen options, for example `python3 clean_notebook_video.py in.mp4 out/video.mp4 --format landscape,vertical --logo logo.png --ai-label --loudnorm --thumbnail`. With several formats, files are named `video_landscape.mp4`, `video_vertical.mp4` and so on.
6. Verify: check durations, view a frame from each format (logo and label placed, framing right), and transcribe the first 4 seconds to confirm the video opens on real content.
7. Deliver each output with SendUserFile and say in one line what was trimmed and what was added.

## Reference values (1280x720 NotebookLM export)

1. Intro sentence runs roughly 0 to 8.5s and ends on a 0.3 to 0.5s pause.
2. The NotebookLM watermark sits bottom right, so the logo defaults to top right and the AI label to bottom left.
3. The title card is on screen for roughly the first 10s, which makes 1.0s a good thumbnail frame.

## Script

```python
#!/usr/bin/env python3
# Copyright (c) 2026 Arun (arunrktech). MIT License, see LICENSE.
'''Notebook video cleaner: repurpose a NotebookLM explainer into a branded, multi platform video.

  1. Trims the spoken opening line ('Welcome to this explainer, where we...').
  2. --logo FILE.png       adds your logo (default top right, so the source tool's own mark stays visible)
  3. --format F            landscape (default), vertical (1080x1920 Shorts/Reels/TikTok),
                           square (1080x1080 LinkedIn/Instagram feed); several, comma separated
  4. --ai-label [TEXT]     adds a small 'Made with AI' tag for platform and EU AI Act disclosure
  5. --loudnorm            levels audio toward -14 LUFS (YouTube/Spotify target)
  6. --intro / --outro     adds your own branded clips (any size; fitted automatically)
  7. --thumbnail           saves a JPG thumbnail from the title card
INPUT may be a single video or a folder (batch). A JSON report is written next to each output.
Source branding (watermark, end card) is left in place; follow the terms of the tool that made the video.

Needs ffmpeg and Pillow (10.1+); pocketsphinx (pip) is optional and sharpens intro detection.
'''
import argparse, json, os, re, subprocess, sys, tempfile

FORMATS = {'landscape': None, 'vertical': (1080, 1920), 'square': (1080, 1080)}
VIDEO_EXT = ('.mp4', '.mov', '.m4v', '.mkv', '.webm')
CORNERS = ['tr', 'tl', 'bl', 'br']

def run(cmd):
    return subprocess.run(cmd, capture_output=True, text=True)

def probe(path):
    out = run(['ffprobe', '-v', 'error', '-show_entries',
               'stream=codec_type,width,height,r_frame_rate:format=duration', '-of', 'json', path]).stdout
    j = json.loads(out)
    v = next(s for s in j['streams'] if s['codec_type'] == 'video')
    has_audio = any(s['codec_type'] == 'audio' for s in j['streams'])
    num, den = v.get('r_frame_rate', '24/1').split('/')
    fps = round(float(num) / float(den or 1), 3) or 24
    return v['width'], v['height'], float(j['format']['duration']), has_audio, fps

# ---------- intro detection ----------

def silences(path, ss, t, noise='-40dB', d=0.25):
    err = run(['ffmpeg', '-v', 'info', '-ss', str(ss), '-t', str(t), '-i', path,
               '-af', f'silencedetect=n={noise}:d={d}', '-f', 'null', '-']).stderr
    starts = [float(x) + ss for x in re.findall(r'silence_start: ([0-9.]+)', err)]
    ends = [float(x) + ss for x in re.findall(r'silence_end: ([0-9.]+)', err)]
    return [(s, ends[i] if i < len(ends) else ss + t) for i, s in enumerate(starts)]

def welcome_time(path):
    '''Time the word welcome starts, via pocketsphinx if installed.'''
    try:
        from pocketsphinx import Decoder
    except ImportError:
        return None
    raw = subprocess.run(['ffmpeg', '-v', 'error', '-t', '20', '-i', path, '-ac', '1',
                          '-ar', '16000', '-f', 's16le', '-'], capture_output=True).stdout
    d = Decoder(samprate=16000)
    d.start_utt(); d.process_raw(raw, full_utt=True); d.end_utt()
    words = [(s.start_frame / 100, s.word.split('(')[0]) for s in d.seg()]
    for i, (t, w) in enumerate(words):
        nxt = [x[1] for x in words[i:i + 4]]
        if w == 'welcome' or ('come' in nxt[:2] and 'explain' in ' '.join(nxt)):
            return t
    return None

def find_start(path):
    w = welcome_time(path)
    base = (w if w is not None else 0.0) + 2.5   # the welcome sentence lasts several seconds
    gaps = [g for g in silences(path, 0, 40) if g[0] > base and g[1] - g[0] >= 0.3]
    if not gaps:
        return 0.0, 'no intro pause found, start kept'
    s, e = gaps[0]
    how = 'speech match' if w is not None else 'silence only'
    return round(s + (e - s) * 0.6, 2), f'pause after welcome sentence {s:.2f}-{e:.2f}s ({how})'

# ---------- rendering ----------

ENC = ['-c:v', 'libx264', '-preset', 'veryfast', '-crf', '18', '-pix_fmt', 'yuv420p',
       '-c:a', 'aac', '-b:a', '192k', '-ar', '44100', '-ac', '2',
       '-map_metadata', '0', '-movflags', '+faststart']

def make_label(text, width, path):
    '''Render the AI disclosure tag as a PNG pill.'''
    from PIL import Image, ImageDraw, ImageFont
    size = max(14, int(width * 0.018))
    font = ImageFont.load_default(size=size)
    l, t, r, b = ImageDraw.Draw(Image.new('RGBA', (1, 1))).textbbox((0, 0), text, font=font)
    pad = size // 2
    im = Image.new('RGBA', (r - l + pad * 2, b - t + pad * 2), (0, 0, 0, 0))
    d = ImageDraw.Draw(im)
    d.rounded_rectangle((0, 0, im.width - 1, im.height - 1), radius=im.height // 2, fill=(0, 0, 0, 150))
    d.text((pad - l, pad - t), text, font=font, fill=(255, 255, 255, 235))
    im.save(path)

def corner(pos, m):
    return {'br': f'W-w-{m}:H-h-{m}', 'bl': f'{m}:H-h-{m}', 'tr': f'W-w-{m}:{m}', 'tl': f'{m}:{m}'}[pos]

def build_graph(src_w, fmt_size, logo_idx, logo_pos, logo_scale, label_idx, label_pos):
    m = int(src_w * 0.015)
    parts = ['[0:v]setsar=1[v0]']
    cur = 'v0'
    if logo_idx:
        parts.append(f'[{logo_idx}:v]scale={max(16, int(src_w * logo_scale))}:-1,format=rgba[lg]')
        parts.append(f'[{cur}][lg]overlay={corner(logo_pos, m)}[v1]')
        cur = 'v1'
    if fmt_size:
        W, H = fmt_size
        parts.append(f'[{cur}]split[a][b]')
        parts.append(f'[a]scale={W}:{H}:force_original_aspect_ratio=increase,crop={W}:{H},boxblur=25:2,eq=brightness=-0.05[bg]')
        parts.append(f'[b]scale={W}:-2[fg]')
        parts.append(f'[bg][fg]overlay=(W-w)/2:(H-h)/2[v2]')
        cur = 'v2'
    if label_idx:   # added last so it sits in the corner of the final frame
        parts.append(f'[{cur}][{label_idx}:v]overlay={corner(label_pos, m)}[v3]')
        cur = 'v3'
    return ';'.join(parts), cur

def fit_clip(clip, W, H, fps, out):
    '''Re-encode a branded intro/outro clip to the target size, fps and audio layout.'''
    _, _, _, has_a, _ = probe(clip)
    vf = (f'scale={W}:{H}:force_original_aspect_ratio=decrease,'
          f'pad={W}:{H}:(ow-iw)/2:(oh-ih)/2:color=black,setsar=1,fps={fps}')
    cmd = ['ffmpeg', '-v', 'error', '-y', '-i', clip]
    if not has_a:
        cmd += ['-f', 'lavfi', '-i', 'anullsrc=r=44100:cl=stereo', '-shortest']
    cmd += ['-vf', vf] + ENC + [out]
    subprocess.run(cmd, check=True)

def concat(files, out):
    with tempfile.NamedTemporaryFile('w', suffix='.txt', delete=False) as f:
        for p in files:
            f.write('file ' + chr(39) + os.path.abspath(p) + chr(39) + chr(10))
        lst = f.name
    subprocess.run(['ffmpeg', '-v', 'error', '-y', '-f', 'concat', '-safe', '0', '-i', lst,
                    '-c', 'copy', '-movflags', '+faststart', out], check=True)
    os.unlink(lst)

def render(a, src, out, start, end, fmt, info):
    w, h, dur, has_audio, fps = info
    size = FORMATS[fmt]
    W, H = size or (w, h)
    with tempfile.TemporaryDirectory() as td:
        cmd = ['ffmpeg', '-v', 'error', '-y', '-ss', str(start), '-to', str(end), '-i', src]
        nxt, logo_idx, label_idx = 1, None, None
        if a.logo:
            cmd += ['-i', a.logo]; logo_idx = nxt; nxt += 1
        if a.ai_label:
            lp = os.path.join(td, 'label.png')
            make_label(a.ai_label, max(W, H), lp)
            cmd += ['-i', lp]; label_idx = nxt
        graph, vout = build_graph(w, size, logo_idx, a.logo_pos, a.logo_scale, label_idx, a.label_pos)
        length = end - start
        afx = ['afade=t=in:d=0.15', f'afade=t=out:st={max(0, length - 0.25):.2f}:d=0.25']
        if a.loudnorm:
            afx.append('loudnorm=I=-14:TP=-1:LRA=11')
        cmd += ['-filter_complex', graph, '-map', f'[{vout}]']
        if has_audio:
            cmd += ['-map', '0:a:0', '-af', ','.join(afx)]
        cmd += ENC
        if not (a.intro or a.outro):
            subprocess.run(cmd + [out], check=True)
            return
        main = os.path.join(td, 'main.mp4')
        subprocess.run(cmd + ['-r', str(fps), main], check=True)
        seq = []
        if a.intro:
            fit_clip(a.intro, W, H, fps, os.path.join(td, 'intro.mp4')); seq.append(os.path.join(td, 'intro.mp4'))
        seq.append(main)
        if a.outro:
            fit_clip(a.outro, W, H, fps, os.path.join(td, 'outro.mp4')); seq.append(os.path.join(td, 'outro.mp4'))
        concat(seq, out)

def thumbnail(src, at, out):
    subprocess.run(['ffmpeg', '-v', 'error', '-y', '-ss', str(at), '-i', src, '-frames:v', '1',
                    '-vf', 'scale=1280:-2', '-q:v', '2', out], check=True)

def out_name(base_out, fmt, multi):
    root, ext = os.path.splitext(base_out)
    return f'{root}_{fmt}{ext or ".mp4"}' if multi else base_out

def process(a, src, out):
    info = probe(src)
    w, h, dur, has_audio, fps = info
    if a.keep_intro:
        start, why_s = 0.0, 'kept (--keep-intro)'
    elif a.start is not None:
        start, why_s = a.start, 'given'
    else:
        start, why_s = find_start(src)
    end, why_e = (a.end, 'given') if a.end is not None else (dur, 'full length')
    fmts = [f.strip() for f in a.format.split(',') if f.strip()]
    for f in fmts:
        if f not in FORMATS:
            sys.exit(f'unknown format {f}; choose from {", ".join(FORMATS)}')
    report = {'input': src, 'source': f'{w}x{h} {dur:.2f}s', 'start': start, 'start_reason': why_s,
              'end': end, 'end_reason': why_e, 'ai_label': a.ai_label, 'outputs': []}
    print(f'{os.path.basename(src)}: {w}x{h}, {dur:.2f}s')
    print(f'  start {start}s ({why_s})')
    print(f'  end   {end}s ({why_e})')
    if a.dry_run:
        return report
    os.makedirs(os.path.dirname(os.path.abspath(out)), exist_ok=True)
    for f in fmts:
        o = out_name(out, f, len(fmts) > 1)
        render(a, src, o, start, end, f, info)
        report['outputs'].append({'format': f, 'file': o, 'duration': round(probe(o)[2], 2)})
        print(f'  wrote {o}')
    if a.thumbnail:
        t = os.path.splitext(out)[0] + '_thumbnail.jpg'
        thumbnail(src, a.thumb_at, t)
        report['thumbnail'] = t
        print(f'  wrote {t}')
    with open(os.path.splitext(out)[0] + '_report.json', 'w') as fh:
        json.dump(report, fh, indent=2)
    return report

def main():
    ap = argparse.ArgumentParser(description='Repurpose NotebookLM explainer videos for every platform.')
    ap.add_argument('input', help='video file or folder of videos')
    ap.add_argument('output', help='output file (single input) or folder (batch)')
    ap.add_argument('--start', type=float); ap.add_argument('--end', type=float)
    ap.add_argument('--keep-intro', action='store_true')
    ap.add_argument('--logo', help='your own logo (PNG with transparency works best)')
    ap.add_argument('--logo-pos', default='tr', choices=CORNERS)
    ap.add_argument('--logo-scale', type=float, default=0.10, help='logo width as share of video width')
    ap.add_argument('--ai-label', nargs='?', const='Made with AI', help='add an AI disclosure tag')
    ap.add_argument('--label-pos', default='bl', choices=CORNERS)
    ap.add_argument('--format', default='landscape', help='landscape, vertical, square (comma separated)')
    ap.add_argument('--loudnorm', action='store_true', help='level audio to about -14 LUFS')
    ap.add_argument('--intro', help='branded clip to put before the video')
    ap.add_argument('--outro', help='branded clip to put after the video')
    ap.add_argument('--thumbnail', action='store_true')
    ap.add_argument('--thumb-at', type=float, default=1.0, help='source time of thumbnail frame (title card)')
    ap.add_argument('--dry-run', action='store_true')
    a = ap.parse_args()
    if os.path.isdir(a.input):
        srcs = sorted(os.path.join(a.input, f) for f in os.listdir(a.input) if f.lower().endswith(VIDEO_EXT))
        if not srcs:
            sys.exit('no videos found in ' + a.input)
        results = []
        for s in srcs:
            name = os.path.splitext(os.path.basename(s))[0] + '_clean.mp4'
            try:
                results.append(process(a, s, os.path.join(a.output, name)))
            except Exception as e:
                print(f'  FAILED {s}: {e}')
                results.append({'input': s, 'error': str(e)})
        if not a.dry_run:
            os.makedirs(a.output, exist_ok=True)
            with open(os.path.join(a.output, 'batch_report.json'), 'w') as fh:
                json.dump(results, fh, indent=2)
        print(f'done: {sum("error" not in r for r in results)}/{len(results)} videos')
    else:
        process(a, a.input, a.output)

if __name__ == '__main__':
    main()
```
