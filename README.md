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
