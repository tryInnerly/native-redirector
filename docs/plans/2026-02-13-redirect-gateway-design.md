# Redirect Gateway Page — Design

## Overview

A minimal loader page that acts as a redirect gateway for ad traffic. Detects in-app WebViews and forces the user into a native browser before redirecting to the target URL.

## File Location

`redirect/index.html` in this repo. Deployed by copying to the React project's `/public/redirect/index.html`, accessible at `https://yourdomain.com/redirect?f=iqtest16`.

## Architecture

Single self-contained HTML file (Approach A). Duplicates WebView detection and redirect logic from `index.html` — no shared modules. Each page is independently deployable.

## UI

### Initial State (Loader)

- Centered vertically and horizontally, mobile-first
- CSS spinner ring (border-based, color `#029a9d`, ~48px)
- Text below spinner from mapping config (e.g. "Loading your IQ test")
- Font: system stack (`-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`)
- No external fonts, no other elements — ultra-clean

### Theme Support

`prefers-color-scheme` media query:

| Theme | Background | Text |
|-------|-----------|------|
| Light | `#f5f5f5` | `#1a1a1a` |
| Dark | `#111` | `#e5e5e5` |

Spinner stays `#029a9d` in both themes.

### Fallback State (after 1.5s)

- Spinner stops/hides
- Text changes to "Open this page in your browser"
- Primary button: "Open in browser" (bg `#029a9d`, white text)
- Secondary button: "Copy link"
- No pointer arrows, no step-by-step instructions

## Logic

### On Page Load

1. Parse `?f=` from URL
2. Look up mapping config. No match → use `_default` entry
3. Set loader text from mapping
4. Detect WebView (same UA patterns as `index.html`)
5. Native browser → immediate redirect to target URL
6. In-app WebView → auto-redirect attempt:
   - Android: Intent URI via `window.location.replace()`
   - iOS: `x-safari-https://` via `window.location.replace()`
7. After 1.5s timeout → show fallback UI

### Query Parameter Forwarding

All query params except `f` are forwarded to the target URL:

```
?f=iqtest16&utm_source=ig&utm_campaign=test
→ https://quiz.tryinnerly.com/iqtest16?utm_source=ig&utm_campaign=test
```

### Mapping Config

```js
var FUNNELS = {
    iqtest16: {
        text: 'Loading your IQ test',
        url: 'https://quiz.tryinnerly.com/iqtest16'
    },
    _default: {
        text: 'Loading your IQ test',
        url: 'https://join.tryinnerly.com/iqtest13/'
    }
};
```

### Copy Link Behavior

Copies the target URL (with forwarded params), not the redirect page URL.

## WebView Detection

Same patterns as `index.html`:

- Instagram, Facebook, TikTok, Twitter, Snapchat, Pinterest, LinkedIn, Telegram, Line, WeChat
- Generic WebView detection for unrecognized apps
- Android vs iOS platform detection

## Redirect Mechanisms

Same as `index.html`:

- **Android:** Intent URI (no package constraint — opens default browser)
- **iOS:** `x-safari-https://` scheme
- **Fallback:** Manual button + copy link (clipboard API → execCommand → prompt)
