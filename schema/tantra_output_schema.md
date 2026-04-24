# TANTRA OUTPUT SCHEMA — Biomechanical Safety System
Version: 1.0 | Status: Integration Ready

---

## PURPOSE

This schema defines the ONLY output format the biomechanical safety system is permitted to produce.
It is consumed by TANTRA (Sarathi + Gated Bridge).

The system does NOT send motor commands.
The system does NOT control hardware.
The system ONLY produces structured decision signals in the format below.

---

## STANDARD OUTPUT SIGNAL

Every evaluation cycle produces one JSON object per joint.

```json
{
  "timestamp_ms": 1714900000123,
  "joint": "knee",
  "safety_class": 1,
  "intent": "flexion_boundary_approach",
  "confidence": 0.91,
  "risk_level": "MODERATE",
  "recommended_action": "reduce_assistance",
  "allowed": true,
  "constraints": {
    "max_torque_percent": 45,
    "control_mode": "tapered_assist",
    "boundary_distance_deg": 12.5
  }
}
```

---

## FIELD DEFINITIONS

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp_ms` | integer | YES | Unix timestamp in milliseconds |
| `joint` | string | YES | `"knee"` or `"elbow"` |
| `safety_class` | integer | YES | 0, 1, 2, or 3 |
| `intent` | string | YES | What boundary event is occurring (see Intent Codes) |
| `confidence` | float | YES | Sensor confidence 0.0–1.0 |
| `risk_level` | string | YES | `"NONE"` / `"LOW"` / `"MODERATE"` / `"HIGH"` / `"CRITICAL"` |
| `recommended_action` | string | YES | What the system recommends TANTRA do |
| `allowed` | boolean | YES | Whether motion/assist is permitted |
| `constraints` | object | CONDITIONAL | Present only when `allowed: true` |
| `execution_block` | boolean | CONDITIONAL | Present only when `allowed: false` |

---

## INTENT CODES

| Intent String | Meaning |
|---|---|
| `"normal_operation"` | Class 0 — no boundary event |
| `"flexion_boundary_approach"` | Class 1 — nearing flexion limit |
| `"extension_boundary_approach"` | Class 1 — nearing extension limit |
| `"flexion_limit_reached"` | Class 2 — at or past flexion soft stop |
| `"extension_limit_reached"` | Class 2 — at or past extension soft stop |
| `"emergency_fallback"` | Class 3 — hard limit breached or override |
| `"sensor_uncertainty"` | UNCERTAIN state — confidence degraded |
| `"bilateral_boundary_conflict"` | Invalid sensor state — both limits triggered |

---

## ALLOWED = TRUE (Constrained Execution)

When motion is permitted, constraints object is mandatory:

```json
{
  "allowed": true,
  "constraints": {
    "max_torque_percent": 45,
    "control_mode": "tapered_assist",
    "boundary_distance_deg": 12.5
  }
}
```

### control_mode values:
| Value | Meaning |
|---|---|
| `"full_assist"` | Class 0 — 100% torque |
| `"tapered_assist"` | Class 1 — progressive reduction |
| `"passive_only"` | Class 2 — no active torque |

---

## ALLOWED = FALSE (Execution Blocked)

When motion is denied, execution_block is mandatory:

```json
{
  "allowed": false,
  "execution_block": true,
  "constraints": null
}
```

This signal means: do not execute. No torque. No command. Passive only.

---

## FULL EXAMPLES BY CLASS

### Class 0 — Normal Operation
```json
{
  "timestamp_ms": 1714900000001,
  "joint": "knee",
  "safety_class": 0,
  "intent": "normal_operation",
  "confidence": 0.97,
  "risk_level": "NONE",
  "recommended_action": "full_assist",
  "allowed": true,
  "constraints": {
    "max_torque_percent": 100,
    "control_mode": "full_assist",
    "boundary_distance_deg": 55.0
  }
}
```

### Class 1 — Boundary Proximity
```json
{
  "timestamp_ms": 1714900000250,
  "joint": "knee",
  "safety_class": 1,
  "intent": "flexion_boundary_approach",
  "confidence": 0.92,
  "risk_level": "MODERATE",
  "recommended_action": "reduce_assistance",
  "allowed": true,
  "constraints": {
    "max_torque_percent": 45,
    "control_mode": "tapered_assist",
    "boundary_distance_deg": 8.2
  }
}
```

### Class 2 — Controlled Restriction
```json
{
  "timestamp_ms": 1714900000500,
  "joint": "elbow",
  "safety_class": 2,
  "intent": "extension_limit_reached",
  "confidence": 0.88,
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
  "confidence": 0.61,
  "risk_level": "CRITICAL",
  "recommended_action": "emergency_stop",
  "allowed": false,
  "execution_block": true,
  "constraints": null
}
```

### UNCERTAIN State — Sensor Degraded
```json
{
  "timestamp_ms": 1714900001000,
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

## WHAT THIS SCHEMA NEVER CONTAINS

- Motor voltage or current commands
- PID parameters
- Raw sensor readings
- Clinical assessments
- User identity data
