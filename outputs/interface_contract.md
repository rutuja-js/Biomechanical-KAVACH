# INTERFACE CONTRACT — Biomechanical Safety System
Version: 1.0 | Status: Final

Parties: Biomechanical Safety Layer ↔ TANTRA (Sarathi + Gated Bridge) ↔ Hardware Layer

---

## WHAT THIS DOCUMENT IS

This is a binding interface contract.
It defines exactly what the hardware layer receives, what is guaranteed safe, and what must never be sent.

Any implementation that violates this contract is not compliant with this system.

---

## SECTION 1 — WHAT HARDWARE LAYER RECEIVES

The hardware layer receives ONLY the following, forwarded by the Gated Bridge:

| Field | Type | Description |
|---|---|---|
| `max_torque_percent` | integer (0–100) | Maximum torque as percentage of motor capacity |
| `control_mode` | string | One of: `full_assist`, `tapered_assist`, `passive_only` |
| `execution_block` | boolean | If true — no command forwarded, motor goes passive |

**Nothing else reaches hardware.**

The hardware layer does not receive:
- Joint angles
- Confidence scores
- Risk levels
- Safety class integers
- Intent strings
- Sensor readings

These fields exist only within the safety system and TANTRA. They are not passed downstream.

---

## SECTION 2 — WHAT IS GUARANTEED SAFE

The following guarantees are made by this system when operating correctly:

### G1 — Hard Limit Enforcement
No powered assistance will be sent if the joint has reached or exceeded its hard limit:
- Knee: -5° (hyperextension) to 160° (flexion)
- Elbow: 0° (full extension) to 150° (flexion)

### G2 — Graduated Response
The system will never jump directly from full assist to emergency stop without passing through intermediate states, unless:
- Sensor confidence drops below threshold
- Hard limit is breached directly (rapid movement)
- Bilateral boundary conflict is detected

### G3 — Passive Fallback on Failure
If ANY of the following occur, the system immediately enters passive mode:
- Sensor confidence < 0.7 (IMU or Encoder)
- Class 3 triggered
- Power interruption
- Bilateral sensor conflict

The motor will never actively resist the user. It will only go passive (backdrivable).

### G4 — No Direct Motor Commands
The safety system never writes directly to motor controllers.
All commands pass through the Gated Bridge, which enforces this contract.

### G5 — Torque Limits by Class

| Safety Class | Max Torque |
|---|---|
| Class 0 | 100% |
| Class 1 | 30–100% (tapered linearly) |
| Class 2 | 0% |
| Class 3 | 0% (motor cut) |
| UNCERTAIN | 0% |

---

## SECTION 3 — WHAT MUST NEVER BE SENT

The following must NEVER appear in any signal passed to the Gated Bridge or hardware layer:

| Prohibited Signal | Reason |
|---|---|
| Torque > 100% of calibrated max | Exceeds mechanical safety margin |
| `control_mode: full_assist` when `safety_class > 0` | Contradiction — would bypass classification |
| Any command when `execution_block: true` | Overrides safety block — not permitted |
| Active braking command in Class 3 | Must be passive only — user must be able to move freely |
| Commands during UNCERTAIN state | Confidence too low to trust boundary enforcement |
| Commands without a valid `timestamp_ms` | Stale data — origin unknown |

---

## SECTION 4 — FAILURE BEHAVIOUR CONTRACT

| Failure Type | System Behaviour | Hardware State |
|---|---|---|
| Sensor confidence drops < 0.7 | UNCERTAIN state, `allowed: false` | Passive |
| Class 3 triggered | `execution_block: true` | Motor power cut |
| Power loss during Class 2/3 | Fail to passive | Mechanically backdrivable |
| Bilateral boundary conflict | Immediate Class 3 | Motor power cut |
| Signal missing `timestamp_ms` | Rejected — not forwarded | No change |
| Bridge receives `allowed: false` | Hard stop — no command issued | Passive |

---

## SECTION 5 — WHAT CANNOT BE CHANGED WITHOUT REVIEW

The following are locked. Any modification requires sign-off from Suraj (Mechanical) + Usman (Electronics):

1. Hard limit values (knee -5°/160°, elbow 0°/150°)
2. Confidence thresholds (IMU <0.7, Encoder <0.7, Alignment <0.8, Strap <0.75)
3. Class 3 manual reset requirement
4. Passive fallback behaviour on power loss
5. Prohibition on direct motor commands from safety layer

---

## SIGNOFF

This contract is authoritative for all integration work on this system.
Questions → refer to HANDOVER_MASTER.md
Incoming developers → refer to FAQ.md
