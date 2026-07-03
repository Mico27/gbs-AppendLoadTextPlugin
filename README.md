# gbs-AppendLoadTextPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that extends the standard text system with the ability to **build up a text string across multiple load steps** before displaying it, and provides more granular control over text rendering options. It separates the "load text into buffer" step from the "display text" step, adds an append mode so consecutive loads accumulate instead of overwrite, lets you switch the render target between the background and overlay layers at any time, and exposes a flexible overlay/text wait event.

An `engineAlt` variant is included for compatibility when the plugin is used alongside the [ScreenScrollPlugin](https://github.com/Mico27/gbs-ScreenScrollPlugin).

All events are in the **Dialogue** group of the script editor.

<img width="403" height="1293" alt="image" src="https://github.com/user-attachments/assets/9e8093cd-688a-4286-9d3c-a6ccec356991" />

<img width="413" height="723" alt="image" src="https://github.com/user-attachments/assets/cc946490-4187-4f11-b610-ff893ca23afd" />

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [How to Use](#how-to-use)
4. [Technicalities and Restrictions](#technicalities-and-restrictions)
5. [Events Reference](#events-reference)
6. [Inner Workings](#inner-workings)
7. [Memory Footprint](#memory-footprint)

---

## Concepts

### The Text Buffer

GB Studio stores text to be displayed in a shared byte array called `ui_text_data`. Normally each **Display Text** event compiles the text inline and fills the buffer in one go. This plugin separates that into two distinct operations:

- **Load text** — writes encoded text bytes into `ui_text_data` and records how many bytes were written in `loaded_text_length`.
- **Display Loaded Text** — triggers the VWF renderer to draw whatever is currently in the buffer onto the tilemap.

Because loading and displaying are separate, you can call **Load text** several times before calling **Display Loaded Text**, optionally building up a longer string by enabling **Append** mode between loads.

### Append Mode

When `load_text_mode` is set to `1` (Append), each subsequent **Load text** call starts writing at the end of the previous load's data rather than at the beginning of the buffer. The `loaded_text_length` counter accumulates across calls, tracking the total bytes written so far. When `load_text_mode` is `0` (Default), every **Load text** call resets to the start of the buffer.

---

## Project Setup

No special scene configuration is required. Install the plugin into your GB Studio project's `plugins` folder. The five new events will appear automatically in the **Dialogue** group of the script editor.

If your project also uses the **ScreenScrollPlugin**, the `engineAlt/ScreenScrollPlugin/src/core/vm_ui.c` file takes priority and provides the same append/load functionality with the scroll-plugin's additional UI changes included.

---

## How to Use

### Basic Load and Display

1. Call **Load text** with the text you want to display.
2. Open the overlay if needed (standard **Show Overlay** / **Move Overlay** events).
3. Call **Display Loaded Text** to render it.
4. Use **Wait for overlay/text to finish displaying** to pause until text drawing is complete.

### Appending Multiple Text Segments

1. Call **Change load text mode** → **Append**.
2. Call **Load text** for the first segment (e.g. a static label).
3. Call **Load text** again for the second segment (e.g. a variable value using `%d`).
4. Call **Load text** once more for any trailing text if needed.
5. Call **Display Loaded Text** to render the combined result.
6. Call **Change load text mode** → **Default** to reset for the next text operation.

### Switching the Target Layer

Call **Change text layer** at any point before **Display Loaded Text** to direct rendering to either the background tilemap or the overlay (window) tilemap. This persists until changed again.

---

## Technicalities and Restrictions

### Buffer Size

The `ui_text_data` array has a fixed size shared with the standard GB Studio text system. Appending across too many **Load text** calls without displaying can overflow the buffer. Keep the total accumulated text within the normal dialogue line limits.

### load_text_mode Persists

`load_text_mode` is a global flag. If you enable Append mode and then branch (condition, jump, etc.) without calling **Change load text mode** → **Default**, subsequent **Load text** calls elsewhere in the script will still append. Always reset to Default mode after you are done appending.

### loaded_text_length Measures from Buffer Start

`loaded_text_length` is updated after every **Load text** call to reflect the total byte count from the beginning of `ui_text_data`, not from the start of the most recent load. This means it correctly accumulates across multiple append calls.

### Format Specifiers

Inside the text field of **Load text**, the following `%` format codes expand variable values at compile/load time:

| Code | Expansion |
|------|-----------|
| `%d` | Signed decimal integer from the next script argument variable. |
| `%D` followed by a digit `n` | Signed decimal integer, zero-padded to `n` digits wide. |
| `%c` | Single character whose ASCII code is taken from the variable value. |
| `%t` | Inline text speed control code (value 0–7 from the variable). |
| `%f` | Inline font-switch code (font index from the variable). |
| `%%` | A literal `%` character. |

### preserve_pos in Display Loaded Text

When **Use previous text position** is checked, the text cursor does not reset to `text_render_base_addr` at the start of rendering. Instead it continues from wherever the previous display call left off. This allows multiple **Display Loaded Text** calls to fill a text box progressively across frames.

### start_tile in Display Loaded Text

The **Specify start tile** option lets you control which VRAM tile slot the VWF renderer starts writing glyph bitmaps into. On CGB this accounts for the split VRAM bank layout (bank 0 tiles 0x00–0x7F, bank 1 tiles 0x80–0xFF). Leave it unchecked to continue from the last written tile.

### Layer Change Persists

The text layer set by **Change text layer** (background or overlay) persists across events and scene scripts until explicitly changed again. It is not reset by **Load text** or **Display Loaded Text**.

---

## Events Reference

All events are in the **Dialogue** group.

---

### Load text

**`EVENT_UI_LOAD_TEXT`**

Encodes the text string (including any `%` format substitutions) into the `ui_text_data` buffer. In **Default** mode the buffer is overwritten from the start. In **Append** mode the new text is appended after the previously loaded content.

Does not render anything — it only populates the buffer. Pair with **Display Loaded Text** to actually draw the text.

| Field | Description |
|-------|-------------|
| Text | The text to load. Supports standard GB Studio text formatting, inline speed codes, colour codes, and `%d`/`%c`/`%t`/`%f` format specifiers for variable substitution. |

---

### Display Loaded Text

**`EVENT_UI_DISPLAY_LOADED_TEXT`**

Triggers the VWF renderer to draw the current contents of `ui_text_data` onto the active text layer (background or overlay). Does not load or change the buffer — only renders what was last loaded.

| Field | Description |
|-------|-------------|
| Use previous text position | When checked, resumes rendering from the cursor position left by the previous display call instead of resetting to the base address. |
| Specify start tile | When checked, enables the **Starting tile** field. |
| Starting tile | The VRAM tile slot index at which the VWF renderer begins writing glyph bitmaps. On CGB, values above `0x100 − TEXT_BUFFER_START` spill into VRAM bank 1. |

---

### Change load text mode

**`EVENT_UI_CHANGE_LOAD_TEXT_MODE`**

Sets the global `load_text_mode` flag that controls how subsequent **Load text** calls behave.

| Field | Description |
|-------|-------------|
| Load text mode | **Default** (0) — each **Load text** overwrites the buffer from position 0. **Append** (1) — each **Load text** adds to the end of the existing buffer content. |

---

### Change text layer

**`EVENT_UI_CHANGE_TEXT_LAYER`**

Redirects text rendering to the chosen VRAM tilemap layer. The change persists until this event is called again.

| Field | Description |
|-------|-------------|
| Location | **Background** — text is written into the background tilemap (`GetBkgAddr()`). **Overlay** — text is written into the window/overlay tilemap (`GetWinAddr()`). |

---

### Wait for overlay/text to finish displaying

**`EVENT_UI_OVERLAY_WAIT`**

A flexible wait event that suspends script execution (or blocks all scripts if modal) until all selected conditions are satisfied. Each condition is independent and all checked conditions must be true simultaneously before the wait resolves.

| Field | Description |
|-------|-------------|
| Modal | When checked, the wait blocks all other script threads (equivalent to `ui_run_modal`). When unchecked the script yields each frame until conditions are met. |
| Wait for overlay | Wait until the overlay window finishes its slide animation (`win_pos == win_dest_pos`). |
| Wait for text | Wait until the VWF renderer has drawn all queued characters (`text_drawn == TRUE`). |
| Wait for button A | Wait until the A button is pressed. |
| Wait for button B | Wait until the B button is pressed. |
| Wait for any button | Wait until any button is pressed. |

---

## Inner Workings

### The `load_text` Function

The core of the plugin is the `load_text` function in `vm_ui.c`. It is called by both `vm_load_text` (inline text in the script bytecode) and `vm_load_text_ex` (indirect pointer from a variable):

```
d = prev_d = ui_text_data;
if (load_text_mode) {
    d += loaded_text_length;   // start writing after existing content
}
```

The function then walks the source string byte-by-byte:
- Regular characters are copied directly.
- `%` format codes consume the next argument from the script stack and expand to digits, a single byte, or a control-code sequence.
- After the loop: `loaded_text_length = d - prev_d` — since `prev_d` is always `ui_text_data` (the buffer origin), this stores the total filled length regardless of where writing began.

This design means `loaded_text_length` always correctly represents "how many bytes of text are ready to display" even after multiple appended loads.

### Format Specifier Expansion

The `%` dispatch switch handles each code:
- **`%d`**: calls `itoa_format(arg, d, 0)` — converts the `INT16` argument to a decimal ASCII string with no padding. Returns the number of bytes written.
- **`%D[n]`**: calls `itoa_format(arg, d, n)` — same but zero-pads to `n` digits, read from the next byte of the source string.
- **`%c`**: writes `(unsigned char)(arg)` — a single character whose ASCII value equals the variable.
- **`%t`**: writes the two-byte speed control code `0x01, arg + 0x02` — matches the GB Studio inline speed format.
- **`%f`**: writes the two-byte font-switch control code `0x02, arg + 0x01` — matches the GB Studio inline font format.

For right-to-left fonts (`vwf_direction != UI_PRINT_LEFTTORIGHT`), `itoa_format` reverses the digit string before returning.

### Switching the Text Layer

`vm_switch_text_layer` sets `text_render_base_addr` to either `GetBkgAddr()` (background VRAM map base) or `GetWinAddr()` (window VRAM map base). The VWF renderer uses `text_render_base_addr` as the origin for all tilemap writes, so changing it before **Display Loaded Text** redirects the entire render pass to the chosen layer without any other changes.

### Display and VRAM Tile Allocation

`vm_display_text` configures the VWF start tile. On CGB it handles the split-bank layout: tile indices below `0x100 − TEXT_BUFFER_START` go into VRAM bank 0; indices above that threshold are remapped into bank 1 by calling `ui_set_start_tile` with `TEXT_BUFFER_START_BANK1`. The `TEXT_OPT_PRESERVE_POS` option flag skips resetting `text_render_base_addr` to the base, allowing the cursor to continue where the previous render left off.

### Overlay Wait

`vm_overlay_wait` is a waitable VM function. In non-modal mode it checks each enabled condition flag against the current state. If any condition is unmet it sets `THIS->waitable = 1` and rewinds `THIS->PC` by the instruction size so the VM re-executes the same wait instruction on the next frame. Once all conditions pass simultaneously the function returns without rewinding, and the script advances. In modal mode it delegates to `ui_run_modal`, which spins in a blocking loop until the flags are satisfied.

---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine (per-file SDCC compile with GB Studio's build flags, default engine settings). Values are the plugin's *delta* versus the stock engine; DMG build, with CGB noted where it differs. ROM cost lands in banked ROM (GB Studio's autobanker spreads it across switchable banks); using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks.

| | Cost |
|---|---|
| WRAM | +3 bytes |
| ROM | +31 bytes |

- **WRAM:** 3 bytes (`load_text_mode`, `loaded_text_length` bookkeeping).
- **Engine WRAM headroom:** the stock GB Studio 4.3.0 engine leaves about **854 bytes** of WRAM free (usable engine WRAM is 7,776 bytes at 0xC0A0–0xDF00; the stock engine uses 6,922 bytes). With this plugin installed roughly **851 bytes** remain. This figure does not depend on how many global variables your project defines: the script memory array has a fixed size of VM_HEAP_SIZE + (VM_MAX_CONTEXTS × VM_CONTEXT_STACK_SIZE) words — 768 + 16 × 64 = 1,792 words (3,584 bytes) with stock engine settings.
- **SRAM:** not used.
