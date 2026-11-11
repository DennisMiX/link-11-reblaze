# link-11-reblaze
Link11 / Reblaze project for the powerfleet marketing team to manage and log permissions and acess issues.

## 2026-11-10 to 2026-11-11 - Joomla/YOOtheme live-preview 502/504 investigation

### Scope and impact

- YOOtheme live-preview updates fail with HTTP 502/504 on **some Joomla articles**, while the same editing workflow works on other articles. The issue is **not specific to heading changes**; changing different YOOtheme elements can trigger it.
- The failure is initiated from Joomla's `/administrator/` editing session, but the captured preview request is a background **POST to a frontend Joomla component URL**. It is not an ordinary public GET and should not be diagnosed using only public-page loads.
- Captured failing request: `POST https://www.powerfleet.com/us/component/content/article/on-site-4/?catid=229` (article ID `3980`, YOOtheme template style `20`). Changing a heading and waiting for live preview was the captured reproduction; the heading did not persist. The user reports other element changes and some other articles also fail.
- Public route: `www.powerfleet.com` through Link11/Reblaze. DNS is hosted on AWS Route 53; both apex and www resolve to `128.65.223.101`. Hostinger confirms origin `92.112.186.38`.

### Link11 event export findings

The dashboard export contains three matching events for source IP `156.155.19.37`, host `www.powerfleet.com`, method POST, and the affected path/query. Exact UTC event data:

| Timestamp | Request ID | Status | Origin/upstream response time | Request length |
| --- | --- | ---: | ---: | ---: |
| 2026-11-10 08:55:59.419357 | `2ef3e7fd871e26c242b4b8f95b4ba5a6` | 504 | 24.347 s | 59,410 bytes |
| 2026-11-10 08:55:58.488920 | `ba38ad350c7a7893ae68175376dc9351` | 502 | 24.074 s | 59,406 bytes |
| 2026-11-10 08:55:57.958961 | `eb80007f2b73dd9e24f77c224619dd84` | 504 | 24.341 s | 59,400 bytes |

For all three events the export records `upstream_addr: 92.112.186.38:443`, matching `upstream_status` and `upstream_response_time`, `blocked: false`, `challenge: false`, and `monitor: true`. Link11 says the content filter is in monitor mode. Its event message says inspection of the `customizer` argument was **skipped** because of an inspection-size limit; that is not evidence the request was blocked or that this limit caused the gateway errors.

The browser capture for the 504 had response date `2026-11-10 08:56:23 GMT` (UTC), `X-L11-Trace: ffm7-lb2`, remote address `128.65.223.101:443`, and Chrome Timing of 26.19 seconds total / waiting for server response. The body was `504 Gateway Time-out` from OpenResty. The response headers did not show a unique request ID; Link11 correlated the event using timestamp, IP, method and path. Browser payload `Content-Length` was 56,526 bytes, while the event export's `request_length` is about 59.4 KB; the relationship between these measurements has not been explained.

**Causation remains unresolved.** The Link11 export's `upstream_status: 502/504` and approximately 24.1-24.3-second upstream time do not, by themselves, prove whether Link11 received an HTTP error from its upstream or represented a locally generated upstream failure. The dashboard labels the result “Blocked by origin,” but this label is not a raw connection error. Link11 has been asked for the raw proxy error entry and confirmation of whether an HTTP response was received versus connect/read/TLS/reset failure.

### Link11 proxy settings and email-thread findings

In the November 10 support email, Link11 identified the `www.powerfleet.com` proxy as **Default Proxy Template** and reported:

| Setting | Link11-reported value |
| --- | ---: |
| Proxy connect timeout | 15 s |
| Proxy send timeout | 30 s |
| Proxy read timeout | 60 s |
| Keepalive timeout | 60 s |
| Send timeout to client | 10 s |

Link11 says none is configured at 23-24 seconds and therefore does not consider the observed duration one of those configured timeout values. This does not independently identify which component generated the status or exclude other connection failures. Link11 confirmed content filtering is monitor-only and did not block these requests. No Link11 configuration changes were made.

Link11 requested a HAR captured during an agreed live preview update, noting that HAR files contain session cookies. Hostinger also requested the Link11 raw event/error details and a coordinated Chrome Network/HAR capture, with no replay of the authenticated preview POST without user coordination. The captured Chrome evidence already available includes Timing, response body and headers, but HAR export/redaction and a coordinated fresh reproduction remain pending.

### Hostinger investigation findings and limits

Hostinger confirmed the Cloud Enterprise Plus site is active, origin IP `92.112.186.38`, with no null route, suspension, or active hosting restriction. In the reviewed configuration, `max_execution_time` and `max_input_time` are both 480 seconds; PHP, POST and memory limits were reported well above the captured request size. Hostinger found no recorded PHP-worker, process, CPU, memory or I/O fault. Historical resource metrics use 20-minute aggregation and cannot rule out a short event.

Hostinger's access-log view is a **completed-request view**. It initially showed no matching failed POST during 2026-11-10 08:55-08:58 UTC; their available vhost error-log source was not a raw frontend proxy/load-balancer trace. Hostinger explicitly said this does not prove a request never reached its infrastructure or exclude a connection that failed before a completed access record was written. Their customer-facing tools do not expose the frontend proxy/TLS trace needed to distinguish connect timeout, read timeout, TLS failure, reset, or premature termination. At the time of those diagnostics, no infrastructure escalation had been confirmed; a later Hostinger live-chat update says the case has now been escalated to "our team." No ticket reference, named receiving team, scope of log access, or investigation result has yet been provided. No service changes were made.

The Hostinger CSV export `hostinger/01a1c911-5b62-73e4-915f-4896f0257cb2.csv` contains five HTTP 200 records, including three successful POSTs to the affected path at 08:46:02, 08:54:07 and 08:54:30 UTC, plus YOOtheme customizer GETs. It contains no records in the 08:55-08:58 failure window. The rows list source addresses `85.131.144.246` and `85.131.144.248`, not the browser/editor IP; whether these are Link11 origin-facing addresses is unconfirmed. The column is named `durationMs` and values for the POSTs are 290.809, 338.436 and 356.164, which correspond to about 0.29-0.36 seconds if the unit label is correct. Hostinger has not yet clarified the duration semantics. These successful requests show that this path sometimes succeeds; they do not capture the failed requests or explain their cause.

Hostinger's read-only direct-origin GET checks used `curl -k` after normal certificate validation failed with an expired-certificate error. With `www.powerfleet.com` retained as hostname/SNI and connection directed to `92.112.186.38`, `/us/on-site/` returned 200 and the affected component URL returned 404 in about 1.20/1.27 seconds. These were unauthenticated GETs, not the authenticated YOOtheme preview POST; they do not reproduce the issue. The 404 may relate to publication/access state and has not been established as the timeout cause. Hostinger reports the origin certificate for apex and www expired on 2026-10-20. Its removal-first Lifetime/custom SSL reinstall has no stated zero-downtime or automatic rollback guarantee. No DNS or SSL changes have been approved.

### Hostinger chat transcript update (through 2026-11-11 09:38 chat time)

The exported chat transcript is `hostinger/chat-transcript.txt`. It adds the following support history and status:

- An initial Hostinger response attributed the SSL/site issue to using AWS nameservers and suggested pointing DNS at Hostinger. This conflicts with the verified and intentional production topology: AWS Route 53 is authoritative DNS and public web traffic points to Link11/Reblaze. The user explicitly instructed Hostinger not to change DNS; **no DNS change was made or authorized**.
- Hostinger then confirmed Cloud Enterprise Plus is active and repeated the diagnostic results and limitations above. The agent stated that the customer-facing tools could not expose frontend proxy/load-balancer or TLS traces. On 2026-11-10 19:29, the Hostinger Agent explicitly could not confirm an infrastructure ticket or assigned infrastructure team and could not verify whether internal infrastructure staff could inspect those traces. No support-policy reference or ticket number was supplied.
- The user requested human support; a later chat agent identified themself as Nancy/Nacy and continued the review. On 2026-11-10 22:07 the agent explicitly said **no infrastructure escalation had been opened**. In a subsequent live-chat response (received 2026-11-11; exact time not supplied), Hostinger said the issue **had been escalated to "our team"** and they would follow up. The newer statement supersedes the earlier status, but does not identify the team as infrastructure or provide a ticket/reference; request those details.
- Hostinger asked for Link11's raw error and a coordinated Chrome Network/HAR capture during a live-preview update. The user supplied the three-event Link11 summary and asked Hostinger to correlate it with frontend infrastructure logs. The transcript ends after that evidence was provided; it does not show a HAR being exported or sent, a controlled authenticated POST reproduction, or a Link11 raw error being received.
- The agent's direct-origin connectivity tests were unauthenticated GETs. The initial certificate-validated request failed with expired-certificate error; subsequent HTTP status checks used diagnostic-only `curl -k`. Results were `/us/on-site/` 200 and the affected component URL 404 at 20:41:05/20:41:06 UTC, taking about 1.20/1.27 seconds. Hostinger did not independently verify matching access-log records. These GET tests are not evidence about the authenticated preview POST.
- Hostinger requested a comparison with a known-working article. The user supplied `https://www.powerfleet.com/us/on-site/` as a working YOOtheme live-preview example and clarified that **any YOOtheme element update** can fail on some articles, while other articles work; the issue is not heading-specific.
- A later Hostinger message again asked whether the issue was related to `powerfleet.com`; the user reconfirmed it was. No production settings, SSL, DNS, proxy, PHP or hosting services were changed, and the transcript records no authorized restart or authenticated POST replay.

**Current support status:** Hostinger now says the case has been escalated to "our team," but has not supplied a ticket reference, named the receiving team, or confirmed it can access frontend proxy/TLS traces. Await the escalation details and findings. The missing evidence remains the raw frontend proxy/TLS connection trace and correlation of the failed authenticated preview POST at both providers.

### Evidence handling

- Email thread: [link-11-support/RE_ Powerfleet.com - Editor Issues.pdf](link-11-support/RE_%20Powerfleet.com%20-%20Editor%20Issues.pdf). Contains Link11's reported timeout values, event IDs and request for event export.
- Link11 event export: `link-11-support/link-11-dashboards.json` (21 events; about 33 MB). This raw export includes cookies/session-related fields and is untracked. Do not commit or forward it unredacted. Relevant incident metadata is summarized above; no cookies, session IDs, or request payloads are copied into this README.
- Hostinger request-log export: `hostinger/01a1c911-5b62-73e4-915f-4896f0257cb2.csv` (5 records; untracked). Treat as operational evidence and do not commit unless deliberately approved.
- Hostinger chat export: `hostinger/chat-transcript.txt` (untracked). It contains support conversation details; retain locally and review/redact before sharing outside approved support channels.
- Chrome screenshots are under `screenshots/10112026/`; some header screenshots expose cookies. Do not forward those screenshots or any HAR without removing cookies, authorization headers, session identifiers and sensitive request data. Prefer a sanitized HAR shared via an approved secure channel.

### Current next steps

1. Obtain Link11's raw proxy error for request `2ef3e7fd871e26c242b4b8f95b4ba5a6`; ask whether the 504 was an actual HTTP response received from `92.112.186.38:443` or a proxy-generated upstream failure, and request connection/TLS/header/response timings and termination reason. Preserve the three request IDs above.
2. Continue Hostinger escalation for frontend proxy/load-balancer/TLS connection evidence in the same UTC window. Correlate using host, path, Link11's source address if identified, and exact timestamps. Absence from the completed access log is inconclusive.
3. If historical logs are unavailable, coordinate **one** controlled YOOtheme element update on a known failing article while both teams capture the same request. Compare against a known working article using the authenticated preview POST workflow, not public GETs. Agree the time, log capture, and safe test article first; do not replay POSTs or repeatedly retry without coordination.
4. Keep DNS, Link11 policy/timeouts, Hostinger SSL, and hosting settings unchanged until the teams identify the failing hop and propose a reviewed change. Do not disable certificate validation, use HTTP for editor login, or treat a customizer inspection-size message as proof of WAF blocking.

## 2026-11-09 - Current routing, origin SSL and safe change order

**Do not approve the pending Hostinger SSL reinstall yet.** Hostinger reports that its reinstall may remove the existing certificate before issuing a replacement, with no guaranteed uninterrupted HTTPS or automatic rollback. Unchanged public DNS does not guarantee availability if Link11 loses its origin connection.

### Verified setup and evidence limits

Public DNS queries on 2026-11-09 returned:

| Layer | Evidence | Meaning |
| --- | --- | --- |
| Authoritative DNS | `ns-1107.awsdns-10.org`, `ns-2000.awsdns-58.co.uk`, `ns-367.awsdns-45.com`, `ns-571.awsdns-07.net` | AWS Route 53 hosts authoritative DNS for `powerfleet.com`. This does not establish that AWS hosts Joomla or proxies its traffic. |
| Public apex | `powerfleet.com A 128.65.223.101`, confirmed directly by authoritative server `ns-367.awsdns-45.com` | Public traffic resolves to the proxy address, not the Hostinger origin. |
| Public www | `www.powerfleet.com A 128.65.223.101`, agreeing through the authoritative server, Cloudflare `1.1.1.1` and Google `8.8.8.8` resolvers | Same public proxy destination. No CNAME or AAAA answer was returned for `www` in the checks. |
| Public HTTPS | Valid certificate verification; test HEAD request returned HTTP `247` and `X-L11-Trace: ffm7-lb2` from `128.65.223.101` | Link11/Reblaze is handling this public request. A command-line challenge does not prove every browser session fails. |
| Hostinger origin | Hostinger support confirms `92.112.186.38`; separate direct HTTPS checks with SNI `www.powerfleet.com` and `powerfleet.com` both fail certificate validation as expired | Local hosts overrides cannot currently provide normally trusted browser HTTPS. |
| Certificate dates | Hostinger reports certificates for apex and www expired on 2026-10-20 | Direct tests independently confirm expiry failures for both names, not the exact expiry date or full certificate coverage. |

The observed architecture is:

```text
AWS Route 53 DNS -> resolves apex/www to 128.65.223.101
Browser -> HTTPS -> Link11/Reblaze public edge -> configured upstream -> Hostinger/Joomla
                                                                        92.112.186.38
```

The upstream address is based on Hostinger's confirmation and the historical origin test, not inspection of current Link11 configuration. **Still unverified:** Link11's configured upstream address, HTTP versus HTTPS, port, SNI hostname, origin certificate-validation policy, failover and current timeouts. Public DNS cannot reveal these settings. No CloudFront or AWS load balancer has been established by these checks; a Route 53 alias or other account-level configuration cannot be ruled out without console access.

DNSChecker (`https://dnschecker.org/#A/www.powerfleet.com`) could not be retrieved from this environment; the evidence above comes from direct DNS queries, not a worldwide propagation audit. No AWS, Hostinger or Link11 account configuration was changed.

### Lowest-risk sequence

1. **Restore editor access through Link11 first, independently of SSL.** Ask the external Link11 team for trusted-source bot-challenge exceptions and targeted WAF false-positive fixes. Include `/administrator/`, relevant AJAX endpoints and actual frontend YOOtheme preview/save routes: the September failing save was `POST /us/icubed-2027/`. Verify current editor egress IPs rather than reusing September addresses. Keep Joomla authentication and unrelated protections enabled; do not make a public, unrestricted admin bypass.
2. **Handle save timeouts separately.** Ask Link11 to confirm upstream timing and timeout settings. Historical origin saves took 286-320 seconds; a 60-120-second timeout alone would not accommodate those durations. Investigate slow Joomla/YOOtheme processing as well.
3. **Get Hostinger's exact certificate-validation requirements before reinstalling.** Request the method, hostnames and any required paths or DNS records. Have Link11 permit only the necessary validation requests if HTTP validation is used. This preparation does not require starting the reinstall.
4. **Prefer issuance before replacement.** Ask whether Hostinger supports installing a pre-issued custom certificate and whether the replacement interrupts HTTPS. DNS validation could issue a certificate without changing website routing, if supported. In Route 53, add only the issuer-specified validation record in the correct public hosted zone, through an authorized DNS administrator. Do not delete existing validation records or share AWS credentials/private keys in tickets or this repo. Establish a supported renewal process and expiry monitoring; a one-off custom install does not establish automatic renewal.
5. **Install only with an agreed recovery procedure.** If Hostinger can only remove first and issue afterward, there is no zero-downtime guarantee: use a low-traffic maintenance window with support available. Have Link11's upstream settings and validation access confirmed beforehand. Recovery must restore working origin TLS; reinstalling an expired certificate is not a trusted-HTTPS repair. Do not switch production traffic to unencrypted HTTP or disable certificate checks as a workaround.
6. **Verify after replacement.** Check trusted direct-origin HTTPS for both hostnames, public HTTPS through Link11, Joomla login and a controlled builder save. Confirm renewal prerequisites and who monitors expiry. A successful HEAD request does not validate a complete editor workflow.

**Keep AWS routing unchanged:** no nameserver changes, no apex/www A/alias/CNAME changes, no move to Hostinger DNS and no public origin exposure to get SSL installed. Route 53 serves DNS; it does not install Hostinger's certificate. An AWS ACM certificate is not automatically deployed to Hostinger; any certificate route must be explicitly supported by the destination and renewal workflow.

### Read-only verification commands

These commands do not change DNS, the hosts file or hosting settings. `--resolve` preserves the URL hostname for SNI and certificate verification while selecting the destination IP. Do not add `-k`.

```zsh
dig @1.1.1.1 powerfleet.com NS +noall +answer
dig @1.1.1.1 powerfleet.com A +noall +answer
dig @1.1.1.1 www.powerfleet.com A +noall +answer
dig @8.8.8.8 www.powerfleet.com A +noall +answer
curl -I --connect-timeout 10 --max-time 20 https://www.powerfleet.com/
curl -I --connect-timeout 10 --max-time 20 --resolve www.powerfleet.com:443:92.112.186.38 https://www.powerfleet.com/
curl -I --connect-timeout 10 --max-time 20 --resolve powerfleet.com:443:92.112.186.38 https://powerfleet.com/
```

Keep normal editor work on the public proxy route. If a local hosts override remains from earlier diagnostics, remove only that mapping and flush local DNS before testing Link11 policy changes. Direct-origin testing should remain approved and temporary, not a substitute for fixing editor access at the WAF.

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

### Priority 1: Restore access for the 3 known users on `powerfleet.com` — ✅ Applied (interim), 2026-09-10
**Status: resolved on `powerfleet.com`.** MD/AU/DF's IPs have been whitelisted and the team can now create/read/update/delete content in the CMS again. This is confirmed **not a long-term fix** — it's the interim IP Bypass workaround described below, not a resolution of the underlying `473` false positive — so the Link11 ticket and Priority 2/3 work below remain open.

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
- [x] Add the 3 Rules above to the ACL Policy on `powerfleet.com` today. — **Done 2026-09-10.**
- [ ] Confirm MD/AU's IPs are static (still worth confirming even though access is currently working, so it doesn't silently break later).
- [x] Re-test save flows across the failing content types (see breakdown below) once applied. — **Confirmed working: CRUD on the CMS is restored for all 3 users on `powerfleet.com`.**

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

**Note (2026-09-10):** `marketing.powerfleet.com` is back up and confirmed CRUD-working in the admin, and is being used as a **staging/test clone** for the Joomla-side performance changes (Redis cache/session handler, OPcache JIT, `behind_loadbalancer`, etc. — see the performance TODO). It has **not** been confirmed to have identical Link11/Reblaze config to `powerfleet.com`, so a clean result there validates the Joomla/PHP-level changes but does **not** confirm the 504/Proxy-Timeout fix for the live site — that still needs to be verified on `powerfleet.com` itself once Link11 actions the timeout increase.

**Performance-testing incident (2026-09-10):** first attempt at switching `cache_handler` to `redis` on the `marketing.powerfleet.com` config (`joomla-system-information/dev/configuration.php`) caused a full **500 fatal error** site-wide. Cause not yet confirmed, but `redis_server_host` is set to `localhost` with a blank `redis_server_auth` — either the Redis service isn't reachable at that host/port from this environment, or auth is required and missing. Reverted `cache_handler` back to `file` and `caching` back to `0` (matching the pre-change working state); site confirmed back up with no errors. **Before retrying:** confirm the actual Redis host/port/auth this hosting environment expects (check hPanel or ask Hostinger support) rather than assuming `localhost` is correct, then retry the cache-handler change in isolation and watch for errors immediately.

**Performance work paused (2026-09-10):** while checking whether Redis is even available on this Cloud Enterprise Plus plan, an SSH connection attempt returned a **"REMOTE HOST IDENTIFICATION HAS CHANGED"** warning (host key mismatch) for the hosting server. Hostinger's own support was asked directly and **could not confirm** whether a server migration/rebuild had occurred, and explicitly advised not to accept the new fingerprint without independent confirmation. Since that confirmation isn't available, and Redis was only an optimization (not a fix for anything currently broken), **the Redis cache/session handler work is dropped for now** — not worth the risk of proceeding on an unverified SSH connection.

**Pivoted to an SSH-free alternative (2026-09-10):** the remaining performance items don't actually require SSH at all — they can be done via hPanel's PHP Configuration page or the Joomla admin/File Manager:
- Enable Joomla's built-in **file-based caching** (`caching: 1`, `cache_handler: file`) via **System → Global Configuration → Cache** in the Joomla admin — zero risk, no file edits or SSH needed, uses the cache handler already confirmed stable. — ✅ **Confirmed correct, 2026-09-10** (System Cache: ON – Conservative caching, Cache Handler: File, all other Cache/Session tab values already correct — see screenshots `global-system-01.png`/`global-system-02.png`).
- Set `opcache.jit_buffer_size` via hPanel's **PHP Configuration** page (not `php.ini` over SSH). — ❌ **Dropped, confirmed 2026-09-10.** Manually re-checked all 8 PHP Configuration screenshots (`screenshots/php-ss/`) field-by-field — no `opcache.jit`/`opcache.jit_buffer_size` field or custom directive box exists. Then asked Hostinger's AI support agent directly, who confirmed: PHP 8.3.33 is active and OPcache is enabled, but JIT is **not exposed as a supported per-site option** on this Cloud Enterprise Plus plan — not via hPanel, not via a `.user.ini` workaround (JIT requires system-level PHP config, and direct `php.ini` access is disabled on Web/Cloud hosting), and not as a support-side override. Hostinger's own recommendation: only a VPS (where you control PHP config directly) supports this; otherwise use the already-available OPcache settings (`opcache.memoryConsumption`, `opcache.maxAcceleratedFiles`, etc., already confirmed set correctly at 384M/10000). Treated as fully closed — no further action possible on this plan tier.
- Set `behind_loadbalancer: true` and clean up the unused Memcached config via a direct `configuration.php` edit through hPanel's File Manager (already how the earlier Redis revert was done).

These are being worked one at a time, same testing discipline as before (change → test → confirm → next). File-based caching is done, OPcache JIT is confirmed unavailable and closed; next up is `behind_loadbalancer`.

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
- **`powerfleet.com` admin access is restored (interim)** — MD/AU/DF's IPs are whitelisted via the ACL Bypass Policy, and the team can CRUD content in the CMS again. This is **not a long-term fix**; it's a workaround pending Link11's root-cause fix for the `473`.
- **Fix not yet extended to the other 4 domains** — each is a separate Web Application in Reblaze, so it needs replicating individually, and may not share `powerfleet.com`'s exact WAF behavior.
- **Root cause not yet confirmed by Link11** — awaiting their log lookup on the Request ID/Session ID above to identify the exact rule/signature; the IP Bypass is a workaround, not a fix.
- **Whitelisting more URLs is not a safe fix on its own** — URL Whitelist entries bypass WAF/WebDDoS/Bot Management entirely for that path, so broad whitelisting of admin routes would reduce protection rather than targeting the specific false positive.
- **No durable onboarding process yet for future editors** (Priority 3) — needed before headcount grows, but intentionally lower priority than restoring today's access.

## 2026-09-11 — 502 Bad Gateway / Proxy Timeout on YOOtheme Pro Page Save (`www.powerfleet.com`)

Sources: [screenshots/502-ange-screenshots-ytp-joomla/](screenshots/502-ange-screenshots-ytp-joomla/), `payload.rtf`, `request headers.rtf`, [screenshots/Link11EventLog-502.txt](screenshots/Link11EventLog-502.txt).

### What happened
- **Date & Exact Timestamp:** `2026-09-11 11:09:17 UTC` (response `Date: Fri, 11 Sep 2026 11:09:17 GMT`).
- **Target URL & Method:** `POST https://www.powerfleet.com/us/icubed-2027/`
- **Initiator:** YOOtheme Pro Customizer (`customizer.js`) saving layout/article `id=3594` (`templateStyle=20`).
- **Payload:** `application/x-www-form-urlencoded`, `Content-Length: 48084` (~48 KB) containing encoded YOOtheme page builder data (`customizer` parameter).
- **Behavior:** The browser stalled for **19.94 seconds** in `Waiting for server response (TTFB)`, after which the proxy returned an HTTP **502 Bad Gateway** HTML page (`<center>openresty</center>`) generated in 0.40 ms.
- **Link11 Edge Trace Header:** `X-L11-Trace: lon2-lb2`
- **Subsequent Traffic:** Subsequent requests (tracking pixels, Google Analytics, clarity, and Joomla session keepalive calls `index.php?option=com_ajax&format=json`) immediately succeeded with HTTP `200`. No WAF challenge or 473 denial was returned.

### Hostinger investigation findings (2026-09-14)

**Historical-scope note (updated 2026-11-11):** the timeout and direct-origin conclusions below refer to the distinct 2026-09-11 save event only. They are not proof of the cause of the November 2026 preview failures. Link11 subsequently reported a 60-second proxy read timeout for the current `www.powerfleet.com` Default Proxy Template; see the 2026-11-10 to 2026-11-11 investigation above.
- **Origin execution duration:** Neighboring successful `POST /us/icubed-2027/` requests at 11:05:15 UTC and 11:07:52 UTC took **286 to 320 seconds (~5 minutes)** each to complete on the origin server.
- **LVE Limits:** Normal across the 10:00–12:00 UTC window (CPU ~5%, RAM ~600MB of 15GB, PHP workers 11 of 400). No account-level throttling or resource exhaustion occurred.
- **2026-09-11 incident interpretation:** The evidence at that time was interpreted as an upstream proxy timeout mismatch: the browser received a 502 after about 20 seconds while neighboring origin saves took roughly 5 minutes. This was the working diagnosis for that specific incident; do not generalize its assumed timeout value or cause to later requests without matching logs.

### Solution & Resolution Testing (2026-09-14)
- **Direct Origin Bypass Test (`/etc/hosts` → `92.112.186.38`):** ✅ **100% SUCCESS.** When bypassing Link11/Reblaze and saving `icubed-2027` directly on the Hostinger origin, the save completed cleanly with **no 502 Bad Gateway and no logout**.
- **Direct-origin test result:** The `icubed-2027` save completed successfully when tested directly against Hostinger on 2026-09-14. This is evidence for that page/test at that time, not proof of the November issue's cause.
- **Historical next action:** A Link11 timeout increase was proposed for this September event. Do not apply that proposal to the November incidents without new evidence; Link11's later-reported current template uses a 60-second read timeout and the November exports do not include a raw connection error.

#### How to test and bypass the Link11 / Reblaze reverse proxy (Direct-to-Origin Testing)

**Historical procedure - not currently ready for use (2026-11-09):** direct-origin HTTPS certificate verification fails as expired. Follow the current SSL/change-order section above before repeating this test. Never ignore certificate warnings for CMS logins. The cache-flush commands below are macOS-specific; browser connection reuse can require a browser restart. A hosts-file entry alone does not prove the actual connection destination.

Use these commands on macOS/Linux to route your local machine's browser directly to the Hostinger origin server (`92.112.186.38`), completely skipping Link11/Reblaze edge processing:

1. **Add the origin override to `/etc/hosts`:**
   ```zsh
   echo "92.112.186.38 www.powerfleet.com" | sudo tee -a /etc/hosts
   ```
   - *Explanation:* Appends (`tee -a`) a mapping from hostname `www.powerfleet.com` to the Hostinger origin IP `92.112.186.38` inside system hosts file `/etc/hosts` using administrator privileges (`sudo`). This forces your local machine to bypass public DNS and connect directly to the origin server.

2. **Flush local macOS DNS cache:**
   ```zsh
   sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
   ```
   - *Explanation:* Flushes Directory Services cache (`dscacheutil -flushcache`) and restarts the macOS multicast DNS daemon (`killall -HUP mDNSResponder`) so the `/etc/hosts` modification takes effect immediately across all open browsers without restarting your Mac.

3. **Verify the active hosts mapping:**
   ```zsh
   grep 'powerfleet' /etc/hosts
   ```
   - *Explanation:* Searches `/etc/hosts` for `powerfleet` to confirm that `92.112.186.38 www.powerfleet.com` is present and active.

4. **Verify in browser & test:**
   - Open a fresh Incognito/Private browser window.
   - Open Developer Tools (`F12`) → **Network** tab.
   - Navigate to `https://www.powerfleet.com/administrator/` and verify the `Remote Address` column shows `92.112.186.38:443` (Hostinger origin) rather than a Link11 edge IP.
   - Perform the failing action (e.g. page builder save).

5. **Revert and restore normal Link11 proxy routing:**
   ```zsh
   sudo sed -i '' '/92.112.186.38 www.powerfleet.com/d' /etc/hosts && sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
   ```
   - *Explanation:* Uses stream editor (`sed -i ''`) with the delete command (`/pattern/d`) to remove the `92.112.186.38 www.powerfleet.com` line from `/etc/hosts`, and immediately flushes the local DNS cache again. Your browser will resume routing traffic through the Link11 / Reblaze reverse proxy.

### Draft request to send to Link11 Support (502 Timeout / Upstream Error)
> **Subject:** 502 Bad Gateway investigation on page save — `www.powerfleet.com` (Trace: `lon2-lb2`, 2026-09-11 11:09:17 UTC)
>
> We are investigating an intermittent `502 Bad Gateway` returned to content editors saving pages via the Joomla CMS / YOOtheme Pro page builder on `www.powerfleet.com`.
>
> **Specific Incident Details:**
> - **Timestamp:** `2026-09-11 11:09:17 UTC` (`Fri, 11 Sep 2026 11:09:17 GMT`)
> - **Domain / Authority:** `www.powerfleet.com`
> - **Request Method & Path:** `POST /us/icubed-2027/`
> - **Referer:** `https://www.powerfleet.com/administrator/index.php?option=com_ajax&p=customizer&templateStyle=20&format=html&site=https%3A%2F%2Fwww.powerfleet.com%2Fus%2Ficubed-2027%2F&return=%2Fadministrator%2Findex.php%3Foption%3Dcom_content%26view%3Darticle%26layout%3Dedit%26id%3D3594`
> - **Payload Size:** `Content-Length: 48084` (~48 KB)
> - **Response Header:** `X-L11-Trace: lon2-lb2`
> - **Response Body / Server:** `502 Bad Gateway` (OpenResty)
> - **Observed Client Wait Time:** ~19.95s (19.94s TTFB)
>
> **Request:**
> 1. Please look up this request in the Link11/Reblaze access logs using timestamp `2026-09-11 11:09:17 UTC`, path `/us/icubed-2027/`, and trace `lon2-lb2`.
> 2. Confirm the Link11 `Request_ID`, whether `sent-to-origin` was true, the upstream host/IP targeted, and the `upstream-status` received.
> 3. Provide the upstream timing metrics (connection time, time-to-first-byte, upstream response time).
> 4. Clarify whether Link11 terminated the request due to an edge proxy timeout (e.g. 20-second upstream timeout) or whether the origin server (`92.112.186.38`) actively returned a `502` / closed the connection.
> 5. If this is a proxy timeout threshold, advise on the process to increase the upstream timeout for authenticated Joomla backend save routes.
