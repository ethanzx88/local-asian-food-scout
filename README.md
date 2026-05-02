# 🍜 Local Asian Food Scout

> Authenticity-aware Asian restaurant finder, packaged as a Claude skill. 中英夹杂 output for the Chinese-American immigrant foodie crowd.

Most apps tell you "this Chinese place is 4.5 stars." That's not what you actually want to know. This skill answers the questions you really have:

- **Is it authentic, or Americanized?** (scored, with signals)
- **Sichuan vs Cantonese vs Hunan vs Northeastern?** (regional sub-cuisine, not one bucket)
- **What should I order here?** (dish-level recs, not just the restaurant)

It does this by reading multiple sources — Reddit (`r/sgv`, `r/FoodNYC`, `r/AskSF`, `r/AsianEats`, native-language keyword searches), The Infatuation, Eater, Tabelog (Japan), Michelin Guide — and fusing the evidence with Claude's reasoning.

## Install

Copy the `SKILL.md` (and this directory) into your Claude skills folder:

```bash
mkdir -p ~/.claude/skills/local-asian-food-scout
cd ~/.claude/skills/local-asian-food-scout

# Option A: download just the SKILL.md
curl -O https://raw.githubusercontent.com/ethanzx88/local-asian-food-scout/main/SKILL.md

# Option B (when local git/xcode tools are set up): clone
# git clone https://github.com/ethanzx88/local-asian-food-scout.git .
```

That's it. Restart Claude (or open a new conversation), and the skill activates on Asian food queries.

## Use it

Just ask, in any language / style:

- `find authentic Sichuan in Manhattan`
- `我在洛杉矶想吃东北菜，求推荐 hidden gems`
- `best dim sum in SF that locals actually go to`
- `where to get good 港式茶餐厅 cha chaan teng in NYC`
- `omakase under $100 in Brooklyn`
- `authentic 担担面 anywhere in Chicago?`

The skill activates automatically on Asian food / restaurant queries (English or 中文).

## Sample output

See [`examples/sample-output.md`](examples/sample-output.md) for a real run on "authentic Sichuan in Manhattan."

## How it works

See [PLAN.md](PLAN.md) for full thesis, research, and architecture. TL;DR pipeline:

1. **Parse** — location + cuisine + vibe from query
2. **Multi-source search** — 3–5 parallel `WebSearch` calls (Reddit, Infatuation, Eater, Tabelog, Michelin)
3. **Deepen** — `WebFetch` top 1–2 promising pages for richer prose
4. **Mine signals** — regional keywords, native-language reviews, "no English menu", 3.5-star sweet spot, owner-from-region, etc.
5. **Score & rank** — authenticity × confidence
6. **Extract dishes** — 3 must-order, sometimes 1 别点 skip
7. **Format** — 中英夹杂 bilingual card with sources

## Why a Claude skill?

- Zero install friction for anyone using Claude
- No API keys (uses Claude's built-in `WebSearch` / `WebFetch`)
- Shareable as a single Git repo — drop into `~/.claude/skills/`
- Differentiation lives in the prompt — easy to fork, tune, contribute

## Limitations (v0)

- Quality scales with public web data — sparse for small US cities
- `WebSearch` results vary run-to-run (we cross-check, but not deterministic)
- No Google Places / Foursquare API integration yet — addresses / hours / phone aren't always exact
- Closed sources (小红书 Xiaohongshu, Beli, Dianping for US) are not yet integrated. See [PLAN.md](PLAN.md) for the full data-source research notes including why these were skipped for v0.

## Roadmap

- [ ] Optional Google Places API integration for structured fallback
- [ ] Static webapp on Vercel for non-Claude users
- [ ] Map-image generation in output
- [ ] More ethnic data sources as access becomes available (Xiaohongshu via authorized vendors, Tabelog deeper integration)

## Hackathon

Built in <1 hour as a hackathon project. See [PLAN.md](PLAN.md) for the full rationale, competitive analysis (no, no one else has shipped this exact angle yet), and data-source research notes.
