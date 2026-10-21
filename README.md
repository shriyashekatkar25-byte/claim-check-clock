# Claim Check Clock

**How long should an identity claim live before the row filter checks again?** An APM Lab decision prototype that uses inverse-CDF Monte Carlo (UCLA Stats 102C, Ch3) to pick a default claim lifetime for per-end-user data filtering.

- **Live demo:** https://shriyashekatkar25-byte.github.io/claim-check-clock/
- **Gallery:** https://shriyashekatkar25-byte.github.io/apm-lab/
- **Author:** Shriya Shekatkar · UCLA, Statistics & Data Science · APM Lab

## The launch that inspired it

On October 19, 2026, the Databricks platform release notes announced **filtering data per end user with OAuth custom identity claims**: a custom identity claim on a service principal's OAuth token can drive row filtering in dynamic views or ABAC policies, so one service principal can serve many users with separate data access.
Source: [Databricks platform release notes, October 2026](https://docs.databricks.com/aws/en/release-notes/product/2026/october)

Related context used for the write-up: Unity Catalog governance features in the same month's notes (governed tag propagation, cross-engine ABAC), from the same source.

## The PM decision

Imagine I am the PM for claim-based filtering. App builders now route thousands of end users through one service principal, with a claim such as `region=EMEA` deciding which rows each person sees. When a user is offboarded or moved, the old claim keeps working until the token is replaced.

**Decision:** what claim lifetime (TTL) should the app template recommend, and do we need a push revocation hook alongside it?

- **Success metric:** P99 time-to-enforce (minutes from an access removal to the filter no longer serving old rows), target ≤ 30 min.
- **Guardrails:** ≤ 2% of removals with a stale read of sensitive rows · P99 time-to-enforce ≤ 30 min · peak token mints ≤ 25/s · ≤ 20 user-visible refresh errors per day.
- **Stakeholders:** app builders, security and compliance, the identity team, Unity Catalog governance PM/eng, support.

**What the simulation shows (simulated data):** TTL alone cannot fix it. Without push revocation, even a 15-minute TTL leaves roughly 18% of removals with a sensitive stale read, and anything shorter overloads the token service. With a push hook at ≥ 95% reliability, a **30-minute TTL** passes every guardrail: the hook sets the median, the TTL only bounds the tail when the hook fails.

## What this prototype shows about product judgment

| Databricks APM skill | Where it shows up |
|---|---|
| Customer problem framing | Starts from the app builder's real worry: "how many rows leak between a removal and token expiry?" |
| Prototyping | A working simulator with sliders, a TTL sweep, and a memo that rewrites itself |
| SQL / Python | Illustrative SQL to measure the real exposure window and a NumPy re-run with fitted rates |
| Data-driven decisions | Four explicit guardrails; the recommendation is the longest TTL that clears all of them |
| Clear writing | Director memo plus a one-sentence VP summary that changes with the evidence |

## Methods

| Method (Stats 102C) | Used for |
|---|---|
| Ch3 inverse CDF, uniform | Time left on a token when a removal lands: `U · TTL` |
| Ch3 inverse CDF, exponential | Gaps between a user's queries `−ln(U)/λ`, and push-hook delay |
| Ch3 finite discrete sampling | Removal type (offboarding / team move / role downgrade) by partitioning [0,1] |
| Ch3 inverse CDF, geometric | Mints until the next refresh failure `⌈ln U / ln(1−p)⌉` |
| Ch1 Monte Carlo basics | Seeded RNG, estimating rates and percentiles from 4,000 simulated removals |

## Design baseline

Four pins from Shriya's Pinterest home feed, each redrawn from scratch in SVG/CSS/canvas. No pin images are downloaded, embedded or hotlinked.

- **Hero** — ["I hope every lotus girly finds this" by Avaniie](https://www.pinterest.com/pin/1091560028460498938/): photo-collage grid, hot lotus pink #E8508A, blush #F7B6CC, pad green #4E7A3A, seed-pod lime, white outlined sticker headline.
- **Cards and buttons** — [pinback button pin by gabriela](https://www.pinterest.com/pin/921197298819669380/): lavender enamel face #E9E7F7, black ring, glossy highlight, lime doodle blob #C5E88A, handwritten labels in slate #2C3A55.
- **KPI badges** — ["Manifesting ✨" by Lois](https://www.pinterest.com/pin/682576887316118515/): stark white tile, heavy black grotesk value over a light-weight "PASS"/"FAIL" result word.
- **Simulation room** — ["Everything starts with a DOT" flash by dear_hitz](https://www.pinterest.com/pin/934285885215735039/): off-white flash sheet with "+" grid marks, stipple/dotwork, cobalt #2F5BD3 accents, four-point sparkles, italic signature caption.

## Run locally

Open `index.html` in a browser. It is a single self-contained file (Google Fonts only).

---

Unofficial concept prototype by Shriya Shekatkar. All data is simulated. Not affiliated with any company.
