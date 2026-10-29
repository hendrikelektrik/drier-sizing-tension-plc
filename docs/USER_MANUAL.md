# USER & OPERATING MANUAL: DRIER SIZING TENSION CONTROLLER
## China FX3U-14MT-6AD-2DA PLC & Fuji Electric FRENIC Inverter Integration

---

## 1. System Overview

This control system synchronizes and regulates the drying cylinder (*drier*) speed on a **Textile Warp Sizing Machine**. 

The system operates **100% automatically** using a closed-loop **Proportional-Integral (PI)** algorithm:
1. It reads continuous yarn tension from an analog dancer roll (**`AD0`**: `0 – 10V DC` $\rightarrow$ `0 – 4000 kg`).
2. It compares the tension against the active tension setpoint (`X0`=1000kg, `X5`=1500kg, `X6`=2000kg, `X7`=2500kg).
3. It filters the tension through an **easily adjustable deadband register (`D125`)** to prevent mechanical hunting.
4. It dynamically calculates the required speed ratio correction (**Parameter `d63`**).
5. It transmits the updated `d63` value directly to a **Fuji Electric Inverter** via **RS-485 Modbus RTU**.

```
  [ Dancer Sensor (0-10V) ] ──► [ AD0 Terminal ]
                                      │
  [ Selectors: Mode / Setpoint ] ──► [ X0, X2-X7 Terminals ]
                                      │
                                      ▼
                        [ China FX3U-14MT PLC Board ]
                    Calculates d63 = Base_Preset - PI_Trim
                    Deadband easily adjusted via register D125
                                      │
                                RS-485 Modbus
                                (Port 1: A+ / B-)
                                      │
                                      ▼
                         [ Fuji FRENIC Inverter ]
                             (Parameter d63)
                                      │
                                      ▼
                           [ Drier Cylinder Motor ]
```

---

## 2. Hardware Specifications

| Item | Specification |
| :--- | :--- |
| **PLC Model** | China FX3U-14MT-6AD-2DA (STM32F103 ARM Cortex-M3 @ 72 MHz) |
| **Power Supply** | **DC 24V** ($\pm 10\%$), 1 Amp regulated power supply |
| **Digital Inputs** | 8 Channels (`X0` – `X7`), NPN Low-Level (Active LOW / Sinking) |
| **Digital Outputs** | 6 Channels (`Y0` – `Y5`), Transistor NPN Open Collector (DC 24V, 500mA max) |
| **Analog Inputs** | 6 Channels (`AD0` – `AD5`), 12-bit resolution (`0 to 4095` counts) |
| **Analog Used** | **`AD0`**: Voltage `0 – 10V DC` connected to Dancer (scaled to `0 – 4000 kg`) |
| **Communication** | **COM1:** RS232 Programming/HMI<br>**COM2:** RS-485 Terminals A/B (Modbus RTU Master) |
| **Target Drive** | Fuji Electric FRENIC Series (MEGA / Ace / Multi / Mini) |

---

## 3. Terminal Wiring Diagram

```
========================================================================================
                                TOP TERMINAL STRIP
========================================================================================
[ 24V ] ─── Connect to External +24V DC Power Supply (+)
[  0V ] ─── Connect to External  0V DC Power Supply (-)

--- DIGITAL INPUTS (NPN: Connect terminal to 0V / GND to activate) ---
[ X0  ] ─── Setpoint Switch: SETPOINT 1000 kg (Dedicated Target 1000 kg)
[ X1  ] ─── Spare (Unused)
[ X2  ] ─── Machine Mode:    SIPUT FLAG       (Selects Base d63 = 2375 via D205)
[ X3  ] ─── Machine Mode:    SPEED 1 FLAG     (Selects Base d63 = 2300 via D210)
[ X4  ] ─── Machine Mode:    SPEED 2 FLAG     (Selects Base d63 = 2225 via D215)
[ X5  ] ─── Setpoint Switch: SETPOINT 1500 kg (Dedicated Target 1500 kg)
[ X6  ] ─── Setpoint Switch: SETPOINT 2000 kg (Dedicated Target 2000 kg)
[ X7  ] ─── Setpoint Switch: SETPOINT 2500 kg (Dedicated Target 2500 kg)

--- ANALOG INPUTS ---
[ AD0 ] ─── Dancer Analog Signal (+) (0 to 10V DC)
[ AD1 ] ─── Spare (0-10V)
[ AD2 ] ─── Spare (0-10V)
[ AD3 ] ─── Spare (0-20mA)
[ AD4 ] ─── Spare (0-20mA)
[ AD5 ] ─── Spare (0-20mA)
[AGND ] ─── Dancer Analog Signal Ground Reference (-)

========================================================================================
                              BOTTOM TERMINAL STRIP
========================================================================================
--- TRANSISTOR OUTPUTS (DC Loads Only) ---
[ Y0  ] ─── Spare
[ Y1  ] ─── Spare
[ Y2  ] ─── Spare
[ Y3  ] ─── Spare
[ Y4  ] ─── Spare
[ Y5  ] ─── Spare
[COM0 ] ─── Connect to DC 0V (GND)

--- ANALOG OUTPUTS ---
[ DA0 ] ─── Spare Analog Output 1 (0-10V)
[ DA1 ] ─── Spare Analog Output 2 (0-10V)
[AGND ] ─── Analog Ground

--- RS-485 SERIAL PORT ---
[ A+  ] (or D+) ─── Connect to Fuji Inverter RS-485 Terminal (+) / Pin 4 (RJ45)
[ B-  ] (or D-) ─── Connect to Fuji Inverter RS-485 Terminal (-) / Pin 5 (RJ45)
========================================================================================
```

> [!CAUTION]
> **Analog Ground vs. Power Ground:** Always connect the dancer sensor's ground wire to **`AGND`**, never to the digital `0V` terminal. This prevents electrical switching noise from corrupting the tension signal.

---

## 4. Fuji Inverter Communication Configuration

Ensure the following parameters are configured on the Fuji Inverter keypad:

| Parameter | Setting Name | Value | Description |
| :--- | :--- | :--- | :--- |
| **`y01` / `y11`** | Station Address | **`1`** | Modbus Slave Station ID = 1 |
| **`y02` / `y12`** | Comms Error Action | **`0`** | Trip or coast to stop on communication loss |
| **`y03` / `y13`** | Comms Error Timer | **`1.0`** | 1.0 second communication error timeout |
| **`y04` / `y14`** | Baud Rate | **`2`** | **9600 bps** (matches PLC register `D8120`) |
| **`y05` / `y15`** | Data Length | **`0`** | **8 Data bits** |
| **`y06` / `y16`** | Parity | **`1`** | **Even Parity** |
| **`y07` / `y17`** | Stop Bits | **`0`** | **1 Stop bit** |
| **`y10` / `y20`** | Protocol Selection | **`0`** | **Modbus RTU Protocol** |
| **`d62`** | Pulse Scaling 1 | *(Fixed)* | Pulse numerator (set per machine gearing) |
| **`d63`** | Pulse Scaling 2 | *(Auto)* | **Controlled dynamically by PLC via Modbus** |

---

## 5. Control Philosophy & Operating Principles

### A. The Sizing Machine Mechanical Dynamics

$$\text{Inverter Frequency } (f^*) = \text{Master Pulses } (Np) \times \left( \frac{\mathbf{d62}}{\mathbf{d63}} \right)$$

Because the dancer roll is positioned **AFTER the drier cylinders** (between Drier and Beamer):
* The **Drier FEEDS yarn** into the dancer.
* The **Beamer PULLS yarn** out of the dancer.

$$\text{Drier Speed } \propto \frac{1}{\mathbf{d63}}$$

* **When Tension is TOO HIGH ($> \text{Setpoint} + \mathbf{D125}$):**
  * The beamer is pulling faster than the drier feeds. Yarn is being stretched.
  * The drier must **FEED MORE YARN (SPEED UP)**.
  * To speed up the drier, the PLC **DECREASES `d63`** via positive PI Trim.
* **When Tension is TOO LOW ($< \text{Setpoint} - \mathbf{D125}$):**
  * The drier is feeding faster than the beamer pulls. Yarn is sagging.
  * The drier must **FEED LESS YARN (SLOW DOWN)**.
  * To slow down the drier, the PLC **INCREASES `d63`** via negative PI Trim.
* **Within Deadband ($\text{Setpoint} \pm \mathbf{D125}$):**
  * Tension is inside the target zone. The PI controller holds steady. **No adjustments are made.**

---

### B. Forward vs. Reverse Action Direction Selector (`M10`)
The Boolean variable **`M10`** defines the control action direction:
* **`M10 = 0` (OFF - Reverse Acting, Default)**:
  $$\text{Final } d63 = \text{Base } (D220) - \text{Trim } (D150)$$
  *Use Case*: Standard setup with the dancer located **AFTER** the drier cylinder. High tension ($PV > SV$) decreases $d63$, accelerating the drier to feed more yarn and relieve tension.
* **`M10 = 1` (ON - Forward Acting)**:
  $$\text{Final } d63 = \text{Base } (D220) + \text{Trim } (D150)$$
  *Use Case*: Setup with the dancer located **BEFORE** the drier cylinder (or inverted tension transducer). High tension ($PV > SV$) increases $d63$, decelerating the drier to relieve tension.

*Control*: `M10` can be toggled via HMI, forced in GX Developer Device Test, or mapped to a physical panel switch (e.g. spare digital input `X001`).

### C. Easy Deadband Adjustment (`D125`)

The deadband is stored in **Register `D125`** (Default = `100 kg`):
* The PLC automatically calculates the negative limit in `D126` (`0 - D125 = -100`).
* **How to change on-site:** Anyone with GX Works2, an HMI, or handheld programmer can simply open register monitor and write a new value into **`D125`** (e.g. `50`, `80`, `120`) **in real time without editing or compiling ladder code!**

---

### C. Dedicated Speed Modes and Base Presets

$$\mathbf{d63} = \text{Active\_Base\_Preset} - \mathbf{PI\_Trim}$$

The 3 machine speed modes are dedicated to `X002`, `X003`, and `X004`:
1. **`SIPUT` Mode (`X002`):** Base `d63 = 2375` (Saved in `D205`). Lowest speed for crawl/threading.
2. **`SPEED 1` Mode (`X003`):** Base `d63 = 2300` (Saved in `D210`). Normal production speed.
3. **`SPEED 2` Mode (`X004`):** Base `d63 = 2225` (Saved in `D215`). High-speed production.

The **PI Trim (`D150`)** continuously floats on top of whichever mode is selected:
$$\text{PI\_Trim} = \text{P\_Term} + \text{I\_Accumulator}$$
* **P-Term:** Instantly reacts to tension displacement.
* **I-Accumulator:** Gently accumulates every 200 ms to eliminate long-term tension offset.

---

### D. Dedicated Tension Setpoint Selection (`X0`, `X5`, `X6`, `X7`)

| Terminal | Target Tension Setpoint | Register `D120` Value |
| :--- | :--- | :--- |
| **`X000`** | **1000 kg** | `D120 = 1000` (Also power-up default) |
| **`X005`** | **1500 kg** | `D120 = 1500` |
| **`X006`** | **2000 kg** | `D120 = 2000` |
| **`X007`** | **2500 kg** | `D120 = 2500` |

---

### E. Built-In Safeguards

1. **VFD EEPROM Memory Protection:**
   The PLC compares current `d63` with the last sent value (`LD<> D230 D231`). Modbus transmission is triggered **only when the value changes**, preventing the inverter's memory chip from wearing out.
2. **Web-Break / Line-Stop Lockout:**
   If the yarn snaps or the line stops, the dancer drops to 0 kg. 
   The PLC checks if `Tension > 150 kg`. If tension is below 150 kg, the PI controller freezes and resets `Trim = 0` so the drier will **never run away to high speed**.
3. **Integral Anti-Windup Clamping:**
   The integral accumulator is hard-clamped between **`-80` and `+80`** counts, preventing massive overshoots when accelerating or starting the machine.

---

## 6. Complete Memory Register Reference

| Device | Name / Function | Range / Default | Description |
| :--- | :--- | :--- | :--- |
| **`X000`** | `SETPOINT 1000KG` | Digital Input | Dedicated 1000 kg tension selector |
| **`X001`** | Spare | — | Unused |
| **`X002`** | `SIPUT FLAG` | Digital Input | Dedicated Siput mode selector (Base `d63 = 2375`) |
| **`X003`** | `SPEED 1 FLAG` | Digital Input | Dedicated Speed 1 mode selector (Base `d63 = 2300`) |
| **`X004`** | `SPEED 2 FLAG` | Digital Input | Dedicated Speed 2 mode selector (Base `d63 = 2225`) |
| **`X005`** | `SETPOINT 1500KG` | Digital Input | Dedicated 1500 kg tension selector |
| **`X006`** | `SETPOINT 2000KG` | Digital Input | Dedicated 2000 kg tension selector |
| **`X007`** | `SETPOINT 2500KG` | Digital Input | Dedicated 2500 kg tension selector |
| **`M2`** | Siput Flag | Internal Relay | Active when in Siput mode |
| **`M3`** | Speed 1 Flag | Internal Relay | Active when in Speed 1 mode |
| **`M4`** | Speed 2 Flag | Internal Relay | Active when in Speed 2 mode |
| **`M10`** | **PI Direction Selector** | **0=REV, 1=FWD** | **Boolean: 0 = Reverse (After Drier), 1 = Forward (Before Drier)** |
| **`M50`** | Modbus Trigger | Internal Relay | Pulses ON when new `d63` needs transmission |
| **`M100`** | `SIMULATOR BYPASS RD3A` | Internal Relay | Turn ON in GX Simulator to test D100 freely without RD3A overwriting |
| **`T20`** | PI Sampling Timer | 200 ms | Controls the integration clock rate |
| **`D100`** | Raw ADC Count | 0 to 4095 | Raw 12-bit analog input from terminal `AD0` |
| **`D104`** | Unfiltered Tension | 0 to 4000 kg | Raw tension calculated from `AD0` via 32-bit `DDIV` |
| **`D106`** | DDIV Remainder | - | 32-bit division remainder output |
| **`D108`** | Filter Temp Register | - | Intermediate calculation register for filter |
| **`D110`** | Filtered Tension (PV)| 0 to 4000 kg | Clean 4-sample running average of dancer tension |
| **`D120`** | Tension Setpoint (SV)| 1000 – 2500 kg| Selected tension target |
| **`D122`** | Raw Error | $\text{PV} - \text{SV}$ | Deviation from setpoint |
| **`D124`** | Effective Error | Error after Deadband | Error fed into PI algorithm (0 inside deadband) |
| **`D125`** | **Deadband Setting** | **100 kg** | **Tuning register: easily adjust deadband live!** |
| **`D126`** | Negative Deadband | Calculated (`-D125`)| Automatic negative threshold (`0 - D125`) |
| **`D128`** | P Product 32-bit | — | Intermediate multiplication register |
| **`D130`** | Proportional Term | $-80$ to $+80$ | $(\text{Error} \times K_p) / 100$ |
| **`D132`** | I Product 32-bit | — | Intermediate multiplication register |
| **`D134`** | Integral Increment | — | Delta step per 200ms |
| **`D140`** | Integral Sum | $-80$ to $+80$ | Accumulated steady-state correction |
| **`D150`** | Total Dynamic Trim | $-100$ to $+100$ | Sum of $P + I$ |
| **`D160`** | **CAL AD0 (0 kg)** | **340** | **Retentive calibration register: 0 kg raw count** |
| **`D161`** | **CAL AD1 (1000 kg)** | **1220** | **Retentive calibration register: 1000 kg count (X000)** |
| **`D162`** | **CAL AD2 (1500 kg)** | **1591** | **Retentive calibration register: 1500 kg count (X005)** |
| **`D163`** | **CAL AD3 (2000 kg)** | **1915** | **Retentive calibration register: 2000 kg count (X006)** |
| **`D164`** | **CAL AD4 (2500 kg)** | **2194** | **Retentive calibration register: 2500 kg count (X007)** |
| **`D165`** | **CAL AD5 (4000 kg)** | **2810** | **Retentive calibration register: 4000 kg count** |
| **`D168`** | Calibration Span AD | Dynamic | Intermediate span calculation register |
| **`D169`** | 32-bit High Word (0)| 0 | Cleared to 0 for DDIV double word division |
| **`D205`** | Base Siput `d63` | 2375 (`H947`) | Base ratio for Siput mode |
| **`D206`** | **Siput Trim Limit** | **60** | **Max trim range for Siput mode (+/- 60)** |
| **`D210`** | Base Speed 1 `d63` | 2300 (`H8FC`) | Base ratio for Speed 1 mode |
| **`D211`** | **Speed 1 Trim Limit** | **100** | **Max trim range for Speed 1 mode (+/- 100)** |
| **`D215`** | Base Speed 2 `d63` | 2225 (`H8B1`) | Base ratio for Speed 2 mode |
| **`D216`** | **Speed 2 Trim Limit** | **120** | **Max trim range for Speed 2 mode (+/- 120)** |
| **`D220`** | Active Base `d63` | 2000 – 3000 | The base ratio for currently active mode |
| **`D222`** | Active Positive Trim Limit | Dynamic | Positive clamp limit (+D206, +D211, or +D216) |
| **`D224`** | Active Negative Trim Limit | Dynamic | Negative clamp limit (-D206, -D211, or -D216) |
| **`D230`** | Final `d63` Output | $\text{Base} - \text{Trim}$| **Value written to Fuji Inverter Parameter `d63`** |
| **`D231`** | Last Sent `d63` | 2000 – 3000 | History register to detect value changes |
| **`D320`** | Proportional Gain $K_p$| **5** | Tuning register for immediate response |
| **`D322`** | Integral Gain $K_i$ | **2** | Tuning register for steady-state offset removal |
| **`D8120`**| RS-485 Config | `H8087` | 9600 bps, 8 Data, Even Parity, 1 Stop bit |
| **`D8129`**| Modbus Timeout | 500 ms | Serial communication timeout |

---

## 7. How to Load the Program into the PLC

### Using Mitsubishi GX Works2:
1. Open **GX Works2**.
2. Go to **Project $\rightarrow$ New...**
   * **Series:** `FXCPU`
   * **Type:** `FX3U / FX3UC`
   * **Project Type:** `Simple Project` (Ladder)
3. Connect your PC to the PLC's programming port using a USB-to-RS232 programming cable (COM port typically set to **38.4 kbps**, 7-E-1).
4. Go to **Project $\rightarrow$ Open Other Format Data $\rightarrow$ Import from Text File...**
   * Select: `export/MAIN export.txt`
5. Go to **Edit $\rightarrow$ Documentation $\rightarrow$ Import from Text File...**
   * Select: `export/COMMENT export.txt`
6. Press **`F4`** (Compile / Rebuild All).
7. Go to **Online $\rightarrow$ Write to PLC...**, check **Program** and **Comment**, and click **Execute**.
8. Turn the PLC RUN/STOP switch to **RUN**. The `RUN` LED should glow solid green.

---

## 8. On-Site Tuning Guide

### 8.1 Procedure to Define the Perfect Base Preset (D205, D210, D215)

In sizing machines, the dancer arm should only perform fine-trimming ($5\%–10\%$), while the Base Preset carries the main synchronized speed ($90\%–95\%$).

```
                  [ Ideal Operating Condition ]
     Base Preset (D220)  ======> Carries 100% of nominal speed
     PI Trim     (D150)  ======> Hovers near 0 (dancer rests in center)
     Output      (D230)  ======> D220 - 0 = D220
```

#### Step-by-Step Machine Calibration:
1. **Run the Machine** in the selected speed mode (e.g. Speed 1 with `X003 = ON`).
2. **Select Tension Setpoint** (e.g. 1000 kg with `X000 = ON`).
3. **Wait for Yarn to Stabilize** at normal running speed and moisture.
4. **Observe Register `D150` (Total Trim)** in GX Works2 monitor (`F3`):
   - **`D150` floats around 0 (-10 to +10)**:
     The Base Preset is **perfect**. The dancer is riding in the middle of its physical stroke.
   - **`D150` consistently sits at +30 to +60**:
     The drier is feeding yarn too slowly $\implies$ needs more speed $\implies$ $d63$ must be lowered.
     - *Correction*: Lower Base Register `D210` by the offset:
       $$\text{New } D210 = \text{Old } D210 - D150$$
       *(e.g., $2300 - 40 = \mathbf{2260}$)*.
   - **`D150` consistently sits at -30 to -60**:
     The drier is feeding yarn too quickly $\implies$ needs less speed $\implies$ $d63$ must be raised.
     - *Correction*: Increase Base Register `D210` by the offset:
       $$\text{New } D210 = \text{Old } D210 + |D150|$$
       *(e.g., $2300 + 40 = \mathbf{2340}$)*.
5. Once dialed in, `D150` will float near `0`, leaving the full trim authority available for unexpected line shocks.

### 8.2 Adjusting Mode Trim Limits (D206, D211, D216)
Each speed mode has its own independent trim limit register:
- **`D206` (Siput Trim Limit, Default = 60)**: Clamps trim to $\pm 60$ during crawl. Keeps slow threading smooth without jerks.
- **`D211` (Speed 1 Trim Limit, Default = 100)**: Clamps trim to $\pm 100$ during standard production.
- **`D216` (Speed 2 Trim Limit, Default = 120)**: Clamps trim to $\pm 120$ during high-speed production.
- *Open-Loop Option*: Set any mode's trim limit register to `0` to completely disable dancer trim and run strictly on fixed base speed.



You can monitor and adjust the controller directly from GX Works2 Online Monitor (`F3`):

### 8.3 Live Recalibrating the Tension Sensor (D160–D165)
The 5-segment piecewise linear calibration is fully held in retentive registers `D160` to `D165`. 
If you ever replace the dancer potentiometer or the sensor mechanical alignment shifts, **you do not need to modify any ladder logic**:
1. Leave dancer completely slack (0 kg) -> read raw count in **`D100`** -> write into **`D160`** (Default = 340).
2. Hang or pull 1000 kg tension -> read **`D100`** -> write into **`D161`** (Default = 1220).
3. Check 1500 kg -> write into **`D162`** (Default = 1591).
4. Check 2000 kg -> write into **`D163`** (Default = 1915).
5. Check 2500 kg -> write into **`D164`** (Default = 2194).
6. Check 4000 kg -> write into **`D165`** (Default = 2810).

The PLC ladder automatically recalculates all span slopes every scan cycle!

### 1. Adjusting the Deadband (`D125`):
* **Default:** `D125 = 100` ($\pm 100\text{ kg}$).
* If you want tighter tension control with narrower tolerance $\rightarrow$ Change **`D125`** to **`50`** ($\pm 50\text{ kg}$) or **`70`** ($\pm 70\text{ kg}$).
* If the yarn has natural slight tension wobble and you want less control activity $\rightarrow$ Change **`D125`** to **`120`** or **`150`**.

### 2. Adjusting Gains (`D320` and `D322`):

```
       DANCER RESPONSE BEHAVIOR                    WHAT TO ADJUST
┌──────────────────────────────────────────────────────────────────────┐
│ The dancer reacts too slowly to line speed changes,                  │
│ and tension wanders outside the deadband for too long:               │
│ ──► Increase Kp (D320) from 5 to 7 or 8.                             │
├──────────────────────────────────────────────────────────────────────┤
│ The dancer is too aggressive, starts bouncing or vibrating,          │
│ and causes the drier speed to oscillate:                             │
│ ──► Decrease Kp (D320) from 5 to 3 or 4.                             │
├──────────────────────────────────────────────────────────────────────┤
│ The dancer settles smoothly, but stays offset just outside           │
│ the deadband without returning to center:                            │
│ ──► Increase Ki (D322) from 2 to 3 or 4.                             │
├──────────────────────────────────────────────────────────────────────┤
│ Tension overshoots the setpoint when returning to center:            │
│ ──► Decrease Ki (D322) from 2 to 1.                                  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 9. Troubleshooting & Diagnostics

| Symptom | Probable Cause | Action |
| :--- | :--- | :--- |
| **`ERR` LED is flashing on PLC** | Hardware watchdog or syntax fault | Check DC 24V power supply voltage. Ensure memory range settings in GX Works2 match FX3U. |
| **Dancer reads `0 kg` (`D100 = 0`)** | Wiring broken or 0-10V signal missing | Measure DC voltage between terminal `AD0` and `AGND` with a multimeter. Ensure sensor power supply is ON. |
| **Dancer reads `4000 kg` constantly** | Open ground wire or input saturation | Check `AGND` terminal connection. Ensure input signal does not exceed 10.0V DC. |
| **Inverter does not change speed** | RS-485 wiring reversed or baud mismatch | 1. Swap terminals `A+` and `B-` on RS-485 port.<br>2. Verify Inverter parameters `y01 = 1`, `y04 = 2` (9600 bps), `y06 = 1` (Even), `y10 = 0` (Modbus). |
| **Inverter trips on Comms Fault (`ErP`)** | Timeout threshold too strict | Check inverter parameter `y03` (set to `1.0` second or higher). Ensure cable shielding is grounded at one end. |
| **Tension increases when drier accelerates** | Inverted dancer physical placement | If dancer was physically moved *before* the drier instead of after, change `SUB D220 D150 D230` to `ADD D220 D150 D230` in Section 7. |