# Redirect Gateway Page — Implementation Plan

Project: docs\plans\2026-02-13-redirect-gateway-design.md

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a minimal loader page (`redirect/index.html`) that redirects ad traffic from in-app WebViews to native browsers.

**Architecture:** Single self-contained HTML file. Copies WebView detection and redirect logic from `index.html`, strips the multi-view UI down to a spinner + fallback button. Uses `?f=` query param to select funnel config (loader text + target URL).

**Tech Stack:** Vanilla HTML/CSS/JS, no dependencies, no build step.

**Reference:** Design doc at `docs/plans/2026-02-13-redirect-gateway-design.md`, existing logic at `index.html:609-887`.

---

### Task 1: Create redirect/index.html — HTML structure + CSS [DONE]

**Files:**
- Create: `redirect/index.html`

**Step 1: Create directory**

```bash
mkdir redirect
```

**Step 2: Write the full HTML + CSS**

Create `redirect/index.html` with:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
    <title>Loading...</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }

        :root {
            --bg: #f5f5f5;
            --text: #1a1a1a;
            --text-secondary: #666;
            --accent: #029a9d;
            --btn-text: #fff;
            --surface: rgba(0,0,0,0.05);
            --border: rgba(0,0,0,0.08);
        }

        @media (prefers-color-scheme: dark) {
            :root {
                --bg: #111;
                --text: #e5e5e5;
                --text-secondary: #999;
                --surface: rgba(255,255,255,0.06);
                --border: rgba(255,255,255,0.1);
            }
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            min-height: 100vh;
            min-height: 100dvh;
            display: flex;
            align-items: center;
            justify-content: center;
            background: var(--bg);
            color: var(--text);
            padding: 20px;
        }

        .container {
            text-align: center;
            max-width: 360px;
            width: 100%;
        }

        /* ─── Spinner ─── */
        .spinner {
            width: 48px;
            height: 48px;
            border: 4px solid var(--surface);
            border-top-color: var(--accent);
            border-radius: 50%;
            animation: spin 0.8s linear infinite;
            margin: 0 auto 24px;
        }

        @keyframes spin {
            to { transform: rotate(360deg); }
        }

        .loader-text {
            font-size: 16px;
            color: var(--text-secondary);
            line-height: 1.5;
        }

        /* ─── Fallback UI ─── */
        .fallback {
            display: none;
        }

        .fallback.visible {
            display: block;
            animation: fadeUp 0.3s ease-out;
        }

        @keyframes fadeUp {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }

        .fallback-text {
            font-size: 16px;
            color: var(--text);
            margin-bottom: 20px;
            line-height: 1.5;
        }

        .actions {
            display: flex;
            flex-direction: column;
            gap: 10px;
        }

        .btn {
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 8px;
            padding: 14px 24px;
            border-radius: 10px;
            font-family: inherit;
            font-size: 15px;
            font-weight: 600;
            cursor: pointer;
            border: none;
            text-decoration: none;
            -webkit-tap-highlight-color: transparent;
        }

        .btn:active {
            transform: scale(0.97);
        }

        .btn-primary {
            background: var(--accent);
            color: var(--btn-text);
        }

        .btn-secondary {
            background: var(--surface);
            border: 1px solid var(--border);
            color: var(--text);
        }

        /* ─── Toast ─── */
        .toast {
            position: fixed;
            bottom: 40px;
            left: 50%;
            transform: translateX(-50%) translateY(20px);
            background: var(--accent);
            color: var(--btn-text);
            padding: 10px 20px;
            border-radius: 10px;
            font-size: 14px;
            font-weight: 600;
            font-family: inherit;
            opacity: 0;
            pointer-events: none;
            transition: opacity 0.2s, transform 0.2s;
            z-index: 100;
        }

        .toast.show {
            opacity: 1;
            transform: translateX(-50%) translateY(0);
        }
    </style>
</head>
<body>
    <div class="container">
        <!-- Loader state -->
        <div id="loader">
            <div class="spinner"></div>
            <p class="loader-text" id="loader-text">Loading...</p>
        </div>

        <!-- Fallback state (hidden by default) -->
        <div class="fallback" id="fallback">
            <p class="fallback-text">Open this page in your browser</p>
            <div class="actions">
                <a class="btn btn-primary" id="btn-open" href="#">Open in browser</a>
                <button class="btn btn-secondary" id="btn-copy">Copy link</button>
            </div>
        </div>
    </div>

    <!-- Toast -->
    <div class="toast" id="toast">Link copied</div>

    <!-- No-JS fallback -->
    <noscript>
        <style>.container { display: none !important; }</style>
        <p style="text-align:center;padding:40px 20px;font-family:sans-serif;">
            Please open this link in Safari or Chrome.
        </p>
    </noscript>

    <script>
    // JS will be added in Task 2
    </script>
</body>
</html>
```

**Step 3: Verify file opens in browser**

Open `redirect/index.html` in a browser. Confirm:
- Spinner animates in `#029a9d`
- "Loading..." text displays centered below spinner
- Light theme on light system preference, dark on dark
- Fallback section is hidden

**Step 4: Commit**

```bash
git add redirect/index.html
git commit -m "feat: add redirect gateway page — HTML structure and CSS"
```

---

### Task 2: Add JavaScript — config, URL parsing, parameter forwarding [DONE]

**Files:**
- Modify: `redirect/index.html` (replace the `<script>` block)

**Step 1: Replace the script placeholder**

Replace `// JS will be added in Task 2` with the funnel config and URL parsing logic:

```js
(function () {
    'use strict';

    // ─── Funnel Config ───
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

    // ─── Parse URL params ───
    var params = new URLSearchParams(window.location.search);
    var funnelKey = params.get('f') || '_default';
    var funnel = FUNNELS[funnelKey] || FUNNELS._default;

    // ─── Build target URL with forwarded params ───
    params.delete('f');
    var targetUrl = funnel.url;
    var remaining = params.toString();
    if (remaining) {
        targetUrl += (targetUrl.indexOf('?') === -1 ? '?' : '&') + remaining;
    }

    // ─── Set loader text ───
    var $ = function (id) { return document.getElementById(id); };
    $('loader-text').textContent = funnel.text;

    // ─── Detection + redirect logic will go here (Task 3) ───
})();
```

**Step 2: Verify in browser**

Open `redirect/index.html?f=iqtest16`. Confirm loader text shows "Loading your IQ test".
Open `redirect/index.html` (no param). Confirm same default text.

**Step 3: Commit**

```bash
git add redirect/index.html
git commit -m "feat: add funnel config and URL parameter parsing"
```

---

### Task 3: Add JavaScript — WebView detection + redirect logic [DONE]

**Files:**
- Modify: `redirect/index.html` (extend the `<script>` block)

**Step 1: Add detection and redirect logic**

Replace `// ─── Detection + redirect logic will go here (Task 3) ───` with the following. This is adapted from `index.html:636-887`:

```js
    var ua = navigator.userAgent || navigator.vendor || '';
    var loaderEl = $('loader');
    var fallbackEl = $('fallback');
    var toast = $('toast');

    // ─── WebView Detection ───
    // Reference: index.html detection patterns
    var apps = {
        instagram: /Instagram/i,
        facebook:  /FBAN|FBAV|FB_IAB|FB4A/i,
        tiktok:    /TikTok|BytedanceWebview|musical_ly|trill/i,
        twitter:   /Twitter/i,
        snapchat:  /Snapchat/i,
        pinterest: /Pinterest/i,
        linkedin:  /LinkedInApp/i,
        telegram:  /Telegram/i,
        line:      /\bLine\//i,
        wechat:    /MicroMessenger/i
    };

    var detectedApp = null;
    for (var key in apps) {
        if (apps[key].test(ua)) {
            detectedApp = key;
            break;
        }
    }

    var isGenericWebview = false;
    if (!detectedApp) {
        if (/wv\)/.test(ua) && /Android/.test(ua)) isGenericWebview = true;
        if (/Android.*Version\/[0-9]/.test(ua)) isGenericWebview = true;
        if (/iPhone|iPad/.test(ua) && !/Safari/.test(ua)) isGenericWebview = true;
        if (/\bWebView\b/i.test(ua)) isGenericWebview = true;
    }

    var isInApp = !!detectedApp || isGenericWebview;
    var isIOS = /iPhone|iPad|iPod/.test(ua);
    var isAndroid = /Android/.test(ua);

    // ─── Native browser → immediate redirect ───
    if (!isInApp) {
        window.location.replace(targetUrl);
        return;
    }

    // ─── In-app → attempt auto-redirect ───
    if (isAndroid) {
        // Intent URI (no package — opens default browser)
        var strippedUrl = targetUrl.replace(/^https?:\/\//, '');
        var intentUrl = 'intent://' + strippedUrl +
            '#Intent;scheme=https;action=android.intent.action.VIEW;' +
            'S.browser_fallback_url=' + encodeURIComponent(targetUrl) + ';end;';
        window.location.replace(intentUrl);

        // Button handler for manual retry
        $('btn-open').addEventListener('click', function (e) {
            e.preventDefault();
            window.location.replace(intentUrl);
        });
    } else if (isIOS) {
        // x-safari-https:// scheme
        var safariUrl = 'x-safari-https://' + targetUrl.replace(/^https?:\/\//, '');
        window.location.replace(safariUrl);

        // Button handler — retry same scheme
        $('btn-open').addEventListener('click', function (e) {
            e.preventDefault();
            window.location.replace(safariUrl);
        });
    } else {
        // Unknown platform — button opens target directly
        $('btn-open').addEventListener('click', function (e) {
            e.preventDefault();
            window.location.href = targetUrl;
        });
    }

    // ─── Fallback: show buttons after 1.5s ───
    setTimeout(function () {
        loaderEl.style.display = 'none';
        fallbackEl.className = 'fallback visible';
    }, 1500);

    // ─── Copy to clipboard ───
    $('btn-copy').addEventListener('click', function () {
        if (navigator.clipboard && window.isSecureContext) {
            navigator.clipboard.writeText(targetUrl).then(function () {
                showToast();
            }, function () {
                fallbackCopy();
            });
            return;
        }
        fallbackCopy();
    });

    function fallbackCopy() {
        try {
            var ta = document.createElement('textarea');
            ta.value = targetUrl;
            ta.style.cssText = 'position:fixed;left:-9999px;top:-9999px;opacity:0';
            document.body.appendChild(ta);
            ta.focus();
            ta.select();
            document.execCommand('copy');
            document.body.removeChild(ta);
            showToast();
        } catch (e) {
            window.prompt('Copy this link:', targetUrl);
        }
    }

    function showToast() {
        toast.className = 'toast show';
        setTimeout(function () {
            toast.className = 'toast';
        }, 2000);
    }
```

**Step 2: Verify in desktop browser**

Open `redirect/index.html?f=iqtest16` in a desktop browser (not a WebView).
Expected: immediate redirect to `https://quiz.tryinnerly.com/iqtest16`.

Open `redirect/index.html?f=iqtest16&utm_source=ig&utm_campaign=test` in desktop browser.
Expected: redirect to `https://quiz.tryinnerly.com/iqtest16?utm_source=ig&utm_campaign=test`.

Open `redirect/index.html` (no params) in desktop browser.
Expected: redirect to `https://join.tryinnerly.com/iqtest13/`.

**Step 3: Commit**

```bash
git add redirect/index.html
git commit -m "feat: add WebView detection, redirect logic, and fallback UI"
```

---

### Task 4: Manual testing checklist

No code changes. Verify the completed page works correctly.

**Step 1: Desktop browser tests**

- [ ] `?f=iqtest16` → redirects to `https://quiz.tryinnerly.com/iqtest16`
- [ ] `?f=iqtest16&utm_source=ig` → redirects with UTM preserved
- [ ] No `?f=` param → redirects to default URL
- [ ] `?f=unknown_value` → redirects to default URL

**Step 2: Theme tests**

- [ ] Light system theme → light gray background, dark text
- [ ] Dark system theme → dark background, light text
- [ ] Spinner is `#029a9d` in both themes

**Step 3: Fallback UI tests (simulate WebView)**

In Chrome DevTools, override User-Agent to `Instagram` and reload `redirect/index.html?f=iqtest16`:
- [ ] Spinner shows for ~1.5s with "Loading your IQ test"
- [ ] Spinner hides, fallback UI appears with "Open in browser" + "Copy link"
- [ ] "Copy link" copies `https://quiz.tryinnerly.com/iqtest16` (not the redirect page URL)
- [ ] Toast "Link copied" appears and fades

**Step 4: Mobile tests (if device available)**

- [ ] Open in Instagram Android WebView → Intent URI fires or fallback shows
- [ ] Open in Instagram iOS WebView → x-safari-https:// fires or fallback shows
- [ ] Open in native mobile browser → immediate redirect
