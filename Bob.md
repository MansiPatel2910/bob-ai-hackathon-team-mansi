# ChronoChip AI — Smart Fab & Geopolitical Risk Copilot

**IBM Bob Innovation Hackathon 2026 — Problem Statement S2 (Semiconductor)**

> An MCP-native copilot that fuses real-time fab floor telemetry with geopolitical
> critical-material risk scoring, so a Fab Operations Director can ask a single
> agent — IBM Bob — "are we going to ship on time, and why not?"

---

## 1. Why This Matters (S2 Problem Framing)

Advanced-node semiconductor fabs run on razor-thin schedule margins. A single
plasma-etch chamber drifting out of match, or a single-country export
restriction on Gallium, can silently add **weeks** to a finished-goods ETA
before a human ever notices the pattern in raw MES logs. ChronoChip AI closes
that gap by giving an IBM Bob Agent **structured, tool-callable access** to
both signal domains — operational (fab floor) and macro (supply chain) — in
one conversational surface.

## 2. System Architecture

```
+-----------------------------------------------------------------------+
|                         IBM Bob Orchestrator                          |
|              (LLM planner - natural-language reasoning)               |
+---------------------------------+---------------------------------- --+
                                  |  MCP tool calls (function-calling)
                                  v
+-----------------------------------------------------------------------+
|                    ChronoChip AI - MCP Tool Server                    |
|                                                                         |
|  fab.get_wip_telemetry     fab.predict_bottleneck   geo.assess_...     |
|         |                          |                       |           |
|         v                          v                       v           |
|  +-------------+          +------------------+   +------------------+  |
|  | Telemetry   |  X-factor | Bottleneck       |   | Geopolitical     |  |
|  | Simulator   |---------->| Engine           |   | Risk Matrix      |  |
|  | (Litho/     |          | (threshold model  |   | (HHI + export-   |  |
|  |  Etch/CVD)  |          |  + 52wk projection)|   |  control scoring)|  |
|  +-------------+          +------------------+   +------------------+  |
+-------------------------------------------------------------------------+
```

**Layer 1 — Telemetry Simulation.** `TelemetrySimulator` models six tool
nodes across the three modules explicitly named in S2: ASML TWINSCAN
NXE:3600D (EUV) and NXT:2000i (DUV) for Lithography, Lam Research Kiyo® FX
Conductor and Applied Materials Centura® AP for Plasma Etch, and Applied
Materials Producer® GT / ASM A412™ for CVD. Each tool carries realistic fab
telemetry: OEE%, queue time (Q-time), WIP lot count, MTBF/MTTR, and state
(`RUN` / `PM` / `DOWN`).

**Layer 2 — Predictive Bottleneck Engine.** Computes **X-factor**
(actual cycle time ÷ theoretical process time) — the standard fab cadence
KPI — per tool, and applies a four-tier threshold model (`nominal → WATCH →
WARNING → CRITICAL`). Breaches are propagated downstream using a fan-out
multiplier keyed to each tool's live WIP load, projecting a finished-goods
ETA slip in **weeks, capped at a 52-week horizon**, with an attached
root-cause hypothesis and confidence score.

**Layer 3 — Geopolitical Risk Matrix.** Scores five critical materials
(Gallium, Germanium, electronic-grade Neon, Palladium, Tungsten) on a
composite 0–100 risk index built from: China supply share %, Herfindahl-
Hirschman Index (HHI, the standard economic concentration metric), active
export-control status, and alternate-source qualification lead time. Tiers
into `MODERATE / ELEVATED / SEVERE`.

**Layer 4 — MCP Tool Server.** `ChronoChipMCPServer` exposes the above as a
`tools/list`-style registry (`fab.get_wip_telemetry`, `fab.predict_bottleneck`,
`geo.assess_material_risk`), structurally identical to a production MCP
stdio/SSE server — swap the in-process dispatch for an actual MCP transport
with no changes to the underlying engines.

**Layer 5 — IBM Bob Conversational Agent.** `BobAgent` implements the
call → synthesize → respond loop a Bob LLM planner performs: parse operator
intent, invoke the relevant MCP tool(s), and return a narrated, data-grounded
answer. The CLI runs a **scripted 4-question demo** (ideal for a 3-minute
video) followed by a **live interactive prompt loop**.

## 3. Data Flow (Request Lifecycle)

1. Fab Ops Director asks Bob a question ("do we have any bottleneck alerts?").
2. `BobAgent.answer()` performs lightweight intent classification.
3. Bob calls the matching MCP tool(s) on `ChronoChipMCPServer`.
4. The tool queries live objects (`TelemetrySimulator`, `BottleneckEngine`,
   `GeopoliticalMatrix`) and returns structured dataclasses.
5. Bob composes a narrated response — headline finding first, full data
   table second — rendered through the zero-dependency ANSI/box-drawing
   kernel for a boardroom-grade terminal UI.

## 4. Why the IBM Bob Integration Is Load-Bearing (Not Cosmetic)

- **Single pane of glass across two domains.** Without Bob, a Fab Ops
  Director checks MES dashboards for tool health *and* a separate
  procurement/geopolitical brief for material risk. ChronoChip AI lets Bob
  correlate both in one answer — e.g., an ETCH bottleneck *and* a Gallium
  export-control alert in the same executive brief.
- **MCP-native, not a chatbot wrapper.** Each capability is a discrete,
  independently callable tool with typed inputs/outputs — exactly the
  interface Bob's planner needs to compose multi-step reasoning (e.g.
  "check bottlenecks, then cross-reference which are on the SEVERE-risk
  material path").
- **Decision-grade output, not raw telemetry.** Bob doesn't just relay
  numbers — it converts X-factor breaches into a **quantified ETA shift in
  weeks with a confidence score**, and material exposure into a
  **prioritized escalation list**, so the human decision-maker acts in
  seconds, not after manual analysis.
- **Deterministic + demo-safe.** The rules-over-telemetry synthesis layer
  means the exact same live-tool-calling pattern Bob would use in
  production runs reliably and repeatably for the judging demo, with zero
  external API dependency or network risk.

## 5. Key Metrics Reference (for judges / demo narration)

| Metric | Definition | Threshold used |
|---|---|---|
| **X-factor** | Actual cycle time ÷ theoretical process time | <2.0 nominal · 2.0–3.0 WATCH · 3.0–4.5 WARNING · ≥4.5 CRITICAL |
| **OEE** | Overall Equipment Effectiveness | Fab-standard tool health % |
| **Q-time** | Minutes a lot waits before a tool processes it | Drives X-factor numerator |
| **HHI** | Herfindahl-Hirschman Index — supply concentration | >2500 = highly concentrated (FTC/DOJ standard) |
| **ETA shift** | Projected finished-goods delivery slip | Capped at 52-week horizon |

## 6. Running the Prototype

```bash
python3 main.py                 # full scripted demo + interactive mode
python3 main.py --no-interactive  # scripted demo only (for recorded video)
```

No `pip install` required — stdlib only. Best viewed in a 96+ column
terminal with 256-color ANSI support (default on macOS Terminal, iTerm2,
Windows Terminal, and most Linux terminal emulators).

## 7. Suggested 3-Minute Demo Script

| Time | Beat |
|---|---|
| 0:00–0:20 | Banner + boot sequence — "establishing MCP transport with IBM Bob" |
| 0:20–0:45 | `fab.get_wip_telemetry` — show the live tool dashboard, call out the Etch X-factor spike |
| 0:45–1:30 | `fab.predict_bottleneck` — Bob narrates the CRITICAL alert and the +N-week ETA slip |
| 1:30–2:15 | `geo.assess_material_risk` — Gallium/Germanium SEVERE tier, tie back to the Etch delay narrative ("even if we fix Etch, we're still exposed on Gallium") |
| 2:15–2:45 | Executive summary panel — the single-screen answer a director actually needs |
| 2:45–3:00 | Switch to interactive mode, ask a live free-form question to prove it's a real agent loop, not a canned script |

## 8. Extending to Production

- Swap `ChronoChipMCPServer`'s in-process dispatch for a real MCP stdio/SSE
  transport (`mcp` Python SDK) — the tool signatures are already compatible.
- Replace `TelemetrySimulator` with a live MES/SECS-GEM or historian feed
  (e.g. Ignition, Camstar, or a Kafka telemetry bus).
- Replace the static `GeopoliticalMatrix.MATERIALS` table with a live feed
  from trade-compliance / export-control data providers.
- Replace `BobAgent`'s rules-based intent router with IBM Bob's native LLM
  planner calling the same three MCP tools — zero changes needed downstream.
