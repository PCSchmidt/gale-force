# Gale Force - Prior Art and Differentiation

**Status:** Draft v1. Limited web survey on 2026-10-30; much of it from publisher and press material. Needs a deeper pass before any novelty claim.

**Correction to earlier planning notes:** the plan cited Virtual Regatta as proof of demand. That stands, but it also means a **large incumbent already owns online sailing competition** in a different segment (strategy and routing, not physical boat handling).

## 1. Existing products found
| Product | Notes |
|---|---|
| [Virtual Regatta (Offshore)](https://apps.apple.com/app/id387893495) | Free-to-play offshore routing/strategy game with real-weather data; official game of major events; publisher claims community of over three million players ([CYCA announcement](https://www.mysailing.com.au/cyca-partners-with-virtual-regatta-for-gold-coast-race/)); optional paid VIP tier |
| [Virtual Regatta Inshore](https://apps.apple.com/us/app/id1182301199) | Inshore fleet racing on mobile; realism claims are publisher and review text |
| [IMOCA Virtual Series (2026)](https://www.imoca.org/en/news/news/imoca-virtual-series-imoca-and-virtual-regatta-launch-the-2026-season-of-the-official-offshore-virtual-racing-championship) | Official virtual championship alongside a real fleet |
| Sailaway, Sailing Simulator and other PC titles | Not covered by this survey; investigate separately |
| Real sailing-simulation research tools | Not surveyed |

## 2. Takeaways
- Demand for real-weather and competitive virtual sailing is proven at scale, but incumbents emphasise routing and mobile accessibility.
- A high-fidelity, physically driven, **boat-handling** racing sim with a rules umpire, photoreal ocean and verified async competition is a different offering; its audience size is unknown.
- Real-weather import (public forecast data) is a common expectation; the open data sources named in the spec make it feasible without paid services `[VERIFY terms]`.

## 3. Intended differentiation (to be validated)
1. Physical boat model validated against published polars, with an explicit calibration and uncertainty story.
2. A racing-rules engine with explainable rulings and a tested golden suite.
3. Photoreal ocean whose visuals and physics share one wave spectrum.
4. Deterministic, replay-verified competition and bot-calibrated ratings.
5. Learned tactical sailors and an optional grounded coach, with published evaluation.

## 4. Open research tasks
- Survey PC sailing sims (physics, rules, multiplayer) and their reviews.
- Review published sailing-performance data for the first class (dinghy) and the planing-hull resistance literature (see spec correction).
- Review World Sailing licensing for rules usage and class-name trademarks.
- Survey existing sailing RL research for baselines.
