# How a Keyboard Matrix Works

>[!NOTE]
>**Scope**: This guide explains matrix wiring and scanning. The first part offers a brief and beginner friendly explanation. The second section [An In-depth Discussion Of Keyboard Matrix](#an-in-depth-discussion-of-keyboard-matrix) offers a more detailed discussion to provide a more advanced understanding while still aiming at beginners.
>
> While the concept of a keyboard or switch matrix is universal, this guide considers implementation details of the QMK firmware. This doesn't affect the wiring (hardware-side implementation) but the firmware level scanning procedure.  
>
>QMK offers two matrix scanning algorithms which are called `COL2ROW` and `ROW2COL`. This guide focuses on the `COL2ROW` algorithm as this is the default configuration that is commonly used. `ROW2COL` will be briefly addressed in the end by highlighting the differences.

---

## A Brief Introduction For Beginners

## Quick Glossary

See appendix for [full glossary](#6-glossary-plain-but-precise) and external educational content links.

- **Current**: Flows from +/`HIGH` to –/`LOW` through a closed path.
- **Diode:** A small electronic device that only allows current to pass in one direction. If current is reversed, it will block. A diode is a electrical one-way barrier.
- **`HIGH`/`LOW`:** Logic levels. `HIGH` usually refers to the logic system voltage (usually 3.3 V or 5 V). `LOW` usually refers to 0 V.
- **Row / Column**: The two sets of wires that form the grid; each switch connects one row to one column.
- **COL2ROW**: Columns are inputs with pull-ups; the selected row is driven LOW.
- **Ghost (phantom) key**: A false keypress that can appear in a diode-less matrix when certain multi-key combos are pressed.
- **Scanning**: The periodical reading of the all key states (pressed/depressed or open/closed in switch terminology) by the microcontroller.
- **Microcontroller**: A chip device that run a program (firmware) to control and add behavior to  the hardware.

## The Concept

- A keyboard matrix aims to simplify wiring of switches by reducing the amount of physical pins required on the microcontroller. It removes complexity from the hardware level.
- In a matrix, all switches are arranged (wired) in rows and columns. Each row and each column is connected to an associated single pin on the microcontroller but to multiple switches ot once.

- Switches (keys) are abstractly viewed as logic circuits, where a open switch represent logical `HIGH` and a closed switch (pressed key) represents logical `LOW`.
- A microcontroller periodically scans all switches. It will visit each switch via the connected input pin to read whether it is `HIGH` (unpressed) or `LOW` (pressed).
- There are edge cases where a key can read pressed (`LOW`) despite the switch being open (`High`). This unwanted effect is called "ghosting". A dedicated diode at each switch is used to eliminate the ghosting effect.

## Before The matrix

Without a switch matrix every switch must be wired directly to a pin on the microcontroller:

![alt text](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/36_switches_direct_wiring_example_basic.png)  
***Figure 1:** 36 switches directly wired to a microcontroller pin with a common GND. Therefore, each key requires a dedicated pin (in this case 36 pins) which scales badly.*

As we can see in Figure 1, each switch requires a dedicated pin on the microcontroller. The microcontroller will then scan each switch independently via the dedicated net. The return line of the switch is usually the shared GND signal.  

To scan the switches, the microcontroller would pull-up the pin that connects to the currently scanned switch (i.e. setting the pin `HIGH`). As a result, the open switch would read `High`. When the switch is closed, the pin is pulled down to GND (0 V or logical `LOW`), which the microcontroller then interprets as a pressed key.

It's apparent that this kind of wiring is not very efficient in terms of scaling. Given the example's 36 switches, we would already exceed the limit of available pins on most common microcontrollers. Building a keyboard this way will be extremely difficult.

For example, a full-size keyboard usually comes with 104 (ANSI) or 105 (ISO) keys. This translates to 105 pins only for the switches. Such a microcontroller is out of scope.

The solution is to share pins between multiple switches by introducing more efficient wiring concept: the matrix design.

## The Matrix To The Rescue

To significantly reduce the amount of pins required for a keyboard, switches are organized in a matrix. A matrix electrically arranges the switches in rows and columns allowing the firmware to address each switch by a matrix index ([row;column]).

These rows and columns are not required to match the actual arrangement of the switches. Instead they describe the electrical wiring. However, to make working with the matrix easier e.g. key mapping, rows and columns usually follow the physical switch arrangement as close as possible.

![alt text](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/matrix_overlay.png)  
***Figure 2:** The actual electrical wiring of the keys overlayed on a keyboard layout to highlight how physical keys and electrical wiring are ideally equal. The virtual grid consisting of rows and columns is clearly visible, where each intersection (matrix coordinate) maps to an individual key.*

 For example, in a 6x17 matrix (6 rows and 17 column) a full-size keyboard consisting of 104 switches only requires 23 pins (6 rows + 17 columns). That's drastically less opposed to the 104 pins when wiring each switch directly. A matrix in this case saves ~78 % pins. And a microcontroller with 23 available pins is easy and cheap to source.

The following example shows a 6x6 matrix including diodes:  
![alt text](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/6x6_switch_matrix_example_basic.png)  
***Figure 3:** A 6x6 matrix including the anti-ghosting diodes. The schematic also highlights how only 12 pins are required to address 36 switches.*

The diodes are necessary to solve the ghosting effect problem.

## Scanning The Matrix

The scan is conducted by the firmware. It dynamically controls the pins of the microcontroller during scanning by configuring them as inputs or outputs and change their logic level from `HIGH`to `LOW` and back.

>[!TIP]
>In microcontroller systems the terms ***input* and *output* do not define the direction of how current/signals flows**. Current can also flow into an output or out of an input as the direction of current merely depends on voltage differences between connected pins (electrical potentials).
>
>*Input* and *output* describe the behavior or purpose of a pin. Furthermore, if a pin is configured to be digital, any analog voltage on that pin is interpreted as a logic signal (`HIGH` or `LOW`).
>
>- **Output:** The *output* pin is **active** (actively changing state). The *output* is **driving the circuit**. In other words, a *output* **sends signals**. If the *output* is digital, the signal is either `High` or `LOW`.
>- **Input:** The *input* pin is **passive** (passively changing state). The *input* is **reading from the circuit**. A *input* **receives signals** sent from an *output*.
>- **Current:** The direction a current flows is always defined by voltage potentials and not pin I/O configuration. Current always flows from the higher potential (higher voltage or a logical `HIGH`) to a lower potential (lower voltage or logical `LOW`).

The logical matrix scan algorithm is as follows:  

- To detect a key press, the rows are **actively driven** `LOW` and the **columns (`HIGH`) are scanned**. Current will flow from column (`HIGH`) through the potentially closed switch to the active row (`LOW`). Therefore, **columns are passive** and **rows are active**.  

>[!IMPORTANT]
>**Columns** are *always* (i.e. permanent) inputs, set to `HIGH` ("pulled up") whereas **rows** are *temporarily* (during activation) configured as outputs and *driven* **`LOW` when activated** and reset to be an input and pulled back `HIGH` when inactive (after all column have been scanned).

- **Next, a row is activated** (pulled `LOW`) and the microcontroller will start to **scan each column** to look for a `LOW` column (the column is pulled `LOW` when connected to another `LOW` signal (the row) via a closed switch).
- After each column was read, the **currently active row is deactivated** by resetting it back to input behavior and pulling it back `HIGH`. **The microcontroller advances to the next row**.
- The microcontroller activates the next row and again reads every column one-by-one to look for a `LOW`.
- The scan-columns-and-advance-to-next-row procedure continues until the full matrix has been read i.e. all rows have been visited.

>[!TIP]
>This logic behavior, where the default logic level is `HIGH` and the activation is signalled by a `LOW` level, is called "active-low" logic (or sometimes also called "negative logic").  
>
>If the logic had been inverted, which is the logic's default is `LOW` and levels go `HIGH`on activation, this would be an "active-high" logic or behavior (sometimes also referred to as "positive logic").  
>
>QMK's matrix scanning logic is active-low: `HIGH` is the default and `LOW` the activation signal, in this case switch closed.

## Controller Pin Equivalent Circuit

Do get an idea how current flows and why the logic levels (voltages) are actually changing (why inputs (columns) change their logic state), we must analyze the electrical equivalent circuit of a microcontroller pin. This can help to understand the logic behavior.  

>[!TIP]
>A microcontroller integrates multiple circuits that consist of multiple integrated parts like diodes, resistors, capacitors, FETs, inductors and other semi-conductor elements.
>
>The pull-up resistors that are depicted in the following images are such integrated parts, that form an integrated circuit.  
The following equivalent circuits are simplifications of such integrated circuits.

![alt text](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/equivalent_circuit_unpressed.png)  
***Figure 4:** The simplified electrical equivalent circuit of microcontroller pins as configured for the switch matrix. The image shows the unpressed key state (the switch `Key1`is open).*

**Figure 4** shows the electrical equivalent circuit of an open switch connection between a row and a column. The **blue arrows symbolize a voltage drop and red arrows a current**.

The image shows how the input pin (on the right) is pulled `HIGH` by a pull-up resistor. The pull-up resistor basically creates an internal connection between the pin and the microcontroller's supply voltage `VDD`, which is most commonly 3.3 V. The pull-up resistor is integrated into the controller. With `VDD = 3.3 V`, a voltage within the range of 2.3-3.3 V on the pin is typically interpreted as logic `HIGH` by the microcontroller.

On the left side we see the activated row pin (hence configured as output and pulled `LOW`). The pin is temporarily pulled `LOW` by internally connecting it to the ground net `GND` (0 V). With `VDD = 3.3 V`, a voltage < 1 V is typically interpreted as logic `LOW` by the microcontroller.

>[!NOTE]
>The thresholds for logic `HIGH` and `LOW` depend on the microcontroller and its supply voltage and can vary. The values mentioned above are typical values for a 3.3 V microcontroller.

Both pins are connected to each other by a switch `Key1` (the actual keyboard key) and a diode `D1`. The diode is required to mitigate the ["ghosting effect"](#).

Since the switch `Key1` is unpressed, no current can flow from input to output: no voltage drop across the internal pull-up `R1` and the external diode `D1`. As the consequence, the input (column) remains `HIGH`, which the firmware interprets as an unpressed key.

>[!TIP]
>The voltage only changes (e.g. from `HIGH` to `LOW`) when a current is flowing between two nodes of different potentials.  
The current then flows from the higher electrical potential (here always logic `HIGH`) to the lower electrical potential (here logic `LOW`). This means, the direction of the current is *always* from `HIGH` to `LOW` and independent from the pin's behavior (input or output).

To change the logic level of the `HIGH` input (column) to `LOW`, a current must flow from the input (column) via the switch to the output (row). This means, a row must be `LOW` (activated) and the switch must be closed (see *Figure 5* below).

![alt text](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/equivalent_circuit_pressed.png)  
***Figure 5:** The simplified electrical equivalent circuit of microcontroller pins as configured for the switch matrix. The image shows the pressed key state (the switch `Key1` is closed).*

**Figure 5** shows the same electrical equivalent circuit as in **Figure 4**. But this time the switch `Key1` is pressed (closed). As a consequence, a tiny current (here 82.5 µA) will now flow from the input pin (column) to the output pin (row). Now, current is flowing through the pull-up resistor `R1` and the diode `D1`, finally causing a voltage drop across both (a 3.0 V drop across `R1`) and the diode `D1` (a 0.3 V drop).

The result is a voltage of 0.3 V on the input pin (column) which the controller interprets as a logic `LOW` (remember, a voltage < 1 V is typically interpreted as logic `LOW`). The QMK firmware will read this `LOW` on the column pin and interpret it as a pressed key.

Now that we understand some of the electric fundamentals of how and why currents flow and voltages drop in a matrix we can get deeper into how scanning of a switch matrix works.

## Example: A Full Matrix Walk

The following section walks through a full matrix scan step-by-step, exemplified by a 2x2 switch matrix.

### 0. Initial State Before Scanning

In the beginning, when the firmware is initialized, **all columns and rows are configured as inputs and pulled up `HIGH`** (see **Figure 6** below).  

![alt text](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/2x2_matrix_scanning_example_step_0.png)  
***Figure 6:** The 2x2 example matrix. The schematic shows two pressed keys `Key1` and `Key4` (one in each column and row). Additionally, the schematic highlights how all pins (rows and columns) are initially `HIGH` and therefore inactive. Especially the rows are inactive which becomes apparent when looking at the row's arrow pointing direction, which points into the circuit ==> input. Later, the row will be activated and becomes an output (arrow pointing out of the circuit) that is pulled `LOW`.*

#### Discussion Initialization Step

- While the **columns will never change their configuration** (they will *always* stay inputs that are pulled up), the **rows will be temporarily configured as outputs and pulled `LOW`**, one after the other.  

- During the matrix scan, there will be only a single row configured as output and pulled `LOW` at the time: the currently active row.

>[!NOTE] Summary
>Columns and rows are ***always*** inputs and `HIGH` by default, whereas **rows are temporarily configured as outputs and pulled `LOW` only when activated**. After all columns of the current roe h ave been scanned, the current row is reset back to a `HIGH` input (disabled, inactive state).

### 1. Scanning First Column Of First Row

The scanning starts with the first column `COLUMN0` of the first row `ROW0` by configuring the `ROW0` pin as output (until all columns of this row have been scanned). Now, to allow a potential current to flow, the row pin is pulled `LOW` by te firmware (current flows from `HIGH` to `LOW` i.e. column to row).  

![alt text](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/2x2_matrix_scanning_example_step_1.png)  
***Figure 7:** `ROW0` is activated (label is bold and blue) by configuring it as output (label arrow points out of the circuit) and pulling the pin `LOW`. Then the first column `COLUMN0`  is scanned (label is bold and blue). In this first column scan the firmware detects a key press, since `COLUMN0` has changed from `HIGH` to `LOW` as the result of the closed switch `Key1`. The green arrows show how the current is flowing.*

#### Discussing Step #1

- Because the switch `Key1` is closed, the current can flow from `COLUMN0` to `ROW0`. The result is that the column `COLUMN0` is also pulled `LOW` (voltage drops as current flows through the column pin's internal resistor and the diode `D1` into the row pin `ROW0` - see [Controller Pin Equivalent Circuit](#controller-pin-equivalent-circuit) to learn about the electrical details).
- When the microcontroller scans the first column it now reads a `LOW` and interprets it correctly as a closed switch (pressed key).
- Since the other column switches of  the current row are not closed (here `Key3`), no current can flow between any of the other columns and the currently activated row `ROW0`. The open switch means that there is no physical connection between column and row.  
However, even *if* the other switches had been closed, the microcontroller will not detect them in the current step as only `COLUMN0` is scanned at the moment. **All other columns are ignored until they are scanned in their respective step.**

>[!NOTE] Summary
>Scanning `COLUMN0` with `ROW0` activated (pulled `LOW`) detects a key press because the switch `Key1` is closed (current flows from `COLUMN0` to `ROW0`). The microcontroller reads a `LOW` on `COLUMN0` and interprets it as a pressed key.

### 2. Scanning Second Column Of First Row

The firmware advances to the next column `COLUMN1` of the same row `ROW0`.

![alt text](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/2x2_matrix_scanning_example_step_2.png)  
**Figure 8:**: `ROW0` is still activated (label is bold and blue) by being configured as output (label arrow points out of the circuit) and pulled `LOW`. Now, the second column `COLUMN1`  is scanned (label is bold and blue). In this second column scan the firmware does not detect a key press, since `COLUMN1` remains `HIGH` as the result of the open switch `Key3`. The green arrows show that no current is flowing.*

#### Discussing Step #2

- Since the switch `Key3` is open, no current can flow from `COLUMN1` to `ROW0`. Therefore, `COLUMN1` remains `HIGH` (the pull-up resistor holds it up).
- When the microcontroller scans the second column it reads a `HIGH` and interprets it correctly as an open switch (unpressed key).
- `COLUMN0` is still `LOW` (pressed key --> current flows from `COLUMN0` to `ROW0`) but is ignored in this step as only `COLUMN1` is scanned at the moment.


For every row that is pulled `LOW` the microcontroller will read every column to detect a `LOW` (a key press). Because every column is `HIGH` by default and the active row

- Keyboard switch matrices are arranged in rows and columns. Without a matrix circuit, each switch would require its own two wires (in and out) directly to the controller.<br>  
Keyboard switch matrices are arranged in rows and columns. Without a matrix, each key typically needs its own controller pin (plus a common return pin, usually the shared GND), so a 104-key board would require about 105 GPIOs (which can make controller selection challenging).  <br>
A matrix reduces this dramatically: for example, 6 rows × 17 columns (6x17 matrix) needs only 23 pins. Controllers with ~20–30 GPIOs are common, so this is practical.<br>
![matrix overlay](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/matrix_overlay.png)  
***Figure 1** Example 75 % keyboard matrix overlay to highlight the electrical arrangement of keys switches in rows and columns.*<br><br>

- Think of a keyboard as a **grid**: **columns** on one side, **rows** on the other. Each key is a **switch** at the intersection of one column and one row, with a **diode** in series so current can only flow **from the column toward the row**.<br>
In other words, in a switch matrix, each switch maps to a unique matrix coordinate described by row number and column number. Using this coordinate, the MCU indexes a lookup table to determine which key was pressed.<br>
![2x2 switch matrix](https://bioniccode.github.io/qmk-documentation-resources/resources/how-a-matrix-works/images/2x2_switch_matrix_basic.png)  
***Figure 2** An example 2x2 switch matrix. It shows how four controller pins (ROW0, ROW1, COLUMN0, COLUMN1) allow the addressing of 4 switches. Without a matrix, 8 pins would have been necessary - two pins for each switch.*<br><br>

- QMK scans by **activating one row at a time**: it **drives that row LOW** (to 0 V).
- All **columns** are **inputs with pull‑ups** (they sit HIGH by default). 
- If a key on the active row is pressed, current flows **from the column’s pull‑up → switch → diode → the LOW row**, pulling that **column input LOW**. QMK reads that LOW as “key pressed.”
- **Diodes prevent ghosting** (false key detections) by letting current flow in only one direction.

---

## An In-depth Discussion Of Keyboard Matrix

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

### Current

Current (I): The rate of flow of electric charge, measured in amperes (A). In circuits, current flows when there is a closed path between two or more electric potentials/voltage differences (ΔV).  

By convention, current is taken to flow from higher electric potential (higher voltage, labelled "+" or in logics "logical HIGH") to a lower potential (lower voltage, labelled "-" or in logics "logical LOW").

**With no voltage difference between two points**, there is no conduction current between them in steady state. In many microcontroller systems, `HIGH` is near the supply (`VCC`, commonly ~5 V or ~3.3 V) and `LOW` is 0 V (`GND`) (see [Logical HIGH/LOW](#logical-highlow)).

See Wikipedia: [Electric current](https://en.wikipedia.org/wiki/Electric_current).

### Logical HIGH/LOW

Digital logic levels that represent a binary state system. In general, `HIGH` is represented by `VCC` (~5 V or ~3.3 V, see [VCC/VDD](#vccvdd)) and `LOW` by 0 V or `GND` (see [GND/VSS](#gndvss)).  

Current flows from `HIGH` to `LOW`.

In practice, the exact voltage thresholds for what constitutes `HIGH` and `LOW` depend on the specific microcontroller and its supply voltage. For example, in a 3.3 V system, a voltage above approximately 2.0 V might be considered `HIGH`, while a voltage below approximately 0.8 V might be considered `LOW`.  
Voltages between these thresholds may be undefined or indeterminate.

In other words, logical `HIGH` and `LOW` are mapped to specific voltage *ranges*, not *exact* voltages. The microcontroller interprets these voltage levels as binary states.

In binary/boolean algebra, `HIGH` often represents a binary `1` (or boolean `TRUE`) and `LOW` a binary `0` (or boolean `FALSE`).  
Based on the actual active state of a system (see [Active-HIGH/Active-LOW](#active-highactive-low)), `HIGH` can represent either an active or inactive state. The same applies to `LOW`.

In QMK’s **COL2ROW** scanning, a `LOW` on a column input (while its row is active) means key pressed. A `LOW` on a row output means that row is active.

See Wikipedia: [Logic level](https://en.wikipedia.org/wiki/Logic_level).

### Active-HIGH/Active-LOW

The terms "active-high" and "active-low" describe how a signal behaves in relation to its logical state.

In an active-low configuration, the signal is considered "active" or "asserted" when it is at a `LOW` voltage level (close to 0 V or `GND`, see [GND/VSS](#gndvss)).

Conversely, in an active-high configuration, the signal is considered "active" when it is at a `HIGH` voltage level (close to `VCC`, typically 3.3 V or 5 V, see [VCC/VDD](#vccvdd)).

See [Logical HIGH/LOW](#logical-highlow) for more information on logic levels.
See Wikipedia: [Active-low](https://en.wikipedia.org/wiki/Active_low) and [Active-high](https://en.wikipedia.org/wiki/Active_high).

### GPIO

General‑Purpose Input/Output pin on the MCU. Can be configured as **input** or **output** (see [Input/Output](#inputoutput)). Opposed to dedicated pins like UART, SPI, I2C, etc. GPIOs are flexible and can be used for various purposes, including reading switches in a keyboard matrix.

See Wikipedia: [General-purpose input/output](https://en.wikipedia.org/wiki/General-purpose_input/output).

### Input/Output

In microcontroller systems the terms **input** and **output** do not define the direction of how current/signals flows. Current can also flow into an output or out of an input as the direction of current merely depends on voltage differences between connected pins (electrical potentials). Instead, these terms define informational behavior or purpose of a pin.

Furthermore, if a pin is configured to be digital, any analog voltage on that pin is interpreted as a logic signal (`HIGH` or `LOW`, see [Logical HIGH/LOW](#logical-highlow)).

#### Input

The pin is **passive** (passively changing state). The pin is **reading from the circuit**. An input **receives signals** sent from an output (external circuit).

#### Output

The pin is **active** (actively changing state and therefore sending information). The pin is **driving the circuit**. An output **sends signals**. If the output is digital, the signal is either `HIGH` or `LOW`.

See Wikipedia: [Input/output](https://en.wikipedia.org/wiki/Input/output).

### Pull‑up Resistor

A resistor that connects a pin to `VCC` (the positive supply voltage, see [VCC/VDD](#vccvdd)) to ensure the pin reads `HIGH`. Such a pin must be actively driven `LOW`  to change state (see [Active-HIGH/Active-LOW](#active-highactive-low)). Pull‑ups prevent floating inputs, which can cause erratic behavior.

A floating pin is a pin that is not connected to a definite voltage level (neither `HIGH` nor `LOW`). This can lead to unpredictable readings, as the pin may pick up noise or interference from the environment, causing voltage to fluctuate, potentially from 0 V to `VCC` resulting in constant and unpredictable toggling between `HIGH` and `LOW`. By using a pull‑up resistor, the pin is tied to a known state (`HIGH`), ensuring reliable and predictable operation.

Typically, pull‑up resistors have high resistance values (e.g., 10 kΩ to 100 kΩ) to limit current flow when the pin is driven `LOW`. Such pull-ups are considered "weak" because they allow the pin to be easily pulled `LOW` by an external circuit (like a switch).

Microcontrollers often have **internal pull‑up resistors** that can be enabled via software configuration, eliminating the need for external components.

See Wikipedia: [Pull-up resistor](https://en.wikipedia.org/wiki/Pull-up_resistor).

### Pull‑down Resistor

A resistor that connects a pin to `GND` (0 V, see [GND/VSS](#gndvss)) to ensure the pin reads `LOW`. Such a pin must be actively driven `HIGH` to change state (see [Active-HIGH/Active-LOW](#active-highactive-low)). Pull‑downs prevent floating inputs, which can cause erratic behavior.

A floating pin is a pin that is not connected to a definite voltage level (neither `HIGH` nor `LOW`). This can lead to unpredictable readings, as the pin may pick up noise or interference from the environment, causing voltage to fluctuate, potentially from 0 V to `VCC` resulting in constant and unpredictable toggling between `HIGH` and `LOW`. By using a pull‑down resistor, the pin is tied to a known state (`LOW`), ensuring reliable and predictable operation.

Typically, pull‑down resistors have high resistance values (e.g., 10 kΩ to 100 kΩ) to limit current flow when the pin is driven `HIGH`. Such pull-downs are considered "weak" because they allow the pin to be easily pulled `HIGH` by an external circuit (like a switch).

Pull-down resistors are less common than pull-up resistors in microcontroller applications, as many microcontrollers provide internal pull-up resistors but not internal pull-down resistors.

See Wikipedia: [Pull-down resistor](https://en.wikipedia.org/wiki/Pull-down_resistor).

### VCC/VDD

The positive supply voltage for a circuit or device. Common values are 3.3 V and 5 V in microcontroller systems. `VCC` is often used interchangeably with `VDD`, although technically `VCC` refers to the collector supply voltage in bipolar junction transistor circuits, while `VDD` refers to the drain supply voltage in field-effect transistor circuits.

See Wikipedia: [V_CC](https://en.wikipedia.org/wiki/V_CC) and [V_DD](https://en.wikipedia.org/wiki/V_DD).

### GND/VSS

The ground reference point in a circuit, typically 0 V. All voltage levels are measured with respect to this point.

`GND` is often used interchangeably with `VSS`, although technically `GND` refers to the ground reference in general, while `VSS` specifically refers to the source supply voltage in field-effect transistor circuits.

See Wikipedia: [Ground (electricity)](https://en.wikipedia.org/wiki/Ground_(electricity)) and [V_SS](https://en.wikipedia.org/wiki/V_SS).

### Diode

A semiconductor device that allows current to flow in one direction only. In keyboard matrices, diodes are used to prevent ghosting by ensuring that current can only flow from the column to the row (in COL2ROW configuration).

The diode has two terminals: the **anode** (positive side) and the **cathode** (negative side, often marked with a band). In COL2ROW, the anode is connected to the column and the cathode to the row (see [Anode/Cathode](#anodecathode)).

Current flows from the anode to the cathode when the diode is forward biased (anode voltage is higher than cathode voltage). If the diode is reverse biased (cathode voltage is higher than anode voltage), it blocks current flow.

In other words, a diode acts like a one-way valve for electric current.

Different types of diodes exist, each with specific characteristics such as forward voltage drop, maximum current rating, and switching speed. For keyboard matrices, small signal diodes are typically used due to their fast switching times and low forward voltage drop.

Famous diode types used in keyboard matrices include 1N4148 (small signal diode) and 1N400x series (general purpose rectifier diodes).

See Wikipedia: [Diode](https://en.wikipedia.org/wiki/Diode).

### Anode/Cathode

- **Anode**: The positive terminal of the diode. In COL2ROW, it connects to the column side of the switch.
- **Cathode**: The negative terminal of the diode (often marked with a band). In COL2ROW, it connects to the row side of the switch.

The cathode is usually marked with a band on the diode package.

See Wikipedia: [Diode#Polarity](https://en.wikipedia.org/wiki/Diode#Polarity).

- **Row / Column**: Two sets of wires forming a grid. Each key connects one row to one column.  
- **Diode**: One‑way valve for current. In COL2ROW, it passes current **from column to row** only (blocks the reverse).  
- **Ghosting**: False key detections caused by unintended current paths when pressing multiple keys without diodes.  
- **Debounce**: Filtering/ignoring rapid on/off chatter when a mechanical switch first opens/closes.

- **Anode**: The plus terminal of the diode.
- **Cathode**: The minus terminal of the diode.
- **Controller**: The micro controller (MCU) that controls the keyboard.
- **Current**: Current (I): The rate of flow of electric charge, measured in amperes (A). In circuits, current flows when there is a closed path between two or more electric potentials/voltage differences (ΔV). By convention, current is taken to flow from higher electric potential (higher voltage, labelled "+" or in logics "logical HIGH") to a lower potential (lower voltage, labelled "-" or in logics "logical LOW"). With no voltage difference between two points, there is no conduction current between them in steady state. In many microcontroller systems, HIGH is near the supply (VCC, commonly ~5 V or ~3.3 V) and LOW is 0 V (GND).
- **Diode**: One‑way valve for current. In *COL2ROW*, it passes current **from column to row** only (and blocks the reverse).
- **GPIO**: General‑Purpose Input/Output pin on the MCU. Can be configured as **input** or **output**.  
- **HIGH / LOW**: Digital logic levels. In QMK’s *COL2ROW* scanning, a **LOW on a column input** (while its row is active) means **key pressed**.
- **Row / Column**: Two sets of physical wires forming a grid (the matrix). Each key connects one row to one column.
- **Scan rate**: The frequency of full matrix scans. A full scan includes activating every row for each currently scanned column. **Activating a row** means setting it **LOW**. **Scanning a column** means detecting a **LOW** or **HIGH**. A full scan usually takes ~0.5-2 ms (0.5-3 kHz) on modern controllers.


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
