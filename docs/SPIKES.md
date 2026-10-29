# Gale Force - Verification Spike Ladder

**Created:** 2026-10-29 | **Status:** Not started

| ID | Question | Build | Pass criteria | If it fails |
|---|---|---|---|---|
| **S1** | Does the force model produce credible boat speeds? | Headless 3-DOF then 6-DOF model for the dinghy class; generate polars over TWS 4-20 kt, TWA 30-180° | Within ±5% of published class performance/polar data across the grid; upwind VMG angle 40-48° TWA in moderate air; plausible hull-speed limits | Recalibrate CL/CD, resistance; widen tolerance only with evidence |
| **S2** | Is the sim deterministic with waves, wind and AI? | 1,000 fleet races replayed from logs; thread-count variation; strict FP | Zero state-hash mismatches | Pin math; isolate non-determinism; verify on identical build only |
| **S3** | Does the visual ocean agree with the physics sampler? | Floating probes vs rendered surface across sea states | RMS height error ≤ 5 cm for Hs ≤ 1 m; foam onset consistent with breaking criterion | Share spectrum parameters; fix phase/time sync |
| **S4** | Can the ocean + boats hold frame budget? | Benchmark with 1/10/30 boats, spray, reflections | Reference tier ≥ 60 FPS 1080p, 1% low reported; Dev laptop fallback tier ≥ 45 FPS | Reduce tessellation/foam resolution; simpler reflections; LOD boats |
| **S5** | Does the rules engine rule correctly? | 200+ labeled geometry cases (port/starboard, overlap, mark-room, OCS) | 100% match on the implemented rule set; every ruling logged with facts | Fix geometry predicates; narrow rule scope |
| **S6** | Do tacks/gybes feel and behave physically? | Tack/gybe sequences with timing; roll-tack energy | Speed loss in a well-sailed tack 10-25% and recovery time plausible; poorly timed tack loses more; documented comparison to published sailing guidance | Tune momentum/added mass and foil stall |
| **S7** | Do wind shifts and puffs create tactical value? | Bots with and without shift awareness over seeded oscillating wind | Shift-aware bots finish ahead with statistical significance (p<0.05 over 500 races) | Increase shift magnitude/period variety; retune |
| **S8** | Does wake/blanketing create covering tactics? | Wake deficit field behind sails | Measurable speed loss in the wind shadow within the defined zone; covering bots beat non-covering bots in matches | Adjust the shadow footprint and decay |
| **S9** | Can async verification reject cheating? | Tampered inputs (impossible trim rates, teleports) | 100% rejected; 0% false rejects on 1,000 legitimate runs | Tighten bounds |
| **S10** | Are the assist levels coherent? | Compare times Cadet→Expert | Monotonic improvement of skilled sailor times with less assistance, and novice lower variance with more | Revisit assist effects |
| **S11** | Can learned sailors be competitive? | Imitation + RL on the headless sim | Learned policy beats scripted bot in ≥ 60% of fleet races; no exploit identified in review | Improve observations/rewards; keep scripted |

## Order
S1 → S2 → S3 → S4 (in parallel with S5) → S6 → S7/S8 → S9/S10 → S11.

## Evidence format
`docs/spike-results/Sx.md` with commit, build, data hashes, plots, verdict, and date. Retain failed attempts.
