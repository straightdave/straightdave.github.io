---
layout: post
title:  "Linear (Live) Streaming Interstitials in SGAI"
date:   2026-11-14 14:23:00 -0800
categories: Streaming
comments: true
---

## Overview

While VOD interstitials pause a finite asset, play an ad, and resume from the exact paused frame, **linear (live) streaming** has a continuously advancing timeline. Bringing HLS Interstitials (`#EXT-X-DATERANGE`) to a linear stream required a different set of manifest, timing, playback, and audio decisions.

This post captures the design choices made for the linear demo in [SGAI](https://github.com/straightdave/sgai) ([`internal/server/live.go`](https://github.com/straightdave/sgai/blob/master/internal/server/live.go) and [`web/src/InterstitialPlayer.jsx`](https://github.com/straightdave/sgai/blob/master/web/src/InterstitialPlayer.jsx)).

---

## 1. Simulating an Endless Linear Stream from VOD

Because public sample live streams are unstable or lack custom `#EXT-X-DATERANGE` markers, the backend synthesizes an endless live HLS stream on the fly by looping the local Big Buck Bunny VOD playlists (`priv/big_buck_bunny/`).

- **Sliding-window live playlists:** Each variant playlist (`/api/live/{variant}/playlist.m3u8`) omits `#EXT-X-PLAYLIST-TYPE:VOD` and `#EXT-X-ENDLIST`, and serves a 6-segment sliding window computed from elapsed wall-clock time since stream start (with a 30-second initial lead so playback can start immediately).
- **Loop boundaries:** When segment sequence numbers wrap from the end of Big Buck Bunny back to segment `0`, the generator emits `#EXT-X-DISCONTINUITY` and increments `#EXT-X-DISCONTINUITY-SEQUENCE` as the boundary slides out of the window.
- **No caching:** Live master and variant playlists are served with `Cache-Control: no-store` so players always receive the latest window and `#EXT-X-DATERANGE` cues.

---

## 2. Aligning `DATERANGE` Breaks with a Sliding Window

In VOD mode, `LEVEL_LOADED` fires once and the first fragment's `programDateTime` corresponds to `video.currentTime = 0`:

```js
time = (dr.startDate.getTime() - basePdt) / 1000
```

In a **live sliding-window** stream, `LEVEL_LOADED` fires periodically as old segments drop off the front of the playlist. `details.fragments[0].programDateTime` advances with the window, while `video.currentTime` continues counting up from when the viewer joined.

To keep break timing accurate in both VOD and live modes, the player anchors `DATERANGE` timestamps using `frag0.start`:

```js
const baseTime = frag0.start || 0
const time = baseTime + (dr.startDate.getTime() - basePdt) / 1000
```

- In **VOD**, `frag0.start` is `0`, preserving the original behavior.
- In **Live**, `frag0.start` tracks the media presentation offset of the first segment in the current window, so `time` always matches `primaryVideo.currentTime`.

---

## 3. Why Linear Fullscreen Ads Omit `X-RESUME-OFFSET=0`

In VOD, fullscreen breaks include `X-RESUME-OFFSET=0` so the player pauses at `START-DATE` and resumes at the exact same frame when the ad finishes.

In a **linear live stream**, `X-RESUME-OFFSET=0` is the wrong semantic:
- Every 5-second fullscreen ad would push the viewer 5 seconds further behind the live edge.
- Over multiple ad breaks, latency accumulates until the viewer falls behind the sliding playlist window and playback stalls.

Per the HLS Interstitials specification, **omitting `X-RESUME-OFFSET`** on live breaks tells the player to resume at `START-DATE + playedDuration` — i.e., **at the live edge**, treating fullscreen ads as broadcast replacements rather than pauses.

---

## 4. Keep Playing (Muted) Instead of Pausing During Live Fullscreen Ads

To match live-edge semantics without seeking or rebuffering when a fullscreen ad ends:

- **VOD (`isLive = false`):** `primaryVideo.pause()` is called when a fullscreen ad starts, and `primaryVideo.play()` is called when it finishes.
- **Linear Live (`isLive = true`):** `primaryVideo` **keeps playing** underneath the fullscreen ad overlay (`z-index: 3`) with its audio muted. When the ad ends, the overlay disappears and primary audio is restored immediately at the live edge with zero seek delay or rebuffer.

---

## 5. Audio-Aware Primary Muting

Both fullscreen ads and L-squeezeback ads play alongside (or over) the primary video, which raises the question of when to mute primary content:

- **Fullscreen ads:** Always mute `primaryVideo` immediately, since the primary video is hidden behind the ad.
- **L-squeezeback ads:** Primary video remains visible in the upper-right 75% frame, while the ad plays in the L-shaped frame around it. If the ad itself is silent, muting the primary stream degrades the viewing experience unnecessarily; if the ad has audio, both tracks would clash.

To handle this cleanly, the player inspects the ad element (`loadeddata`, `playing`, `timeupdate`) using browser and `hls.js` signals (`hlsInstance.audioTracks`, `mozHasAudio`, `webkitAudioDecodedByteCount`, `video.audioTracks`) and mutes `primaryVideo` **only when the ad actually contains audio**, restoring the viewer's prior mute state (`wasMutedRef`) when the ad finishes.
