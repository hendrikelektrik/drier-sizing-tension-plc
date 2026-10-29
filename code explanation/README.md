# Code Explanation & Reference Documentation: Drier Sizing Ratio Controller

This folder contains the complete, line-by-line documented Instruction List (IL) programs for the **China FX3U PLC** controlling the **Drier Sizing** Fuji inverter via RS-485 Modbus RTU.

---

## Files in this Folder

1. **[`REVISED_PROGRAM.txt`](./REVISED_PROGRAM.txt)** / **[`PI_TRIM_DANCER_AUTO.txt`](./PI_TRIM_DANCER_AUTO.txt)** (CURRENT REVISED PROGRAM)
   * **100% Automatic Closed-Loop PI Dancer Tension Controller** using `AD0` (0–10V / 340 to 2876 counts).
   * **5-Segment Piecewise Linear Calibration (Easily Adjustable Live via Registers `D160`–`D165`)**:
     * Automatically scales raw counts into true physical kilograms ($0\text{–}4250\text{ kg}$).
     * Anchored to the exact calibration curve from [`source/table_tension.txt`](../source/table_tension.txt).
     * Hit your 4 production setpoints with **0.0% calibration error** (`1000 kg`, `1500 kg`, `2000 kg`, `2500 kg`).
     * If the sensor is replaced or recalibrated in the future, simply update registers `D160`–`D165` via HMI/GX Works2 without touching any ladder code!
   * **Forward / Reverse Action Selector (`M10`)**:
     * **`M10 = OFF (0)` [Default - Reverse Acting]**:
       $$d63 = \text{Base } (D220) - \text{Trim } (D150)$$
       Used when the dancer is located **AFTER** the drier. High tension ($PV > SV$) decreases $d63$, accelerating the drier to feed more yarn.
     * **`M10 = ON (1)` [Forward Acting]**:
       $$d63 = \text{Base } (D220) + \text{Trim } (D150)$$
       Used when the dancer is located **BEFORE** the drier (or inverted sensor). High tension ($PV > SV$) increases $d63$, decelerating the drier to relieve tension.
   * **Adjustable Trim Limits per Speed Mode:**
     * `X002`: SIPUT Mode (Base `d63 = 2375` via `D205`, Trim Limit `+/- 60` via `D206`)
     * `X003`: SPEED 1 Mode (Base `d63 = 2300` via `D210`, Trim Limit `+/- 100` via `D211`)
     * `X004`: SPEED 2 Mode (Base `d63 = 2225` via `D215`, Trim Limit `+/- 120` via `D216`)
   * **Live Adjustable Deadband:** Stored in Register **`D125`** (Default = 100 kg). Easily changed via HMI or GX Developer without modifying ladder rungs!
   * **Dedicated Tension Setpoint Inputs:**
     * `X000`: 1000 kg (Dedicated Target 1000 kg Selector)
     * `X005`: 1500 kg (Dedicated Target 1500 kg Selector)
     * `X006`: 2000 kg (Dedicated Target 2000 kg Selector)
     * `X007`: 2500 kg (Dedicated Target 2500 kg Selector)
   * **Simulator Testing Support:** Bit `M100` allows running GX Developer Simulator without `RD3A` overwriting `D100`.
   * **VFD EEPROM Protection:** Writes over RS-485 occur only when the calculated `d63` value actually changes.
   * **Web-Break Safety:** Integrator accumulator resets if tension drops below 150 kg (broken yarn / line stopped).

2. **[`CURRENT_PROGRAM_COMMENTED.txt`](./CURRENT_PROGRAM_COMMENTED.txt)**
   * Exact replica of the original `source/MAIN export.txt` with line-by-line comments for historical reference.

---

## 5-Segment Tension Calibration Guide (Live Adjustable)

Because the dancer potentiometer has a zero-offset of `340 counts` and mechanical curvature, the PLC uses **6 retentive calibration registers** to map counts into true engineering kilograms:

| Register | Calibration Anchor Point | Default Count | Matching Selector | Purpose |
| :---: | :---: | :---: | :---: | :--- |
| **`D160`** | **0 kg** Point | **340** | - | Sensor zero offset when dancer is slack |
| **`D161`** | **1000 kg** Point | **1220** | `X000` | Target anchor for Setpoint 1000 kg |
| **`D162`** | **1500 kg** Point | **1591** | `X005` | Target anchor for Setpoint 1500 kg |
| **`D163`** | **2000 kg** Point | **1915** | `X006` | Target anchor for Setpoint 2000 kg |
| **`D164`** | **2500 kg** Point | **2194** | `X007` | Target anchor for Setpoint 2500 kg |
| **`D165`** | **4000 kg** Point | **2810** | - | Full stroke electrical limit |

### How to Recalibrate in the Future (No Ladder Changes):
If the potentiometer is replaced, mechanical linkage shifts, or you install a new tension transducer:
1. Allow dancer to rest slack (0 kg) $\rightarrow$ read raw count in **`D100`** $\rightarrow$ write into **`D160`**.
2. Hang/pull 1000 kg calibration weight $\rightarrow$ read raw count in **`D100`** $\rightarrow$ write into **`D161`**.
3. Repeat for 1500, 2000, 2500, and 4000 kg into **`D162`–`D165`**.
4. The PLC ladder automatically calculates segment spans ($\Delta AD = D16_{x+1} - D16_x$) every scan, instantly updating the curve in real time!

---

## Forward vs. Reverse Action Direction (`M10`)

The Boolean variable **`M10`** allows instantly switching the control action without rewiring or changing ladder math:

| Mode | `M10` State | Mathematical Output | When to Use | Tension High ($PV > SV$) | Tension Low ($PV < SV$) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Reverse Acting** *(Default)* | **`0` (OFF)** | $D230 = D220 - D150$ | Dancer is **AFTER** the drier cylinder | $d63$ decreases $\implies$ Drier **Speeds Up** | $d63$ increases $\implies$ Drier **Slows Down** |
| **Forward Acting** | **`1` (ON)** | $D230 = D220 + D150$ | Dancer is **BEFORE** the drier cylinder | $d63$ increases $\implies$ Drier **Slows Down** | $d63$ decreases $\implies$ Drier **Speeds Up** |

---

## Engineering Guide: Defining the Perfect Base Preset

In web and yarn tension handling, the dancer arm should only provide **fine-trimming** ($5\%–10\%$), while the **Base Preset** carries the bulk of the synchronized line speed ($90\%–95\%$).

```
                  [ Ideal Operating Condition ]
     Base Preset (D220)  ======> Carries 100% of nominal speed
     PI Trim     (D150)  ======> Hovers near 0 (dancer rests in center)
     Output      (D230)  ======> D220 +/- 0 = D220
```

### 3-Step Calibration Procedure at the Machine:

1. **Run the Machine in Steady State**:
   - Select the target speed mode (e.g. Speed 1 via `X003`).
   - Select the target tension setpoint (e.g. 1000 kg via `X000`).
   - Wait for line speed and yarn temperature to stabilize.

2. **Observe Register `D150` (Total PI Trim)**:
   - **Case A (`D150` floats near 0, between -10 and +10)**:
     The Base Preset is already **perfect**. The dancer is riding in the middle of its physical stroke.
   - **Case B (`D150` consistently sits at +30 to +60)**:
     With default Reverse Action (`M10=0`), the drier is feeding yarn too slowly $\implies$ $d63$ must be lowered:
     $$\text{New Base } D210 = \text{Old } D210 - D150$$
     *(e.g., $2300 - 40 = \mathbf{2260}$)*.
   - **Case C (`D150` consistently sits at -30 to -60)**:
     With default Reverse Action (`M10=0`), the drier is feeding yarn too quickly $\implies$ $d63$ must be raised:
     $$\text{New Base } D210 = \text{Old } D210 + |D150|$$
     *(e.g., $2300 + 40 = \mathbf{2340}$)*.

3. **Lock In & Verify**:
   - Once tuned, `D150` will hover around `0`.
   - The entire trim range (`D211 = +/- 100`) is now fully available to absorb roll diameter changes, moisture variations, and speed ramps without hitting mechanical limits.

---

## Memory & Register Reference Map

| Device | Name / Function | Range / Default | Description |
| :--- | :--- | :--- | :--- |
| **`X000`** | `SETPOINT 1000KG` | Digital Input | Dedicated 1000 kg tension selector |
| **`X001`** | Spare / Optional Switch | Digital Input | Can be mapped to toggle `M10` (Forward/Reverse) |
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
| **`M100`** | Simulator Bypass RD3A | Internal Relay | Turn ON in GX Simulator to test D100 freely without RD3A overwriting |
| **`T20`** | PI Sampling Timer | 200 ms | Controls the integration clock rate |
| **`D100`** | Raw ADC Count | 340 to 2876 | Raw 12-bit analog input from terminal `AD0` (0–10V) |
| **`D102`** | Raw Product 32-bit | Double Word | Intermediate multiplication product (D102, D103) |
| **`D104`** | Calibrated Tension | 0 to 4250 kg | True physical tension in kg from 5-segment calibration |
| **`D106`** | Calibration Delta AD | Dynamic | Intermediate $\Delta AD$ count register |
| **`D108`** | Filter Temp Register | - | Intermediate calculation register for filter |
| **`D110`** | Filtered Tension (PV)| 0 to 4250 kg | Clean 4-sample running average of true tension |
| **`D120`** | Tension Setpoint (SV)| 1000–2500 kg | Selected tension target in true kg |
| **`D122`** | Raw Error | PV - SV | Deviation from setpoint in physical kg |
| **`D124`** | Effective Error | Error outside Deadband | Filtered error fed into PI loop (0 inside deadband) |
| **`D125`** | **Deadband Setting** | **100 kg** | **Live tuning register: easily adjust deadband** |
| **`D126`** | Negative Deadband | Calculated (`-D125`)| Automatic negative threshold (`0 - D125`) |
| **`D128`** | P Product 32-bit | Double Word | Intermediate multiplication register |
| **`D130`** | Proportional Term | Dynamic | Proportional trim $(D124 \times K_p) / 100$ |
| **`D132`** | I Product 32-bit | Double Word | Intermediate multiplication register |
| **`D134`** | Integral Increment | Dynamic | Delta step per 200 ms |
| **`D140`** | Integral Sum | Clamped to Trim Limit | Accumulated steady-state correction |
| **`D150`** | Total Dynamic Trim | Clamped to Trim Limit | Sum of $P + I$ |
| **`D160`** | **CAL AD0 (0 kg)** | **340** | **Retentive calibration register: 0 kg raw count** |
| **`D161`** | **CAL AD1 (1000 kg)**| **1220** | **Retentive calibration register: 1000 kg count** |
| **`D162`** | **CAL AD2 (1500 kg)**| **1591** | **Retentive calibration register: 1500 kg count** |
| **`D163`** | **CAL AD3 (2000 kg)**| **1915** | **Retentive calibration register: 2000 kg count** |
| **`D164`** | **CAL AD4 (2500 kg)**| **2194** | **Retentive calibration register: 2500 kg count** |
| **`D165`** | **CAL AD5 (4000 kg)**| **2810** | **Retentive calibration register: 4000 kg count** |
| **`D168`** | Calibration Span AD | Dynamic | Calculated span between adjacent calibration points |
| **`D169`** | 32-bit High Word (0)| 0 | Cleared to 0 for DDIV double word division |
| **`D205`** | Base Siput `d63` | 2375 (`0x0947`) | Base ratio for Siput mode |
| **`D206`** | **Siput Trim Limit** | **60** | **Max trim range for Siput mode (+/- 60)** |
| **`D210`** | Base Speed 1 `d63` | 2300 (`0x08FC`) | Base ratio for Speed 1 mode |
| **`D211`** | **Speed 1 Trim Limit** | **100** | **Max trim range for Speed 1 mode (+/- 100)** |
| **`D215`** | Base Speed 2 `d63` | 2225 (`0x08B1`) | Base ratio for Speed 2 mode |
| **`D216`** | **Speed 2 Trim Limit** | **120** | **Max trim range for Speed 2 mode (+/- 120)** |
| **`D220`** | Active Base `d63` | 2000–3000 | Active base ratio for current speed mode |
| **`D222`** | Active Positive Trim Limit | Dynamic | Positive clamp limit (+D206, +D211, or +D216) |
| **`D224`** | Active Negative Trim Limit | Dynamic | Negative clamp limit (-D206, -D211, or -D216) |
| **`D230`** | **Final `d63` Output** | $\text{Base} \pm \text{Trim}$ | **Value written to Fuji Inverter Parameter `d63`** |
| **`D231`** | Last Sent `d63` | 2000–3000 | History register to detect value changes |
| **`D320`** | Proportional Gain $K_p$| **5** | Tuning register for immediate response |
| **`D322`** | Integral Gain $K_i$ | **2** | Tuning register for steady-state offset removal |
