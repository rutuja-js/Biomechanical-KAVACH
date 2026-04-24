# HANDOVER MASTER DOCUMENT
## Biomechanical Safety System — Exosuit / Prosthetic (Knee + Elbow)
Version: 1.0 | Status: Final Handover

Author: Rutuja
Receiving Teams: Suraj (Mechanical), Usman (Electronics), Akash (Backend), Incoming Developers

---

## 1. SYSTEM OVERVIEW (Plain Language)

This system is the safety brain for a wearable robotic device that assists knee and elbow joints.

Its only job is to answer one question every few milliseconds:

> **"Is it safe to assist this joint right now — and if so, how much?"**

It reads sensor data (joint angle, speed, measurement confidence), checks it against safe limits, and outputs a structured signal saying either:
- "Yes, assist — but cap torque at X%"
- "No — stop all assistance immediately"

It does NOT control motors. It does NOT diagnose injuries. It only produces decision signals.

Those signals go to TANTRA (Sarathi → Gated Bridge), which is what actually gates hardware commands.

---

## 2. FULL FLOW — INPUT TO OUTPUT

```
[IMU Sensor]    ──┐
[Encoder]       ──┼──→ [Sensor Fusion] ──→ [Confidence Calculation]
[Strap Sensor]  ──┘                                │
                                                   ▼
                                      [Boundary Classification]
                                      Compare angle to ROM limits
                                                   │
                                                   ▼
                                      [State Machine — Class 0–3]
                                                   │
                                                   ▼
                                      [Fallback Logic]
                                      (confidence × class = final state)
                                                   │
                                                   ▼
                                      [JSON Output Signal]
                                      intent / confidence / risk_level
                                      allowed / constraints
                                                   │
                                                   ▼
                                      [TANTRA — Sarathi]
                                      Routes or blocks
                                                   │
                                          ┌─────────────┐
                                          │             │
                                    [Blocked]     [Gated Bridge]
                                    Motor passive   Enforces torque cap
                                                   │
                                                   ▼
                                           [Hardware Layer]
                                           Motor command (if allowed)
```

---

## 3. STATE MACHINE EXPLANATION

The system operates as a 4-class state machine. Each class represents a safety state.

### Class 0 — Normal
- Joint is well within safe range (0–80% of safe assist range)
- Full assistance permitted
- Transparent to user — no alerts

### Class 1 — Boundary Proximity
- Joint is 80–95% of safe range OR velocity will reach boundary within 200ms
- Assistance tapers from 100% down to 30%
- Amber visual indicator, subtle haptic feedback
- User can still move — system is warning, not blocking

### Class 2 — Controlled Restriction
- Joint is 95–105% of safe range OR user pushed against Class 1 resistance
- Zero active torque — passive mode only
- Soft resistance applied (virtual spring — not hard wall)
- Red warning, firm haptic feedback
- Motor emergency cutoff circuit armed

### Class 3 — Emergency Fallback
- Joint past 105% safe range OR 3+ seconds of force against Class 2 OR sensor drops in Class 2
- Motor power cut immediately
- Full passive mode (user can still move limb freely — no active locking)
- Flashing red + audio alert
- **Requires manual reset** — cannot auto-recover

### Rules:
- Classes can only escalate (0→1→2→3), never skip downward automatically
- Exit thresholds are lower than entry thresholds (hysteresis) to prevent oscillation
- Multi-joint: system class = whichever joint is in the highest class

---

## 4. SAFETY PHILOSOPHY — WHY DECISIONS EXIST

### Why graduated response instead of a hard stop?
An abrupt stop at the boundary creates a jerk force on the joint that can itself cause injury. Progressive resistance gives the user natural feedback and time to self-correct.

### Why passive fallback instead of active braking on Class 3?
If the system locks the joint, the user cannot move freely. In an emergency — a fall, a stumble — a locked joint is dangerous. The system always chooses to let the user move over preventing movement.

### Why require manual reset after Class 3?
An automatic reset could re-enable assistance before the cause of the emergency is understood. The user must physically return to neutral and confirm the reset. This forces awareness.

### Why does sensor uncertainty trigger the same response as a safety violation?
If the system doesn't know where the joint is, it cannot enforce boundaries. Uncertain measurement is treated as a worst-case scenario — reduce capability, expand safety margins.

### Why does the system never send motor commands directly?
Separation of responsibility. The safety layer decides. TANTRA executes. This ensures no single bug in the safety classification can directly drive hardware — there is always a gate.

---

## 5. KNOWN LIMITATIONS

| Limitation | Detail |
|---|---|
| ROM values are population averages | Must be calibrated per user before first use |
| Velocity-only prediction can false-trigger | A fast movement that self-reverses may briefly enter Class 1 unnecessarily |
| Sensor uncertainty detection is not instant | Up to 10 seconds before confidence metrics stabilize post-recovery |
| Bilateral asymmetry detection requires both limbs active | Single-joint mode cannot detect alignment faults via comparison |
| Environmental factors not accounted for | Extreme temperature, water exposure, or EMI may degrade sensor performance beyond what confidence metrics detect |
| System does not account for tissue loading history | A joint within ROM limits may still be at risk if already fatigued or injured |

---

## 6. WHAT NOT TO CHANGE

The following must NOT be modified without mechanical and electronics sign-off:

| Item | Why It's Locked |
|---|---|
| Hard limit values (knee -5°/160°, elbow 0°/150°) | Derived from anatomical max — changing risks injury |
| Confidence thresholds (IMU 0.7, Encoder 0.7, Alignment 0.8, Strap 0.75) | Tuned against failure mode data — lowering risks missed detections |
| Class 3 manual reset requirement | Intentional safety gate — automation removes user awareness |
| Passive fallback on any failure | Core safety guarantee — never leave user without joint mobility |
| Monotonic escalation rule | Prevents rapid oscillation and gaming of the state machine |
| Output format (JSON schema) | TANTRA expects exact field names — changes break integration |

---

## CONTACTS

| Team | Person | Responsible For |
|---|---|---|
| Mechanical | Suraj | Physical motion validation |
| Electronics | Usman | Actuator constraint mapping |
| Backend | Akash | Log integration + structured outputs |

For questions on this document → read FAQ.md first.
