---
layout: post
title:  "Server-Driven Ad Decision Pattern in SGAI"
date:   2026-11-14 14:20:00 -0800
categories: Streaming
comments: true
---

## Overview

In Server-Guided Ad Insertion (SGAI) with HLS Interstitials (`#EXT-X-DATERANGE`), an ad break involves three parties:

1. **The HLS Manifest** — describes the ad opportunity (slot) in neutral terms.
2. **The Player** — advertises what it is capable of rendering.
3. **The Ad Server** — combines slot context and player capabilities to decide which creative and metadata to return.

Rather than encoding format-specific instructions in the playlist (such as `type=squeezeback`) or putting negotiation logic in the player, the [SGAI demo](https://github.com/straightdave/sgai) uses a **server-driven decision pattern** with strict separation of concerns.

---

## The Problem with Manifest-Driven or Player-Driven Negotiation

### 1. Hardcoding Formats in the Manifest Breaks Unsupported Players
If a manifest hardcodes `X-ASSET-LIST="/api/ads?type=squeezeback"` and the server unconditionally returns an L-shape squeezeback video (`squeezeback_l_10s.mp4`), any player that does not understand L-shape squeezebacks (for example, Safari's native HLS Interstitials or a basic `hls.js` player) will play the squeezeback video fullscreen. The viewer sees an empty black 75% region where the primary video was supposed to sit.

### 2. Conditional Negotiation on the Player Adds Complexity
If the manifest uses custom attributes (e.g. `X-AD-TYPE="continue"`) and the player has to inspect each break, decide which capabilities to send, and map attributes into query parameters, the client accumulates branching logic that must be duplicated across every player implementation (web, iOS, Android, TV).

---

## The Decision Pattern

The architecture splits responsibilities cleanly across the three layers:

```text
+-------------------+         +-----------------------+         +------------------+
|   HLS Manifest    |         |        Player         |         |    Ad Server     |
| (Slot Context)    |         | (Fixed Capabilities)  |         | (Decision Logic) |
+-------------------+         +-----------------------+         +------------------+
         |                               |                               |
         | X-ASSET-LIST with             |                               |
         | slot, duration, playback      |                               |
         |------------------------------>|                               |
         |                               | GET X-ASSET-LIST              |
         |                               |   + clickable=1&squeezeback=1 |
         |                               |------------------------------>|
         |                               |                               | Match slot + caps
         |                               |  ASSETS[0]: URI, LAYOUT,      |
         |                               |             BUTTON            |
         |                               |<------------------------------|
         |                               | Render LAYOUT + BUTTON        |
         |                               |                               |
```

### 1. Neutral Manifest (Slot Metadata Only)
Each `#EXT-X-DATERANGE` tag in the HLS playlists carries only neutral, format-agnostic slot descriptors inside `X-ASSET-LIST`:

```m3u8
#EXT-X-DATERANGE:ID="ad-break-10s",CLASS="com.apple.hls.interstitial",START-DATE="2026-01-01T00:00:10.000Z",DURATION=5.0,X-ASSET-LIST="/api/ads?slot=ad-break-10s&duration=5.0&playback=block",X-RESUME-OFFSET=0
#EXT-X-DATERANGE:ID="ad-break-40s",CLASS="com.apple.hls.interstitial",START-DATE="2026-01-01T00:00:40.000Z",DURATION=10.0,X-ASSET-LIST="/api/ads?slot=ad-break-40s&duration=10.0&playback=continue"
```

- `slot`: Identifies the break (`ad-break-10s`, `ad-break-30s`, `ad-break-40s`).
- `duration`: Slot duration in seconds (`5.0`, `10.0`).
- `playback`: Whether primary content should pause (`block`) or may keep playing (`continue`) during the break.

Because this metadata lives in `X-ASSET-LIST`, even off-the-shelf players that ignore custom `X-*` attributes automatically forward the full slot context to `/api/ads`.

### 2. Stateless, Capability-Declaring Player
The player does not inspect break types or make format decisions before fetching. When a break is reached, it takes `X-ASSET-LIST` as-is and unconditionally appends its own fixed capability flags:

- `clickable=1` — the player can render an interactive CTA button overlay.
- `squeezeback=1` — the player can shrink the primary video into a 75% top-right viewport and play an L-shape ad behind it.

When the server responds, the player simply obeys the returned `LAYOUT` and `BUTTON` fields:
- `LAYOUT === "l-squeezeback"` → enter `squeezeback` mode and keep the primary video playing.
- Otherwise → enter `fullscreen` mode and pause the primary video until the ad ends.
- If `BUTTON` is present → display the CTA button (`LABEL`) and execute its `ACTION` (`"link"` opens `URL` in a new tab; `"request"` sends a `POST` to `URL`, e.g. `/api/ads/click`).

### 3. Single Decision Point on the Server
In [`internal/server/server.go`](https://github.com/straightdave/sgai/blob/master/internal/server/server.go), `handleAds` combines the slot context (`playback`) with the player's declared capabilities (`squeezeback`, `clickable`):

| Slot `playback` | Player `squeezeback` | Player `clickable` | Returned Asset | Returned `LAYOUT` | Returned `BUTTON` |
|---|---|---|---|---|---|
| `block` (or omitted) | any | absent | `/priv/ads/countdown_5s.mp4` | *(omitted)* | *(omitted)* |
| `block` (or omitted) | any | `1` / `true` | `/priv/ads/countdown_5s.mp4` | *(omitted)* | `Learn More` (`link`) |
| `continue` | absent | any | `/priv/ads/countdown_5s.mp4` | *(omitted)* | *(per `clickable`)* |
| `continue` | `1` / `true` | `1` / `true` | `/priv/ads/squeezeback_l_10s.mp4` | `l-squeezeback` | `Claim Offer` (`request`) |

---

## Why This Pattern Works Well

1. **Safe graceful degradation:** A player that does not send `squeezeback=1` or `clickable=1` automatically receives a standard fullscreen asset (`countdown_5s.mp4`) with no broken layout or unused metadata.
2. **Minimal client logic:** Adding a new ad format (such as picture-in-picture or a new interactive overlay) requires updating the server's decision table and adding support to capable players—no manifest changes or per-break client branching.
3. **Format-agnostic playlists:** Manifests describe *constraints and opportunities* (`duration`, `playback`), not UI formats, so the same playlist can serve web, mobile, and connected-TV players simultaneously.
