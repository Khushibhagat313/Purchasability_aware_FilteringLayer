# Purchasability-Aware Recommendation Layer

**Status: design proposal, not yet built. Feedback from practitioners is very welcome.**

When a shopper filters in search and then opens a product, the recommendation carousel below it often ignores those filters: items in the wrong size, out of stock, or far over budget, or sale badges that don't apply to the shopper's size. This repo proposes a small layer that carries the shopper's filters from search into the recommendation step, so everything shown is something they can actually buy.

This README summarises the design. Longer design notes will go in [`docs/`](docs/).

## The problem

Five related problems:

| | Problem |
|---|---|
| P1 | Filter context is lost between search and recommendations |
| P2 | The user's profile is ignored |
| P3 | Trust decays when shoppers see items they can't buy (a consequence of P1 and P2) |
| P4 | The profile becomes a ceiling and discovery narrows |
| P5 | The same items are shown repeatedly |

The claim is deliberately narrow: **filter context is not always propagated from search into recommendation surfaces.** The claim is *not* that platforms ignore purchasability in general. It comes from my own experience as a shopper and is not yet validated at scale; the evaluation plan below is how I intend to test it.

## The idea

A layer that sits after the recommender and before display. The recommender is untouched, so the change is revertible and A/B testable. It needs one extra field in the request:

```
POST /recommend
{ user_id, product_id, session_filters: { size, max_price } }
```

## Treat constraints differently

Only stock and size are facts. Everything else is a preference the system can be wrong about, and being wrong there is recoverable and measurable.

| Constraint | Behaviour |
|---|---|
| Stock | **Hard wall.** Genuinely unbuyable items are never shown. |
| Size | **Steep slope.** Vanity sizing is real, so any tolerance is learned per user and category later; v1 treats size as near-hard. |
| Price | **Gentle slope.** Over budget is a decision, never a wall, but stretches are small, labelled and capped. |
| Style | **Explorable.** Aim for a novelty band instead of maximum similarity. |
| Category | **Explorable.** Same occasion, different garment. |

## The 10 slots

| Slots | Type | What |
|---|---|---|
| 5 | Matched | In band, familiar, same category. Capped at 1.2x the stated filter. |
| 3 | Stretch | Same category, one deviation (price or style, never both). Capped at 1.75x the profile and 1.5x the session filter. |
| 2 | Adjacent | Different category, same occasion. Conservative on everything else. |

Example: a filter of INR 2,200 gives matched items up to INR 2,640, stretch items up to INR 3,300, and nothing above. Two rules: one novelty per slot, and label every deviation ("also for weddings" turns a mistake into a suggestion).

## Three tools, three jobs

- **Plain code** for size, budget and brands. Exact answers exist; an LLM here would be slower, costlier and non-deterministic.
- **An LLM, offline, batched and cached,** for style and occasion tags from product text. No formula exists for that.
- **Embeddings** for style matching and novelty bands, because discrete tags can't express "adjacent but not identical".

Code computes numbers, the LLM extracts meaning, embeddings compare meaning at speed.

## Byproducts

The same data can surface: inventory gaps (for example, a size band that is sold out in most sessions), the share of sale ads where the shopper's size isn't discounted, demand by occasion, and ad spend wasted on items the shopper can't buy.

## Scope, honestly

- **Addressed:** P1 and P2.
- **Indirect:** P3, because it follows from P1 and P2. This is argued, not measured.
- **Partial:** P4 and P5. Exploration mitigates them; a full solution needs live traffic.
- **Research territory:** bandits and diversity-aware re-ranking.

**Known conflict:** fixing P1 and P2 can worsen P4 and P5 (filter to what you can buy, show more of what you already buy, and the loop tightens). The resolution proposed here is that exploration runs on price, style and category while buyability stays fixed.

## Limitations

- Synthetic data would validate the pipeline, not business impact.
- All parameters (caps, bands) are placeholders to be set by real data and swept for sensitivity.
- Adjacent sizes can increase returns, which are expensive in Indian fashion.
- Showing items above a user's explicit price filter may conflict with the filter itself; this needs testing, and may need to be opt-in.
- Marketplaces and aggregators may not have real-time size-level stock, so the design needs an "unknown stock" state with confidence weighting.

## Proposed evaluation

1. An audit of about 100 shopping sessions across 3 apps, measuring how often recommendations ignore the filters set.
2. Offline metrics against a baseline recommender: survival rate (how much of the baseline top-K remains after the constraints), relevance retained (nDCG vs baseline), coverage and novelty.
3. A sensitivity sweep on the caps and novelty bands.
4. A hand-labelled sample to check the LLM style and occasion tags.

## Roadmap

1. A catalogue with size-level stock built on a public fashion dataset, plus a baseline recommender.
2. The layer and the `/recommend` endpoint.
3. The evaluation harness and sensitivity sweeps.
4. A public write-up of results, including where the layer hurts.

## Feedback wanted

If you work on recommendations, search or fashion e-commerce, I'd value your view on:

1. How do real platforms carry filters into recommendations today, and where does it break down?
2. Is a labelled price stretch above an explicit max-price filter acceptable, or a trust risk?
3. How would you learn size tolerance without returns data?
4. How would you evaluate this without live traffic?
5. How should the layer behave when stock data is stale or unavailable?

Please open an issue or a discussion.

## About

By Khushi, an AI engineer. Related work: [HireView](https://github.com/Khushibhagat313/HireView-V2), a live explainable candidate-ranking system ([demo](https://hireview-ai.vercel.app)), and ongoing research on prompt-injection defences for tool-calling agents.
