# Key Technical Decisions

> **Architecture Decision Records (ADR)** — the "why" behind the design.

---

## ADR 1: On-Device CoreML vs. Cloud API

**Context:** Hypoglycemia prediction requires real-time inference with sensitive health data.

**Decision:** Use CoreML with bundled `.mlpackage` models running entirely on-device.

**Rationale:**
- **Latency:** No network round-trip. Inference completes in <50ms on iPhone 15 Pro.
- **Privacy:** Blood glucose and insulin data never leaves the device.
- **Reliability:** Works offline in areas with poor signal (e.g., subway, gym basement).
- **Cost:** No server infrastructure or API costs for the developer or user.

**Trade-offs:**
- Model size: ~2MB per model × 4 models = ~8MB. Acceptable for modern iOS apps.
- Update friction: Model updates require App Store release, not hot-swappable.
- No federated learning: Cannot improve models from real-world usage data.

**Status:** ✅ Accepted, no plans to change.

---

## ADR 2: HealthKit as Primary CGM Source, Nightscout as Fallback

**Context:** The app needs glucose data. Options: (a) direct Dexcom/Libre API, (b) Nightscout, (c) HealthKit.

**Decision:** HealthKit as primary, Nightscout as fallback and treatment source.

**Rationale:**
- **HealthKit:** Lower latency, works offline, requires no third-party setup. Many CGM systems (Dexcom G7, Libre 3) now write to HealthKit automatically.
- **Nightscout:** Still needed for treatment data (bolus, carbs) and historical CGM when HealthKit is sparse.
- **Fallback:** If HealthKit has <12 glucose points in the last 3.5 hours, switch to Nightscout.

**Trade-offs:**
- HealthKit CGM access varies by region and CGM firmware version.
- Nightscout requires user setup (server URL, API token), raising barrier to entry.

**Status:** ✅ Accepted. HealthKit-first strategy has proven stable.

---

## ADR 3: 4-Model Cascade vs. Single Multi-Horizon Model

**Context:** Should we use one model that predicts all horizons, or separate models?

**Decision:** 4 separate models (h1, h2, h3, h6) with specialized training.

**Rationale:**
- **Accuracy:** Short-term models (5–10 min) learn fast dynamics (insulin absorption, meal spikes). Long-term models (30 min) learn slow trends (basal drift, nocturnal patterns).
- **Calibration:** Each horizon has its own iso-calibrator, ensuring probability scores are well-calibrated across time scales.
- **Escalation logic:** The cascade naturally maps to the app's 4-tier alert system.

**Trade-offs:**
- 4× model storage vs. 1 model.
- 4× training compute.
- Slightly more complex inference orchestration.

**Status:** ✅ Accepted. RMSE improvements justify the complexity.

---

## ADR 4: WALSH Bilinear IOB vs. Exponential Decay

**Context:** Insulin On Board (IOB) is critical for prediction. Multiple pharmacokinetic models exist.

**Decision:** Walsh bilinear model with DIA=300 minutes (5 hours).

**Rationale:**
- **Clinical standard:** Walsh model is widely accepted in diabetes research.
- **Bilinear:** Captures the "peak and decay" shape of insulin absorption better than simple exponential decay.
- **Parameter simplicity:** Only one parameter (DIA) — easier to tune than multi-compartment models.

**Trade-offs:**
- Does not account for insulin type (rapid-acting vs. ultra-rapid). Could be extended with type-specific kernels.
- Assumes constant absorption rate; real-world absorption varies by injection site, temperature, exercise.

**Status:** ✅ Accepted. Future: allow user-configurable DIA in Settings.

---

## ADR 5: XcodeGen for Project Management

**Context:** `.xcodeproj` files are notoriously difficult to merge and prone to corruption.

**Decision:** Generate `.xcodeproj` from `project.yml` using XcodeGen.

**Rationale:**
- **Version control friendly:** `project.yml` is human-readable, diffable, and mergeable.
- **Reproducibility:** Any developer can clone the repo and generate the exact same project.
- **Clean git history:** No 50KB `.xcodeproj` diffs in PRs.

**Trade-offs:**
- Additional build step: `xcodegen generate` before opening in Xcode.
- Less "visual" editing of project settings (must edit YAML).
- Some advanced Xcode features may require manual `project.pbxproj` tweaks.

**Status:** ✅ Accepted. Has prevented multiple merge conflicts.

---

## ADR 6: App Group for Widget Data Sharing

**Context:** The widget needs real-time data from the main app, but iOS sandboxes app extensions.

**Decision:** Use `com.apple.security.application-groups` with `UserDefaults(suiteName:)`.

**Rationale:**
- **Standard iOS pattern:** Apple-approved mechanism for sharing small data between app and extension.
- **Low latency:** No network or file I/O required. Read is synchronous.
- **Sufficient for widget data:** Only ~20 key-value pairs (glucose, risk, TIR, delta, timestamp).

**Trade-offs:**
- Limited to ~1MB total storage. Not suitable for large history arrays.
- Data is not persistent across device restores if iCloud Keychain is not enabled.

**Status:** ✅ Configured. Real-time data flow pending widget provider update.

---

## ADR 7: No Automated Insulin Dosing

**Context:** A "closed loop" system could theoretically control insulin pumps automatically.

**Decision:** The app is strictly **predictive and advisory**. It never doses insulin, controls pumps, or makes dosing decisions.

**Rationale:**
- **Safety:** FDA clearance for automated dosing requires extensive clinical trials. A student/hobby project cannot safely assume this risk.
- **Liability:** The developer (me) is not a medical device manufacturer.
- **User autonomy:** The app recommends actions ("consider eating 15g carbs"). The user decides.
- **Ethical boundary:** "First, do no harm." False positives in dosing are life-threatening.

**Trade-offs:**
- Less "magical" user experience compared to commercial closed-loop systems (e.g., Tandem Control-IQ, Medtronic 780G).
- Requires user engagement rather than passive automation.

**Status:** ✅ Accepted as non-negotiable design constraint. Every feature is audited against this principle.

---

## ADR 8: Danish UI with English Codebase

**Context:** The app targets Danish users (initially), but the developer works in international environments.

**Decision:** UI is Danish. Code, comments, and documentation are English.

**Rationale:**
- **User comfort:** Danish diabetes terminology ("kulhydrat", "insulin", "blodsukker") is familiar to users. English medical terms increase cognitive load.
- **Code accessibility:** English variable names and comments enable international code review, open-source contribution, and employer evaluation.
- **Future localization:** The codebase is structured for easy localization (`Localizable.strings` can be added later).

**Trade-offs:**
- Bilingual mental model: developer must switch between Danish UX copy and English code logic.
- Some Danish medical terms lack direct English equivalents (e.g., "hypoglykæmi" is clear, but "målområde" vs. "target range" requires context).

**Status:** ✅ Accepted. Working well in practice.

---

*All ADRs are living documents. Revisit when assumptions change.*
