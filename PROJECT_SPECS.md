# PhysicSense Defense — Project Specifications

**Author:** Hossam Elshahaby  
**Document date:** 7 November 2026  
**Application:** https://physicsense-helmet-signal.lovable.app  
**Demo video:** https://youtu.be/aQnM90rgW6U  
**Challenge:** CurveSYS — Physical Truth: Smart Helmet Impact Detection & Verified Alerts

## 1. Scope and status

A browser research prototype demonstrates integrity-first, explainable multi-sensor verification on eight synthetic helmet scenarios. It includes signal visualization, rule-based verdicts, simulated human review, incident inspection and downloads. “AI-powered” is product positioning: the shipped classifier uses deterministic rules and hand-written scoring, not a trained model.

**Not certified ballistic protection, blast dosimetry or medical diagnosis.** No real helmet data, operational response, emergency-service contact or validated injury prediction is implemented.

## 2. Problem, solution and intended impact

| Problem | Implemented response | Intended benefit, not measured impact |
| --- | --- | --- |
| An acceleration spike lacks context | Correlate acceleration, rotation and pressure | Better differentiation of equipment and personnel events |
| Corrupt data can imitate physical events | Integrity gate precedes physical interpretation | Prevent unsupported physical claims on faulty inputs |
| Ambiguous evidence encourages overconfidence | Human-review classification and evidence list | More accountable interpretation |
| Decisions are difficult to inspect | Incident timeline and TXT/JSON/CSV exports | Repeatable engineering review, subject to provenance limits below |

No numerical injury reduction, field accuracy, response-time improvement or operational false-alert reduction is claimed.

## 3. Architecture and data flow

```text
Seeded synthetic generator → feature extraction → ordered verification
                         → dashboard/HUD → incident review and exports
                         → baseline-vs-fusion benchmark
```

- `src/lib/sim/engine.ts`: seeded waveform generation, magnitudes, orientation helpers, filtering.
- `src/lib/sim/detect.ts`: features, quality gate, physical rules, scores and evidence.
- `src/lib/sim/benchmark.ts`: seeded comparisons and confusion matrix.
- `src/lib/sim/store.ts`: in-memory client configuration and incidents.
- `src/lib/sim/export.ts`: synthetic CSV, JSON and text reports.
- `src/components/ps/`: shared shell, live monitor, chart and presentation controls.
- `src/routes/`: nine TanStack content pages.

Pure-TS simulation/detection/benchmark logic is UI-independent. The store and displays currently generate scenarios directly; a hardware adapter is proposed, not delivered. Reload resets prototype state. Identifiers and initial incidents are fictional.

## 4. Sample contract and signal characteristics

```ts
interface Sample {
  t: number;                         // seconds
  ax: number; ay: number; az: number; // g
  gx: number; gy: number; gz: number; // degrees/second
  p: number;                         // absolute pressure, kPa
  temp: number;                      // degrees Celsius
  valid: boolean;                    // false denotes a dropped sample
}
```

One pressure field exists; separate left/right pressure, battery, strain and CRC/sequence UI labels are not implemented sample channels. Sensor self-test displays are static. There is no certified sensor range, resolution or bit-depth specification.

- Six-second synthetic windows; default 200 Hz produces 1,200 samples.
- Mulberry32 seeded PRNG, analytic waveforms and uniform noise.
- Acceleration magnitude includes gravity; it is not gravity-compensated linear acceleration.
- Roll/pitch derive from acceleration, not full inertial orientation fusion.
- Reconstruction supports a one-pole low-pass acceleration view at 15 Hz.
- Temperature is displayed, not a classifier input.

## 5. Configuration

| Parameter | Default | UI choices/range |
| --- | --- | --- |
| Sampling rate | 200 Hz | 100, 200, 400, 800 Hz |
| Impact threshold | 20 g | 5–80 g |
| Blast overpressure threshold | 8 kPa | 1–40 kPa |
| Pressure/acceleration peak-time window | 50 ms | 5–500 ms |
| Fall stillness duration | 2 s | 0.5–2.5 s |
| Baseline acceleration threshold | 6 g | 2–30 g |
| Pressure and gyro availability | Enabled | On/off |
| Seed | Editable | Used for reproducible synthesis |

All thresholds are illustrative and uncalibrated. Blast Analysis uses a local seed of 11 and at least 400 Hz for its dedicated synthesis.

## 6. Feature extraction

Peak acceleration/time, peak gyro, total above-5-g sample duration, longest below-0.3-g free-fall run, median-referenced overpressure peak/time, pressure–acceleration peak-time difference, post-event acceleration variability/gyro mean/stillness, orientation change, pre-event rotation, missing fraction, freezing and out-of-range metrics.

Post-event analysis begins 0.5 s after the acceleration peak. Stillness uses acceleration within 0.05 g of 1 g and gyro below 8 degrees/s. The frozen metric compares selected axes (`ax`, `gy`, `az`), not all IMU components. “Correlation” means peak-time separation, not waveform cross-correlation. “Impulse duration” sums qualifying samples rather than measuring one contiguous pulse.

## 7. Verification rules in execution order

Strict comparison boundaries matter; these describe the shipped engine, not a safety standard.

| Order / output | Predicate |
| --- | --- |
| 1 — SENSOR FAULT | Missing >5%; freezing >300 ms; any out-of-range sample; or enabled pressure value <50 kPa. Range checks include acceleration >200 g, enabled gyro >4,000 degrees/s, and nonfinite acceleration. |
| 2 — POTENTIAL BLAST EXPOSURE | Overpressure > configured threshold; acceleration >8 g; peak-time separation < configured window. Pressure-only evidence above threshold becomes human review. |
| 3 — HELMET DROPPED | Free-fall >200 ms, acceleration >6 g, post-event acceleration standard deviation <0.004 g, mean gyro <2 degrees/s. |
| 4 — POSSIBLE FALL | Stillness ≥ configured duration, orientation change >40 degrees, acceleration >3 g. Positive micro-motion and pre-contact rotation are not required gates. |
| 5 — VERIFIED PHYSICAL EVENT | Acceleration > impact threshold, gyro >700 degrees/s, branch score ≥80. Otherwise above-threshold impact becomes POTENTIAL IMPACT. |
| 6 — HUMAN REVIEW REQUIRED | Remaining acceleration >6 g. |
| 7 — MONITORING | Acceleration >2 g or gyro >250 degrees/s. |
| 8 — NORMAL | Remaining valid inputs. |

Evidence score (0–100) uses branch-specific formulas; displayed evidence weights are not summed. Sensor quality (0–100) is separate. Neither score is an injury probability.

## 8. Synthetic scenario inventory

| Scenario | Benchmark truth | Signature / expected default behavior |
| --- | --- | --- |
| Normal movement | NORMAL | Walking/turning/bending; NORMAL |
| Running + sudden head turn | NORMAL | Running and 320 degrees/s turn; MONITORING |
| Accidental helmet drop | DROP | Free-fall/contact/stillness; HELMET DROPPED |
| Blunt helmet impact | IMPACT | Synthetic 38-g transient with rotation; verified event |
| Blast exposure signature | BLAST | Synthetic 28-kPa pressure transient and acceleration; potential exposure |
| Fall + immobility | FALL | Rotation/contact/sustained stillness; possible fall |
| Sensor malfunction | FAULT | Frozen/missing/invalid signals; sensor fault |
| Ambiguous transient | UNCERTAIN | Moderate spike/weak corroboration; human review |

Figures are generator parameters, not hardware measurements; sampled peaks vary with seed, intensity and sampling rate.

## 9. Application pages and reviewer workflow

| Path | Capability |
| --- | --- |
| `/` | Overview, scenario playback and completed-verdict HUD |
| `/sensors` | Simulated signal charts, temperature/orientation, static self-test display |
| `/detection` | Eight-scenario feature/verdict comparison |
| `/blast` | Dedicated blast synthesis and peak-time relationship |
| `/alerts` | Acknowledge, request verification, false-positive action, simulated medical escalation, close and export |
| `/reconstruction` | Incident time-slider inspection and raw/filtered acceleration |
| `/config` | Seed, sampling, thresholds and channel availability |
| `/benchmark` | Seeded metrics, confusion matrix, chart and JSON export |
| `/docs` | Architecture, limitations and proposed hardware path |

Closure requires at least five trimmed note characters in the UI. This is not a server-enforced permission or workflow. Reviewer identity is simulated. Overrides append review history without replacing the original machine verdict. No outside service is contacted.

## 10. Benchmark protocol and interpretation

- Eight scenarios; normal and running share NORMAL, producing seven truth classes.
- UI default: 20 trials per scenario, 160 total; library default: 25, 200 total.
- Six seconds per record; deterministic intensity range approximately 0.45–1.55×.
- Baseline: configured single acceleration threshold. Fusion: full verification engine.
- Personnel-positive truth: IMPACT, BLAST, FALL and UNCERTAIN. DROP/FAULT are personnel-negative, although the app records them as incidents.
- Computes precision, recall, F1, false-positive rate, false alerts per simulated hour, fault accuracy, confusion matrix and per-class counts.
- Reported latency is assigned from verdict timing formulas, not measured execution or end-to-end delivery; live playback verifies after six seconds.
- No benchmark score is asserted here. Any reported run must include seed, settings and trial count.

The generator and classifier are developed together. Synthetic in-distribution evaluation cannot establish real-world safety or generalization.

## 11. Export, reconstruction and provenance limitations

CSV includes sample fields, derived magnitudes and validity. JSON includes incident/verdict, scenario metadata and regenerated samples. TXT includes explanation, evidence and review history. Simulation disclaimers accompany exports.

**Important:** incidents store scenario and seed, not original raw samples or complete acquisition/classifier configuration. Export and reconstruction regenerate at the current sampling rate. Changing settings after an incident can make regenerated evidence differ from the stored verdict. JSON does not preserve original thresholds/channel settings. Benchmark export can also reflect controls changed after the result. Preserve settings during a walkthrough; immutable provenance remains future work.

## 12. Technology and verification coverage

TanStack Start v1 / React 19, TypeScript, Tailwind v4, Recharts, Vitest and Testing Library. Current manifest specifies Vite 8.1.5. No durable data service, login or actual hardware acquisition is implemented.

Existing coverage consists of eight classifier tests and one route-matching smoke test. These are narrow examples, not full boundary, provenance, workflow, hardware or field validation. Test files: `src/test/classifier.test.ts` and `src/test/app-routing.test.tsx`. Use `npm run test`; local development uses `npm i` and `npm run dev`. Test execution status is reported separately rather than implied by this specification.

## 13. Proposed development and validation path

1. Add time-synchronized IMU/pressure acquisition and transport health handling.
2. Calibrate mounted sensors and investigate wear-state ambiguity and motion artifacts.
3. Capture immutable raw samples, acquisition settings and classifier version per incident.
4. Validate on controlled headform experiments and independent datasets; test disabled channels, corruption and threshold boundaries.
5. Measure real processing/delivery latency and reviewer usability separately from assigned event timing.
6. Before operational use, address durable storage, authentication, access control, security, privacy, availability and applicable certification.

These are proposed milestones, not implemented or funded commitments. Personnel protection and situational awareness only; no targeting, weapons or engagement functionality.