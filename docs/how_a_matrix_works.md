# How a Keyboard Matrix Works (QMK Default: COL2ROW)

> **Scope**: This guide explains QMK’s **default** matrix wiring and scanning: **diodes from column to row** (`DIODE_DIRECTION = COL2ROW`). There is an optional, “inverted” setup called **ROW2COL** (roles swapped). Because COL2ROW is far more common in QMK, we focus on it here and summarize ROW2COL at the end.

---

## Quick Glossary

See appendix for [full glossary](#6-glossary-plain-but-precise).

- **Anode**: The plus terminal of the diode.
- **Cathode**: The minus terminal of te diode.
- **Controller**: The micro controller (MCU) that controls the keyboard.
- **Current**: Electrical charge that flows from **HIGH** to **LOW**. Current can never flow between same potentials like **HIGH** and **HIGH** or **LOW** and **LOW**. Usually **HIGH** is represented by **VCC** (~5 V or ~3.3 V) and **LOW** by 0 V or GND.
- **Diode**: One‑way valve for current. In *COL2ROW*, it passes current **from column to row** only (and blocks the reverse).
- **GPIO**: General‑Purpose Input/Output pin on the MCU. Can be configured as **input** or **output**.  
- **HIGH / LOW**: Digital logic levels. In QMK’s *COL2ROW* scanning, a **LOW on a column input** (while its row is active) means **key pressed**.
- **Row / Column**: Two sets of physical wires forming a grid (the matrix). Each key connects one row to one column.
- **Scan rate**: The frequency of full matrix scans. A full scan includes activating every row for each currently scanned column. **Activating a row** means setting it **LOW**. **Scanning a column** means detecting a **LOW** or **HIGH**. A full scan usually takes ~0.5-2 ms (0.5-3 kHz) on modern controllers.

## TL;DR (for impatient beginners)

- Keyboard switch matrices are arranged in rows and columns. Without a matrix circuit, each switch would require its own two wires (in and out) directly to the controller.<br>  
Keyboard switch matrices are arranged in rows and columns. Without a matrix, each key typically needs its own controller pin (plus a common return pin, usually the shared GND), so a 104-key board would require about 105 GPIOs (which can make controller selection challenging).  <br>
A matrix reduces this dramatically: for example, 6 rows × 17 columns (6x17 matrix) needs only 23 pins. Controllers with ~20–30 GPIOs are common, so this is practical.<br>
![matrix overlay](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/matrix_overlay.png)  
**Figure 1** Example 75 % keyboard matrix overlay to highlight the electrical arrangement of keys switches in rows and columns.<br><br>

- Think of a keyboard as a **grid**: **columns** on one side, **rows** on the other. Each key is a **switch** at the intersection of one column and one row, with a **diode** in series so current can only flow **from the column toward the row**.<br>
In other words, in a switch matrix, each switch maps to a unique matrix coordinate described by row number and column number. Using this coordinate, the MCU indexes a lookup table to determine which key was pressed.<br>
![2x2 switch matrix](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/2x2_switch_matrix_basic.png)  
**Figure 2** An example 2x2 switch matrix. It shows how four controller pins (ROW0, ROW1, COLUMN0, COLUMN1) allow the addressing of 4 switches. Without a matrix, 8 pins would have been necessary - two pins for each switch.<br><br>

- QMK scans by **activating one row at a time**: it **drives that row LOW** (to 0 V).
- All **columns** are **inputs with pull‑ups** (they sit HIGH by default). 
- If a key on the active row is pressed, current flows **from the column’s pull‑up → switch → diode → the LOW row**, pulling that **column input LOW**. QMK reads that LOW as “key pressed.”
- **Diodes prevent ghosting** (false key detections) by letting current flow in only one direction.

---

## 1 Rows vs. Columns and What “COL2ROW” Means

- **Columns (COLUMN)**: nets named `COLUMN0…COLUMNn`. In COL2ROW, columns are **inputs with pull‑ups** (not driven).  
- **Rows (ROW)**: nets named `ROW0…ROWm`. In COL2ROW, rows are **outputs** that are **activated one at a time** by driving them **LOW**.  
- **Diode orientation** for COL2ROW: **anode on the column side**, **cathode on the row side** (typically marked with a bar on the device package):
![alt text](resources/images/how_a_matrix_works/diode_symbol.png)
 This lets current flow from **column to row** (anode to cathode) only. Any reverse current from row to column is blocked by the diode.

Why is it named “COL2ROW”? Because that’s the **current’s allowed direction** through the diode: **from the column to the row**.

---

## 2) How Scanning Works (Step by Step)
Let’s imagine a 2×2 example (2 columns × 2 rows). The MCU repeats this loop very quickly:

1. **Prepare inputs**: Configure **all column pins** as **inputs with pull‑ups**. They idle at **HIGH**.
2. **Select one row**: Configure **ROW0** as an output and **drive it LOW** (leave other rows as inputs or otherwise inactive).  
3. **Read columns**: Instantly read every **column input**.  
   - If **no key** on ROW0 is pressed, the column inputs remain **HIGH** (the pull‑ups hold them up).  
   - If **key at (ROW0, COL1)** is pressed, current flows from **COL1 pull‑up → key switch → diode → ROW0 (LOW)** and **COL1 reads LOW** → QMK marks `(row=0, col=1)` as **pressed**.
4. **Unselect the row**: Return **ROW0** to inactive (often input or driven HIGH depending on board options).  
5. **Move to the next row**: Repeat steps 2–4 for **ROW1**, then ROW2, etc. This repeats thousands of times per second.

**Important polarity detail**: In COL2ROW, QMK uses **active‑LOW sensing**. A **LOW** on a column input (during that row’s active window) means **“key pressed.”**

---

## 3) Why Diodes Are Needed (Ghosting vs. N‑Key)
Without diodes, pressing certain **key combinations** can create unintended **current paths** that make the controller “see” extra keys that aren’t actually pressed. This is called **ghosting**.  

**Diodes** in **series with each switch** ensure current can **only** flow **from column → row**. That **blocks back‑paths** that would otherwise light up other rows/columns and cause ghost keys. With correctly placed diodes (one per switch), you prevent those false connections and enable **rollover** beyond 2 keys (often full NKRO, subject to firmware/OS limits).

**Orientation matters**:
- **Correct for COL2ROW**: **anode at column, cathode at row** (`column →|— row`).  
- If you **flip** a diode, that key will never read (or will act strangely), because current can’t reach the active LOW row.

---

## 4) What You Configure in QMK
Where you put it (legacy and modern layouts vary):
- **`DIODE_DIRECTION`** → usually `COL2ROW`. Commonly set in **`config.h`** (C define) or in **`rules.mk`** in older setups. In JSON keyboards, **`info.json`** often uses `diode_direction`.  
- **Matrix size** → `MATRIX_ROWS`, `MATRIX_COLS` in `config.h`, and per‑key wiring in `info.json`/keymaps.  
- **Pin lists** → `MATRIX_ROW_PINS[]`, `MATRIX_COL_PINS[]` (C arrays in `config.h`) *or* corresponding JSON fields in `info.json` (e.g., `matrix_pins`).

At runtime, QMK’s matrix code will:
- Set **columns** to **input‑pull‑up**.
- For each row in turn, **drive that row LOW**, read all columns, record which columns read **LOW** (pressed), then move on.

> Tip: Some advanced options (e.g., `MATRIX_UNSELECT_DRIVE_HIGH`) change how “inactive” rows are parked (input vs. driven), but **do not** change the fundamental **active‑LOW** sensing in COL2ROW.

---

## 5) Common Pitfalls
- **Wrong diode direction**: The most frequent wiring mistake. For COL2ROW, remember **`column →|— row`** (cathode to row).  
- **Floating inputs**: If your columns aren’t set to **pull‑up**, they can float and cause random presses. In QMK, columns are configured with internal pull‑ups.  
- **Crossed nets**: A single swapped row/column pin mapping can make an entire block of keys appear in the wrong positions. Double‑check `MATRIX_ROW_PINS`, `MATRIX_COL_PINS`.  
- **Debounce**: Real switches bounce; enable QMK’s debounce (e.g., `DEBOUNCE` in `config.h`) so brief oscillations aren’t read as multiple presses.

---

## 6) Glossary (Plain but Precise)
- **GPIO**: General‑Purpose Input/Output pin on the MCU. Can be configured as **input** or **output**.  
- **Input (with pull‑up)**: The MCU pin is reading a signal. “Pull‑up” means an internal resistor ties it weakly to **VCC**, so it idles **HIGH** unless pulled **LOW** by external circuitry.  
- **Output (driven LOW/HIGH)**: The MCU pin is actively forcing **0 V (LOW)** or **VCC (HIGH)**.  
- **HIGH / LOW**: Digital logic levels. In QMK’s COL2ROW scanning, a **LOW on a column input** (while its row is active) means **key pressed**.  
Current flows from **HIGH** to **LOW**. Current can never flow between same potentials like **HIGH** and **HIGH** or **LOW** and **LOW**. Usually **HIGH** is represented by **VCC** (~5 V or ~3.3 V) and **LOW** by 0 V or GND. 
- **Row / Column**: Two sets of wires forming a grid. Each key connects one row to one column.  
- **Diode**: One‑way valve for current. In COL2ROW, it passes current **from column to row** only (blocks the reverse).  
- **Ghosting**: False key detections caused by unintended current paths when pressing multiple keys without diodes.  
- **Debounce**: Filtering/ignoring rapid on/off chatter when a mechanical switch first opens/closes.

---

## 7) What Changes in ROW2COL (Quick Reference)
ROW2COL simply **swaps roles**:  
- **Columns** are **driven LOW one at a time**, and **rows** are **inputs with pull‑ups**.  
- **Diode orientation flips** accordingly: **anode at the row, cathode at the column** (`row →|— column`).  
- Firmware logic is otherwise the same idea: drive one side, read the other; a **LOW** on the input side (during that line’s active window) means pressed.

---

## 8) Schematic for the Docs
A KiCad 9 schematic named **`col2row_matrix_example.kicad_sch`** is provided alongside this document. It shows a **2×2 matrix** wired for **COL2ROW** (diodes from column to row). You can open it in KiCad and export an SVG/PNG to embed in this page.

> When in doubt about polarity: **the diode’s black band (cathode) faces the ROW.**

---

### Credits & Notes
- This rewrite aligns the explanation and the diode orientation with QMK’s **actual default COL2ROW** scanning behavior.  
- For deeper dives, see QMK’s `matrix.c` and your keyboard’s `config.h`/`info.json` for exact pin lists and options.
