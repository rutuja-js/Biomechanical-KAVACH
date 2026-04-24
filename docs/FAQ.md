# FAQ — Incoming Developer Guide
## Biomechanical Safety System

Read this before touching anything.

---

## Q1: What is Class 0, 1, 2, 3?

These are safety states the system assigns to each joint in real time.

| Class | Name | What It Means | What Happens |
|---|---|---|---|
| 0 | Normal | Joint is well within safe limits | Full assistance, no alerts |
| 1 | Proximity | Joint is approaching a boundary | Assistance tapers, amber alert |
| 2 | Restriction | Joint is at the soft stop zone | No active torque, red alert |
| 3 | Emergency | Hard limit breached or override detected | Motor cut, manual reset required |

**The number only goes up** until the user reverses direction. It never jumps down on its own.

If you see Class 3 in logs, a human must have physically reset the device before operation resumed.

---

## Q2: When does the system block execution?

Execution is blocked (`allowed: false`) in any of these conditions:

- Safety class reaches **2 or 3**
- Sensor confidence drops below threshold (UNCERTAIN state)
- A bilateral boundary conflict is detected (both limits triggered simultaneously — impossible physically, means sensor fault)
- Power loss occurs during Class 2 or 3

When blocked:
- `allowed: false` is set in the JSON output
- `execution_block: true` is set
- Sarathi does not forward the signal to the Gated Bridge
- Motor enters passive mode

---

## Q3: What happens if sensors fail?

Four sensor failure modes are tracked:

| Failure | Detection Method | System Response |
|---|---|---|
| IMU Drift | Gyro continues changing during stationary periods | Switch to encoder-only, reduce torque 70% |
| Encoder Offset | Sudden >2° jump or slow creep | Switch to IMU-only, reduce torque 70% |
| Limb Misalignment | Angle exceeds anatomical max from calibration | Alert user, contract boundaries 10°, reduce torque 50% |
| Strap Shift | Neutral angle drifts >3° over 5 minutes | Alert user to refit, contract boundaries 10% |

If BOTH primary sensors fail → **passive-only mode**. Motor disabled. User can still move the joint freely.

The system never leaves the user unable to move. It only removes assistance, never adds resistance.

Confidence thresholds:
- IMU: must stay above **0.7**
- Encoder: must stay above **0.7**
- Alignment: must stay above **0.8**
- Strap: must stay above **0.75**

If any drops below → UNCERTAIN state → `allowed: false`

---

## Q4: How do I test safely?

### Before any hardware test:

1. **Run simulation mode first** — inject test angles and verify JSON outputs match expected class
2. **Test each class transition individually** — do not test Class 3 first
3. **Verify passive fallback** — simulate sensor failure, confirm motor goes passive not active
4. **Never skip calibration** — ROM values are user-specific. Population defaults are reference only

### Safe test sequence:
```
Step 1: Verify Class 0 output at mid-range angle
Step 2: Increase angle to 85% safe range → expect Class 1
Step 3: Increase to 100% safe range → expect Class 2, passive mode
Step 4: Check manual reset requirement after Class 3
Step 5: Simulate IMU confidence drop → verify UNCERTAIN state triggers
```

### What NOT to do:
- Do not test on a user before simulated validation passes
- Do not bypass confidence checks "just to test motion"
- Do not manually set `allowed: true` in the output — the system must compute it
- Do not test Class 3 recovery without first confirming passive fallback works

---

## Q5: What breaks the system?

These will cause incorrect or dangerous behaviour:

| Action | Why It Breaks Things |
|---|---|
| Skipping user calibration | ROM limits stay at population defaults — may be wrong for this user |
| Modifying confidence thresholds without testing | Lowering them means sensor failures go undetected |
| Removing hysteresis from state machine | Class oscillates rapidly at boundary → undefined behaviour |
| Sending Class 3 reset signal programmatically | Bypasses the manual reset requirement — user is not informed |
| Adding new joint types without updating schema | TANTRA schema expects only `"knee"` or `"elbow"` |
| Changing JSON field names | Sarathi and Bridge parse by exact field name — silent failures |
| Enabling auto-recovery from Class 3 | System was designed to require human confirmation — automation removes that |
| Writing directly to motor controllers from safety layer | Violates the TANTRA gate — safety layer must never touch hardware directly |

---

## Q6: Where do I start?

Read in this order:
1. `docs/HANDOVER_MASTER.md` — understand the system
2. `schema/tantra_output_schema.md` — understand what this system produces
3. `outputs/interface_contract.md` — understand what is guaranteed and what is forbidden
4. `outputs/sarathi_bridge_interface.md` — understand how TANTRA consumes the output
5. `review_packets/review_packet_final.md` — see a real flow end to end

---

## Q7: Who do I talk to if something is unclear?

| Area | Contact |
|---|---|
| Physical joint limits feel wrong | Suraj — Mechanical Systems |
| Actuator / hardware behaviour | Usman — Electronics |
| Log structure / backend integration | Akash — Backend |
| This safety system's logic | Refer to HANDOVER_MASTER.md first |
