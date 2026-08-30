# From Testbed to Robot — project blog

Source for <https://air-hockey-rl.github.io/>.

A single-page Jekyll site. GitHub Pages builds it on push; there is no local build step
and no theme gem, so nothing needs Ruby or Bundler.

```
_config.yml          site metadata, baseurl
_layouts/post.html   page shell
assets/css/main.css  all styling (hand-rolled; no theme gem, no build step)
assets/favicon.svg   tab icon
assets/media/        transcoded video (H.264 mp4 + jpg posters)
index.md             the post
```

## Editing

Edit `index.md` and push to `main`. Reference assets through `relative_url` so they
resolve correctly if `baseurl` ever changes:

```liquid
<video src="{{ '/assets/media/name.mp4' | relative_url }}" autoplay loop muted playsinline></video>
```

## Media

Source footage lives in the research repo under `blog/Videos/` (gitignored) and is
transcoded before landing here — the originals total ~110 MB, the shipped versions 4 MB.
GIFs and HEVC `.mov` files are not web-safe; convert with:

```bash
# animated GIF -> mp4
ffmpeg -i in.gif -vf "scale=trunc(iw/2)*2:trunc(ih/2)*2" \
  -c:v libx264 -crf 23 -pix_fmt yuv420p -movflags +faststart out.mp4

# HEVC/QuickTime -> 720p H.264
ffmpeg -i in.mov -vf "scale=-2:720" -c:v libx264 -crf 26 -preset slow -an \
  -movflags +faststart out.mp4

# poster frame for any video using `controls` without autoplay
ffmpeg -ss 3 -i out.mp4 -frames:v 1 -q:v 4 out.jpg
```
