# Deploying to acknowledgerfirst.com (CloudPanel / nginx) — domain root

This build has **plain root-absolute paths** (`/_next/...`, `/brand/...`,
`/team/...`, `/video/...`) with **no subfolder prefix**. It is built for
serving directly at `https://acknowledgerfirst.com/` — the domain root,
not `/AFPL` or any other subfolder.

## Why the site was broken before

The previous build you deployed had every asset path hard-coded with an
`/AFPL/` prefix (made for a `domain.com/AFPL` subfolder deployment). You
deployed it to the domain root instead, so the browser requested
`/AFPL/_next/...`, `/AFPL/video/enterprise.mp4`, etc., which don't exist
there — hence the wall of 404s. This build fixes that: it has no `/AFPL`
anywhere, confirmed by grepping every HTML/JS/CSS file.

## Important: the video file is NOT included in this zip

`video/enterprise.mp4` is unchanged from what you already deployed (it
was never touched by the `/AFPL` fix), and it's 55MB, so it's left out
here to keep this download small. **Do not delete your existing
`video/enterprise.mp4` on the server** — just leave that one file in
place and overwrite everything else from this zip. If you do a full
clean wipe of the htdocs folder, copy `video/enterprise.mp4` back in
from your current GitHub repo or server backup before going live.

## Deploy steps (matches your CloudPanel + GitHub setup)

Your site root on the server is:
```
/home/ackfirstweb/htdocs/acknowledgerfirst.com/
```

1. **Clear out the old broken files**, but keep the video so you don't
   have to re-upload 55MB:
   ```bash
   cd /home/ackfirstweb/htdocs/acknowledgerfirst.com/
   mkdir -p /tmp/video-backup && cp video/enterprise.mp4 /tmp/video-backup/ 2>/dev/null
   rm -rf ./* ./.htaccess
   mkdir -p video && cp /tmp/video-backup/enterprise.mp4 video/ 2>/dev/null
   ```
   (if that cp fails because `video/enterprise.mp4` wasn't there to begin
   with, just remember to add it back in step 2 below)
2. **Push this corrected `out/` folder's contents** to your
   `sharathwamshi/acknowledgerFirst_website` GitHub repo (replacing
   whatever is there now — including removing any `/AFPL`-prefixed
   files that may currently be committed, and keeping
   `video/enterprise.mp4` in the repo since it's unchanged), then on
   the server:
   ```bash
   cd /home/ackfirstweb/htdocs/acknowledgerfirst.com/
   git clone https://github.com/sharathwamshi/acknowledgerFirst_website.git .
   ```
   (or `git pull` if you keep the clone elsewhere and copy/symlink the
   contents of `out/` into the htdocs folder — just make sure
   `index.html`, `_next/`, `brand/`, `team/`, `video/` end up **directly**
   inside `acknowledgerfirst.com/`, not nested one level deeper.)
3. Visit `https://acknowledgerfirst.com/` and hard-refresh
   (Ctrl/Cmd+Shift+R) to bypass any cached 404s from the browser.

## Note on `.htaccess`

nginx (what CloudPanel uses) ignores `.htaccess` entirely — it's included
here only for portability in case you ever move to an Apache host. On
your actual CloudPanel/nginx setup:
- gzip compression, correct MIME types, and HTTP range requests (needed
  for video scrubbing) are already handled by nginx's defaults for static
  sites, so no extra config is required.
- If you want explicit long-lived caching for `_next/`, `brand/`,
  `team/`, and `video/` plus no-cache on `index.html`, add this to the
  site's nginx vhost (`/etc/nginx/sites-enabled/acknowledgerfirst.com.conf`)
  inside the existing `server { ... }` block, then
  `clpctl site:reload:nginx` (or `systemctl reload nginx`):

  ```nginx
  location ~* \.(?:js|css|png|jpg|jpeg|mp4)$ {
      expires 1y;
      add_header Cache-Control "public, immutable";
      add_header Accept-Ranges bytes;
  }
  location = /index.html {
      add_header Cache-Control "no-cache";
  }
  ```

## What's different from the dev-mode version

- The `/api/chat` server route isn't included — static hosting can't run
  server-side code. This costs nothing today: the chatbot already runs
  entirely client-side (`lib/chatResponder.ts`), answering from the
  site's own data with no server needed.
- If you ever want the live-Claude-API chatbot, that specifically needs a
  Node.js-capable host, not this static export.
