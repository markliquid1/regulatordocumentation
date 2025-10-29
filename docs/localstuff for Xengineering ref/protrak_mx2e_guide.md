# ProtoTRAK MX2e .CAM File Programming Guide (Proven Style)

This guide is based **only** on known-working `.CAM` programs that run correctly on the MX2e. All examples and formatting rules below come directly from those files. No speculative formats are included.

## NOTE
- Read this WHOLE file including the conclusions before proceeding

## Format Rules

### Program Header & Footer
- Start with `%`
- Second line must be `O0000`
- End with `M30` and `%`

### N-Numbering
- Use N-numbers incrementing by 10 (N100, N110, …)
- Every block must have its own N-number

### Coordinates
- Always include a trailing period for whole numbers
  - `X0.` not `X0`
  - `F15.` not `F15`
- Decimals are kept short (no `0.0000` → use `0.` or `-.4375`)

### Arcs (I/J)
- Use concise decimal style:
  - `I0. J.3` not `I0.000 J0.300`
  - `I-.4375 J0.` not `I0.000 J-0.4375`
- Zero values must be explicit with a period (`I0. J0.`)

### Feed Specification 
- Feedrates **must be specified on a motion block**, never alone
- Known working style: combine feed with a small nudge move. Example:
  ```
  N110 G01 X-2.7372 Y-2.964 F15.
  ```
  This ensures the control accepts the feed and avoids "point" errors

### Modal Feed Rate Management 
- **Feed rates are modal** - they stay active until explicitly changed
- **Plunge feeds contaminate cutting feeds** - always reset after Z moves
- **Reset cutting feed immediately** after any slow plunge or positioning move

### CRITICAL FUCKS UP TO BE AVOIDED BY AI
- **ALL FEED RATES MUST BE THE SAME IN FINAL RESULT YOU DELIVER TO ME** - Check this at the very end- rescan the file and make sure no other feed rates are present.  Just the one i ask for, generally 54.  Ask if you have questions about this.
- **EXTRANEOUS BULLSHIT** - Do not add or keep COMMENTS in the code.  Do not add or keep blank lines.  The programs need to look like my examples, no extra bullshit.
- **NUDGE AMOUNT MAGNITUDE** 0.0001" nudge is not enough. Nudge by 0.001".  Double check at the end that there are no point events.  You consistently fuck this up.
- **2D MACHINE** The MX2e is 2D only.  Strip out all Z events, always.
- Never Omit Complete Operations
When converting G-code, **every distinct machining operation must be preserved**. Missing operations create incomplete parts with wrong geometry.

### How to Identify Separate Operations
Look for these indicators of operation breaks:

1. **Rapid positioning to new start points** - Large coordinate jumps often signal new operations
2. **Z-level changes** - Different cutting depths usually indicate separate operations  
3. **Distinct toolpath patterns** - Pocket vs contour vs island machining have different motion signatures
4. **Helical entry sequences** - Ramping entries often start new operations    NOTE WE DO NOT DO HELICAL MOVES, WE ARE 2D

### Operation Analysis Checklist
Before converting, analyze the original G-code:

**Step 1: Count Operations**
- Identify all rapid moves to new start coordinates
- Look for repeated entry patterns (G18 arcs, ramping sequences)
- Note major toolpath pattern changes

**Step 2: Verify Each Operation**

**Step 3: Preserve All Operations**
- Convert each operation separately in the .CAM file
- Use comments `(OPERATION NAME)` to mark sections during development
- Remove comments in final .CAM but maintain operation separation.   Operations give opportunity for the operator to change the Z height when needed (manual quill)


### When in Doubt
- **Preserve rather than combine** - Better to have extra operations than missing features
- **Analyze the part geometry** - What features require what operations?
- **Check for circular arcs** - These often indicate island or boss operations that are easily missed

**Problem example:**

### 3D Moves and Plane Changes  
- **NEVER use G18 or G19 plane arcs** - MX2e only handles G17 (XY plane)
- **Convert all K values to separate moves** - no XZ or YZ arcs allowed
- **Break helical interpolation into linear segments**
- When converting G-code: if you see `K` values or plane changes, split into separate X, Y, Z moves

### Point Error Fix (5104)
- When the end of one block matches the start of the next, ProtoTRAK may throw error **5104 ("point event")**
- Fix: shift by 0.001" (imperceptible to the part). Example:
  ```
  N140 X-2.4371 Y2.665
  ```
  instead of
  ```
  N140 X-2.4372 Y2.665
  ```

### Arc Quadrant Handling  
The MX2e arc processor has **quadrant limits**. Arcs that span more than 180° or terminate exactly on a quadrant axis (0°, 90°, 180°, 270°) can cause:

- **Error 5104 ("point event")**  
- The control taking the **short arc path** instead of the intended sweep  
- Crashes when I/J math says one thing but the interpreter resolves another  

**Safe programming rules:**

1. **Never program arcs >180°.** Break long arcs into two or more smaller arcs (e.g. a 270° arc → three 90° arcs).  
2. **Avoid exact axis endpoints.** If an arc lands exactly on X0. or Y0., nudge by ±0.001" to bypass the "point" trap.  
   - Example: use `X0.001 Y0.5` instead of `X0. Y0.5`  
3. **Always include explicit I and J with a trailing period,** even when one is zero (`I0. J.5`).  
4. **Check direction.** For climb vs conventional, confirm your G02/G03 matches the intended tool side — the MX2e will happily cut the wrong side if I/J signs are flipped.  
5. **Simulate by plotting.** The MX2e "Plot" function shows arc paths; verify every quadrant before cutting.  

**Example — bad vs. safe 270° CCW arc:**

❌ Risky single arc (often errors out):  
```
N120 G03 X0.5 Y0.5 I0. J0.5
```

✅ Safe split into quadrants:  
```
N120 G03 X0. Y0.5 I0. J0.5
N130 G03 X-0.5 Y0. I-.5 J0.
N140 G03 X0. Y-0.5 I0. J-.5
```

This avoids quadrant crossing errors and ensures predictable climb milling.

## Example Programs

### Example 1 — Drilling Pattern

From a known good file (`00027.CAM`):

```
%
O0000
N100 G00 X1.000 Y1.000
N110 G81 X1.000 Y1.000 R0.1 Z-0.500 F5.
N120 G80
N130 G00 X0. Y0.
N140 M30
%
```

**What it does:** Drills a hole at (1.0, 1.0) to depth -0.500 with feed 5.0 ipm. (not that the depth matters, Z is a manual quill on this machine)

### Example 2 — Contour Milling with Nudge Feed

From a known good file (`00065.CAM`):

```
%
O0000
N100 G00 X-3.5482 Y-.1386
N110 G01 X-3.5482 Y-.1396 F5.
N120 G01 F6.42
N130 X-3.4086 F5.
N140 G03 X-3.269 Y0. I0. J.1396
N150 G02 X0. Y3.269 I3.269 J0.
N160 X3.269 Y0. I0. J-3.269
N170 X0. Y-3.269 I-3.269 J0.
N180 X-3.269 Y0. I0. J3.269
N190 G03 X-3.4086 Y.1396 I-.1396 J0.
N200 G01 X-3.5482
N210 F6.42
N220 G00 X0. Y0.
N230 G01 X0. Y0.001
N240 M30
%
```

**What it does:** Mills a rounded rectangular contour, using climb arcs.
- Feed applied on a nudge line (N110)
- 0.001" shift used at N140 to prevent error 5104

### Example 3 — Rectangle Mill

From a DEFINITELY good file:

```
%
O0000
N100 G00 X-2.3125 Y-2.3125
N110 G01 X-2.3115 Y-2.3125 F36.
N120 X2.3125 Y-2.3125
N130 X2.3125 Y2.3115
N140 X2.3125 Y2.3125
N150 X-2.3115 Y2.3125
N160 X-2.3125 Y2.3125
N170 X-2.3125 Y-2.3135
N180 G00 X0. Y0.
N190 M30
%
```

**What it does:** Mills a rectangular box.

## Preferred Workflow For Programming

### CAD/CAM Setup
- **Fusion360**: Import Solidworks model to Fusion 

When you import an IGES file into Fusion 360, it often comes in as **surface bodies** instead of solids. This can prevent pocketing operations from recognizing closed boundaries and islands.

Better choices for coming out of SolidWorks → Fusion:

1. STEP (.STEP / .STP) → Best general-purpose. It carries solids (BREP) cleanly, Fusion stitches them correctly.
2. Parasolid (.X_T / .X_B) → NO, MY LICENSE WON'T ACCEPT IT
3. Native SW file (.SLDPRT / .SLDASM) → Fusion can open these directly if you enable the SolidWorks translator in Fusion. It preserves parametrics where possible.  (NEED TO TEST, MIGHT NOT WORK)
👉 Rule of thumb: STEP > IGES.

---
When stuck with IGES- Identifying Surface Bodies
- In the **Browser**, expand *Bodies*.
- Icons:
  - **Cube** = Solid Body
  - **Soup can wrapper with a slit** = Surface Body
- Right-click → *Properties*:
  - Solid shows **Volume + Area**
  - Surface shows **Area only**

---

Step 1: Stitch Surfaces
1. Switch to the **Surface** tab.
2. Use **Stitch**.
3. Select all imported surfaces (e.g., 77 bodies).
4. Set **tolerance** (start with `0.001 in` / `0.025 mm`).
5. Confirm:
   - If the surfaces form a closed volume → Fusion creates a **Solid Body**.
   - If gaps remain → Stitch fails, leaving Surface Bodies.

---

Step 2: Repair if Needed
- Use **Surface → Patch** to fill missing holes.
- Re-run **Stitch** until Fusion produces a solid.

---

Step 3: Verify
- Browser now shows **Solid Bodies (1)** (or more, depending on geometry).
- Right-click → *Properties* should show **Volume**.

---

Step 4: Create Pocket Toolpath
1. Switch to **Manufacture workspace**.
2. Use **2D Pocket**.
3. For **Pocket Selections**, click:
   - The **outer loop** of the pocket.
   - Any **inner loops** (islands).
4. Fusion will now generate a toolpath clearing the region while leaving islands intact.

---

- **Programming**: Generate toolpaths this way - so you can get proper offsets, lead in/lead out, tool size compensation, etc. Use Autodesk Generic 3 Axis machine and Artsoft mach3mill.cps post. (Southwestern Industries prototrak conversational.cps sounded better initially, but threw some errors.) These choices were random but seem to work.

### AI Conversion
- **AI**: Use Claude to convert G code from above into .CAM file, using this guide also attached to the prompt. Make sure the line endings are handled correctly:

## Line Endings Fix - HTML Download Method

### The Problem
- Files downloaded from browsers (Claude site) often convert to Unix LF endings (`\n`) 
- The MX2e requires **Windows CRLF line endings** (`\r\n`) to read .CAM files properly
- Standard copy/paste or download methods lose the proper line ending format

### Solution: Claude HTML Download Method
1. **AI creates interactive HTML page** with embedded JavaScript
2. **Click download button** in the AI's response
3. **Get .CAM file** with proper Windows line endings automatically
4. **Ready for MX2e** - no further conversion needed
5. **WARNING** - DO NOT MAKE A STUPID BLOATED HTML PAGE.  I JUST NEED A BUTTON TO CLICK, NOT YOUR DECORATIVE GARBAGE>

### Verification
After download, the file should show:
- `CRLF: [number of lines]` when analyzed
- `LF only: 0` 
- Escape sequences display `\r\n` throughout

### Why This Works
- **JavaScript `.join('\r\n')`** forces Windows format regardless of host OS
- **Browser blob API** preserves explicit line ending characters  
- **Direct .CAM download** bypasses text editor conversions
- **No external tools** required - works with any AI that can create HTML artifacts, doesn't requrie text file downloads

## File Naming
- Always name the final downloadable file 00001.CAM, except instead of 00001, it can be 00002 , 00022, 00435, etc.  Any number that fits in 5 digits.  Pick a random one.

### Final Steps
- **Manual Check**: Cut air first

## Tool Compensation for Outside Cutting ONLY WHEN USING AI TO GENERATE TOOLPATHS FROM SCRATCH.  THE BELOW DOES NOT APPLY TO TRANSLATING FUSION TOOLPATHS!!!!!

### Manual Offset Calculation Required
- The MX2e requires **manual tool compensation** in the .CAM file coordinates
- No G41/G42 cutter compensation codes - all offsets must be calculated beforehand
- For outside cutting (leaving an island), move tool center **away from desired edge** by tool radius

### Outside Cut Compensation Formula
- **Tool radius** = Tool diameter ÷ 2
- **Left edges**: X_toolpath = X_part - tool_radius
- **Right edges**: X_toolpath = X_part + tool_radius
- **Bottom edges**: Y_toolpath = Y_part - tool_radius
- **Top edges**: Y_toolpath = Y_part + tool_radius

### Arc Compensation  
When compensating arcs for tool radius:
- **Outside cutting**: Arc radius increases by tool radius
- **Formula**: I/J_toolpath = I/J_part + tool_radius (for outside arcs)
- **Example**: Part arc radius 0.4375" + tool radius 0.3125" = toolpath arc radius 0.75"
- Must recalculate **both** the arc endpoints AND the I/J center offsets

## Climb vs Conventional Milling Direction

### Arc Direction for Climb Milling
- **CW spindle + outside cutting + climb milling = G02 arcs**
- **CW spindle + inside cutting + climb milling = G03 arcs**
- Always verify arc direction matches your intended climb/conventional choice

## Complex Contour Programming Checklist

Before finalizing a complex contour .CAM file:

1. ✅ **Geometry verification**: Plot the toolpath to verify part dimensions
2. ✅ **Arc calculations**: Verify all I/J values match compensated arc centers
3. ✅ **Direction check**: Confirm G02/G03 direction for desired climb/conventional
4. ✅ **0.001" shifts**: Apply where consecutive blocks might have identical coordinates
5. ✅ **Format compliance**: All MX2e format rules from main guide followed

## Machining Parameters

**Setup:** Bridgeport @ **3,000 rpm**, **1/2"**, 2-flute upcut, hardwood.

### Roughing (recommended)
• **Speed (S):** 3,000 rpm  
• **DOC (axial):** 0.25" per pass (→ 4 passes for 1")  
• **Stepover (radial):** 0.30" (~60% D)  
• **Feed (F):** **4.5 ft/min (54 ipm)** ≈0.009"/tooth  

### Finish wall
• **Speed:** 3,000 rpm  
• **DOC:** full 1"  
• **WOC:** leave 0.02–0.03"  
• **Feed:** **5.0 ft/min (60 ipm)** (light cut, climb)  

### Species tweak
• **Oak:** drop feeds ~15%  
• **Cherry:** roughing up to **5.0 ft/min (60 ipm)** if chips clear cleanly  

Keep chipload ≥ **0.008"/tooth**; slowing feed burns. Vacuum; climb on finish

## Future Improvements

**Someday:** A single solution will hopefully come out that does not involve such a convoluted workflow to program the MX2e mill

## Summary of Lessons Learned

1. Always use **O0000**
2. Always include trailing periods (`X0.` not `X0`)
3. Put feeds (`Fxx.`) on a **motion block with a nudge**, not on their own line
4. Use `.001"` nudges to prevent point errors
5. CRLF endings are mandatory
6. Do not invent formats — stick exactly to what's shown in working files
7. **Manual tool compensation required** — calculate all offsets before programming
8. **Arc compensation is critical** — both endpoints AND I/J centers must be recalculated
9. **Verify arc direction** for intended climb/conventional milling
10. **Cut Air First** — complex contours need verification
11. The Mill is a 2D device. Ignore helical moves and other such 3D requests
12. Tool changes are manual. Ignore tools.
13. Coolant is manual. Ignore coolant.
14. CRITICAL FUCK UP TO BE AVOIDED.  ALL FEED RATES MUST BE THE SAME IN FINAL RESULT YOU DELIVER TO ME.  Check this at the very end- rescan the file and make sure no other feed rates are present.  Just the one i ask for, generally 54.  Ask if you have questions about this.

---

This document is derived strictly from verified MX2e programs that run successfully.