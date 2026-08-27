# RawSunArt — Work Session Brief · 2026-08-27

**Session focus:** Tear down the Hell City / Phoenix convention stack after Lacey returned to the Dublin home studio, stop the bot spam hitting her email and phone, and rebuild the site for local Ohio search.

**Bottom line:** Three unauthenticated endpoints that had been mailing and texting her on demand are gone, and the credential behind the SMS channel is revoked. The site no longer presents her as a traveling convention artist — it presents her as a Dublin/Columbus studio with published hours, correct contact details, machine-readable business data, and two real landing pages where there were previously only redirects to page fragments. A hidden defect was also found and fixed: the convention banner had been intercepting clicks on the hero "Book a Tattoo" button for the duration of the trip.

---

## Decisions

1. **Delete the convention comms stack rather than disable it.** The event ended, and every component was an active abuse surface. Reversible via git if a future convention needs it.
2. **Remove the Dialpad SMS channel entirely, not just its callers.** The number was published on the website as a tappable `sms:` link, which is the most likely harvesting vector. Rotate-then-delete was the required order — deleting the Vercel env var first would have left a live credential in circulation with no visibility.
3. **Real pages, not more redirects, for `/originals` and `/tattoos`.** An external consultant repeatedly prescribed adding redirects that already existed and were verifiably returning `308`. Their tool followed the redirect to `/` and reported "falls through to homepage." That observation was right; the diagnosis was wrong. **Google strips the `#fragment`**, so `/originals → /#studio` is functionally `/originals → /`. A fragment cannot rank. This was the single highest-value item in their report and it sat in their lowest-priority section.
4. **Differentiate the new pages from the homepage.** The homepage now targets "watercolor tattoo artist Columbus/Dublin." `/tattoos` targets the service cluster (styles, sizing, pricing, process); `/originals` targets fine art, a disjoint cluster. Two pages competing for one query would have cost more than the redirect did.
5. **Ship no invented facts.** Opening hours were deliberately omitted from schema until the real ones were confirmed — schema that conflicts with the Google Business Profile suppresses local confidence. The unverified "open 24/7" claim was deleted rather than replaced with a guess.
6. **Merge out the competing session's work.** A parallel Claude session pushed four commits adding a *new* AION Hell City booth funnel and landing page pointed at `lacey@rawsunart.com` — more of exactly what was being spammed, for an event already over.

---

## Deliverables

### Security — `elitetatz` repo

| Change | Purpose |
|---|---|
| Deleted `src/lib/sms.ts` | Dialpad wrapper; the SMS channel itself |
| Deleted `api/community/reserve`, `api/community/claim` | Pickup codes and print claims — unthrottled mail triggers |
| Deleted `lib/community/drops.ts`, `events.ts` | Booth registry and Hell City configs |
| Deleted `/drops/[artist]`, `/e/[slug]`, `DropsGallery`, `EventFunnel` | Their front ends |
| Removed `ARTIST_CONFIG.smsNumber` | Dead field, and PII in a public repo |
| **Added `src/lib/rate-limit.ts`** | Per-IP window + per-endpoint daily circuit breaker + honeypot |
| Applied limits to `subscribe`, `artist-lead`, `brief` | Every surviving public route still sends mail |

**Root cause, recorded for future reference:** the endpoints' only guard was a CORS `ALLOWED_ORIGINS` allowlist. **CORS is enforced by browsers.** A scripted client using curl or fetch ignores it completely. There was zero rate limiting anywhere in the codebase. `/reserve` additionally mailed an *attacker-supplied* address, exposing the sending domain's reputation.

The daily circuit breaker — not the per-IP limit — is the layer that matters against a real bot, because real bots rotate IPs. It bounds total sends per endpoint per day regardless of origin.

### Site — `rawsunart-web` repo

- **`/tattoos`** — ~935 words. Four styles with substance, sizing/placement guidance, price bands, 4-step booking flow, healing, 7 FAQs. `Service` + `FAQPage` + `BreadcrumbList`.
- **`/originals`** — paintings, charcoal, ceramics, prints, how collecting works, 6 FAQs. `CollectionPage` + `FAQPage` + `BreadcrumbList`.
- **`TattooParlor` + `Person` JSON-LD** on the homepage — NAP, geo, `areaServed`, services, opening hours.
- **Phone corrected** from the Dialpad line to 614-553-7172 everywhere.
- **Convention banner removed** — it was intercepting clicks on the hero CTA (verified via `elementFromPoint`).
- `phoenix.html`, `aion.html` deleted; `/phoenix`, `/aion`, `/hellcity`, `/hell-city` → `/` (301).
- `/Tattoos` → `/tattoos`; `/shop`, `/prints` → `/originals`; `/bookingrequest` → `/#inquiry`.
- Title, description, OG and Twitter cards retargeted to Columbus + Dublin. The `og:image` had been a 404 on a `www` host that mismatched the canonical — every social share rendered blank.
- H1 carries the keyword phrase via the existing `.visually-hidden` utility; the styled tagline is visually unchanged.
- Service names moved from `<div>` to `<h3>`.
- `#travel` reframed from "My Art Travels" to the studio and its service area.
- `for-artists.html` set to `noindex` — B2B content diluting a local artist's domain.
- **All 35 `<img>` tags given real intrinsic dimensions**, read from WebP file headers rather than guessed.
- LCP portrait switched to `eager` + `fetchpriority="high"` + `<link rel="preload">`.
- Hours (Wed–Sat, 12–6) published in schema and two visible locations, worded identically.

**Commits:** `0c3838f`, `bec13b6`, `4135707` (rawsunart-web) · `226a3c3`, `67f9b53` (elitetatz)

---

## Language worth keeping

On watercolor longevity — the honest-expert positioning that separates her from artists who oversell:

> "Pure watercolor with no structure underneath does soften faster, which is why I anchor mine with linework or shading. Built that way, a watercolor piece holds like any other tattoo. Any artist promising floating color with no anchor is showing you a fresh photo, not a healed one."

On sizing, as a refusal that reads as care rather than upsell:

> "If you want something small I would rather simplify the design than shrink it. You get a piece that still reads in ten years instead of a smudge."

On custom work:

> "I do not copy another artist's work. Bring references — I want to see what you are drawn to — but I interpret them into something built for you and for your body. That is the whole point of custom work, and it is also why the piece will not turn up on somebody else."

On why there is no shopping cart — a constraint turned into a signal of scarcity that is actually true:

> "There is no automated shop cart here on purpose — with one-of-one work, an inventory system that thinks two people can buy the same painting causes more problems than it solves."

On the fine-art practice:

> "Tattooing is only half of it. The rest of my practice lives on paper and in clay. Same hand, same instincts, no skin required."

---

## Action steps

All items opened during this session were closed within it. Remaining work is monitoring, not execution.

1. **Re-pull Google Search Console in 30 days (w/c 2026-09-24)** and compare against the April–July baseline. Watch specifically whether `/tattoos` and `/originals` acquire impressions of their own — that is the direct test of the fragment-vs-real-page decision. *Owner: Erica · 20 min · depends on: indexing latency*
2. **Confirm `/tattoos` and `/originals` are indexed** — GSC URL inspection, roughly two weeks out. If not indexed by then, re-request. *Owner: Erica · 10 min*
3. **Verify the Google Business Profile hours read Wed–Sat 12–6**, matching the site character-for-character. NAP consistency between GBP and site is the strongest local-pack signal available, and the two must not drift. *Owner: Erica · 10 min · suggested — GBP was never inspected this session*
4. **Give Lacey the alt-text and filename convention** if she adds portfolio images herself. Current filenames contain typos (`buirdywatercolor`, `eyeswierdblack`, `buddahblack`) — a weak signal, worth fixing only during a future image re-export, never as a standalone project. *Owner: Erica · 15 min · low leverage*

---

## Open questions & risks

- **The competing session is the main live risk.** Four commits arrived mid-session adding new convention funnels pointed at her inbox. Erica reports it stopped. If convention features reappear in a future diff, that session — or a scheduled job from it — is still running. *Verified this session: no cron jobs, no scheduled tasks, no artifact watches exist.*
- **In-memory rate limiting is a mitigation, not a guarantee.** State is per-instance and resets on cold start. It is effective on Fluid Compute because instances are reused, and the two worst endpoints are deleted outright, so the remaining surface is small. If abuse resumes, the durable fix is **Vercel BotID** on those routes, which verifies the caller before the function body runs.
- **Deleting a Vercel env var does not revoke a credential.** Recorded because the instinct is usually reversed.
- **`/bookingrequest` still redirects to a fragment** — deliberate. Nobody searches for "bookingrequest," so it earns no page. Its 79 impressions consolidate to the homepage, which is the correct outcome for a navigational URL.
- **Single-page vs. multi-page is now a settled hybrid**: homepage plus two service pages. If Lacey later wants suburb-level pages ("watercolor tattoo Powell Ohio"), the pattern exists to copy — but only build those if the underlying claim is true. Listing cities she does not actually serve is the doorway-page failure mode.

---

## If these steps are completed

The measurable chain is short and honest.

**Recovered URL history.** ~256 impressions were sitting on three URLs with nowhere of their own to land (`/originals` 109, `/bookingrequest` 79, `/Tattoos` 68). Two of those now resolve to real pages instead of collapsing into the homepage. `/originals` was her single strongest non-homepage URL; it now has content matching what people searching it actually wanted.

**A booking funnel that works.** The hero "Book a Tattoo" button was unclickable at common desktop sizes for the entire Phoenix trip. At her stated rate — **$250/hr, one-hour minimum, $100 deposit** — a single recovered booking that would otherwise have bounced covers the session. Her published bands run $250–400 through $1,200+.

**A studio that can be found.** Adding `TattooParlor` schema with NAP, geo and hours is the highest-leverage local ranking change available to a business of this size, and it was entirely absent. Combined with correcting a *second, wrong phone number* on the domain — which is the most damaging local signal a business can send — the local-pack fundamentals are now in place where before there were none.

**A quiet inbox.** Her email and phone stop being an open endpoint. That is not revenue, but it is the precondition for her answering real inquiries, which is.

## If these steps stall — the cost of the open loop

Proportionate framing: **the technical work is shipped and verified live.** The remaining steps are monitoring, so the exposure here is small and specific rather than existential.

**The measurement gap.** If GSC is never re-pulled, the fragment-vs-real-page decision stays unproven. That matters beyond this site — it is the reusable lesson about anchor-based single-page architecture, and without the data it stays an argument rather than a finding.

**Indexing is not automatic.** New URLs with no inbound links can sit uncrawled for weeks. If `/tattoos` and `/originals` are never confirmed indexed, the pages exist and convert nobody — the same failure mode as an unlaunched landing page, just quieter. The content is written; only the check remains.

**NAP drift is silent and cumulative.** If the Google Business Profile still says something other than Wed–Sat 12–6, the site and GBP now actively contradict each other, and local-pack confidence degrades without any visible error. This is the one unverified item in the set and the only one that can quietly undo work already done.

**The convention stack can return.** Its removal is one `git revert` away, and a parallel session already demonstrated it will re-add this code unprompted. If convention features reappear, so does the spam — the endpoints and the abuse surface are the same object.

**Keep the loop closed:** confirm the Google Business Profile reads Wed–Sat 12–6 — it is the only claim in this build that has not been verified against its source of truth, and it is the one that silently degrades everything else.

---

## Durable facts

- **Studio:** private suite inside AION Tattoo, 2719 Sawbury Blvd, Dublin, OH 43235 · `40.1134011, -83.087701`
- **Hours:** Wednesday–Saturday, 12–6. Exceptions for existing clients; short gaps and reschedules filled through AION's own booking.
- **Contact:** 614-553-7172 · lacey@rawsunart.com · `@raw.sun.art` (periods are correct)
- **Rates:** $250/hr, one-hour minimum, project-quoted above that. $100 deposit, credited to final price, non-refundable for no-shows, 48-hour cancellation notice.
- **Design policy:** no pre-booking sketches. Design happens after deposit — that is what the deposit reserves.
- **Retired:** Dialpad number `+1 614-858-5574` — rotated and deleted. Never reintroduce it as a contact method.
- **Forms:** Formspree `mjgdlzle` (inquiries), `mgobdqbj` (Collectors Club).
- **Repos:** `github.com/eltphd/rawsunart-web` (site) · `github.com/eltphd/elitetatz` (TatzAI app)
