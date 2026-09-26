# Planned upstream improvements

Working notes for fixes to send to upstream (hababr/Imagus-Reborn).
This file is local tracking only; leave it out of upstream PRs.

## 1. Re-hover of extensionless video shows red load error — FIXED (9e579c0)

- **Symptom:** Reddit GIF-as-mp4 posts (`preview.redd.it/….gif?format=mp4`)
  play on first hover, show a red spinner on the second.
- **Cause:** after showing a video, `IMGS_c` cached `PLAYER.src()`, which has
  lost the `#mp4` type hint that `set()` strips. On re-hover `isVideoUrl()`
  doesn't recognise the URL, so it's loaded into `<img>` and fails.
- **Fix:** `set()` keeps the hinted URL in `PVI.videoSrc`; `assign_src()`
  caches it when it matches the player's source (`src/content/content.js`).
- **Status:** committed, needs browser testing, then its own upstream PR.

## 2. Hovering a link shows a previously seen video — INVESTIGATING

- **Symptom:** persistent wrong video for a given link until page reload,
  maybe more on Firefox.
- **Suspected cause:** async resolve race. Video rules' `res` functions
  (YouTube, Reddit, Twitch, …) write to `this.TRG` (current hover) instead of
  the element the request was made for; `onMessage` reads
  `PVI.TRG.IMGS_ext_data` (content.js ~3630); `[Extension]` rule keys the
  album cache `PVI.stack` off `this.TRG`. A wrong entry then sticks, made
  worse by upstream `4606ec9` no longer clearing `PVI.stack` on SPA
  navigation, and `resetNode` not clearing `IMGS_ext_data`.
- **Next:** reproduce (hover A, move to B before A resolves), then fix by
  pinning `PVI.TRG = trg` while `rule.res` runs.

## 3. Seeking into unbuffered part of a video closes the popup — INVESTIGATING

- **Symptom:** clicking the timeline past the buffered range closes the popup.
- **Suspected cause:** the seek triggers a new HTTP range request which fails
  (server without range support, or an expired or throttled signed URL), so
  Video.js fires `error` → `content_onerror` → `show("R_load")`, which hides
  `PVI.DIV`. The error handler is meant for initial load, not mid-playback.
- **Next:** confirm with the `Load error > …` console line, then keep the popup
  open for errors after playback started (and maybe retry the source at the
  target time).
