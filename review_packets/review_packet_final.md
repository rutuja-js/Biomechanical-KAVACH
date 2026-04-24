# REVIEW PACKET — FINAL
## Biomechanical Safety System
Version: 1.0 | Status: Handover Complete

---

## 1. ENTRY POINT

The system starts here:

```
/biomech_safety/schema/tantra_output_schema.md
```

This file defines everything the system produces.
If you understand the output schema, you understand the system's job.

The system reads sensors → classifies joint state → emits one JSON signal per joint per cycle.
That is all it does.

---

## 2. CORE 3 FILES

| File | What It Is | Why It Matters |
|---|---|---|
| `schema/tantra_output_schema.md` | Output format definition | Every signal the system produces must match this exactly |
| `outputs/interface_contract.md` | Integration guarantees | Defines what is safe, what is forbidden, what hardware receives |
| `docs/HANDOVER_MASTER.md` | Full system explanation | Architecture, philosophy, state machine, limitations |

Supporting files:
- `outputs/sarathi_bridge_interface.md` — how TANTRA consumes signals
- `docs/FAQ.md` — incoming developer quick reference

---

## 3. REAL FLOW — END TO END

**Scenario: User bends knee toward flexion limit during assisted walking**

```
T=0ms    Knee angle: 72° | Velocity: +8°/s | IMU confidence: 0.95 | Encoder confidence: 0.93
         → Boundary check: 72° = 65.5% of safe range (110°)
         → Class 0 (below 80% threshold = 88°)
         → Output: allowed=true, full_assist, 100% torque

T=150ms  Knee angle: 91° | Velocity: +12°/s | IMU confidence: 0.94
         → 91° = 82.7% of safe range → Class 1 triggered
         → Velocity check: 91° + (12°/s × 0.2s) = 103.4° → boundary in <200ms
         → Output: allowed=true, tapered_assist, 45% torque, amber alert

T=300ms  Knee angle: 106° | Velocity: +4°/s | IMU confidence: 0.93
         → 106° = 96.4% of safe range → Class 2 triggered
         → Output: allowed=false, execution_block=true, passive_only, red alert

T=310ms  Sarathi receives signal → allowed=false → blocks → Bridge not reached
         Motor enters passive mode
         User feels firm but yielding resistance (virtual spring)

T=600ms  Knee angle: 98° (user reversing) | Velocity: -6°/s
         → 98° = 89.1% — still above Class 2 exit threshold (90% = 99°)
         → Class 2 maintained

T=900ms  Knee angle: 96° → 87.3% → below 90% exit threshold
         → Class 2 exits → Class 1 re-entered
         → allowed=true, tapered_assist resumed

T=1500ms Knee angle: 80° → 72.7% → below Class 1 exit (75% = 82.5°)
         → Class 0 restored → full assist resumed
```

---

## 4. SAMPLE OUTPUT JSON — ALL STATES

### Class 0 — Normal Operation
```json
{
  "timestamp_ms": 1714900000001,
  "joint": "knee",
  "safety_class": 0,
  "intent": "normal_operation",
  "confidence": 0.95,
  "risk_level": "NONE",
  "recommended_action": "full_assist",
  "allowed": true,
  "constraints": {
    "max_torque_percent": 100,
    "control_mode": "full_assist",
    "boundary_distance_deg": 38.0
  }
}
```

### Class 1 — Boundary Proximity (velocity-triggered)
```json
{
  "timestamp_ms": 1714900000150,
  "joint": "knee",
  "safety_class": 1,
  "intent": "flexion_boundary_approach",
  "confidence": 0.94,
  "risk_level": "MODERATE",
  "recommended_action": "reduce_assistance",
  "allowed": true,
  "constraints": {
    "max_torque_percent": 45,
    "control_mode": "tapered_assist",
    "boundary_distance_deg": 7.8
  }
}
```

### Class 2 — Controlled Restriction
```json
{
  "timestamp_ms": 1714900000300,
  "joint": "knee",
  "safety_class": 2,
  "intent": "flexion_limit_reached",
  "confidence": 0.93,
  "risk_level": "HIGH",
  "recommended_action": "restrict_motion",
  "allowed": false,
  "execution_block": true,
  "constraints": null
}
```

### Class 3 — Emergency Fallback
```json
{
  "timestamp_ms": 1714900000750,
  "joint": "knee",
  "safety_class": 3,
  "intent": "emergency_fallback",
  "confidence": 0.91,
  "risk_level": "CRITICAL",
  "recommended_action": "emergency_stop",
  "allowed": false,
  "execution_block": true,
  "constraints": null
}
```

### UNCERTAIN — Sensor Confidence Degraded
```json
{
  "timestamp_ms": 1714900001200,
  "joint": "elbow",
  "safety_class": 2,
  "intent": "sensor_uncertainty",
  "confidence": 0.58,
  "risk_level": "HIGH",
  "recommended_action": "emergency_stop",
  "allowed": false,
  "execution_block": true,
  "constraints": null
}
```

---

## 5. FAILURE CASES — INPUT → OUTPUT → SYSTEM BEHAVIOUR

### F1: IMU Drift Detected

**Input:**
- Gyro continues integrating during confirmed stationary period
- IMU angle vs encoder angle diverges to 6°
- IMU_confidence drops to 0.62

**Expected Output:**
```json
{
  "intent": "sensor_uncertainty",
  "confidence": 0.62,
  "risk_level": "HIGH",
  "allowed": false,
  "execution_block": true
}
```

**System Behaviour:**
- UNCERTAIN state entered
- Torque reduced to 30% → then 0% (allowed=false)
- Boundary margins contracted by 10°
- User notified: "SENSOR UNCERTAINTY – REDUCED ASSIST"
- Recovery: hold stationary 10s → gyro recalibrated → compare to encoder → if within 2°, resume

---

### F2: Encoder Sudden Jump

**Input:**
- Encoder reads 85° at T=0
- At T=10ms encoder reads 91° (6° jump, no velocity signature)
- Encoder_confidence drops to 0.55

**Expected Output:**
```json
{
  "intent": "sensor_uncertainty",
  "confidence": 0.55,
  "risk_level": "CRITICAL",
  "allowed": false,
  "execution_block": true
}
```

**System Behaviour:**
- Immediate Class 3 escalation
- Motor power cut
- Log entry: encoder_jump, delta=6°, timestamp
- Recovery: passive range-of-motion check, if encoder tracks IMU smoothly → resume in IMU-only degraded mode

---

### F3: Bilateral Boundary Conflict

**Input:**
- System detects joint simultaneously approaching extension AND flexion limit
- Physically impossible — sensor fault

**Expected Output:**
```json
{
  "intent": "bilateral_boundary_conflict",
  "confidence": 0.0,
  "risk_level": "CRITICAL",
  "allowed": false,
  "execution_block": true
}
```

**System Behaviour:**
- Immediate Class 3
- Log flagged as sensor fault
- Full recalibration required before reset accepted

---

### F4: Boundary Oscillation

**Input:**
- Detected angle oscillates ±3° around safety boundary
- More than 3 transitions in 5 seconds

**Expected Output:**
```json
{
  "intent": "sensor_uncertainty",
  "confidence": 0.61,
  "risk_level": "HIGH",
  "allowed": false,
  "execution_block": true
}
```

**System Behaviour:**
- Interpreted as sensor noise, not real motion
- Locked to Class 3
- Diagnostic check required before reset

---

### F5: Power Loss During Class 2

**Input:**
- System in Class 2, passive mode
- Power interrupted

**Expected Output:**
- No output (system offline)

**System Behaviour:**
- Motor fails to passive (mechanical backdrive — user can move freely)
- No active locking or braking
- State written to non-volatile memory before shutdown
- On restore: Class 3 required, manual reset before operation

---

## 6. PROOF — VERIFICATION CHECKLIST

The following must be confirmed before this system is considered validated for deployment:

### Functional
- [ ] Class 0 output confirmed at 50% safe range angle
- [ ] Class 1 output confirmed at 85% safe range angle
- [ ] Class 1 velocity trigger confirmed at 190ms-to-boundary simulation
- [ ] Class 2 output confirmed at 100% safe range angle
- [ ] Class 3 output confirmed at 107% safe range angle
- [ ] Manual reset required after Class 3 confirmed
- [ ] Hysteresis verified — Class 2 exit at 90%, not 95%
- [ ] Multi-joint class = max of joint classes confirmed

### Failure Modes
- [ ] IMU drift injection → UNCERTAIN state triggered
- [ ] Encoder jump injection → Class 3 triggered
- [ ] Both sensors disabled → passive-only mode confirmed
- [ ] Bilateral conflict → Class 3 + sensor fault log confirmed
- [ ] Power loss in Class 2 → passive fallback confirmed (no active brake)
- [ ] Rapid oscillation (>5 transitions/second) → Class 3 lock confirmed

### Integration
- [ ] JSON output schema validated against tantra_output_schema.md
- [ ] Sarathi correctly blocks all `allowed: false` signals
- [ ] Gated Bridge enforces `max_torque_percent` cap
- [ ] All state transitions logged with millisecond timestamp
- [ ] Akash backend receiving structured log stream

---

## HANDOVER STATUS

| Deliverable | Status |
|---|---|
| Output schema (TANTRA-ready JSON) | ✅ Complete |
| Sarathi + Bridge interface definition | ✅ Complete |
| Interface contract | ✅ Complete |
| Handover master doc | ✅ Complete |
| FAQ for incoming developers | ✅ Complete |
| This review packet | ✅ Complete |
| Folder structure | ✅ Complete |

**System is ready for integration.**
