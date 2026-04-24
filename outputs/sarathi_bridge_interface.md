# SARATHI + GATED BRIDGE INTERFACE DEFINITION
Version: 1.0 | Status: Integration Ready

---

## OVERVIEW

The biomechanical safety system produces structured JSON signals.
These signals flow into TANTRA via two components:

```
[Biomech Safety] → [Sarathi] → [Gated Bridge] → [Hardware Layer]
```

- **Sarathi** — receives intent + risk + confidence, makes routing decision
- **Gated Bridge** — receives allow/deny + constraints, gates hardware execution

Neither component sends raw commands to hardware.
The Bridge is the last software gate before physical actuation.

---

## WHAT SARATHI RECEIVES

Sarathi consumes the top-level decision fields from the output signal:

```json
{
  "timestamp_ms": 1714900000250,
  "joint": "knee",
  "safety_class": 1,
  "intent": "flexion_boundary_approach",
  "confidence": 0.92,
  "risk_level": "MODERATE",
  "recommended_action": "reduce_assistance",
  "allowed": true
}
```

### Sarathi Decision Logic

| risk_level | confidence | Sarathi Action |
|---|---|---|
| NONE | > 0.8 | Pass through to Bridge — full execution |
| LOW | > 0.8 | Pass through to Bridge — constrained |
| MODERATE | 0.7–0.8 | Pass through to Bridge — constrained + flag |
| HIGH | any | Pass through to Bridge — deny, log event |
| CRITICAL | any | Block at Sarathi — do not forward to Bridge |

If `allowed: false` — Sarathi does NOT forward to Bridge. It returns an immediate block.
If `allowed: true` — Sarathi forwards to Bridge with constraint envelope.

---

## WHAT GATED BRIDGE RECEIVES

Bridge only receives signals where Sarathi has passed `allowed: true`.
It receives the full constraints object:

```json
{
  "timestamp_ms": 1714900000250,
  "joint": "knee",
  "allowed": true,
  "constraints": {
    "max_torque_percent": 45,
    "control_mode": "tapered_assist",
    "boundary_distance_deg": 8.2
  }
}
```

### Bridge Behavior

**IF `allowed: true`:**

Bridge enforces constraints before any hardware signal is passed:
- Caps torque output at `max_torque_percent`
- Sets control mode flag for motor controller
- Passes constraint envelope to hardware layer

**IF `allowed: false` (should not reach Bridge, but defensive check):**
```json
{
  "execution_block": true
}
```
Bridge must treat any signal with `execution_block: true` as a hard stop.
No command is forwarded. Motor is set to passive mode.

---

## FULL FLOW BY CLASS

### Class 0 → Full Execution
```
Safety System → Sarathi (NONE risk, passes) → Bridge (100% torque, full_assist) → Hardware
```

### Class 1 → Constrained Execution
```
Safety System → Sarathi (MODERATE risk, passes + flags) → Bridge (45% torque, tapered_assist) → Hardware
```

### Class 2 → Execution Blocked
```
Safety System → Sarathi (HIGH risk, allowed=false) → BLOCK → Bridge not reached → Motor passive
```

### Class 3 → Emergency Stop
```
Safety System → Sarathi (CRITICAL, allowed=false) → BLOCK at Sarathi → Bridge not reached → Motor cutoff
```

### UNCERTAIN → Emergency Stop
```
Safety System (confidence < threshold) → allowed=false → Sarathi BLOCK → Motor cutoff → Alert user
```

---

## LOGGING CONTRACT

Every signal processed by Sarathi must be logged with:
- Full JSON payload
- Sarathi routing decision (pass / block)
- Bridge execution result (if passed)
- Timestamp at each hop

Logs are forwarded to Akash (Backend) via structured event stream.

---

## WHAT NEITHER SARATHI NOR BRIDGE DOES

- Does not compute joint angles
- Does not make safety classifications
- Does not interpret sensor data
- Does not produce feedback commands (haptic/audio/visual)
- Does not modify the safety class
- Does not override an `allowed: false` signal under any condition
