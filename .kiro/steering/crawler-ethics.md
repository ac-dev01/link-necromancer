# Crawler ethics and security (non-negotiable)

## SSRF protection
- Only `http:` and `https:` URLs.
- Resolve DNS before every request and refuse private, loopback, link-local, unique-local, multicast, unspecified and cloud metadata addresses (e.g. 127.0.0.0/8, 10/8, 172.16/12, 192.168/16, 169.254/16, ::1, fc00::/7, fe80::/10).
- Re-check after every redirect hop. Max 5 redirects.
- All network code goes through one guarded fetch function in `src/core`. Never call fetch directly elsewhere.

## Politeness
- Respect robots.txt for the crawled site.
- Max 4 concurrent requests per host, small delay between requests to the same host.
- Identifying User-Agent: `LinkNecromancer/1.0 (+<project URL>)`.
- Per-request timeout 10s; max response body read 2 MB.
- Use HEAD first, fall back to GET when HEAD is rejected (405/501) or unreliable.

## Abuse limits
- Max 200 pages per scan; rate-limit scans per client IP.
- Jobs expire from memory after a TTL.
