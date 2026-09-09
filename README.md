# link-11-reblaze
Link11 / Reblaze project for the powerfleet marketing team to manage and log permissions and acess issues.

## 2026-09-06 to 2026-09-08 — Site-down incident (Hostinger nullroute + Link11 WAF DDoS)

Sources: [down-site-logs/fresh-new-site-issues.txt](down-site-logs/fresh-new-site-issues.txt), [down-site-logs/transcript-08-09-26.txt](down-site-logs/transcript-08-09-26.txt), [down-site-logs/transcript-08-09-26-1235.txt](down-site-logs/transcript-08-09-26-1235.txt) (fullest/final version of the Hostinger support chat, running 07:55–12:34 UTC on 2026-09-08), [incident-report/Incident Report - Powerfleet DDoS 2026-09-08.docx](incident-report/Incident%20Report%20-%20Powerfleet%20DDoS%202026-09-08.docx), [incident-report/Link11WAF-www.powerfleet.comLogs_2026-09-08.json](incident-report/Link11WAF-www.powerfleet.comLogs_2026-09-08.json).

### What happened
- **07:55 UTC 08/09** — `powerfleet.com` and `marketing.powerfleet.com` went down (all sites on the shared Hostinger Cloud Enterprise Plus account). This is reported as the **second occurrence**, following a similar incident just after the 4 July weekend.
- Hostinger's chat agent initially misdiagnosed this as a **missing DNS A record** on `marketing.powerfleet.com`, then walked that back after the user pushed back and confirmed DNS was resolving correctly (`www.powerfleet.com → 92.112.186.38`).
- The real cause was confirmed later: Hostinger had **nullrouted the shared hosting IP `92.112.186.38`** as an automatic response to a **Layer‑7 DDoS / bot-scraping flood**, dropping all traffic before it reached any website on the account (including `powerfleet.com`, `marketing.powerfleet.com`, and others sharing the IP).
- A parallel, unrelated finding surfaced during troubleshooting: an account-level malware scan had detected **26 compromised Joomla-related PHP files** (first flagged 2026-09-06). Hostinger reported these as "cleaned," but the user was told not to trust that the Joomla installation was fully intact, and to hold off on any backup restore until this was verified.
- The user manually added a `RewriteEngine On` / `RewriteRule ^ - [R=503,L]` block to `marketing.powerfleet.com`'s `.htaccess` to hard-disable that non-production site during the outage (fully reversible — remove those two lines to restore).
- The account-wide nullroute was lifted around **11:15 UTC**, but `powerfleet.com` itself continued to time out afterward — attributed to the still-unresolved Joomla file compromise and/or WAF/403 behavior, not the nullroute itself.
- Hostinger's own tooling gave **repeatedly contradictory status updates** during the incident (e.g. claiming `powerfleet.com` had no current errors while access logs simultaneously showed **zero requests reaching the site for 6+ hours**), which significantly slowed root-cause confirmation.
- When escalated to a human agent ("Restry"), the resolution pushed was to **adopt Cloudflare** (Under Attack mode, Bot Fight mode, Block AI Bots) since Hostinger's own CDN/WAF requires the domain to use Hostinger nameservers (`ns1/ns2.dns-parking.com`) to actually take effect — the account's Hostinger CDN "Under Attack" toggle was confirmed **not actually applying** because the domain isn't on Hostinger nameservers. Hostinger support explicitly said site-level DDoS mitigation (rate limiting, IP drops, caching) is "web development," which is outside their support scope, and suggested hiring a freelance developer.
- Independently, a **Link11 WAF log export** (`www.powerfleet.com`, 2026‑09‑08, 15:56:40–15:59:57 UTC, 1,000-row capped sample) confirms a genuine, distinct **Layer‑7 cache-busting DDoS** against the site that day — see full breakdown below. This is Link11's own edge WAF (separate from Hostinger's account-level nullroute) and shows the attack was real, not solely a false positive.

### Link11 WAF incident detail (2026-09-08, 15:56:40–15:59:57 UTC)
From the incident report docx and the raw JSON export (1,000 sampled rows, ~3.3 minutes — a capped sample, not the full incident duration):
- **Target:** the Resource Center — 68% of sampled requests hit `/us/resources/`, another 6% hit `/eu/resources/`. Everything else (product pages, homepage, static JS/media, WAF challenge callback) was background traffic.
- **Technique:** cache-busting flood — of 678 requests to `/us/resources/`, 677 carried a distinct, randomized filter-parameter query string, guaranteeing an edge-cache miss and a fresh backend query on almost every hit. Not an exploit attempt (no SQLi/XSS payloads observed).
- **Scale:** 812 unique source IPs across 441 ASNs and 89 countries in the sample; no single IP dominated (top IP = 41/1,000 requests). Source ASNs were overwhelmingly residential/mobile carriers (Techtel Argentina, Reliance Jio India, Charter/Comcast US, Türk Telekom, Emirates Internet, Rogers Canada, Sky UK) — consistent with a compromised-device botnet or residential-proxy network, not a single scripted origin.
- **Disguise:** plausible Chrome/Edge/mobile-Safari user-agents, and a `Referer` header spoofed to `www.powerfleet.com` itself on 99% of resource-page hits.
- **Traffic pattern:** bursty (botnet-wave style), not a flat flood — peak observed was 141 requests / 5 seconds, 133 of them (~96%) against the resource pages.
- **WAF disposition:** 86% of sampled requests scored `bot: true`; 81.4% received an active JS challenge (HTTP 247) via the "Pointer Default" ACL rule (`deny_bot` → `challenge`), not a hard block. 0% tripped rate-limit, generic-filter, or content-filter/exploit rules.

### Root causes (combined)
1. **Hostinger account-level automatic nullroute** — triggered when the shared hosting IP was judged under DDoS load; this drops all traffic to every domain on the account, including sites not being targeted, and cannot be worked around from within Joomla or `.htaccess`.
2. **Layer‑7 bot flood against `/us/resources/` and `/eu/resources/`** — a cache-busting scraping/crawling attack exploiting the Resource Center's combinable filter parameters (industry/type/topic/products/platform), which is CPU/DB-expensive and nearly impossible to cache as-is.
3. **Unverified Joomla file compromise** (26 flagged files, first seen 2026-09-06) — contributed to confusion about whether the continued downtime after the nullroute lifted was attack-related or a site-integrity issue.
4. **Hostinger's own CDN/WAF was not actually protecting the domain** — it was "Active" in the panel but non-functional because the domain isn't delegated to Hostinger nameservers, so "Under Attack" mode had no effect.

### Solutions applied / attempted
- Confirmed DNS was not the actual problem (A records were correct at the registrar).
- Hard-disabled `marketing.powerfleet.com` via a reversible `.htaccess` 503 rule while it wasn't meant to be live anyway.
- Escalated repeatedly to a human Hostinger agent, who confirmed the nullroute cause via LVE/analytics logs and pointed at the Resource Center bot traffic.
- Declined to blindly block the ~10 candidate IPs initially suggested by the Hostinger bot agent (later confirmed as not useful — distributed attack, none reappeared in follow-up log review); any manually blocked IPs from that list were subsequently unblocked.
- Did **not** restore the 2026-08-30 backup or delete/replace files while cause was still unconfirmed, per Hostinger's own guidance.
- Site-side mitigations recommended by Link11's incident report have **not yet been implemented** (see Next Steps).

### Next steps
- [ ] Move the Link11 WAF "Pointer Default" action on `/us/resources/` and `/eu/resources/` from **challenge to block** for confirmed-bot traffic (0% false-positive signal in-sample).  — *Security & Compliance*
- [ ] Request the **full, unsampled Link11 export** for the 2026-09-08 incident window to confirm true peak RPS, total volume, and attack duration (the 1,000-row/3.3-minute sample is capped). — *Security & Compliance*
- [ ] Add **path-based rate limiting** on `/us/resources/` and `/eu/resources/`, independent of query string, so randomized filters can't dodge volumetric controls. — *Security Engineering*
- [ ] Add **short-TTL (30–60s) caching** keyed on normalized/sorted filter parameters for the resource-filter query, to blunt the cache-busting technique at the origin/edge. — *Web Engineering*
- [ ] Review Link11 ACL/challenge thresholds for other combinatorial/filterable pages (search, product configurators) with the same cache-busting exposure. — *Security Engineering*
- [ ] Add monitoring/alerting for sustained bot-flagged bursts (e.g. >100 bot-scored requests/5s) on the Resource Center path. — *SOC / Monitoring*
- [ ] Fully audit and clean the Joomla installation (26 flagged files from 2026-09-06) — verify core files, extensions, templates, admin accounts — before trusting the site is fully clean; rotate all hosting/Joomla/DB/FTP/email credentials afterward.
- [ ] Decide on a durable DDoS/bot mitigation strategy for `powerfleet.com` that doesn't require ceding DNS to Cloudflare (current Hostinger CDN/WAF cannot help without moving to Hostinger nameservers), or evaluate accepting the nameserver change if Link11/Reblaze-level protection is judged insufficient on its own.
- [ ] Track false-positive reports from legitimate users on `/resources/` for two weeks after any WAF rule tightening.
- [ ] File the incident report and full log export in a permanent case record and formally close out `2026-09-08-DDOS` once the above actions are complete.

### Current main blockers
- **No durable mitigation in place yet** — the Resource Center's cache-busting exposure (root cause) has not been rate-limited or cached differently since the incident; a repeat is likely if traffic resumes.
- **Hostinger's own DDoS/CDN protection is non-functional for this domain** without switching to Hostinger nameservers, and Hostinger has explicitly deprioritized further hosting-side support for this, framing site-level DDoS defense as "web development" outside their remit.
- **Reluctance/inability to move DNS to Cloudflare** — the team does not want to hand over DNS management to enable Cloudflare's Under Attack/Bot Fight mode, leaving no confirmed alternative path to account-level protection yet.
- **Joomla integrity is not yet independently verified** — the "cleaned" status of the 26 flagged files from 2026-09-06 has not been confirmed by the team's own review, so trust in the current install is still open.
- **Only a capped/sampled Link11 log export (1,000 rows / ~3.3 minutes)** is available; the full incident window, true peak RPS, and total attack volume are still unconfirmed pending a fuller export request.

## 2026-09-09 — Joomla `/administrator/` blocked on save (not limited to YOOtheme Pro builder)

Priorities below are ordered by what unblocks work fastest: (1) get the 3 known users working on `powerfleet.com`, (2) replicate that to the other 4 domains, (3) — most time-consuming, so last — build a durable process for adding future editors across all domains. Background/diagnostic detail follows after the action plan.

### Priority 1: Restore access for the 3 known users on `powerfleet.com`
Reblaze ACL Rules support a **Bypass** operation on an **IP Address** match — per Link11's [Profile Concepts](https://waap.docs.link11.com/v2.16/product-walkthrough/security/profiles/profile-concepts) and [ACL Policies](https://waap.docs.link11.com/v2.16/product-walkthrough/security/profiles/acl-policies) docs, "the requestor will be granted access to the requested resource, without further evaluation or filtering" — including WAF/content-filter checks (still logged as `reason:bypassed`, so it's auditable). This is different from **Allow**, which still runs the WAF; **Bypass** is what's needed since the block is content-filtering, not an ACL denial. It does **not** require Link11 to identify the exact triggering rule first — it sidesteps the false positive immediately while the Link11 ticket (below) works the permanent fix in parallel.

**Action:** in the Reblaze console, edit (or create) the ACL Policy assigned to the `/administrator` resource — mapped under **Web Proxy → Security Profiles** — and add one Rule per known editor: **Match: IP Address**, **Operation: Bypass**.

| ID | Client IP | Evidence |
| --- | --- | --- |
| MD (User 1) | `142.114.5.61` | Request_ID `70179df7ff29a91516fa0150275466dd`, 2026-09-08T17:29:20 UTC |
| AU (User 2) | `156.155.11.93` | Request_ID `35ffd1546685738c93a128a76b18eacb`, 2026-09-09T06:41:27 UTC |
| DF (User 3) | `156.155.21.7` | Request_ID `445331450774f260d135efbefeb4ab2b`, 2026-09-09T08:16:29 UTC (same IP also seen at 07:56:17 UTC, Request_ID `4c9c993f9e1cfe1e776a00314b56dede`) |

- Name the policy clearly, e.g. **"Trusted CMS Editors — Admin Bypass"**, so its purpose and scope are obvious to anyone reviewing it later.
- **Scope tightly:** apply only to the `/administrator` resource definition (not the whole domain) and to these individual IPs (not a range), so this doesn't open a general WAF bypass on public-facing pages.
- Confirm with MD and AU whether `142.114.5.61` and `156.155.11.93` are static IPs — DF's `156.155.21.7` is already confirmed stable (repeated across two separate blocked requests). A Bypass Rule silently stops matching if the IP changes.
- [ ] Add the 3 Rules above to the ACL Policy on `powerfleet.com` today.
- [ ] Confirm MD/AU's IPs are static.
- [ ] Re-test save flows across the failing content types (see breakdown below) once applied.

### Priority 2: Replicate the same access to the other 4 domains
The same access is needed on:
- `marketing.powerfleet.com`
- `storage.powerfleet.com`
- `compliance.mixtelematics.com`
- `www.mixtelematics.com`

Per Link11's [Web Proxy](https://waap.docs.link11.com/v2.16/product-walkthrough/settings/web-proxy) docs, each protected domain is its own **Web Application** in the Reblaze console (selected via the pulldown at the top of the Web Proxy page), each with its own Security Profiles/ACL Policy assignments — so the Priority 1 fix **does not automatically carry over**; it has to be applied per domain.

- [ ] Re-create/re-assign the same "Trusted CMS Editors — Admin Bypass" Policy (MD/AU/DF) against each of the 4 domains' own `/administrator` resource definitions.
- [ ] Test whether the same `473` false positive even reproduces on each domain before assuming it does — `compliance.mixtelematics.com` and `www.mixtelematics.com` may run a different CMS/version or WAF configuration than the Joomla powerfleet.com sites, so don't assume identical behavior.
- [ ] Widen the Link11 support ticket (draft below) to cover all 5 domains up front, rather than raising a separate ticket per domain later.

### Priority 3 (most time-consuming): durable process for onboarding future editors, across all domains
This is the long-tail work — lower priority than getting today's 3 users unblocked, but necessary so this doesn't become a recurring bottleneck as headcount grows.

- [ ] Define who owns adding new editors to the ACL Policy (needs Reblaze console access) and how quickly it happens once someone is confirmed.
- [ ] Decide whether new editors are expected to have a **static IP** for admin work, or whether the team standardizes on a fixed egress point (office IP/VPN) specifically so admin access stays manageable as more people are added — this matters more here than for the initial 3, since new hires/contractors are less likely to already have known, stable IPs.
- [ ] Decide whether the same named ACL Policy is reused/extended across all 5 domains for every new editor (added once, replicated 5x) or whether domains diverge over time — document whichever is chosen so it doesn't drift.
- [ ] Once Link11 confirms and fixes the actual `473` root cause (Priority-1/2 tickets), re-evaluate whether the IP Bypass is still needed at all, since a proper rule-level fix would let any authenticated admin edit without per-person IP entries — this would remove most of the ongoing onboarding burden.
- [ ] Until then, keep a single source of truth (this document, or wherever the team tracks it) listing every editor's name, IP, and domain(s) they're added to, so onboarding/offboarding is auditable.

---

### Background: what's happening
- `https://www.powerfleet.com/administrator/` has been whitelisted, but **admin users (including the account owner) are still blocked** when saving/editing content in the backend — this started after Link11/Reblaze was put in front of the site.
- **Initial reproduction** suggested a clean split:
  - A standard Joomla article (a whitepaper), edited at `.../administrator/index.php?option=com_content&view=article&layout=edit&id=2841`, **saved fine**.
  - A page built with **YOOtheme Pro builder elements**, edited at `.../administrator/index.php?option=com_content&layout=edit&id=3623`, **could not be saved** — the block fired even when clicking Save with **no changes made** in the builder.
- **Correction (same day):** this is **not builder-specific** — some standard Joomla articles, including **blog posts** with no YOOtheme builder content, also return the same `access denied` error on save.
- Error returned to the browser:
  ```
  access denied.
  =======================
  Client IP: 156.155.21.7
  Host_Domain: www.powerfleet.com
  Timestamp: 2026-09-09T07:56:17.591637132+00:00
  Status_Code: 473
  Request_ID: 4c9c993f9e1cfe1e776a00314b56dede
  Session_ID: 4f311528de05125abd482624ef5f5336f408cd19ad2173095f43c125
  ```
- `473` is **not a standard HTTP status code** (Link11's own [HTTP Response Codes reference](https://waap.docs.link11.com/v2.16/reference-information-1/response-codes) only documents the standard 1xx–5xx set) — it appears to be a Reblaze-internal WAF/content-filter block code, not an origin-server error. This points to the **Link11 WAF itself** rejecting the request, not Joomla or the hosting server.

### Background: content-type breakdown (2026-09-09)
Testing across content types shows the block is **consistent per content type**, not random or per-article:

| Content type | Status |
| --- | --- |
| Case studies | ✅ Working |
| Blogs | ❌ Not working |
| White papers | ✅ Working |
| Industries | ❌ Not working |
| Videos | ✅ Working |
| Solution briefs | ❌ Not working |
| One-pagers | ❌ Not working |
| Product sheets | ✅ Working |
| Customer stories | ❌ Not working |
| Webinars | ✅ Working |
| Flyers/Brochures | ✅ Working |
| Value added services | ✅ Working |

This tracks consistently with **content type** (likely the Joomla category, custom fields/layout, or template associated with each type), rather than random article content. It doesn't yet explain *why* those specific types fail — that needs Link11's log lookup — but it's a much more targeted starting point than "some saves fail." Worth checking whether the failing types share a common category structure, custom field type (e.g. a repeatable/JSON field), or template override that the working types don't use.

### Background: why whitelisting `/administrator/` alone likely isn't enough
Per Link11's own docs on the [URL Whitelist](https://docs.link11.com/product-guides/web-ddos/interface/url-whitelist) feature: entries are treated as **exact paths** and only exempt requests that target that literal path from WAF/WebDDoS/Bot Management mitigation. Joomla admin saves happen via various component routes and AJAX calls under `/administrator/` (not just the bare `/administrator/` path), so a whitelist entry for `/administrator/` alone would not cover all of them. Given that plain articles/blogs can also be blocked, not just YOOtheme builder pages, the block looks less like a single fixed rule tied to builder markup, and more like a **content/payload-dependent WAF signature** (e.g. a generic-attack, XSS, or content-filter rule matching specific characters/patterns) that can trigger on any article whose content/type happens to match it. See Link11's [Signatures reference](https://waap.docs.link11.com/v2.16/reference-information-1/reblaze-signatures) (Generic Attacks, XSS, `bypassed@dpi-max-length`) and its [False Positive guidance](https://waap.docs.link11.com/v2.16/using-the-product/best-practices/dealing-with-false-positive), which explicitly calls out CMS POST content as a common false-positive source needing a WAF/IPS policy adjustment, not just a URL whitelist entry.

### Draft request to send to the Link11 team
> **Subject:** False-positive WAF block (473) on Joomla admin saves — `powerfleet.com`, `marketing.powerfleet.com`, `storage.powerfleet.com`, `compliance.mixtelematics.com`, `www.mixtelematics.com`
>
> We've whitelisted `https://www.powerfleet.com/administrator/`, but admin users are consistently blocked when saving certain **content types** in the Joomla backend, while other content types save fine. This tracks with content type, not with random article content:
>
> - **Blocked on save:** Blogs, Industries, Solution briefs, One-pagers, Customer stories (all content types, not just YOOtheme Pro builder pages).
> - **Saves fine:** Case studies, White papers, Videos, Product sheets, Webinars, Flyers/Brochures, Value added services.
> - Example blocked request: `POST .../administrator/index.php?option=com_content&layout=edit&id=3623` (YOOtheme Pro builder page), blocked with no content changes made.
> - Example working request (at time of testing): `POST .../administrator/index.php?option=com_content&view=article&layout=edit&id=2841` (plain whitepaper article).
> - Error shown: `Status_Code: 473`, `Request_ID: 4c9c993f9e1cfe1e776a00314b56dede`, `Session_ID: 4f311528de05125abd482624ef5f5336f408cd19ad2173095f43c125`, `Client IP: 156.155.21.7`, `Timestamp: 2026-09-09T07:56:17Z`.
> - We need the same admin access working across all 5 domains listed in the subject line, not just `powerfleet.com`.
>
> Please can you:
> 1. Look up the above Request ID/Session ID in the WAF logs and confirm which rule/signature triggered the `473` block (ACL, content-filter, custom signature, or payload-size), and whether the same rule is responsible across the blocked examples and across all 5 domains.
> 2. Advise the correct way to exempt legitimate Joomla `/administrator` backend saves from that rule — e.g. an ACL/content-filter exception scoped to authenticated admin sessions and/or the relevant component route(s), rather than broadening the existing exact-path URL whitelist entry, so we don't reopen the admin area to unauthenticated abuse.
> 3. Confirm whether this needs a policy change on Link11's side, or a configuration change we can make ourselves in the WAF/IPS Policies interface, on each of the 5 domains.

### Current blockers (admin-blocking issue)
- **Admin users cannot reliably save/edit Joomla content on `powerfleet.com`** — affecting both YOOtheme builder pages and several standard content types (see breakdown above), blocking day-to-day content work.
- **The Priority 1 IP Bypass fix has not yet been applied** — this is the fastest known path to restoring editing for MD/AU/DF, pending someone with Reblaze console access implementing it.
- **Fix not yet extended to the other 4 domains** — each is a separate Web Application in Reblaze, so it needs replicating individually, and may not share `powerfleet.com`'s exact WAF behavior.
- **Root cause not yet confirmed by Link11** — awaiting their log lookup on the Request ID/Session ID above to identify the exact rule/signature; the IP Bypass is a workaround, not a fix.
- **Whitelisting more URLs is not a safe fix on its own** — URL Whitelist entries bypass WAF/WebDDoS/Bot Management entirely for that path, so broad whitelisting of admin routes would reduce protection rather than targeting the specific false positive.
- **No durable onboarding process yet for future editors** (Priority 3) — needed before headcount grows, but intentionally lower priority than restoring today's access.
