# drive_learning: virtual SINAMICS S210 (telegram 3) for TIA Portal + PLCSIM

A learning kit for the PLC ↔ drive interface **without a real drive**.

Startdrive can't go online to a home-made simulator, because its online protocol is
proprietary. This kit simulates the drive **inside the PLC** instead. Your PLC program
sends the real telegram 3 words (STW1, NSOLL_B, …) and a virtual S210 block answers like
a drive would (ZSW1, NIST_B, G1_XIST1, …). Everything runs in **PLCSIM**.

```
 watch table ──► DB_Drive1.Cmd ──► FB_Tlg3_Control ──► TlgOut (STW1, NSOLL_B, STW2, G1_STW)
                                        ▲                   │
                                        │                   ▼  SimMode = TRUE
                 DB_Drive1.Status ◄─────┴── TlgIn ◄── FB_VirtualS210_Tlg3
                                                    (state machine, ramp, motor, encoder)

                                                       SimMode = FALSE → real S210 via PROFINET
```

## What is simulated

| Simulated | Not simulated |
|-----------|---------------|
| PROFIdrive state machine S1…S5 + fault (STW1 → ZSW1) | Startdrive online access, parameters, trace |
| OFF1 / OFF2 / OFF3, enable operation, RFG bits 4–6, control by PLC (bit 10) | Torque/current limits, load, inertia (only a first-order speed model) |
| Ramp-function generator (p1120 / p1121 / p1135) | Safety Integrated (STO, SS1 …) |
| 32-bit speed scaling NSOLL_B / NIST_B with p2000 | Isochronous mode, sign-of-life monitoring |
| Encoder G1_XIST1 (32-bit, wraps), G1_XIST2 absolute on request | Reference mark search / measuring probe (G1_STW bits 0–7) |
| G1_STW / G1_ZSW: absolute request, parking, encoder fault + acknowledge | Real fault reactions, exact firmware timing |
| Fault / alarm / encoder fault injection, slave sign-of-life | Device-specific bits of STW1/ZSW1/STW2/ZSW2 |

Treat it as a training aid. Bit-level details can differ from a real S210, so check
against the S210 manual before using your program on hardware.

## Files

| File | Content |
|------|---------|
| `plc/01_UDT_Tlg3_Setpoint.udt` | PLC data type: telegram 3 PLC → drive |
| `plc/02_UDT_Tlg3_Actual.udt` | PLC data type: telegram 3 drive → PLC |
| `plc/03_FB_VirtualS210_Tlg3.scl` | **The virtual drive** |
| `plc/04_FB_Tlg3_Control.scl` | Example PLC-side control block (switch-on sequence, scaling) |
| `plc/05_DB_Drive1.db` | Global DB: commands, sim settings, status, telegram images |
| `plc/06_FB_DriveSim_Main.scl` | Wires everything together, SimMode switch |
| `plc/07_DB_DriveSim_Main.db` | Instance DB of the main FB |
| `docs/telegram3_reference.md` | Telegram 3 bit tables, scaling, state machine |

## Setup in TIA Portal (V17 or newer, S7-1500)

1. Create a project with an **S7-1500 CPU**, for example a CPU 1511-1 PN. Telegram 3 with an S210 is
   normally used with S7-1500.
2. In the project tree under the CPU, open **External source files → Add new external file**
   and add all files from `plc/`.
3. Right-click each file → **Generate blocks from source**, **in the numbered order 01 … 07**.
   Later files depend on earlier ones.
4. Add a **Cyclic interrupt OB** (Add new block → Organization block → Cyclic interrupt,
   e.g. OB30 with 4 ms) and write one line in it (SCL):

   ```
   "DB_DriveSim_Main"();
   ```

   You can call it from OB1 instead. The simulator measures its own cycle time.
5. Compile, start **PLCSIM**, download, and set the CPU to RUN.
6. Create a **watch table** with the tags from `DB_Drive1` (Cmd, Sim, Status, TlgOut, TlgIn).
   Show `TlgOut.STW1`, `TlgIn.ZSW1`, `TlgIn.G1_ZSW` in **binary** or **hex** format.

## Try it

1. `Cmd.Speed_rpm = 1000`, `Cmd.Enable = TRUE`. `Status.SimState` goes 1 → 2 → 3 → 4,
   STW1 changes from `16#047E` to `16#047F`, and the speed ramps to 1000 rpm in about 0.33 s
   (ramp time is referred to 3000 rpm).
2. Watch `TlgIn.NIST_B` and `TlgIn.G1_XIST1`. Check the scaling yourself:
   1000 rpm / 3000 rpm × 16#4000_0000 = 357913941.
3. `Cmd.Enable = FALSE`. OFF1 ramps down and the state goes 51 → 2.
4. `Sim.InjectFault = TRUE`. ZSW1.3 = 1, the motor coasts, and `Status.SimFault = 7900`.
   Set it FALSE, then pulse `Cmd.AckError`. The state goes to 1 and the drive restarts if Enable is still TRUE.

## Exercises

1. **Your own switch-on sequence.** Write your own FB instead of `FB_Tlg3_Control`, using a
   state machine (CASE) with timeouts. If the drive doesn't reach "operation enabled"
   within 2 s, raise an error.
2. **OFF2 vs OFF3.** Add an emergency-stop input: quick stop (OFF3, STW1.2 = 0) and coast
   (OFF2, STW1.1 = 0). Compare how `Status.SpeedAct_rpm` behaves.
3. **Position from G1_XIST1.** Calculate the axis position in mm (for example 10 mm per motor rev)
   from the difference of `G1_XIST1` between cycles, so that the 32-bit wrap does no harm.
4. **Absolute value.** Set `Cmd.RequestAbsPosition` and check G1_ZSW.13 and G1_XIST2.
5. **Encoder fault.** `Sim.InjectEncoderFault = TRUE`, then handle G1_ZSW.15 in your program
   and acknowledge it with G1_STW.15. On a real drive this also trips the drive, so
   combine it with `Sim.InjectFault` to practise that.
6. **Wrong p2000.** Set `Cfg.RefSpeed_rpm` differently in the controller and the drive
   (edit the call in `FB_DriveSim_Main`) and see the speed error it causes.
7. **Bit 10.** Clear STW1.10 while running and see that the drive keeps its last command.

## Moving to a real S210 later

1. Add the S210 in the device configuration, network it with the CPU, and select
   **standard telegram 3**. Note the I/Q start addresses of the telegram.
2. Create two PLC tags on those addresses with the data types `"UDT_Tlg3_Actual"` (inputs)
   and `"UDT_Tlg3_Setpoint"` (outputs). For example, `S210_Tlg3_In` at `%I256.0` and
   `S210_Tlg3_Out` at `%Q256.0`. Use your own addresses.
3. Change the OB call to

   ```
   "DB_DriveSim_Main"(RealIn := "S210_Tlg3_In", RealOut => "S210_Tlg3_Out");
   ```

   and set `DB_Drive1.SimMode = FALSE`. Set p2000 in the drive and `Cfg.RefSpeed_rpm` to the same value.

In real projects an S210 with telegram 3 is usually driven through a **technology object**
(TO_SpeedAxis / TO_PositioningAxis + MC_Power, MC_MoveVelocity, MC_MoveAbsolute …). The
TO builds the telegram for you and can itself be run in simulation in PLCSIM. Learning
the raw telegram first, as done here, makes the TO much easier to understand and debug.

## Startdrive without hardware

Startdrive works **offline** without a drive. Add an S210, select the motor, set
the telegram, look through the parameters (p2000, p1120, p1121, p1135 …) and use the
commissioning wizard. You only need the real drive for going online.
