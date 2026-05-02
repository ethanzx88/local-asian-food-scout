# Local Asian Food Scout — Plan

> Hackathon project, target ≤1h MVP. This doc captures the thesis, research, scope, and build plan so anyone landing on the repo (judges, future contributors, future-me) can ramp in 5 min.

## Vision (one-liner)

An **authenticity-aware Asian restaurant finder**, shipped as a **Claude skill** in this repo. Given a location and a craving, it returns ranked picks with a pronounced regional/authenticity lens and tells you **what to order**, not just where to go.

## The problem we're solving

General-purpose apps (Yelp, Google Maps, Fantuan, HungryPanda) optimize for popularity, convenience, and ad revenue — not authenticity. Three concrete gaps:

1. **Authenticity is invisible.** Asian-American foodies rely on folk heuristics ("3.5-star Yelp ≈ most authentic", "no English menu", "whole fish on the table", "cash only", native-language reviews). Nobody surfaces those signals.
2. **Cuisines get flattened.** "Chinese" hides Sichuan / Hunan / Cantonese / Hakka / Northeastern / Dongbei / Yunnan. Same for Japanese (Kansai vs Kanto), Korean (regional banchan styles), Thai (Isaan vs Bangkok), etc.
3. **You don't know what to order.** Walking into a regional place with a Chinese-character menu, even ABC/2nd-gen diners get stuck. Existing apps stop at "this restaurant is good."

## Differentiation — what nobody else does

| Capability | Yelp / Google | Fantuan / HungryPanda | The Infatuation / Eater | Beli | **Us** |
|---|---|---|---|---|---|
| Authenticity score from review mining | ❌ | ❌ | partial (editorial) | ❌ | ✅ |
| Regional sub-cuisine filter (Sichuan vs Cantonese vs …) | ❌ | ❌ | ❌ | ❌ | ✅ |
| Dish-level "what to order" | ❌ | ❌ | partial | ❌ | ✅ |
| Multi-source fusion (Reddit + Infatuation + Tabelog + Eater + Michelin) | ❌ | ❌ | ❌ | ❌ | ✅ |
| Zero install / API keys | ❌ | ❌ | ❌ | ❌ | ✅ (Claude WebSearch) |

## Format & sharing

- **Format:** Claude skill (`SKILL.md` + supporting prompts) in this repo.
- **Install:** `git clone` into `~/.claude/skills/local-asian-food-scout/` (or copy the dir). README has one-liner.
- **Distribution:** GitHub repo only for hackathon. Future: package as Claude plugin, add web UI.
- **Why a skill (not webapp / MCP / plugin):**
  - Zero infra, zero hosting, zero cost.
  - Claude provides WebSearch and WebFetch natively → no API keys → demo can't break on stage from a billing/quota issue.
  - Shareable artifact is a single directory in a Git repo — exactly what the hackathon prompt asked for.

## Architecture

```
User query  →  Claude (with skill loaded)  →  multi-source web search  →  evidence mining  →  ranked card

                ┌─ WebSearch: site:reddit.com/r/{sgv|FoodNYC|AskSF|AsianEats|Sichuan…}
                ├─ WebSearch: site:theinfatuation.com  (Asian-American guides)
                ├─ WebSearch: site:eater.com/maps
                ├─ WebSearch: site:tabelog.com         (Japan queries only)
                ├─ WebSearch: site:guide.michelin.com  (high-tier)
                └─ WebFetch:  top 1–2 promising URLs for richer prose
```

**Pipeline inside the skill:**

1. (Optional) one clarifying Q — "what kind of vibe / sub-cuisine / dietary?"
2. Run 3–5 parallel `WebSearch` calls across the source list above
3. `WebFetch` the most promising 1–2 URLs for full text
4. Claude reasons over collected evidence:
   - extract candidate restaurants
   - score authenticity (regional-keyword count, native-language review presence, folk-signal presence)
   - tag regional sub-cuisine
   - extract specific dish names mentioned approvingly
5. Return a structured Markdown card:
   - 3–5 picks ranked by authenticity score
   - each with: name, neighborhood, regional cuisine, why-authentic blurb, **3 dishes to order** (and 1 to skip if signal is strong)

## Data sources — research summary

Two research passes were done (general data sources, then ethnic/niche sources). Full notes below; here's the v0 pick list:

### v0 sources (zero API keys, accessible via Claude's WebSearch / WebFetch)

| Source | Why it's in | Use for |
|---|---|---|
| **Reddit** (r/sgv, r/FoodNYC, r/AskSF, r/AsianEats, r/Sichuan, …) | Best signal-per-effort. Authenticity-rich threads with native-speaker presence. Free tier survives for `site:` searches. | Authenticity signals, hidden gems, dish-level recs |
| **The Infatuation** | Public web, no paywall, explicit Asian-American/SGV/Chinatown guide pages. WebFetch confirmed working. | Curated picks, neighborhood scoping |
| **Tabelog** (Japan only) | Canonical Japanese-language scoring (3.5+ ≈ very good); pages render to WebFetch fine. | Japan / sushi / ramen origin checks |
| **Eater maps** | Strong city-level Asian heatmaps; reachable via WebSearch on Google index even if direct fetch is blocked. | City-level scope |
| **Michelin Guide / World's 50 Best Asia** | High-tier bookend; small N but tier-defining. | Upscale / tasting-menu picks |

### Sources researched and **skipped for v0** (auth / anti-bot / coverage gaps)

- **小红书 (Xiaohongshu / Rednote)** — login wall + monthly-rotating signed requests; would need paid scrapers (Apify $$) or vendor APIs ($20–50k/yr enterprise). Best authenticity signal in the world for Chinese diaspora picks, but inaccessible in 1h.
- **大众点评 (Dianping)** — strong inside China, weak US coverage, anti-bot.
- **AMap / Gaode** — Chinese ID + Alipay required for individual API access; weak US POI.
- **Beli** — closed app, social-graph gated. Strong Asian-American power-user concentration but no public surface.
- **Resy / OpenTable** — affiliate-only / reverse-engineered; "hard to book" is a noisy proxy anyway.
- **Naver / Kakao / Mangoplate (KR)** — auth friction; Mangoplate effectively zombie since Naver/Kakao ate the market.
- **Instagram / TikTok** — closed APIs in 2026, paid-scraper-only.
- **Yelp Fusion** — free tier killed in 2024.
- **Google Maps direct scraping** — CAPTCHA risk too high for live demo.
- **Google Places API (New)** — best structured data but needs CC + GCP project (10+ min setup; demo-fragility risk). Defer to v1.
- **Foursquare** — free tier collapses 10K → 500 calls on **2026-06-01**; usable today but not durable. Defer.
- **OSM Overpass** — free, no key, but `cuisine=*` tag coverage is patchy outside dense metros.

### Differentiator data trick

Most existing AI/restaurant tools use **one** structured API and call it a day. We're doing **multi-source web fusion** with Claude reasoning over the prose. That's what makes "authenticity score" possible — you can't compute it from a Yelp JSON blob; you compute it by *reading* the reviews and Reddit threads.

## Competitive landscape

A focused 2024–2026 search across Product Hunt, YC batches, Show HN, Devpost, VC announcements, and the Claude/MCP/plugin ecosystem turned up nobody shipping the exact combo. Verdict: **angle is unoccupied.**

### Closest adjacent products (none direct)

| Product | Year | What it does | Tackles authenticity? | Tackles regional? | Tackles dish-level? | Threat |
|---|---|---|---|---|---|---|
| **[Savor](https://www.savortheapp.com/)** | 2024 | Personal dish-rating log w/ AI dish recognition, 1–10 scale per dish | ❌ | ❌ | ✅ | Adjacent — closest dish-level player; if they add discovery + cuisine intelligence they converge |
| **BiteSight** (YC W24) | 2024 | TikTok-style food video discovery + delivery | ❌ | ❌ | ❌ | Not really — different paradigm |
| **MenuMystic / Dishlingo** | 2024–25 | Snap-a-menu translator / decoder | ❌ | ❌ | ✅ (decoder, not "what to order") | Adjacent — solves "what is this dish" not "where is it best" |
| **[Bites on ChatGPT](https://www.prnewswire.com/news-releases/bites-launches-on-chatgpt-unlocking-ai-powered-direct-ordering-for-restaurants-and-pos-platforms-302742535.html)** | Oct 2025 | AI-native ordering inside ChatGPT | ❌ | ❌ | weak | Adjacent — bigger structural threat is ChatGPT itself + OpenTable, not this product |
| **[Ontopo Restaurant Finder](https://mcpmarket.com/tools/skills/ontopo-restaurant-finder)** | 2025 | Claude Code skill for Israel restaurant search w/ availability + bilingual menus | ❌ | ❌ | ❌ | **Format precedent** — proves "restaurant finder as Claude skill on GitHub" pattern works; we own the Asian/authenticity twist |
| **Spoonhunt** | older | English-menu Chinese restaurants (mostly in-China use) | weak | partial | ❌ | Adjacent — geographically misaligned for US users |
| **The Infatuation** | older | Editorial guides incl. SGV / Asian-American | partial (editorial) | ❌ | partial | Adjacent — human-curated, not AI/agentic |
| **Beli** | 2021+ | Social ranking app, Asian-cuisine-heavy power-user base | ❌ | ❌ | ❌ | Closed app — biggest convergent threat if they add authenticity/regional filters |

### Searched-but-empty buckets (validates the gap)

- No "authenticity scoring" consumer app in US — Fantuan markets *on* authenticity but doesn't *score* it
- No regional sub-cuisine filter in any English-language app (Dianping/AMap have it; nobody else does)
- No Asian-American creator-led app conversion (Foodog, Tiffy Chen, Newt Nguyen all stay in social/blog format)
- No 2024–25 Devpost winner matching this thesis
- No Show HN / GitHub Claude skill for Asian-authenticity restaurant finding
- B2B restaurant-AI is hot ([Magic/Loyalist $10M Oct 2025](https://fortune.com/2025/10/20/mario-carbone-magic-loyalist-major-food-group-ai-restaurants-dining-crm/), Franzy $2.2M, Vox $8.7M) but it's CRM/voice/ops — not consumer discovery

### Validation: it's not just us

- **[Mochi Magazine](https://www.mochimag.com/lifestyle/food-rating-bias-asian-restaurants/)** documents Yelp's "authenticity trap" bias — 85% of "authenticity" mentions on non-white restaurants tied to "dirt floors / plastic stools" stereotype. Real problem; nobody's shipped a product that addresses it.
- **[Modern Restaurant Management](https://modernrestaurantmanagement.com/hyper-regional-authenticity-is-defining-asian-food-in-2026/)** trend piece: "Hyper-regional authenticity is defining Asian food in 2026" — consumer demand is there.

### Our defensible differentiation

1. **Explicit authenticity scoring methodology** (countering documented Yelp bias)
2. **Regional sub-cuisine taxonomy** (Sichuan/Cantonese/Hunan/Chiuchow/Hakka/Northeastern, not one "Chinese" bucket)
3. **Dish-anchored ranking** ("best dan dan noodles in SF" not "best Chinese restaurant")
4. **Open Claude-skill format** in a public GitHub repo (Ontopo precedent for non-Asian; we're first for Asian/authenticity)

## Scope (build budget: 1 hour)

### v0 must-ship

- [ ] `SKILL.md` with prompt + clear input/output spec
- [ ] One example invocation that works end-to-end on a real query (e.g., "authentic Sichuan in Manhattan")
- [ ] `README.md` with install instructions + sample transcript
- [ ] Public GitHub repo

### v0 non-goals (resist scope creep)

- ❌ Web UI / hosted version
- ❌ Real map embed
- ❌ Group features / lists / save-for-later
- ❌ Auth / accounts
- ❌ API key wiring (Google Places / Foursquare) — defer to v1
- ❌ Multi-turn conversation memory beyond what Claude does natively

## v1 ideas (post-hackathon backlog)

- Optional Google Places / Foursquare integration for structured cross-check
- Static webapp on Vercel (form → /api → same prompt) for non-Claude users
- Image-rendered map of picks
- "Save my picks" via local file (no auth)
- Plugin marketplace listing
- Locale: 中文 query → 中文 output (skill prompt is multilingual already if Claude handles it)

## Risks & mitigations

| Risk | Mitigation |
|---|---|
| WebSearch result variance run-to-run | Multi-query + Claude cross-check + ask for ≥3 sources per claim |
| Live WebSearch latency on stage (3–5 queries × 2–5s) | Pre-record a sample transcript as backup; consider showing one cached run alongside one live |
| Source pages blocked by WebFetch (Eater, Michelin) | Fall back to WebSearch text snippets — already verified workable |
| Claude hallucinating restaurants | Explicit "every claim must cite a URL" instruction in skill prompt |

## Build plan — 1-hour timer

| min | step |
|---|---|
| 0–10 | Repo skeleton: `SKILL.md`, `README.md`, `examples/` |
| 10–35 | Core skill prompt — authenticity scoring, regional tagging, dish extraction, source fusion |
| 35–45 | Test on 2–3 real queries; tune prompt |
| 45–55 | README polish: install steps, sample transcript, screenshot |
| 55–60 | Push, tag, demo prep |

## Open questions for the build

1. Skill should ask a clarifying Q first, or jump straight to results? (Lean: jump straight, but offer a "want me to refine?" line at the end.)
2. Card verbosity: one-screen tight, or detailed report? (Lean: tight default, with a "tell me more about #2" affordance.)
3. Hard-code the data source list, or let Claude pick dynamically per query? (Lean: hard-code for v0 — predictable demo.)

---

## Appendix — research notes

### A. General data source comparison (full pass)

| API | Free tier | Setup time | Asian cuisine quality | 1h verdict |
|---|---|---|---|---|
| Google Places (New) | 10K Essentials/mo + $300 trial | CC + GCP, ~10 min | Excellent — has `chinese_restaurant`, `japanese_restaurant`, etc. native types | Defer (setup time + demo fragility) |
| Yelp Fusion | 30-day trial only (no permanent free since 2024) | Yelp acct | Excellent | Skip (paywall risk) |
| Foursquare Places | 10K Pro/mo until **2026-06-01**, then 500/mo | Email, no CC | Good metros | Defer (cliff in 30 days) |
| OSM Overpass | Unlimited, no key | None | Patchy `cuisine=*` outside dense metros | Skip (coverage) |
| Mapbox / TomTom / HERE | Various; TomTom 2.5K/day no-CC easiest | 2–10 min | Decent POIs, weak cuisine subtype | Defer |
| Google Maps direct scrape | Free | None | Best | Skip — CAPTCHA on live demo |
| **Claude WebSearch + WebFetch** | Built-in | None | Variable, LLM parses prose | **Picked for v0** |

### B. Ethnic / niche data source pass

(Top 5 picked above; full table is in the build notes thread — short version: Xiaohongshu / Dianping / AMap / Beli / Naver / Kakao / Resy / Instagram / TikTok all skipped due to auth or anti-bot in 2026.)

### C. Folk authenticity heuristics (from Reddit / HN / community threads)

These map directly to features in our scoring prompt:

- **Yelp 3.5-star sweet spot** — 4.5+ often = Americanized crowd-pleaser, 3.5 ≈ authentic (gruff service, fish heads, no concessions to non-native palate)
- **No English menu / Chinese-character menu only** — strong authenticity signal
- **Cash only / cash discount** — older signal but still correlated
- **Whole fish served head-on** — Cantonese / Sichuan / Korean signal
- **Native-language reviews present** — strongest single signal
- **Regional keywords** — "麻辣" "豆瓣酱" "陈醋" (Sichuan); "煲仔饭" "烧鹅" (Cantonese); "锅包肉" (Northeastern); "kushikatsu" "okonomiyaki" (Osaka); "isaan" "som tum" (NE Thai)
- **Specific dish names referenced > restaurant referenced** — community knows the dish, not just the brand

### D. Reference: Claude skill format

A skill is a directory with at minimum a `SKILL.md` describing when/how to activate it. Other files (prompts, scripts, examples) sit alongside. Distribution is "copy the directory into `~/.claude/skills/`".
