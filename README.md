# Robot Air Hockey — project site

Source for <https://air-hockey-rl.github.io/>.

A small Jekyll site: a landing page about the air hockey project, plus blog posts under
`_posts/`. GitHub Pages builds it on push; there is no local build step and no theme gem,
so nothing needs Ruby or Bundler.

```
_config.yml          site metadata, baseurl
_layouts/post.html   page shell
assets/css/main.css  all styling (hand-rolled; no theme gem, no build step)
assets/favicon.svg   tab icon
assets/media/        transcoded video (H.264 mp4 + jpg posters) and figures
index.md             landing page (/)
_posts/              one file per post, each with its own `permalink:`
```

Every page uses the same `post` layout, keyed off front matter:

| key | effect |
| --- | --- |
| `home: true` | landing page only. Suppresses the "&larr; Robot Air Hockey" back-link and the `&middot; site.title` suffix in `<title>`. |
| `author:` | shows the hero byline and the footer copyright line. Set it on posts; omit it on the landing page, which is about the project rather than by anyone. |
| `date:` | shows in the hero byline and orders the post index. Jekyll takes it from front matter, not the filename — keep the two in step anyway. |
| `permalink:` | the post's URL. Without it a post lands under `/YYYY/MM/DD/`. |
| `description:` | meta and Open Graph description, falling back to `site.description`. |
| `eyebrow:`, `image:` | hero label and Open Graph image. |

There is no site-level `author`, so attribution never leaks onto a page that did not ask
for it. The landing page's `## Posts` section loops over `site.posts`, so a new file in
`_posts/` appears there on its own.

## Editing

Edit the page and push to `main`. Reference assets through `relative_url` so they
resolve correctly if `baseurl` ever changes:

```liquid
<video src="{{ '/assets/media/name.mp4' | relative_url }}" autoplay loop muted playsinline></video>
```

To eyeball a page locally without Ruby, `python3 .preview.py [page.md]` writes a rough
`_preview*.html` next to it (gitignored, not a build step).

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
