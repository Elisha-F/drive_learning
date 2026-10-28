# Standard telegram 3: quick reference (S210)

Telegram 3 = **speed setpoint (32-bit) + one encoder (position actual value)**.
It is the usual telegram for an S210 driven by a S7-1500 positioning or speed axis
technology object. The position is calculated in the PLC from `G1_XIST1`.

> Bit meanings below are the PROFIdrive/SINAMICS standard ones. Device-specific bits
> (marked "device specific") can differ between drive families and firmware versions.
> Always check the *SINAMICS S210 operating instructions* and the parameter list for your
> firmware before you rely on them with real hardware.

## Layout

| PZD | PLC → drive | | Drive → PLC | |
|----|-------------|---|-------------|---|
| 1 | `STW1` | control word 1 | `ZSW1` | status word 1 |
| 2–3 | `NSOLL_B` | speed setpoint, 32 bit | `NIST_B` | speed actual, 32 bit |
| 4 | `STW2` | control word 2 | `ZSW2` | status word 2 |
| 5 | `G1_STW` | encoder 1 control word | `G1_ZSW` | encoder 1 status word |
| 6–7 | | | `G1_XIST1` | position actual value 1 |
| 8–9 | | | `G1_XIST2` | position actual value 2 |

Sizes: PLC → drive is 5 words (10 bytes) and drive → PLC is 9 words (18 bytes).

## Speed scaling

```
NSOLL_B = speed_rpm / p2000 * 16#4000_0000      (16#4000_0000 = 1073741824 = 100 %)
```

With p2000 = 3000 rpm, 1500 rpm gives 16#2000_0000, and -3000 rpm gives -16#4000_0000.
The PLC and the drive must use the **same p2000**. If they don't, every speed is wrong
by that ratio, which is a classic commissioning mistake.

## STW1: control word 1

| Bit | Meaning (1 = …) |
|----|------------------|
| 0 | ON (0 = OFF1: ramp down, then pulses off) |
| 1 | No OFF2 (0 = coast stop, pulses off immediately) |
| 2 | No OFF3 (0 = quick stop with the OFF3 ramp) |
| 3 | Enable operation (0 = pulses off) |
| 4 | Enable ramp-function generator (0 = RFG output set to 0) |
| 5 | Continue ramp-function generator (0 = freeze RFG) |
| 6 | Enable speed setpoint (0 = setpoint 0, ramps down) |
| 7 | Acknowledge faults (**rising edge**) |
| 8, 9 | Device specific / reserved |
| 10 | Control by PLC (0 = drive ignores STW1 and setpoint) |
| 11–15 | Device specific |

Typical values:

| STW1 | Meaning |
|------|---------|
| `16#047E` | Ready, OFF1 active, used to leave "switching on inhibited" |
| `16#047F` | Run |
| `16#04FE` | `047E` + acknowledge bit (pulse bit 7, then remove it) |
| `16#047C` | OFF2 (coast) |
| `16#047A` | OFF3 (quick stop) |

## ZSW1: status word 1

| Bit | Meaning (1 = …) |
|----|------------------|
| 0 | Ready for switching on |
| 1 | Ready for operation |
| 2 | Operation enabled (drive follows the setpoint) |
| 3 | Fault present |
| 4 | No coast-down active (OFF2 inactive) |
| 5 | No quick stop active (OFF3 inactive) |
| 6 | Switching on inhibited |
| 7 | Alarm present |
| 8 | Speed setpoint/actual deviation within tolerance |
| 9 | Control requested (PLC may control) |
| 10 | Comparison speed reached or exceeded (p2141) |
| 11 | I, M or P limit not reached |
| 12 | Holding brake open |
| 13 | No motor overtemperature alarm |
| 14 | Motor rotates forwards (n_act ≥ 0) |
| 15 | No thermal overload alarm, power unit |

## STW2 / ZSW2

- STW2 bits 12–15: **master sign-of-life**. The PLC counts 1…15 (0 is skipped) each cycle.
- ZSW2 bits 12–15: **slave sign-of-life**. The drive counts 1…15.
- The drive only monitors the sign-of-life in isochronous (IRT) operation. The other bits are device specific.

## G1_STW / G1_ZSW: encoder interface

| Bit | G1_STW (request) | G1_ZSW (status) |
|----|------------------|-----------------|
| 0–7 | Reference mark search / measuring probe functions | Status of those functions |
| 8, 9 | – | Probe 1 / probe 2 deflected |
| 11 | – | Encoder fault acknowledge active |
| 13 | Request cyclic absolute value in G1_XIST2 | Absolute value is being sent cyclically |
| 14 | Request parking encoder | Parking encoder active |
| 15 | Acknowledge encoder fault | Encoder fault (G1_XIST2 then holds an error code) |

- `G1_XIST1` is a 32-bit counter that wraps around. The PLC works with *differences* between
  cycles, so a wrap from +2147483647 to -2147483648 is harmless if you subtract as DINT.
- `G1_XIST2` holds the absolute position when requested (G1_STW.13), or an error code when G1_ZSW.15 = 1.

## State machine

```
            power on / fault acknowledged
                       │
                       ▼
            ┌───────────────────────┐
            │ S1 switching on       │ ZSW1.6 = 1
            │    inhibited          │
            └──────────┬────────────┘
     STW1.0=0 and      │ ▲ OFF2 or OFF3
     STW1.1=1, .2=1    ▼ │
            ┌───────────────────────┐
            │ S2 ready for          │ ZSW1.0 = 1
            │    switching on       │
            └──────────┬────────────┘
             STW1.0=1  │ ▲ STW1.0=0
                       ▼ │
            ┌───────────────────────┐
            │ S3 switched on        │ ZSW1.0,1 = 1
            └──────────┬────────────┘
             STW1.3=1  │ ▲ STW1.3=0 (pulses off, coast)
                       ▼ │
            ┌───────────────────────┐
            │ S4 operation          │ ZSW1.0,1,2 = 1
            └───┬──────────┬────────┘
       STW1.0=0 │          │ STW1.2=0
                ▼          ▼
         S5 OFF1 ramp   S5 OFF3 quick stop
         → standstill   → standstill
         → S2           → S1

  OFF2 (STW1.1=0) from any state: pulses off immediately → S1
  Any fault: → FAULT (ZSW1.3 = 1), acknowledge with a rising edge on STW1.7 → S1
```

The most common beginner bug: sending `16#047F` straight away after a fault or power-up.
The drive stays in S1 (switching on inhibited) because it needs to see **ON = 0 first**.
Send `047E`, wait for ZSW1.6 = 0, then send `047F`.
