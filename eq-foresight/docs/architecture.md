# Architecture Deep Dive

This document provides a high-level technical overview of Eq Foresight's architecture. The full implementation details, source code, and ML training scripts are maintained in private repositories.

---

## 1. Philosophy: On-Device, Privacy-First, Reactive

Three principles guided every architectural decision:

1. **On-Device AI:** All model inference happens locally. No network call is required for predictions. The app works offline if HealthKit data is available.
2. **Privacy-First:** We do not collect, store, or transmit user data. Nightscout integration is read-only + optional write (treatment logs). The app does not phone home.
3. **Reactive, Not Closed-Loop:** The app warns and recommends. It never doses insulin or controls pumps. This is a critical safety boundary.

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User (Patient)                          │
└──────────────────────┬──────────────────────────────────────────┘
                       │
┌──────────────────────┴──────────────────────────────────────────┐
│                      Eq Foresight iOS App                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                        SwiftUI Layer                     │   │
│  │  DashboardView  │  HistoryView  │  SettingsView  │ Widget│   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     ViewModel Layer                      │   │
│  │            DashboardViewModel (MVVM, @MainActor)         │   │
│  │         ┌────────────────┐    ┌────────────────────┐     │   │
│  │         │  AppStorage    │    │  UserDefaults      │     │   │
│  │         │  (physiology)  │    │  (risk history,    │     │   │
│  │         │  (targets)     │    │   alarm cooldown)  │     │   │
│  │         └────────────────┘    └────────────────────┘     │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     Service Layer                        │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐  │   │
│  │  │ Nightscout   │  │ HealthKit    │  │ Notification   │  │   │
│  │  │ Service      │  │ Service      │  │ Manager        │  │   │
│  │  │ (SHA1 auth)  │  │ (singleton)  │  │ (escalation    │  │   │
│  │  │              │  │              │  │  + cooldown)   │  │   │
│  │  └──────────────┘  └──────────────┘  └────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     Model Layer                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐  │   │
│  │  │  DataBuffer  │  │  Inference   │  │  HypoPredictor │  │   │
│  │  │  (23 feat.   │  │  Engine      │  │  (4 CoreML     │  │   │
│  │  │   48 steps)  │  │              │  │   models)      │  │   │
│  │  └──────────────┘  └──────────────┘  └────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                       │
┌──────────────────────┴──────────────────────┐
│           External Data Sources             │
│  ┌────────────┐  ┌────────────┐  ┌────────┐ │
│  │ Nightscout │  │  HealthKit │  │ CGM    │ │
│  │  Server    │  │  (Apple)   │  │ Device │ │
│  └────────────┘  └────────────┘  └────────┘ │
└─────────────────────────────────────────────┘
```

---

## 3. The ML Cascade: 4 Horizons, 4 Models

Most ML apps use a single model. Eq Foresight uses **4 specialized models** because prediction accuracy degrades non-linearly with horizon length.

| Model | Horizon | Use Case | Trigger Threshold |
|-------|---------|----------|-------------------|
| `h1` | 5 min | Immediate warning | Critical (urgent) |
| `h2` | 10 min | Early warning | Critical (urgent) |
| `h3` | 15 min | Planning window | Warning (elevated) |
| `h6` | 30 min | Long-term risk | Advisory (monitor) |

**Why a cascade?**
- A single 30-minute model would miss short-term dynamics.
- A single 5-minute model would produce false alarms at longer horizons.
- The cascade allows **context-aware escalation**: h1+h2 trigger urgent alerts; h3+h6 provide context.

**Input tensor:** `[batch=1, timesteps=48, features=23]` (float32)
**Output:** `hypo_probability[0]` as Double, calibrated to mmol/L with iso-calibrators.

---

## 4. Feature Engineering: 23 Features from 4 Data Streams

The `DataBuffer` maintains a rolling window and computes:

| Category | Features | Source |
|----------|----------|--------|
| Glucose dynamics | `glucose`, `delta`, `delta2`, `roll_mean`, `roll_std`, `slope_30` | CGM (Nightscout or HealthKit) |
| Physiology | `heart_rate`, `steps` (placeholder), `gsr` (placeholder) | HealthKit |
| Time context | `time_sin`, `time_cos`, `is_nocturnal`, `time_into_night` | Derived from timestamp |
| Treatments | `bolus_dose`, `carbs_g`, `mins_since_bolus`, `mins_since_meal` | Nightscout treatments |
| Pharmacokinetics | `iob_units`, `cob_g`, `iob_velocity`, `basal_rate` | Derived from treatment history using Walsh bilinear IOB model |

**Key design decisions:**
- **48 timesteps = 4 hours** of 5-minute data. This captures meal absorption and insulin decay curves.
- **Nocturnal features:** Hypoglycemia patterns differ dramatically between day and night. The model learns this via `is_nocturnal` and `time_into_night`.
- **IOB/COB modeling:** Instead of raw insulin/carb inputs, we model the *active* amounts using pharmacokinetic kernels. This dramatically improves prediction accuracy.

---

## 5. Alert Escalation System

Not all risks are equal. The notification system uses **4 tiers** with intelligent hysteresis:

```
Probability
    │
0.95 ┤ ████████████████  Pump Stop Alert  (critical sound + action buttons)
     │
0.80 ┤ ██████████        Alarm Alert      (strong warning + haptic)
     │
0.65 ┤ ██████            Warning Alert    (subtle notification)
     │                    (Night mode: 0.65 threshold)
0.40 ┤ ██                Monitoring       (no notification, shown in UI)
     │
     └──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────→ Time
           5m     10m    15m    20m    25m    30m
```

**Cooldown rules:**
- 15-minute cooldown per escalation level prevents notification spam.
- Escalating from Warning → Alarm resets the lower level's cooldown.
- Night mode (22:00–07:00) lowers the threshold to 0.65 for sharper monitoring.

---

## 6. Background Processing

iOS is strict about background execution. Eq Foresight uses a multi-pronged strategy:

| Trigger | Mechanism | Action |
|---------|-----------|--------|
| Every 5 min | `Timer` (foreground) | Poll HealthKit + Nightscout, refresh UI |
| New heart rate | `HKObserverQuery` | Throttled to 5-min intervals → trigger inference |
| iOS decides | `BGTaskScheduler` | `handleAppRefresh` with 25s semaphore timeout |
| Widget refresh | `WidgetKit` | Reads shared `UserDefaults` (App Group) |

**Safety guard:** If the latest 15 minutes lack real CGM data, inference is **refused**. The app shows a "CGM gap" warning rather than guessing.

---

## 7. Data Flow: From Sensor to Prediction

```
[CGM Device] ──→ [Nightscout] ──→ [NightscoutService] ────┐
                                                          │
[Apple Watch] ──→ [HealthKit] ──→ [HealthKitService] ─────│ ──→ [DataBuffer]
                                                          │       │
                                                          │       ▼
                                                          │  [buildFeatureMatrix]
                                                          │       │
                                                          │       ▼
                                                          │  [InferenceEngine]
                                                          │       │
                                                          │       ▼
                                                          │  [HypoPredictor]
                                                          │       │
                                                          │       ▼
                                                          │  [DashboardViewModel]
                                                          │       │
                                                          │       ├──→ [SwiftUI View]
                                                          │       ├──→ [Notification]
                                                          │       └──→ [Widget + Live Activity]
```

---

## 8. Build & Deployment

The project uses **XcodeGen** for reproducible builds:

```yaml
# project.yml (simplified)
targets:
  ClosedLoopRing:
    type: application
    platform: iOS
    deploymentTarget: "17.0"
    sources: [ClosedLoopRing]
    dependencies: [target: ClosedLoopRingWidget]
    entitlements:
      com.apple.security.application-groups: [group.com.andy.ClosedLoopRing]
```

This ensures:
- Clean `.gitignore` (no checked-in `.xcodeproj`)
- Easy bundle ID changes
- Reproducible builds across machines

---

## 9. Testing Strategy

- **Unit tests:** `DataBuffer` feature matrix correctness (verified against Python `FEATURE_COLS`)
- **Integration tests:** `NightscoutService` mock mode, `InferenceEngine` with known inputs
- **Device tests:** CoreML models only run on physical devices (not Simulator)
- **Manual QA:** Real-world CGM data validation, alarm cooldown verification

---

## 10. Future Roadmap

| Priority | Feature | Technical Approach |
|----------|---------|-------------------|
| 🔴 High | Apple Watch app | WatchKit + WatchConnectivity for 1-tap carb logging |
| 🔴 High | Widget real-time data | App Group `UserDefaults` (already configured) |
| 🟡 Medium | Siri Shortcuts | `AppIntent` for voice-activated logging |
| 🟡 Medium | Workout auto-detection | `HKWorkoutType` observation → auto Activity Mode |
| 🟢 Low | Step count integration | Retrain model with 5-min step aggregates |
| 🟢 Low | PDF clinical reports | `PDFKit` + `UIGraphicsPDFRenderer` |

---

*Last updated: June 2026 · Architecture version 1.2*
