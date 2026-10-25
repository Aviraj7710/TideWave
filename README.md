# Tidewatch

An AI agent that works as an always-on maritime analyst for protected areas. When a fishing vessel switches off its transponder near the Galápagos Marine Reserve, Tidewatch scores the risk, gathers evidence from satellites and drones, sends the fastest available patrol boat, and leaves only the boarding decision to a human.

Built for the Invisible Interfaces challenge, Dispatch track.

## Run it

Open `index.html` in a browser. No install or backend needed. Press **New night** for a fresh random scenario, or open a specific one by adding a seed to the URL, for example `index.html#n22222` (vessel caught) or `index.html#n10003` (research ship identified, patrol stood down).

## The problem

Fishing vessels broadcast their position with AIS transponders. Switching AIS off ("going dark") right at the edge of a marine reserve is a common way to hide illegal fishing. Countries already have the sensors to spot this: satellite radar, night-light satellites, vessel tracking. The slow part is the human workflow between a detection and a response. An analyst at 3am has to notice the alert, choose the next check, find a patrol boat that may be 3 hours away, and build a case for the commander.

## How Tidewatch works

```
DETECT -> VERIFY -> SCORE -> DISPATCH -> ESCALATE (human)
```

1. **Detect.** Watches every vessel's AIS reports. A gap of 3+ minutes opens a track. Signed distance to the reserve boundary (point-in-polygon plus distance to the boundary line) decides whether the gap is suspicious. Licensed local boats going quiet inside known reception dead zones are dismissed automatically.
2. **Verify.** Projects the vessel's last speed and heading forward into a growing search area, then gathers evidence:
   - VIIRS night-light detections inside the search area
   - Satellite radar returns matched to AIS positions. A hull with no matching transponder is a dark vessel.
   - A cost-aware imagery choice: wait for the free Sentinel-1 pass, or buy commercial radar ($900) when the free pass is far away and risk is high, within a nightly budget.
3. **Score.** Logistic regression over the evidence:

   | Feature | Weight |
   |---|---|
   | Bias | -4.0 |
   | Transponder off near boundary | 1.6 |
   | No reserve fishing license | 0.8 |
   | Earlier AIS gaps in 90 days (per gap) | 0.3 |
   | Night lights match squid fishing | 1.1 |
   | Radar hull inside reserve, no transponder | 1.9 |
   | Drone sees fishing gear | 1.5 |
   | Drone sees research ship, no gear | -3.0 |

   `p = 1 / (1 + e^-z)`. Weights stay fixed. Each new piece of evidence adds a term and the score is recalculated. Weights are hand-set for the demo and would be learned from enforcement outcomes in production.
4. **Dispatch.** Actions are tiered by cost, using the rule "act when p > false alarm cost / (false alarm cost + miss cost)":
   - p >= 0.25: watch, free imagery
   - p >= 0.50: send the nearest available drone
   - p >= 0.75: call the patrol boat with the shortest ETA, skipping busy or low-fuel boats
5. **Escalate.** When a patrol is moving and the drone confirms fishing gear, the commander gets the evidence package and one decision: board or keep shadowing. If the drone finds an innocent vessel instead, the score drops and the agent stands the patrol down.

Guardrails: imagery budget cap, drone sortie limit, automatic false alarm dismissal, patrol stand-down.

## What is real and what is simulated

- **Simulated:** vessels, coastlines, sensor feeds, radar and thermal images. Each night is generated from a random seed.
- **Live code:** gap detection, boundary geometry, search projection, radar-to-AIS matching, the logistic model, action tiers, the imagery decision, the ETA dispatcher, guardrails, and every log line (generated from the run's actual values).

## Next steps

- Learn the weights from real enforcement outcomes
- Plug into real feeds: Global Fishing Watch, Skylight, Sentinel-1, VIIRS
- A* routing around islands and intercept points for moving targets
- Underwater acoustic detections as a new feature
- Same engine beyond reserves: national fishing zones, ship-to-ship transfers, sanctions evasion, undersea cable protection, search and rescue
