# Gale Force - Agent Layer (secondary scope)

**Status:** Draft v1 - design only. Reuses the Dark Fleet agent harness (`HRAgentKit`) and rules ([Dark Fleet AGENTS.md](https://github.com/PCSchmidt/dark-fleet/blob/main/docs/AGENTS.md)): agents use the same typed, role-limited API as a human; outputs are grounded by a deterministic checker; the game is fully playable without them.

## 1. Agents
| Agent | Mode | Tools | Output |
|---|---|---|---|
| **Race Coach** | Post-race | `get_timeline`, `get_wind_history`, `get_boat_state`, `list_rule_events`, `compare_to_optimal_path` | A review of key decisions (tacks, layline calls, starts, rule events) with timestamps; each statement cites event IDs |
| **Rules Explainer** | Any time | `get_ruling`, `get_geometry_at(t)` | Plain-language explanation of a ruling from the logged geometric facts; never overrides the engine |
| **Tactician** (practice only) | Optional hints | Same observations as the player | Suggests a tack/lane choice with uncertainty; labelled as hints, disabled in ranked divisions |
| **Crew chatter** (cosmetic) | Any | Structured race state in; text out | Short in-character calls constrained to true facts |

## 2. Evaluation
- *Grounding/faithfulness:* deterministic checker verifies that each claim's cited event IDs exist and support it (position change, rule event, wind shift).
- *Rules Explainer accuracy:* agreement with the engine's logged ruling on a labelled set (target: 100% consistent; any contradiction is a bug in the explanation layer).
- *Tactician utility:* effect on a bot sailor's finishing position vs unaided and vs the learned policy; cost and latency.
- *Ablation:* with/without tools; local vs hosted model.
A small benchmark (GaleForce-Bench) reports these with confidence intervals on fixed seeds.

## 3. Security and risks
Rules-based boundaries: the engine is authoritative; agent text is advisory. Untrusted text fields (venue names, player names) are handled as in Dark Fleet. Risk: agents misexplaining rules; mitigated by grounding to the logged facts and by the contradiction check.
