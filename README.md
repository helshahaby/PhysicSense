# PhysicSense
AI-Powered Military Protective Helmet Monitoring &amp; Verified Impact Detection
# PhysicSense Defense

**AI-Powered Military Protective Helmet Monitoring & Verified Impact Detection**
Hackathon prototype for the CurveSYS challenge — *Physical Truth: Smart Helmet Impact Detection & Verified Alerts*.

**Author / presenter:** Hossam Elshahaby  
**Live application:** https://physicsense-helmet-signal.lovable.app  
**Demo video:** https://youtu.be/aQnM90rgW6U

> ⚠️ **SIMULATED DATA — RESEARCH PROTOTYPE.** PhysicSense Defense is **not** a certified ballistic protection, blast dosimetry, or medical diagnostic system. All sensor data is synthetic, all thresholds are illustrative and uncalibrated, and the verification score expresses **algorithmic evidence strength — never an injury probability**. No emergency services are contacted by any action in this app.

---

## 1. Mission

A simple accelerometer threshold cannot reliably distinguish an equipment drop from a personnel event. PhysicSense Defense demonstrates **integrity-first, multi-sensor verification with explainable rule-based verdicts**. Uncertain evidence and sensor faults remain visible for review. The current engine is deterministic signal processing, not a trained machine-learning model.

The prototype:

- Correlates acceleration, rotation, and pressure measurements instead of reacting to a single spike.
- Verifies data integrity **before** making any physical claim (frozen IMU, dropped packets, out-of-range values → SENSOR FAULT, not impact).
- Separates a **dropped helmet** (equipment event) from a **personnel event** using post-event micro-motion.
- Requires **two independent modalities** (pressure + acceleration, time-correlated) before raising a blast-exposure signature.
- Routes ambiguous evidence to a **human reviewer** instead of guessing.
- Produces an **explainable evidence list**, a reproducibility record (scenario + seed + sample rate), and exportable incident reports.

## 2. Live demo

Published app: **https://physicsense-helmet-signal.lovable.app**  
Video walkthrough: **https://youtu.be/aQnM90rgW6U**

### Guided demo script (9 steps)

1. **Overview (`/`)** — read the mission, then press **Run live scenario** with *Normal movement*. The console stays quiet: `NORMAL`.
2. Run *Running + sudden head turn* — a 320 °/s head rotation while running. Expected result at default settings: `MONITORING`, **no personnel alert**.
3. Run *Accidental helmet drop* — free-fall, ground impact, then perfect stillness. Result: `HELMET DROPPED` — classified as equipment, **not** a casualty.
4. Run *Blunt helmet impact* — 38 g / 11 ms transient with 1400 °/s rotation. Result: `VERIFIED PHYSICAL EVENT` (accelerometer + gyroscope corroborate). A new incident appears in the Alert Center.
5. Run *Blast exposure signature* — 28 kPa overpressure within ~2 ms of an acceleration disturbance. Result: `POTENTIAL BLAST EXPOSURE`.
6. Run *Sensor malfunction* — frozen IMU, dropped packets, impossible values. Result: `SENSOR FAULT` — the spike is **not** treated as an impact.
7. Run *Ambiguous transient* — moderate spike, weak corroboration. Result: `HUMAN REVIEW REQUIRED`.
8. Open **Alert Center (`/alerts`)** — acknowledge, escalate, or mark false positive (a written reason is required to close). Export the incident as a text report, JSON evidence package, or raw CSV.
9. Open **Benchmark (`/benchmark`)** — run the seeded benchmark comparing the naive threshold baseline vs. the multi-sensor verification engine.

The soldier HUD preview on the home page updates its message after every run.

## 3. Application map

| Route | Page | Purpose |
| --- | --- | --- |
| `/` | Overview | Mission, guided demo, live scenario console, HUD preview |
| `/sensors` | Sensor Dashboard | Simulated acceleration, gyro and pressure charts; temperature/orientation readouts |
| `/detection` | Detection Console | Feature extraction and verdict comparison across all eight scenarios |
| `/blast` | Blast Analysis | Overpressure transient, pressure↔accel correlation window |
| `/alerts` | Alert Center | Incident list, human review workflow, exports (TXT / JSON / CSV) |
| `/reconstruction` | Event Reconstruction | Time-slider inspection with evidence markers and raw/filtered acceleration |
| `/config` | Sensor Config | Sampling rate, thresholds, correlation window, sensor enable/disable |
| `/benchmark` | Benchmark Lab | Seeded threshold-vs-fusion metrics and confusion matrix |
| `/docs` | Architecture | System diagram, verification order, limitations, hardware integration path |

## 4. Architecture

```text
 ┌────────────────────────┐   samples @ fs Hz   ┌──────────────────────────┐
 │ 1. Helmet Sensor        │ ──────────────────▶ │ 2. Physical Event         │
 │    Simulation Engine    │  (ax..gz, p, T,     │    Detection Engine       │
 │  seeded · deterministic │   valid flag)       │  feature extraction       │
 │  ⇄ swap for HW driver   │                     └────────────┬─────────────┘
 └────────────────────────┘                                   │ features
                                                              ▼
 ┌────────────────────────┐   verdict + reasoning ┌──────────────────────────┐
 │ 4. Personnel Safety     │ ◀──────────────────── │ 3. Multi-Sensor           │
 │    Dashboard + HUD      │                       │    Verification Engine    │
 └───────────┬────────────┘                       │  integrity gate → blast → │
              │ alerts                            │  drop → fall → impact     │
              ▼                                   │  evidence score 0–100     │
 ┌────────────────────────┐                       └──────────────────────────┘
 │ 5. Incident & Alert     │ ── CSV / JSON / report export
 │    Management (human    │ ── reconstruction · benchmark lab
 │    review, simulated)   │
 └────────────────────────┘
```

### Source layout

```text
src/
├── lib/sim/
│   ├── engine.ts      # Scenario generator — deterministic, seeded, hardware-agnostic
│   ├── detect.ts      # Feature extraction + multi-sensor verification (explainable)
│   ├── benchmark.ts   # Seeded benchmark: threshold baseline vs. fusion engine
│   ├── store.ts       # In-memory incident/config store (resets on reload — no backend)
│   └── export.ts      # CSV / JSON / TXT incident export with disclaimer
├── components/ps/     # AppShell, LiveMonitor, charts, shared UI panels
├── routes/            # TanStack Start file-based routes (one per page above)
└── test/
    ├── classifier.test.ts    # One test per classification rule
    └── app-routing.test.tsx  # Route smoke tests
```

The simulation, detection, and benchmark layers are **pure TypeScript modules with no UI dependencies**. A proposed hardware driver would emit the same `Sample` shape. Hardware integration is not implemented: the UI, store, exports and reconstruction still generate synthetic scenarios and would need acquisition/provenance adapters.

## 5. Detection & verification logic

### Features extracted per 6 s window

Peak acceleration magnitude including gravity & time, peak gyro magnitude, total time above 5 g (not contiguous pulse width), longest free-fall run (< 0.3 g), overpressure peak & time, pressure↔accel peak-time separation (not waveform cross-correlation), post-event stillness and acceleration σ, post-event gyro mean, orientation change, pre-event rotation, missing-sample %, frozen metric, out-of-range count, invalid-pressure flag. Temperature is visualized but is not used for classification.

### Verification order (and why)

1. **Integrity gate first.** Missing, frozen, or physically impossible data cannot support any physical claim → `SENSOR FAULT`.
2. **Blast** requires pressure **and** acceleration **and** correlation within the configured window (default 50 ms). One modality alone → `HUMAN REVIEW REQUIRED`.
3. **Drop**: free-fall + contact + *absence* of physiological micro-motion → `HELMET DROPPED` (equipment event).
4. **Fall**: orientation change > 40°, peak acceleration > 3 g and stillness at least the configured duration → `POSSIBLE FALL`. Pre-contact rotation and positive micro-motion are extracted/narrated but are not required by this predicate.
5. **Impact**: peak above the impact threshold (default 20 g), gyroscope corroboration (> 700 °/s), and branch score ≥ 80 → `VERIFIED PHYSICAL EVENT`; otherwise the above-threshold event is `POTENTIAL IMPACT`.
6. Anything else above 6 g → `HUMAN REVIEW REQUIRED`; elevated activity → `MONITORING`; otherwise `NORMAL`.

### Scores

- **Evidence score (0–100)** uses branch-specific formulas expressing algorithmic support. Displayed evidence weights are not summed into the returned score. It is **not** an injury probability.
- **Sensor quality (0–100)** is reported separately, derived from completeness, freezing, and range checks.

### Default configuration

| Parameter | Default | Meaning |
| --- | --- | --- |
| `fs` | 200 Hz | Sampling rate |
| `impactG` | 20 g | Impact threshold |
| `blastKpa` | 8 kPa | Overpressure threshold |
| `immobilityS` | 2 s | Stillness duration for fall detection |
| `correlationMs` | 50 ms | Max pressure↔accel lag for blast correlation |
| `thresholdBaselineG` | 6 g | Naive baseline comparator |
| `pressureEnabled` / `gyroEnabled` | on | Sensor availability toggles |

All values are editable on the Sensor Config page. They are **illustrative placeholders**, uncalibrated against any helmet, headform, or human data.

## 6. Simulated scenarios

| Scenario | Ground truth | Signature |
| --- | --- | --- |
| Normal movement | NORMAL | Walking, turning, bending |
| Running + sudden head turn | NORMAL | 320 °/s rotation with running gait — must **not** alert |
| Accidental helmet drop | DROP | Free-fall → 55 g contact → bounces → perfect stillness |
| Blunt helmet impact | IMPACT | 38 g / 11 ms transient + 1400 °/s rotation, wearer keeps moving |
| Blast exposure signature | BLAST | 28 kPa overpressure correlated with acceleration disturbance |
| Fall + immobility | FALL | Rotation → moderate contact → stillness with breathing micro-motion |
| Sensor malfunction | FAULT | Frozen IMU, 25% dropped packets, out-of-range spike, stuck pressure |
| Ambiguous transient | UNCERTAIN | 11 g spike with weak corroboration → human review |

Scenarios are deterministic for the same scenario, seed, sampling rate, duration and intensity. **Provenance limitation:** incidents retain scenario and seed but not original configuration or raw samples. Export/reconstruction regenerates samples at the current sampling rate, and JSON does not preserve original thresholds. Keep settings unchanged during an incident walkthrough; exact historical evidence reproduction is not guaranteed.

## 7. Benchmark

The Benchmark Lab runs N seeded trials per scenario at randomized intensities (0.45–1.55×). The UI defaults to 20 trials per each of eight scenarios (160 records); the library defaults to 25 (200 records). Normal movement and running share one truth class, so the matrix has seven classes. It compares:

- **Baseline:** single acceleration threshold (`thresholdBaselineG`).
- **Fusion:** the full multi-sensor verification engine.

Computed metrics: precision, recall, F1, false-positive rate, model-assigned detection latency, false alerts per simulated hour, fault-detection accuracy, confusion matrix and per-class counts. Per-class accuracy is not separately rendered. Latency comes from assigned verdict timestamps, not measured processing or end-to-end alert timing; live playback verifies after its six-second window. DROP and FAULT are personnel-negative in the benchmark even though the app logs them as incidents.

> Benchmark results are optimistic, in-distribution evaluations on the engine's own synthetic data. They do not establish real-world accuracy, operational false-alarm reduction, or injury prevention. Record settings and seed alongside any reported run.

## 8. Tech stack

- **TanStack Start v1** (React 19; manifest specifies Vite 8.1.5) — file-based routing, SSR
- **TypeScript** — strict, pure-TS algorithm modules
- **Tailwind CSS v4** + shadcn-style components — dark tactical UI (Chakra Petch / IBM Plex Sans / JetBrains Mono)
- **Recharts** — signal visualization
- **Vitest** + Testing Library — classifier and routing tests
- No backend: prototype state is an in-memory client store and resets on reload

## 9. Development

Built with [Lovable](https://lovable.dev). To run locally you need Node.js and npm ([install with nvm](https://github.com/nvm-sh/nvm#installing-and-using-nvm)):

```sh
git clone <this-repository-url>
cd <repository-name>
npm i
npm run dev
```

| Command | Action |
| --- | --- |
| `npm run dev` | Start the dev server |
| `npm run build` | Production build |
| `npm run test` | Run classifier + routing tests |
| `npm run lint` | ESLint |

Classifier rules are covered by `src/test/classifier.test.ts` — update the tests whenever thresholds or the verification order change.

## 10. Hardware integration path

Replace `generateScenario()` with a driver emitting the same `Sample` shape:

```ts
interface Sample {
  t: number;                       // seconds
  ax: number; ay: number; az: number;  // g
  gx: number; gy: number; gz: number;  // deg/s
  p: number;                       // kPa absolute
  temp: number;                    // °C
  valid: boolean;                  // false = dropped packet
}
```

Proposed path: 6-DoF IMU + MEMS absolute pressure sensor → MCU → BLE/UART → acquisition adapter → pure-TS detection engine. Time synchronization, calibration, transport handling, recorded raw data and independent validation are future work. A server service would require a separate implementation/deployment; it is not present in this prototype.

## 11. Limitations & safety

- All data is synthetic. Thresholds are illustrative and uncalibrated against any helmet, headform, or human data.
- The system does **not** diagnose injury, determine ballistic protection effectiveness, confirm projectile penetration, or measure blast dose.
- All reviewer actions are simulated; no emergency services are contacted.
- No weapons, targeting, or engagement functionality — personnel protection and situational awareness only.
- This browser prototype runs the engine client-side; state resets on reload.
