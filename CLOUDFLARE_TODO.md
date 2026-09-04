# Cloudflare domain migration

Complete these steps after deploying the repository changes.

## Pages custom domain

- [ ] In **Workers & Pages**, open the existing Pages project.
- [ ] Under **Custom domains**, add `faillearnrepeat.net`.
- [ ] Wait until the domain and its TLS certificate are active.
- [ ] Confirm the Pages production build uses:
  - Build command: `hugo --gc --minify`
  - Build output directory: `public`
  - Environment variable: `HUGO_VERSION=0.156.0`

## Redirects

- [ ] Disable the existing catch-all redirect from `faillearnrepeat.net` to GitHub.
- [ ] Create a **301 Dynamic Redirect** for `blog.faillearnrepeat.net`:
  - Condition: `http.host eq "blog.faillearnrepeat.net"`
  - Target expression: `concat("https://faillearnrepeat.net", http.request.uri.path)`
  - Preserve query string: enabled
- [ ] Create the equivalent **301 Dynamic Redirect** for `www.faillearnrepeat.net`:
  - Condition: `http.host eq "www.faillearnrepeat.net"`
  - Target expression: `concat("https://faillearnrepeat.net", http.request.uri.path)`
  - Preserve query string: enabled
- [ ] Keep the DNS records for `blog` and `www` proxied so the Redirect Rules run at Cloudflare's edge.
- [ ] Remove the existing Cloudflare `/cv` redirect after verifying the Pages `_redirects` rule.
- [ ] Add an account-level **Bulk Redirect** from the exact production `<project>.pages.dev` hostname to `https://faillearnrepeat.net`, preserving paths and query strings. Do not match preview deployment hostnames.
- [ ] Purge the Cloudflare cache after the cutover.

## Services and verification

- [ ] Update the Google Analytics web stream URL to `https://faillearnrepeat.net`.
- [ ] Add `https://faillearnrepeat.net` to Google Search Console and submit `/sitemap.xml`.
- [ ] Use Search Console's Change of Address tool if `blog.faillearnrepeat.net` has its own URL-prefix property.
- [ ] Verify the apex homepage and representative `/blog/<slug>/` pages return `200`.
- [ ] Verify `/cv` returns `302` to the existing Google Doc.
- [ ] Verify `blog`, `www`, and the production `pages.dev` hostname return one direct `301` to the equivalent apex URL, preserving paths and query strings.
- [ ] Verify unknown apex paths return the Hugo `404` and that there are no TLS errors or redirect loops.
- [ ] Keep the old-host redirects for at least one year, preferably indefinitely.
