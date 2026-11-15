# ASTROTAG DUAL-PLATFORM DEPLOYMENT GUIDE (VERCEL + CLOUDFLARE PAGES + RENDER)

**Document Version:** 1.0.0  
**Effective Date:** November 15, 2026  
**Audience:** DevOps Engineers, Cloud Administrators, Developers  

---

## 1. Architectural Overview

Astrotag is engineered to run seamlessly and interchangeably on two production hosting stacks without code modifications:

```mermaid
flowchart TD
    User["Visitor Browser (astrotag.in / www.astrotag.in)"]
    
    subgraph Track_A ["Stack 1: Vercel Native (Current Default)"]
        V_Edge["Vercel Edge Network (vercel.json)"]
        V_Func["Vercel Serverless Function (FastAPI main.py)"]
        V_Edge -->|Static Files / HTML| User
        User -->|API_BASE = '' (Same-Origin /api/*)| V_Func
    end
    
    subgraph Track_B ["Stack 2: Cloudflare Pages + Render"]
        CF_Edge["Cloudflare Pages (_headers)"]
        Render_API["Render Web Service (astrotag-api.onrender.com)"]
        CF_Edge -->|Static Files / HTML| User
        User -->|API_BASE = 'https://astrotag-api.onrender.com'| Render_API
    end
```

---

## 2. Frontend Origin-Aware Dynamic Router

In `public/index.html` (and root `index.html`), the client automatically detects its hosting host:

```javascript
const RENDER_BACKEND_URL = "https://astrotag-api.onrender.com";
const API_BASE = window.location.hostname.endsWith("pages.dev")
  ? RENDER_BACKEND_URL
  : "";
```

- **When hosted on Vercel** (`*.vercel.app` or `astrotag.in` pointing to Vercel):
  `API_BASE` resolves to `""` (relative same-origin `/api/*` invoking Vercel's native FastAPI serverless routes).
- **When hosted on Cloudflare Pages** (`*.pages.dev` or preview builds):
  `API_BASE` automatically routes `/api/*` calls cross-origin to `https://astrotag-api.onrender.com`.
- **Switching `astrotag.in` DNS to Cloudflare Pages**:
  Update the condition to:
  ```javascript
  const API_BASE = (window.location.hostname.endsWith("pages.dev") || window.location.hostname.endsWith("astrotag.in"))
    ? RENDER_BACKEND_URL
    : "";
  ```

---

## 3. Backend Hardened CORS Security Layer

In `main.py`, CORS is locked down against unauthorized cross-origin requests while fully permitting your legitimate platforms:

```python
ALLOWED_ORIGINS = [
    "https://astrotag.in",
    "https://www.astrotag.in",
    "https://astrotag.pages.dev",
    "https://astrotag.vercel.app",
    "http://127.0.0.1:8000",
    "http://localhost:8000",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,
    allow_origin_regex=r"^https://([a-z0-9-]+\.)?astrotag\.(in|pages\.dev|vercel\.app)$",
    allow_credentials=False,
    allow_methods=["GET", "POST", "OPTIONS"],
    allow_headers=["Content-Type"],
    max_age=3600,
)
```

The regex `^https://([a-z0-9-]+\.)?astrotag\.(in|pages\.dev|vercel\.app)$` automatically allows all subdomains and branch previews (e.g., `git-feature.astrotag.pages.dev` and `astrotag-*.vercel.app`).

---

## 4. Edge Security Headers & Content Security Policy (CSP)

Both platforms enforce identical, modern enterprise security headers:

### A. Cloudflare Pages (`public/_headers` and `_headers`)
```
/*
  Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()
  Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' https://pagead2.googlesyndication.com https://partner.googleadservices.com https://tpc.googlesyndication.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https://astrotag-api.onrender.com https://countriesnow.space https://pagead2.googlesyndication.com; frame-src https://googleads.g.doubleclick.net https://tpc.googlesyndication.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'
```

### B. Vercel (`vercel.json`)
```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "Strict-Transport-Security", "value": "max-age=63072000; includeSubDomains; preload" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=(), payment=()" },
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-inline' https://pagead2.googlesyndication.com https://partner.googleadservices.com https://tpc.googlesyndication.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https://astrotag-api.onrender.com https://countriesnow.space https://pagead2.googlesyndication.com; frame-src https://googleads.g.doubleclick.net https://tpc.googlesyndication.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'"
        }
      ]
    }
  ]
}
```

---

## 5. GoDaddy DNS Configuration for `astrotag.in`

### Route Option 1: Pointing to Vercel (Current Stack)
Log in to GoDaddy Domain Portfolio &rarr; `astrotag.in` &rarr; **Manage DNS**:

| Type | Name | Value | TTL |
| :--- | :--- | :--- | :--- |
| **A** | `@` | `76.76.21.21` | 1 Hour / Automatic |
| **CNAME** | `www` | `cname.vercel-dns.com` | 1 Hour / Automatic |

### Route Option 2: Pointing to Cloudflare Pages (Failover / Alternative Stack)
In Cloudflare Dashboard &rarr; **Pages** &rarr; `astrotag` &rarr; **Custom Domains**:
Add `astrotag.in` and `www.astrotag.in`. Cloudflare will provide CNAME targets:

| Type | Name | Value | TTL |
| :--- | :--- | :--- | :--- |
| **CNAME** | `@` | `astrotag.pages.dev` (or Cloudflare Nameservers) | 1 Hour / Automatic |
| **CNAME** | `www` | `astrotag.pages.dev` | 1 Hour / Automatic |

---

## 6. Render Backend Setup (`astrotag-api.onrender.com`)

If deploying the backend on Render for Cloudflare Pages:
1. Log in to [render.com](https://render.com) and click **New** &rarr; **Web Service**.
2. Connect your GitHub repository (`pkjghs-bot/Astrotag`).
3. Set the service settings:
   - **Environment:** `Python 3`
   - **Build Command:** `pip install -r requirements.txt`
   - **Start Command:** `uvicorn main:app --host 0.0.0.0 --port $PORT`
4. In **Environment Variables**:
   - `GEMINI_API_KEY`: `AIzaSy...`
   - `ALLOWED_ORIGINS`: `https://astrotag.in,https://www.astrotag.in,https://astrotag.pages.dev`
5. Click **Create Web Service**. Your API will be live at `https://astrotag-api.onrender.com`.
