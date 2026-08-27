# Try Before You Buy (TBYB) — Changes Made (August 11, 2026)

This document summarizes the changes made to the **Try Before You Buy / Prescription Frames flow** in `try-before-you-buy.html`.

## 1. Prescription Frames Selection

Added/updated the **Prescription Frames** flow where customers can:

* Browse prescription-compatible frames.
* View the available colors for each frame.
* See the prescription/Rx range supported by each frame.
* Select a frame and color before entering prescription information.

The selected frame is visually highlighted and the button changes from **Select Frame** to **Selected**.

The customer's selection is carried into the prescription process and displayed in the format:

`Selected: Cover Over — Black — $69`

---

## 2. Previous Try Before You Buy Order

After selecting a prescription frame, the customer is asked:

**Did You Complete Try Before You Buy Before This?**

The customer can optionally enter their previous TBYB **Order #**.

The purpose is to allow an existing Try Before You Buy deposit/order to be associated with the prescription order.

If they did not previously complete TBYB, they can leave the field blank and continue normally.

---

## 3. Prescription Flow

The prescription configuration is organized into a **5-step flow**.

A numbered progress indicator shows:

`1 — 2 — 3 — 4 — 5`

The selected frame, color, and price remain visible during the process.

---

## 4. Prescription Entry

The prescription step contains separate fields for:

### OD — Right Eye

* Sphere
* Cylinder
* Axis

### OS — Left Eye

* Sphere
* Cylinder
* Axis

The page also displays the supported sphere range for the selected frame.

Example:

`Sphere power for this frame must fall within -6.00 to +4.00.`

---

## 5. PD Input

The PD portion supports both:

* A single PD number
* Two different PD numbers

When the customer selects **I have 2 different PD numbers**, separate **Left PD** and **Right PD** inputs are displayed.

---

## 6. Dual-PD Ranges

The dual-PD dropdown ranges were updated to match the required prescription ranges.

### Left PD

`24.5 – 35`

in increments of:

`0.5`

### Right PD

`25 – 37`

in increments of:

`0.5`

---

## 7. Left/Right PD Ordering

The PD inputs were reordered so that the UI displays:

**Left PD on the left**

and

**Right PD on the right**

This ordering was also updated in the review/summary so the display remains consistent throughout the flow.

---

## 8. Removed "None" From Dual-PD Fields

Previously, the **Left PD** and **Right PD** dropdowns contained a `None` option.

That option was removed.

When the customer chooses:

**I have 2 different PD numbers**

they are expected to provide an actual value for both PD fields.

---

## 9. Dual-PD Validation

Validation was added so a customer cannot continue through the PD step without completing the required PD information.

### Single PD

If the customer is using one PD number:

* The single PD field must contain a value.

### Dual PD

If **I have 2 different PD numbers** is selected:

* Left PD is required.
* Right PD is required.
* The customer cannot continue until both have been selected.

---

## 10. PD Validation Step Fix

An issue was discovered where the validation existed but was checking the **wrong step**.

The PD fields actually live in **Step 2**, while the validation had originally been attached to **Step 3**.

Because of this, a customer could select **I have 2 different PD numbers**, leave both PD fields empty, and still continue.

The validation was moved to the correct step.

### Correct behavior

When the user reaches **Step 2**:

* Single PD selected → PD is required.
* Two PD numbers selected → both Left PD and Right PD are required.
* Missing required PD information prevents the user from continuing.

---

## 11. PD Informational Text Cleanup

Extra explanatory text around the PD fields was removed.

Step 2 now keeps only the **"Please note"** informational message/banner.

The additional PD explainer paragraph was removed to make the step cleaner and avoid redundant information.

---

## Final PD Behavior

The completed PD behavior should now be:

**One PD number**

`PD: [required dropdown]`

**Two PD numbers**

`Left PD: [24.5–35]    Right PD: [25–37]`

Both dual-PD dropdowns:

* Move in `0.5` increments.
* Do not contain `None`.
* Are required.
* Are validated during Step 2.
* Prevent continuing when incomplete.

The Left/Right ordering is also consistent in both the input UI and final review summary.

