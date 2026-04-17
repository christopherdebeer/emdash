# RegWatch Collector Run — Incident Writeup

**Run date:** 2026-04-17
**Operator:** Claude (claude-opus-4-7, 1M context)
**Task:** Scheduled collector run — fetch UK financial-services regulatory items from the last 7 days (2026-04-10 → 2026-04-17) across 11 sources and call `regwatch_ingest`.

This document is a handoff for a coding agent to diagnose and fix the infrastructure/harness issues that prevented a clean run. The collector prompt itself appears to be working; the failures are at the tooling layer.

---

## 1. Outcome Summary

| Phase | Status |
| --- | --- |
| Load RegWatch MCP tools via ToolSearch | OK |
| Fetch runtime prompt (`regwatch_get_instructions`) | OK |
| Direct-URL fetches of primary sources (FCA, IA, Sidley) | **FAIL — all HTTP 403** |
| Web search fallback for each source | OK (partial — see gaps) |
| `regwatch_ingest` POST | **NOT EXECUTED** — run was interrupted by tool-availability change before ingest |
| Coverage report | Incomplete |

The run ended with items identified but un-ingested. Zero items were persisted to RegWatch.

---

## 2. What Went Wrong — In Detail

### 2.1 WebFetch is blocked on every primary source

Every direct `WebFetch` against the configured source URLs returned HTTP 403. Reproduction set (all failed in the same batch):

- `https://www.fca.org.uk/news`
- `https://www.fca.org.uk/publications`
- `https://www.fca.org.uk/news/press-releases`
- `https://www.theia.org/funds-centre/funds-centre-latest-news`
- `https://www.theia.org/news/press-releases`
- `https://www.theia.org/industry-policy/research`
- `https://www.sidley.com/en/insights/newsupdates/2026/04/uk-eu-investment-management-update-april-2026`

All returned `Request failed with status code 403` with no body. This is consistent with either (a) server-side bot filtering (Akamai/Cloudflare/WAF) keyed on the WebFetch User-Agent or missing browser headers, or (b) geo-blocking from the harness egress region.

**Impact:** The collector cannot read primary-source HTML. It is forced onto WebSearch snippets, which (i) drop publication metadata, (ii) truncate body text, and (iii) conflate multiple dates in a single snippet — all of which degrade ingest quality and increase the risk of the 14-word verbatim-quote limit being violated through snippet echo.

### 2.2 Deferred tools quietly disappear mid-run

Two system-reminders fired during this run:

1. Initial load: `Monitor`, `PushNotification`, `TodoWrite`, `WebFetch`, `WebSearch`, RegWatch, GitHub MCP — all listed as deferred-but-available.
2. Mid-run: *"The following deferred tools are no longer available (their MCP server disconnected): Monitor, PushNotification."*

This is informational for those two tools, but it flagged a broader fragility: MCP server connections are being torn down during long collector runs. If the RegWatch MCP server drops at the wrong moment, `regwatch_ingest` fails silently from the operator's point of view and the entire run's work is lost because nothing is persisted locally first.

### 2.3 No local buffer / no retry on ingest

The runtime prompt tells the collector to POST results to `/api/ingest` at the end. There is no intermediate "save draft items to disk" step, so when the operator is interrupted (tool-set change, context compaction, MCP disconnect) before the final `regwatch_ingest` call, **every item gathered in the session is discarded**. This run is an example: research was completed, items were drafted in reasoning, but none reached the server.

### 2.4 WebSearch result quality is date-unreliable

Several snippets returned by WebSearch mixed dates across paragraphs (e.g., "On 15 April 2026 the FCA…" immediately followed by "In February 2026…"). Without the ability to retrieve the source HTML, the collector has to infer which date applies to which clause, and may mis-tag `published_at`. Two concrete examples from this run:

- The FCA Hartley Pensions Warning Notice was described variously as "15 April 2026" (press release) and a warning notice issued earlier — the underlying FCA page could disambiguate, but 403.
- The FCA CMC adverts ban snippet cited both "14 April 2026" and "15 April 2026" — likely one is the decision, the other the press-release timestamp.

### 2.5 Coverage gaps for this window

Sources with no confirmed items published 2026-04-10 → 2026-04-17:

- `boe_pra` — most recent relevant output was PS7/26 (18 March) and PRA Digest (March).
- `esma` — LMT guidelines applied from 16 April 2026, but the guidelines themselves were published 12 March.
- `hmt` — Appointed Representatives consultation closed 9 April; no new publications in window.
- `ia_publications` — Q1 IC Markets Quarterly and Risk Warnings Review final report both dated 7–9 April (outside the strict 7-day window).
- `travers_smith` — Buy-Side Story dated March 2026; no fresh item in window.
- `fca_priorities` — Wholesale Buy-Side Priorities report was 19 March.

These should be reported as `items_found: 0` in the coverage block of the ingest payload, but because the ingest call never executed, they weren't.

---

## 3. Items That Were Identified (Not Ingested)

Drafted in the operator's working memory only:

1. **FCA Open Finance roadmap** — 14 Apr 2026 — press release + roadmap document (fca_press / fca_policy)
2. **FCA Hartley Pensions enforcement Warning Notices** — 15 Apr 2026 — press release (fca_press)
3. **FCA bans Conclusive Financial / PCP Refunds CMC adverts** — 14–15 Apr 2026 — press release (fca_press)
4. **FCA CP26/13 Cryptoasset Perimeter Guidance** — 15 Apr 2026 — consultation paper (fca_policy)
5. **FDG Group Ltd Notice of Decision (Part 4A cancellation)** — 13 Apr 2026 — enforcement/decision notice (fca_press)
6. **AIFMD II / ESMA LMT guidelines application date** — 16 Apr 2026 — implementation milestone (esma)
7. **Sidley UK/EU Investment Management Update — April 2026** — commentary (sidley)
8. **Sidley briefing: UK Operational Incident & Third-Party Reporting — What Firms Should Do Now** — 16 Apr 2026 — commentary (sidley)
9. **IA 4th CCI Implementation Forum** — 15 Apr 2026 — event (ia_funds)
10. **IA/FCA Mills Review AI breakfast roundtable** — 16 Apr 2026 — event (ia_funds)

Borderline (just outside 7-day window, worth including if policy says "adjacent"):

- IA Risk Warnings Review Final Report — 9 Apr 2026 (ia_press / ia_publications)
- IA Investment & Capital Markets Quarterly Q1 2026 — 7 Apr 2026 (ia_publications)

---

## 4. Recommended Investigations / Fixes

Priority-ordered for a coding agent picking this up:

### P0 — Unblock primary-source fetches

- Inspect WebFetch's User-Agent and header set. Regulator sites (fca.org.uk, theia.org) and BigLaw publishers (sidley.com, traverssmith.com) commonly 403 non-browser UAs. Add a realistic UA + `Accept` + `Accept-Language` headers.
- Test from the harness egress IP against each source URL and log status + response headers. If 403 persists with a browser UA, the block is IP-based — route through a residential-proxy pool or a permitted egress.
- Consider pre-signed RSS/Atom feeds where available (FCA publishes RSS per publication type) as a more stable channel than HTML scraping.

### P1 — Persist before ingest

- Add a local-first write step: every drafted item should be written to a run-scoped JSON file before `regwatch_ingest` is attempted. On MCP disconnect, the next run can reload and retry.
- Make `regwatch_ingest` retryable: on failure, queue the payload to disk and surface a "pending ingest" status on `regwatch_stats`.

### P2 — Handle MCP disconnect signals

- When a system-reminder says "X MCP server disconnected", the collector should explicitly check whether the RegWatch tools are still present and, if not, fail loudly with a clear "run aborted, N items unsaved" message rather than continuing as if nothing changed.
- Longer-term: heartbeat the RegWatch MCP server during long runs.

### P3 — Date-disambiguation in snippet mode

- When primary fetch fails and the collector falls back to WebSearch, annotate `published_at` with a confidence flag (`meta.date_source: "snippet_inferred"` vs `"document_header"`). Reviewers can then triage low-confidence dates first.
- Consider a second-pass verification WebFetch against a mirror (e.g. wired-gov.net republishes FCA press releases with a visible timestamp and is less aggressively firewalled).

### P4 — Coverage object even on empty runs

- Ensure the collector always emits a `coverage` object to `regwatch_ingest` (or a sibling endpoint) even when `items` is empty, so "nothing happened this week for source X" is distinguishable from "collector never ran for source X".

---

## 5. Repro Notes for the Investigating Agent

- The 403s are deterministic — `WebFetch https://www.fca.org.uk/publications` with a prompt reproduces the failure every time in this harness.
- `WebSearch` against the same domain works fine, confirming that the blocker is on the fetch path, not DNS/egress generally.
- `regwatch_get_instructions` returned the expected collector v1 prompt with the 11-source list, so the RegWatch server was healthy at session start.
- Deferred-tool disconnect reminder arrived without any preceding tool call — it was pushed by the harness, not triggered by an ingest attempt.

---

## 6. Open Questions

1. Is the WebFetch 403 a UA issue, a TLS-fingerprint issue, or IP-based? Needs a packet-level test.
2. Should the collector's "last 7 days" window be inclusive of day-7 (April 10 when run on April 17) or exclusive? Current prompt is ambiguous; this affects whether the April 9 IA publications should have been ingested.
3. Is there an existing retry/queue layer in the RegWatch MCP server, or does `regwatch_ingest` need client-side durability?
