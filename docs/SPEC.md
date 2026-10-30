# Gale Force - Architecture and Scoping Specification

**Created:** 2026-10-29
**Status:** Draft v1 - design only; nothing implemented or verified
**Platform dependency:** [hyperreal-platform](https://github.com/PCSchmidt/hyperreal-platform) (HRCore, HRScenario, HRReplay, HRAtmosphere, HROcean, HRFlow, HRGeo, HRRating, HRNet)
**Companion documents:** [REQUIREMENTS.md](./REQUIREMENTS.md) (user stories, FR IDs, session walkthrough) | [ML.md](./ML.md) | [AGENTS.md](./AGENTS.md) | [SPIKES.md](./SPIKES.md) | [PRIOR_ART.md](./PRIOR_ART.md)

> `[VERIFY]` marks assumptions not yet checked against the engine, a license or measurement.

---

## 1. Vision

**Pitch.** Competitive, physically grounded sailboat racing with a photoreal ocean. Boat speed comes from honest aerodynamics and hydrodynamics; tactics come from real wind shifts, puffs, current and the racing rules.

**Player fantasy.** Win the start, find the favoured side of the course, hold a lane in a header, round the mark with a rival on your hip, and feel the boat accelerate in a puff - rendered so well that a replay looks like broadcast footage.

**Pillars**
1. **Wind is the opponent.** Shifts, gusts and geographic effects reward reading, not memorizing.
2. **Boat feel is physical.** Heel, weather helm, leeway, stall and momentum loss in a tack are consequences of the model, not animation.
3. **Rules create tactics.** Right-of-way and mark-room are enforced by an automated umpire, producing real covering, luffing, port-starboard and penalty decisions.
4. **Spectacle is data-honest.** Spray, wake and sail shape come from the same state the sim uses.
5. **Accessible depth.** An assisted mode makes the sport approachable; the unassisted mode has a very high ceiling.

**Non-goals (v1):** offshore crew simulation, below-deck systems, multihull and foiling classes (post-v1), photoreal individual crew animation beyond helm and hiking poses, ocean-scale (multi-day) real-time hosting beyond a defined mode, damage/collision destruction.

---

## 2. Competitive design

### 2.1 Formats

| Format | Description | Netcode |
|---|---|---|
| **Time trial / Beat the ghost** | Solo, fixed course and conditions; compete against recorded ghosts | None |
| **Match race (1v1)** | Two boats, pre-start dance, rules enforced, best-of series | Live (phase 2) or async ghost match |
| **Fleet race** | 6-30 boats, windward-leeward and triangle courses, starts with line bias | Live (phase 2) |
| **Daily Course** | Seeded wind + course + class; ranked by finish time and rule penalties | None |
| **Regatta series** | Multi-race series with a discard, points scoring | None / Live |
| **Offshore Real-Time** | Long race using real forecast data (compressed timeline option); routing decisions | None (async) |
| **Team racing** | 3v3 with team scoring (later) | Live |

### 2.2 Scoring
- Fleet: low-point system (position = points), with penalties (turn penalties or disqualification-lite).
- Match race: win/loss with penalty system.
- Time trial: elapsed time with penalty seconds for rule infractions.
- Ratings: Glicko-2 / TrueSkill by format and boat class.

### 2.3 Rules engine (the umpire)
Automate a documented subset of the Racing Rules of Sailing (World Sailing; current edition `[VERIFY]`). The rule *text* is not copied; the logic is independently implemented with descriptive names. Implemented first:
1. Port/starboard.
2. Windward/leeward when overlapped on the same tack.
3. Clear astern/clear ahead and overlap determination.
4. Mark-room at the three-boat-length zone.
5. Starting rules (OCS, general recall behavior) and the start line.
6. Tacking/gybing and "keep clear".
7. Contact avoidance and penalty turns (one-turn / two-turn).
8. Obstruction rules where the course includes fixed obstructions.

Edge cases (proper course, luffing rights nuances, redress) are deferred or approximated; the umpire logs every decision with the geometric facts, so rulings are explainable in replay. A "protest-free" assisted mode applies automatic penalties; an advanced mode permits player-initiated flags with an umpire panel (post-v1).

---

## 3. Simulation architecture

### 3.1 State and step
- Fixed step **5-10 ms** (Spike S1 decides) for boat dynamics; 20-30 Hz network snapshots.
- 6 degrees of freedom for the hull: surge, sway, heave, roll, pitch, yaw. Heave/pitch are small (waves) but required for realistic bow-slam and speed loss in chop.
- Rigid-body + force-model design, **independent of Chaos**. A custom integrator (semi-implicit or RK4) in `double`.

### 3.2 Force model

| Force | Model | Inputs |
|---|---|---|
| **Aerodynamic - sails** | Lift/drag from coefficient curves `CL(α)`, `CD(α)` per sail (main, jib/genoa, spinnaker/gennaker) as functions of angle of attack, twist, camber and trim state; applied at a center of effort that moves with trim and heel | Apparent wind vector (true wind − boat velocity), sail trim (sheet, traveller, vang, outhaul, cunningham, halyard tension), heel |
| **Mast/rig/crew windage** | Quadratic drag | Apparent wind |
| **Hydro - hull resistance** | Frictional (ITTC-57 line) + residuary resistance. **Source depends on class:** the Delft Systematic Yacht Hull Series regressions apply to displacement **keel yachts** and are suitable for the keelboat/cruiser classes only. Planing and semi-planing **dinghies** need a different basis: published planing-hull resistance methods and/or measured performance data used to fit a speed-dependent resistance curve, with the planing transition modelled explicitly `[VERIFY sources before S1]`. Heel- and trim-dependent wetted surface in all cases | Boat speed, heel, trim |
| **Hydro - appendages** | Keel/centerboard and rudder as lifting foils: `CL(α)`, `CD(α)`, stall, induced drag via aspect ratio; ventilation/cavitation ignored (v1) | Local flow velocity and leeway angle |
| **Added resistance in waves** | Speed loss as a function of significant wave height and heading; slam penalties in short steep chop | Wave field (HROcean sampler) |
| **Righting moment** | Hull form stability curve `GZ(φ)` + crew hiking (ballast moment) + water ballast/keel | Heel angle, crew position |
| **Damping** | Roll damping from appendages and bilges | Roll rate |
| **Added mass** | Constant matrices per axis | Yaw and sway accelerations |
| **Current** | Hydrodynamic velocity = boat − current; HRFlow sample | Tide/river/eddy field |

### 3.3 Sail shape
- **Physics:** a parametric model (twist, camber, draft position, luff tension) yields effective lift/drag. A lookup informed by public data and optionally calibrated against a vortex-lattice (VLM) or panel-method tool offline `[VERIFY approach]` - offline tooling is a one-time dataset generation step, not runtime CFD.
- **Rendering:** a cosmetic cloth sim or morph-target blend driven by the same parameters; luffing occurs when `α` falls below the stall-low threshold, with flogging sound.

### 3.4 Control model
- **Helm:** rudder angle with rate limits and weather-helm feedback; tiller extension / wheel options.
- **Trim:** mainsheet, traveller, jib sheet, vang, outhaul, cunningham, backstay (class-dependent).
- **Crew:** hiking (dinghy), crew-weight placement (keelboat), roll tacks/gybes with timing minigame (skill); crew actions have physical effects (roll-tack energy).
- **Assist levels:** (A) Cadet: auto-trim and helm assist; (B) Club: auto-trim only; (C) Pro: manual helm and trim; (D) Expert: manual with no performance display. The leaderboard is separated by assist level.
- **Instruments:** apparent/true wind (speed, angle), boat speed, VMG, heading, laylines, heel angle, polars (optional); Expert mode hides most.

### 3.5 Wind
- **True wind field** (HRAtmosphere): mean speed/direction by scenario, oscillations (shifts with period 3-15 min), persistent shifts, gust field (spatially coherent puffs advected downwind, with a lull tail), shear with height (boundary layer), thermal sea breeze models (building through the day), and geographic effects (headland acceleration, lee shadows, wind bend near coast).
- **Real weather mode:** initialize from a weather model snapshot (NOAA GFS/HRRR or ECMWF open data `[VERIFY terms]`) at a chosen place/time and downscale with a statistical turbulence layer.
- **Wake and blanket-air effects:** a boat to windward of another casts wind shadow and bad air; modelled as a wake velocity deficit field behind sails - the basis of covering tactics.

### 3.6 Ocean
- **Wave model:** spectral (JONSWAP or Pierson-Moskowitz with directional spreading) with local swell, wind-sea and fetch effects. The **CPU sampler** (for physics) and **GPU renderer** (for visuals) use the same spectrum so buoyancy and foam agree (HROcean).
- **Choppy regimes:** short-wavelength steep chop from the wind; shallow-water transformation near shore (refraction/shoaling approximated).
- **Current effects:** tide streams and eddies from HRFlow; chop opposing a current steepens waves.
- **Visual:** foam/whitecaps (breaking criterion), spray and bow wave (Niagara driven by hull speed and wave heights), wake persistence on the surface, subsurface scattering and refraction from above, reflections of sails and sky.

### 3.7 Determinism
Fixed step, `double`, deterministic ocean sampler, counter-based RNG for gust generation, strict FP. Rendering-only systems (foam, spray) may use independent randomness.

---

## 4. Boat classes and progression

| Order | Class (generic) | Why | Notes |
|---|---|---|---|
| 1 | **Single-handed dinghy** (ILCA-like, generic name) | Simple rig, huge competitive base, rich tactics | Hiking, roll tacks, single sail; use generic naming to avoid trademark issues |
| 2 | **Two-person planing dinghy** (470-/Flying Junior-like) | Spinnaker and crew work | Adds spinnaker aerodynamics, trapeze |
| 3 | **Sportsboat/keelboat** (J/70-like) | Keelboat stability, match racing | Crew-weight placement |
| 4 | **Cruiser-racer** (offshore) | Offshore mode | Sail inventory and routing |
| Post-v1 | **Foiling monohull/catamaran** | Spectacle (AC-type) | Requires foil lift/ride-height control and stability; large new physics |

Each class ships with: hull parameters (public design data or own estimates), sail coefficient set, appendage data, stability curve, polars derived from the model for validation.

---

## 5. Courses and venues

- **Course types:** windward-leeward (with gates), triangle (Olympic), trapezoid, coastal/pursuit, offshore waypoints.
- **Course builder:** deterministic generation with wind direction, laylines and mark-boat placement; the RC (race committee) picks the line with bias.
- **Venues:** Lake/inland (thermal), coastal bay (sea breeze, current), open ocean (swell), archipelago (shifty), river estuary (tide). Venue data from bathymetry (GEBCO/NOAA), shoreline (OSM/Natural Earth), land elevation (DEM) for wind shadow, and tide/current data (NOAA CO-OPS, HYCOM) `[VERIFY terms]`.
- **Scenarios:** time-of-day, cloud cover, haze, season, and weather presets. Daily seeds use seeded wind scripts for fairness.

---

## 6. Rendering and presentation

| Element | Approach |
|---|---|
| Ocean | HROcean: FFT/Gerstner-hybrid surface from a spectrum; tessellated near camera; foam/whitecaps; subsurface scattering; depth-based color; reflections via Lumen/SSR/RT |
| Boat | Nanite hull and rig, procedural livery, physically based materials (gelcoat, carbon, Dacron/laminate sails) |
| Sails | Cloth/morph targets from trim parameters; translucent laminate with seam detail; sunlight transmission |
| Weather | Volumetric clouds, haze, rain, squall lines |
| Spray/wake | Niagara fed by relative speed and wave interaction |
| Cameras | Cockpit, helm, mast-top, drone/follow, broadcast director (automatic framing of tactical moments) |
| UI | Diegetic where possible; instrument displays, tactical overlay (laylines, wind arrows - mode-dependent), post-race analytics |
| Replay | HRReplay-driven with free camera and telemetry overlays (VMG, wind shift graph, rule events) |

**Frame budget:** ≥ 60 FPS at 1080p on the Reference tier with the full ocean; Dev laptop fallback tier with reduced spray and cheaper reflections `[VERIFY S4]`.

---

## 7. Audio
Wind in rigging, water hiss at speed scaling with relative velocity, slap on the hull in chop, sail flog when luffing, sheet creak under load, crew calls, horns and flags at starts, distant boats; spatialized, environment-correct.

---

## 8. Networking

- **Authoritative server** simulates all boats; clients send control inputs (helm, trim) at 30-60 Hz; server snapshots at 20 Hz; interpolation for remote boats; own boat predicted.
- Boats are slow (< ~12 m/s in v1), so latency up to ~150 ms is tolerable for fleet racing; close-quarters match racing is more sensitive: the server resolves rules at its tick and the client displays predicted boat plus authoritative overrides.
- **Async first:** time trials, daily courses and "ghost match" (race the replay of a real opponent). This delivers competition with no live infrastructure.
- **Scaling:** 30 boats × 20 Hz × compact state is small; bandwidth is not the main risk. Deterministic wind/waves are generated from the seed on both ends and not replicated.
- **Anti-cheat:** server re-simulation of inputs; bounds on control rates; telemetry screening.

---

## 9. AI and ML

| Use | Approach |
|---|---|
| Ghost fleets | Replay recordings |
| Scripted opponents | Polar-following with rule-aware heuristics and tactical rules (cover, tack on a lift, take the lane) by difficulty |
| Learned sailors | Imitation learning from replays, then self-play RL for tactics (start timing, layline calls, covering) on the headless sim |
| Wind-reading coach | A trained assistant that flags shifts and puffs and explains tactical options (post-race) |
| Rating calibration | Large bot tournaments to calibrate Glicko-2 |
| Opponent diversity | Style parameters (aggressive, conservative) to avoid homogeneous fleets |
| Evaluation | Bot-vs-bot Elo; human-vs-bot results; check for exploits (e.g. circling start line) |

---

## 10. Technical architecture

```
GaleForce (game module)
 ├─ GFBoat          6-DOF dynamics, force model, class parameter sets
 ├─ GFAero          sail coefficients, trim state, wind shadow/blanketing
 ├─ GFHydro         resistance, appendages, wave interaction, current
 ├─ GFRules         umpire: geometry queries, right-of-way, penalties, log
 ├─ GFCourse        course generation, marks, line/gate logic, laylines
 ├─ GFRace          starts, finishes, scoring, series
 ├─ GFAI            scripted + learned sailors
 ├─ GFRender        sail/rig visuals, spray/wake drivers
 └─ GFUI            instruments, tactical overlay, analytics
```
Platform: HRCore, HRScenario, HRReplay, HRAtmosphere, HROcean, HRFlow, HRGeo, HRNet, HRRating, HRTelemetry, HRAgents.

**Data flow:** Scenario (seed, venue, class, weather) → wind/wave/current fields → per-tick boat dynamics → rules engine → race state → HRReplay; render reads the sim state only.

**Tooling:** a Python/Jupyter polar-generator that runs the model headless to output polars for validation; offline panel/VLM calibration scripts; scenario previewer.

---

## 11. Data and licensing
- Weather: NOAA GFS/HRRR (public domain), ECMWF open data and ERA5 (Copernicus license) `[VERIFY]`.
- Bathymetry: GEBCO, NOAA NCEI; tides: NOAA CO-OPS; coastlines: OSM (ODbL) and Natural Earth (public domain).
- Hull/sail data: public design literature (Delft series etc.); do not copy proprietary velocity-prediction-program data.
- Rules: implement logic; do not reproduce rule text; credit World Sailing as the rules source and confirm any permitted use `[VERIFY]`.
- Class names: use generic names (no trademarks).
- Assets/audio: registered in the manifest.

---

## 12. Verification strategy
See [SPIKES.md](./SPIKES.md). Summary:

| Layer | Evidence |
|---|---|
| Boat physics | Model-generated polars within a stated tolerance of published polars/performance data for the class (e.g. ≤ 5% boat speed across TWA/TWS grid) |
| Rules | A suite of >= 200 hand-labeled geometry cases with correct rulings; 100% pass for implemented rules |
| Determinism | 1,000 fleet races re-simulated: zero hash mismatches |
| Ocean consistency | Floating probe follows rendered wave surface within 5 cm RMS |
| Performance | Frame-time percentiles for 1, 10 and 30 boats |
| Competitive depth | Bot ladder shows consistent skill ordering; gust/shift strategy measurably affects finishing position |

---

## 13. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Sail aerodynamics fidelity vs. effort | Medium | High | Coefficient-curve model calibrated to polars; no runtime CFD |
| Rules engine complexity | High | Medium | Staged subset; logged explainable rulings; golden test suite |
| Ocean visual quality vs. performance | Medium | High | HROcean spikes; tiered quality; frame budget gates |
| "Feels like a boat" is subjective | High | High | Early tuning spikes with owner and (if possible) sailing-literate testers; record honestly |
| Determinism with floating-point and waves | Medium | High | Deterministic sampler, strict FP, S2 |
| Close-quarters live netcode | Medium | Medium | Async-first; live later |
| Weather data pipeline fragility | Medium | Low | Cache snapshots by hash; offline scenarios |
| Foiling expectation (AC hype) | Medium | Low | Post-v1 |
| Hyper-real ocean art demands | High | Medium | Procedural and data-driven; limited hero boats |

---

## 14. Milestones (gated)

| Gate | Deliverable | Exit evidence |
|---|---|---|
| G-0 | Spikes S1-S3 | Polars within tolerance; rules golden cases; ocean consistency |
| G-1 | Single-handed dinghy time trial | Solo sailing playable; replay; ghost |
| G-2 | Rules engine v1 + AI fleet | Fleet race with umpire; scripted bots |
| G-3 | Visual vertical slice | Hero venue, cameras, spray/wake, broadcast replay |
| G-4 | Verified async daily course | Server re-sim leaderboard |
| G-5 | Learned sailors + calibration | Rating tables and exploit review |
| G-6 | Second class (planing dinghy) + spinnaker | Class reuse test |
| G-7 | Live match race (optional) | Two-client desync tests |

---

## 15. Open decisions

| Decision | Default | Resolve by |
|---|---|---|
| First class | Single-handed dinghy | G-0 |
| Foiling | Post-v1 | After G-5 |
| Real-weather mode in v1 | Included as seeded snapshots (offline), live fetch later | G-3 |
| Series/regatta framing | Season ladder with named venues | G-5 |
| Umpire protest mechanics | Auto-penalty only in v1 | G-2 |
| VR | Out of scope | Later |
