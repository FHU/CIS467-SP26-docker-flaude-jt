# Hardening a Containerized Static Site (Flaude) with nginx

---

## Learning Objectives

By the end of this lab, students will be able to:

1. Serve a static site from a custom nginx Docker container
2. Incrementally apply and verify server-side configurations
3. Use `curl` and browser DevTools to observe and validate HTTP behavior
4. Explain the purpose and tradeoff of each configuration decision
5. Write a minimal but production-realistic `nginx.conf`

---

## Lab Setup

### Project Structure

```
CIS467-SP26-docker-flaude/
├── Dockerfile
├── README.md
├── nginx.conf
├── index.html
├── src/
│   ├── app.js      
│   └── style.css
```

### Base Dockerfile

Students start with this and do not modify it — all changes happen in `nginx.conf`:

```dockerfile
FROM nginx:alpine
COPY site/ /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
```

### Rebuild Helper

You can use this command throughout the lab to build and run your image:

```bash
docker build -t flaude-nginx . && docker run --rm -p 8080:80 flaude-nginx
```

---

## Checkpoint 0 — Baseline (Just Serve Files)

### Goal
Confirm the site loads before any custom configuration is applied.

### nginx.conf

```nginx
events {}

http {
    include /etc/nginx/mime.types;

    server {
        listen 80;
        root /usr/share/nginx/html;
        index index.html;
    }
}
```

### Verification

```bash
curl -I http://localhost:8080/
```

**Expected:** `200 OK`, no special headers, no compression.

### 0.1 - Reflection Question
> What headers does nginx send by default? Are any of them surprising?

> The Last-Modified suprised me as the docker file was made later than the Last-Modified date. I didn't know ETag was for caching either.
```
Server: nginx/1.29.5
Date: Wed, 11 Mar 2026 18:21:48 GMT
Content-Type: text/html
Content-Length: 19820
Last-Modified: Mon, 09 Mar 2026 18:24:46 GMT
Connection: keep-alive
ETag: "69af106e-4d6c"
Accept-Ranges: bytes
```
---

## Checkpoint 1 — Compression

### Goal
Reduce asset transfer size for text-based files using gzip.

### Changes to `nginx.conf`

Add inside the `http` block:

```nginx
gzip on;
gzip_types text/plain text/css application/javascript application/json;
gzip_min_length 1024;
```

### Verification

```bash
curl -I -H "Accept-Encoding: gzip" http://localhost:8080/index.js
```

Look for: `Content-Encoding: gzip`

Also verify in browser DevTools → Network tab → select a JS or CSS file →
check the **Response Headers** panel.

### 1.1 Reflection Question
> Why does `gzip_min_length` exist? What's the cost of compressing a 200-byte file?

> To not waste time processing small files that would be faster to just send over the network uncompressed. The cost of compressing a 200-byte file is that even
> if it was to become a 100-byte file, the overhead of the function would not even be made up by the transfer speed.


---

## Checkpoint 2 — Cache Control

### Goal
Apply appropriate caching strategies: aggressive caching for fingerprinted assets,
no caching for HTML entry points.

### Changes to `nginx.conf`

Add inside the `server` block:

```nginx
# HTML — always revalidate
location ~* \.html$ {
    add_header Cache-Control "no-cache, must-revalidate";
}

# Fingerprinted assets — cache for 1 year
location ~* \.(js|css|png|jpg|woff2|mp4)$ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

### Verification

```bash
curl -I http://localhost:8080/index.html
curl -I http://localhost:8080/assets/My_Differential_Equation.cs8Lh0j6.mp4
```
(had to change curl link for mp4, due to it changing in the dist)

Confirm different `Cache-Control` values on each response.

### 2.1 - Reflection Question
> Why would caching `index.html` aggressively be dangerous for a single-page app?
> What would happen if a user's browser cached a stale `index.html` pointing to
> old JS bundles?

> A single-page app would likely change the index.html every single update.
> The old JS bundles might still work, but be unoptimized or actually simply broken due to relying on certain APIs.

---

## Checkpoint 3 — Security Headers

### Goal
Protect users from common browser-level attacks by adding standard security headers.

### Changes to `nginx.conf`

Add inside the `server` block (or a dedicated location):

```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";
add_header Referrer-Policy "strict-origin-when-cross-origin";
add_header Permissions-Policy "geolocation=(), camera=(), microphone=()";
add_header Content-Security-Policy
    "default-src 'self'; script-src 'self'; style-src 'self';";
```

### Verification

```bash
curl -I http://localhost:8080/
```

All five headers should appear in the response.

Also check: https://securityheaders.com (enter `http://localhost:8080` if using
a tunneling tool, or deploy to a VPS for full scoring).

### 3.1 - Reflection Questions
> Break the CSP intentionally — add an inline `<script>` tag to `index.html`
> and observe the browser console error. What does this teach you about
> how CSP is enforced?

> I added an alert script to the index, and it still loaded the page, but the alert did not appear. So it blocked the inline-script while still loading the page.

---

## Checkpoint 4 — SPA Routing Fallback

### Goal
Ensure that client-side routes (e.g., `/dashboard`, `/profile/42`) return
`index.html` instead of a 404, allowing JavaScript frameworks to handle routing.

### Setup

Add a link in `index.html` to a route that has no corresponding HTML file:

```html
<a href="/dashboard">Go to Dashboard</a>
```

Without the fallback, clicking this returns a 404.

### Changes to `nginx.conf`

Replace or update the default `location` block:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

### Verification

```bash
curl -I http://localhost:8080/dashboard
```

**Expected:** `200 OK` with the content of `index.html` — not a 404.

Also add a custom 404 page to handle truly missing assets:

```nginx
error_page 404 /404.html;
```

### 4.1 - Reflection Questions
> If every route returns `index.html` with a 200, what are the SEO implications?
> How do SSR frameworks like Next.js solve this problem?

> Crawlers for SEO might have a harder time judging the quality of the site due to the fact that every page returns a 200 (instead of a 404 for obviously false pages). Also, there would be duplicate data given to the crawlers. (also if javascript/typescript is used to renderstuff, that rendering might be too long for a crawler to care about so it might just get the base HTML)
> Next.js solves this by actually making the separate HTML files to serve to the crawlers rather than assuming they will let the javascript run, so the application's server does the rendering of the HTML page, and sends the full version to the crawler.

---

## Checkpoint 5 — Rate Limiting

### Goal
Protect the server from abusive request patterns using nginx's built-in
rate limiting directives.

### Changes to `nginx.conf`

Add to the `http` block:

```nginx
limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
```

Apply it in the `server` block:

```nginx
limit_req zone=general burst=20 nodelay;
limit_req_status 429;
```

### Verification

Use a loop to fire rapid requests:

```bash
for i in $(seq 1 30); do curl -s -o /dev/null -w "%{http_code}\n" \
  http://localhost:8080/; done
```

Some responses should return `429 Too Many Requests` once the burst is exhausted.

### 5.1 - Reflection Question
> Rate limiting on a static site might seem overkill — when would it actually
> matter in production?

> When the site starts using API calls to any server. Or when some crazy person sets up a bot to ruin your site. Static sites still eat up bandwidth, so if the site gets super popular, it might have to limit some people so it doesn't just crash and burn. So I would say it almost always matters in production, even in a static site.
---

## Checkpoint 6 — Block Sensitive Paths

### Goal
Prevent accidental exposure of configuration files, version control artifacts,
or environment files that might exist in the container.

### Changes to `nginx.conf`

```nginx
location ~ /\. {
    deny all;
    return 404;
}

location ~* \.(env|git|yml|yaml|config)$ {
    deny all;
    return 404;
}
```

### Verification

```bash
# Create a test file to block
echo "SECRET=abc123" > site/.env

# Rebuild and test
curl -I http://localhost:8080/.env
```

**Expected:** `404` — not the file contents.

### 6.1 - Reflection Question
> Why return `404` instead of `403 Forbidden`? What information does each
> status code leak to an attacker?

> To give out a 403 would signal to an attacker that the specific file they are rooting around for is actually in the production server, and they would focus their attacks there specifically. (As they would gain knowledge about the folder structure.)
> 403 Forbidden is used mainly for when security tokens (for stuff that people already can know is there) are not valid.

---

## Final nginx.conf

You should have a complete, working config. Review it as a whole and identify any ordering issues or redundancies.

---

## Deliverable: Written Reflection (Individual)

Submit a short written response (200-500 words) answering the following:

1. Which configuration had the most visible impact when you verified it? Why?
> I would say the most impact would be the security headers, probably because it added the most amount of headers and the most amount of code in the nginx.conf. However, the most "bang-for-your-buck" that happened is the rate limiting of checkpoint 5. This is because it gave the logic to block a lot of concurrent calls with just three lines of code. When I verified it, it simply gave back the 429 without me having to change any other files.
2. Choose one header or directive you added. Research what a real-world attack
   looks like that it mitigates, and describe it briefly.
> Looking at https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Referrer-Policy the Refferer-Policy, with "strict-origin-when-cross-origin", the main use of it is to prevent leaking data to other sites (and thus attackers). It does seem, based on https://security.stackexchange.com/questions/66165/does-referrer-header-checking-offer-any-real-world-security-improvement that it could be circumvented, but it would be better to make someone have to circumevent than to just let "the low hanging fruit" also be able to attack you.
3. What does this lab reveal about what managed hosting platforms like Netlify
   are silently doing on your behalf?
> Caching, Security, and API limiting seem to be the three aspects that you don't have to keep up with (as much) when using a hosting platform. With the caching, I had to choose myself what files and for how long they needed to be cached for my users. I also had to specify the security headers, even though a lot of them are what is normally used for deployed applications. For the API rate limiting, I had to manually specify what is the limit and for who; managed hosting platforms could have more advanced rate limiting than me because they have a firmer grasp on it than I do now.

---

## Grading Rubric

| Component | Points |
|---|---|
| All 6 checkpoints complete with working config | 40 |
| Verification commands run and output documented (screenshots or paste) | 20 |
| Written reflection — depth and specificity | 30 |
| Config is clean, commented, and well-organized | 10 |
| **Total** | **100** |
