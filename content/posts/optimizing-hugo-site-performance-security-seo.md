+++
title = "Optimizing My Hugo Site: Performance, Security, and SEO"
date = "2026-06-23T00:00:00+01:00"
author = "Igor Mihajlov"
cover = ""
tags = ["hugo", "nginx", "cloudflare", "performance", "security", "seo", "devops"]
keywords = ["hugo performance", "nginx security headers", "content security policy", "web vitals", "seo optimization", "gzip nginx", "hugo configuration"]
description = "After setting up the initial deployment pipeline, I ran the site through several audit tools and found a number of issues worth fixing — redirect errors, render-blocking resources, missing security headers, and SEO gaps. Here's what I changed."
showFullContent = false
readingTime = true
hideComments = false
+++

After setting up the [initial deployment pipeline](/posts/deploying-hugo-site-github-actions-docker-swarm/), I ran the site through PageSpeed Insights, Google Search Console, and a few security scanners. The results surfaced a number of issues worth fixing. This post covers everything I changed — from redirect errors and performance to security headers and SEO.

## Fixing Redirect Errors

Google Search Console flagged a redirect error preventing pages from being indexed. The cause was a two-hop redirect chain triggered by URLs without trailing slashes:

1. Googlebot requests `https://mysite.com/about`
2. The Hugo container's nginx issues a `301` to `http://mysite.com/about/` — absolute, HTTP, because the container only speaks HTTP internally
3. The reverse proxy catches the HTTP request and issues another `301` to `https://mysite.com/about/`

Adding `proxy_redirect http:// https://;` to the reverse proxy collapses this into a single redirect. The backend's `http://` location headers get rewritten to `https://` before they reach the client:

```nginx
location / {
    proxy_pass http://host.docker.internal:81;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_redirect http:// https://;
}
```

## Performance

### Gzip Compression

The Hugo container serves uncompressed responses by default. Adding gzip at the reverse proxy level compresses HTML, CSS, and JavaScript before they reach the client:

```nginx
gzip on;
gzip_types text/plain text/css application/javascript application/json image/svg+xml;
gzip_min_length 1024;
gzip_proxied any;
gzip_vary on;
```

`gzip_vary on` adds a `Vary: Accept-Encoding` header so Cloudflare's CDN caches compressed and uncompressed versions separately.

### Long Cache TTLs for Static Assets

Hugo fingerprints CSS and JavaScript files by content hash — the filename changes whenever the content changes. This makes them safe to cache indefinitely. A separate `location` block sets a one-year TTL:

```nginx
location ~* \.(css|js)$ {
    proxy_pass http://host.docker.internal:81;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_redirect http:// https://;
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

### Deferring JavaScript

The theme's `bundle.min.js` was loaded in the footer without `defer`, keeping it on the critical render path. Overriding the theme's `footer.html` partial and adding `defer` removes it:

```html
<script type="text/javascript" src="{{ $bundle.RelPermalink }}" defer></script>
```

### Font Preloading

The FiraCode font was being discovered late — only after the CSS was parsed — adding it to the critical path chain. Preloading it in `<head>` starts the download in parallel with the CSS:

```html
<link rel="preload" href="/fonts/FiraCode-Latin.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/fonts/FiraCode-LatinExt.woff2" as="font" type="font/woff2" crossorigin>
```

This dropped the maximum critical path latency from 220ms to 177ms.

## Hugo Configuration

A few Hugo settings that improve the site without requiring any template changes:

```toml
enableGitInfo = true

[markup.highlight]
  lineNos = true
  lineNumbersInTable = false
  noClasses = false

[markup.goldmark.renderer]
  unsafe = true

[params]
  showLastUpdated = true
  Toc = true
  readingTime = true
```

`enableGitInfo` pulls `lastmod` from git history for each file, keeping the sitemap accurate without manually updating front matter. Pair it with `fetch-depth: 0` in the GitHub Actions checkout step, otherwise the shallow clone only sees the most recent commit:

```yaml
- uses: actions/checkout@v4
  with:
    submodules: true
    fetch-depth: 0
```

`noClasses = false` is required by the terminal theme — without it, Hugo uses inline styles for syntax highlighting instead of CSS classes, overriding the theme's `syntax.css`.

## SEO

### H1 Heading

The homepage content was using `##` (H2) as the main heading. Changed to `#` (H1) — each page should have exactly one.

### Page Title

The `<title>` tag was just the site name. Expanded it to include role and keywords while staying within the 50–60 character sweet spot that tools like Yoast recommend:

```toml
title = 'Igor Mihajlov | Go & TypeScript Backend Engineer'
```

### Charset in HTTP Header

UTF-8 was declared in the HTML `<meta>` tag but missing from the `Content-Type` HTTP response header. One line in the nginx server block:

```nginx
charset utf-8;
```

### Custom 404 Page

Hugo generates a `404.html` from any `layouts/404.html` template. The terminal theme includes one, but nginx needs to be told to serve it on 404 errors:

```nginx
location / {
    proxy_pass http://host.docker.internal:81;
    ...
    proxy_intercept_errors on;
    error_page 404 /404.html;
}
```

## Security Headers

### The nginx Inheritance Gotcha

Before adding headers, there's an important nginx behaviour to understand: `add_header` directives in a `location` block **replace** the parent server block's `add_header` directives — they do not merge. If you define headers at the server block level and then use `add_header` inside a `location` block, the server-level headers disappear for responses from that location.

The fix is to repeat all security headers in every `location` block that uses `add_header`.

### The Headers

```nginx
server_tokens off;

add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), accelerometer=(), gyroscope=(), magnetometer=(), interest-cohort=()" always;
```

- `server_tokens off` — stops nginx from advertising its version number in error pages and response headers
- `HSTS` — tells browsers to always use HTTPS for this domain, even if the user types `http://`
- `nosniff` — prevents browsers from MIME-sniffing responses away from the declared content type
- `SAMEORIGIN` — blocks the page from being embedded in iframes on other domains
- `Referrer-Policy` — limits referrer information sent when users navigate away from the site
- `Permissions-Policy` — disables browser APIs the site doesn't use (camera, microphone, geolocation, etc.)

### Content Security Policy

CSP is the most impactful security header and the most complex to get right. It whitelists every source of scripts, styles, fonts, and images — anything not on the list gets blocked.

For this site everything is self-hosted, so the policy can be strict:

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self'; font-src 'self'; img-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests;" always;
```

A few things worth noting:

- Cloudflare's email obfuscation script is served from `/cdn-cgi/scripts/...` — a path on your own domain — so `'self'` covers it
- JSON-LD `<script type="application/ld+json">` blocks are data, not executable JavaScript, so they're not affected by `script-src`
- `frame-ancestors 'none'` is the CSP equivalent of `X-Frame-Options` and is preferred by modern browsers
- `upgrade-insecure-requests` auto-upgrades any stray `http://` resource loads to `https://`

Before deploying a CSP, check the page source for inline scripts or styles — they require `'unsafe-inline'`, which significantly weakens the policy.

## The Complete Nginx Config

```nginx
server {
    listen 80;
    server_name mysite.com www.mysite.com;
    return 301 https://mysite.com$request_uri;
}

server {
    listen 443 ssl;
    server_name www.mysite.com;

    ssl_certificate /etc/nginx/ssl/mysite.com.pem;
    ssl_certificate_key /etc/nginx/ssl/mysite.com.key;

    return 301 https://mysite.com$request_uri;
}

server {
    listen 443 ssl;
    server_name mysite.com;

    ssl_certificate /etc/nginx/ssl/mysite.com.pem;
    ssl_certificate_key /etc/nginx/ssl/mysite.com.key;

    server_tokens off;

    gzip on;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_vary on;

    charset utf-8;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self'; font-src 'self'; img-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests;" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), accelerometer=(), gyroscope=(), magnetometer=(), interest-cohort=()" always;

    location ~* \.(css|js)$ {
        proxy_pass http://host.docker.internal:81;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_redirect http:// https://;
        expires 1y;
        add_header Cache-Control "public, immutable";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self'; font-src 'self'; img-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests;" always;
        add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), accelerometer=(), gyroscope=(), magnetometer=(), interest-cohort=()" always;
    }

    location / {
        proxy_pass http://host.docker.internal:81;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_redirect http:// https://;
        proxy_intercept_errors on;
        error_page 404 /404.html;
    }
}
```

## The Result

After these changes the site scores A+ on [securityheaders.com](https://securityheaders.com), passes Core Web Vitals, and has no redirect errors in Google Search Console. The remaining render-blocking resources are CSS — unavoidable without inlining critical styles, which isn't worth the complexity for a terminal-themed site where essentially all CSS is critical.

The full git history for these changes is on [GitHub](https://github.com/irog-c/igormihajlov.com).
