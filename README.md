# gbs-AppendLoadTextPlugin

**Version 4.3.0. Requires GB Studio 4.3.0 or newer.**

Splits GB Studio's text handling into two steps, preparing the text and drawing it, so a script can
assemble a line out of several pieces before anything appears on screen.

That gets you things the stock **Display Text** event cannot do: a shop line that reads
"You bought 3 Potions for 900G" built from three separate pieces, a name the player entered dropped
into the middle of a sentence, text drawn onto the background instead of the dialogue box, and a
wait that ends on the overlay finishing, the text finishing, or a button press.

A compatibility variant ships for the
[ScreenScrollPlugin](https://github.com/Mico27/gbs-ScreenScrollPlugin).

All events are in the **Dialogue** group.

<img width="403" height="1293" alt="image" src="https://github.com/user-attachments/assets/9e8093cd-688a-4286-9d3c-a6ccec356991" />

<img width="413" height="723" alt="image" src="https://github.com/user-attachments/assets/cc946490-4187-4f11-b610-ff893ca23afd" />

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Events Reference](#events-reference)
5. [FAQ](#faq)
6. [Memory Footprint](#memory-footprint)
7. [Bank 0 (HOME) Usage](#bank-0-home-usage)
8. [Changelog](#changelog)

---

## Concepts

### Two steps instead of one

GB Studio keeps the text it is about to show in one shared area. The stock **Display Text** event
fills that area and draws it in a single move. This plugin separates them:

- **Load text** puts text into the area and records how long it is.
- **Display Loaded Text** draws whatever is in the area at that moment.

Because they are separate, you can call **Load text** several times and then draw the result once.

### Append mode

**Change load text mode** set to **Append** makes each **Load text** carry on from where the last
one stopped. Set back to **Default**, each **Load text** starts over from the beginning.

---

## Project Setup

Copy the plugin into your project's `plugins` folder. The five events appear in the **Dialogue**
group. There is nothing to configure.

If your project also uses the **ScreenScrollPlugin**, GB Studio picks the matching variant
automatically.

### Loading and displaying

1. Call **Load text** with the text you want.
2. Open the dialogue box if you need one, with the stock **Show Overlay** and **Move Overlay**
   events.
3. Call **Display Loaded Text**.
4. Call **Wait for overlay/text to finish displaying** to pause until the drawing has finished.

### Building a line out of pieces

1. Call **Change load text mode** and choose **Append**.
2. Call **Load text** for the first piece, for example `You bought `.
3. Call **Load text** for the next piece, for example `%d` to insert a variable.
4. Call **Load text** for the rest, for example ` Potions.`.
5. Call **Display Loaded Text** to draw the whole line.
6. Call **Change load text mode** and choose **Default** so the next text starts fresh.

### Drawing on the background instead of the dialogue box

Call **Change text layer** at any point before **Display Loaded Text**. The choice sticks until you
change it again.

---

## Size Limits and Restrictions

### The text area has a fixed size

It is the same area the stock text system uses. Appending too much before drawing overruns it. Keep
the total within normal dialogue length.

### The append setting sticks

It is a single global setting. Turning on Append and then branching elsewhere in your scripts
without setting it back means those scripts append too. Set it back to Default when you are done.

### Substitutions inside Load text

| Code | What it inserts |
|------|-----------------|
| `%d` | A whole number from the next variable, with a minus sign if negative. |
| `%D` then a digit | The same, padded with zeros to that many digits. |
| `%c` | One character, whose code comes from the variable. |
| `%t` | A text speed change, 0 to 7, from the variable. |
| `%f` | A font change, using the font number in the variable. |
| `%%` | A literal percent sign. |

### Use previous text position

With this ticked, **Display Loaded Text** carries on from where the last draw stopped instead of
starting at the top of the box. That is how you fill a text box a piece at a time.

### Specify start tile

Chooses which tile slot the text renderer starts writing letters into. On Game Boy Color this also
decides which tile bank they land in. Leave it unticked to carry on from the last one used.

### The layer choice sticks

The layer set by **Change text layer** stays until you change it again. Loading or displaying text
does not reset it.

---

## Events Reference

All events are in the **Dialogue** group.

### Load text

Puts text into the shared text area, expanding any `%` substitutions. In **Default** mode it starts
from the beginning. In **Append** mode it carries on from the last load.

Nothing appears on screen. Follow it with **Display Loaded Text**.

| Field | Description |
|-------|-------------|
| Text | The text to load. Supports the usual GB Studio formatting, speed and colour codes, and the `%d`, `%c`, `%t` and `%f` substitutions. |

### Display Loaded Text

Draws whatever is currently in the text area, onto the background or the overlay. It does not
change the text.

| Field | Description |
|-------|-------------|
| Use previous text position | Carry on from where the last draw stopped instead of starting at the top. |
| Specify start tile | Turns on the **Starting tile** field. |
| Starting tile | The tile slot the renderer starts writing letters into. On Game Boy Color, higher values move into the second tile bank. |

### Change load text mode

| Field | Description |
|-------|-------------|
| Load text mode | **Default** starts each **Load text** from the beginning. **Append** carries on from the last one. |

### Change text layer

Sends text to the chosen layer. The choice sticks until changed.

| Field | Description |
|-------|-------------|
| Location | **Background** draws into the scene's background. **Overlay** draws into the dialogue box layer. |

### Wait for overlay/text to finish displaying

Pauses until every condition you tick is satisfied at the same time.

| Field | Description |
|-------|-------------|
| Modal | Holds every script, not just this one. Unticked, only this script waits and the rest keep running. |
| Wait for overlay | Wait until the overlay has finished sliding. |
| Wait for text | Wait until every queued character has been drawn. |
| Wait for button A | Wait for A. |
| Wait for button B | Wait for B. |
| Wait for any button | Wait for any button. |

---

## FAQ

**How do I show a sentence with a variable in the middle?**
Turn on Append, then call **Load text** three times: the words before, `%d` for the number, and the
words after. Call **Display Loaded Text**, then set the mode back to Default.

**Can I do that with the stock Display Text event?**
A single **Display Text** already handles one `%d`. This plugin is for cases where the pieces come
from different places in your script, or where you want to decide what to include before showing
anything.

**How do I write text directly onto the scene background?**
Call **Change text layer** and choose **Background** before **Display Loaded Text**. Handy for a
title screen, a sign, or a heads-up display that is not in a dialogue box.

**My text kept growing across unrelated dialogue. Why?**
Append mode was left on. It is one global setting, so set it back to Default as soon as you have
drawn the assembled line.

**How do I reveal a text box a line at a time?**
Tick **Use previous text position** on the second and later **Display Loaded Text** calls. Each one
carries on where the last stopped.

**How do I wait until the text has finished drawing?**
Use **Wait for overlay/text to finish displaying** with **Wait for text** ticked. Add **Wait for
button A** to also require a keypress.

**What happens if I append too much?**
The text area has a fixed size shared with the stock text system, and going past it overruns it.
Keep the total to normal dialogue length.

**Does this replace the stock Display Text event?**
No. The stock events keep working. Use these when you need to assemble text before showing it.

**Does it work with the ScreenScroll plugin?**
Yes. A compatibility variant ships with it and GB Studio picks it automatically.

**Can I change font or speed partway through a line?**
Yes. Use `%f` for the font and `%t` for the speed, taking the value from a variable.

---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine at default engine settings, report of
2026-08-13. Figures are the difference against a stock project. Each event you use also compiles a
few bytes of script into your project, on top of the fixed cost below.

| Budget | Cost |
|---|---|
| Bank 0 (HOME) | +31 bytes |
| WRAM | +3 bytes |
| Banked ROM | 0 bytes |

- **Bank 0:** 31 bytes sit in the fixed bank, in the text event code. Everything else lives in a
  switchable bank. See [Bank 0 (HOME) Usage](#bank-0-home-usage).
- **WRAM:** 3 bytes to remember the append setting and how much text is loaded.
- **Banked ROM:** no change. The plugin's text event code compiles to the same banked size as the
  stock version, so its whole cost shows up in bank 0.
- **Engine WRAM headroom:** a stock GB Studio 4.3.0 project leaves about **854 bytes** of WRAM
  free (the engine has 7,776 bytes to work with and uses 6,922 of them). With this plugin
  installed roughly **851 bytes** remain. Adding more global variables to your project does not
  change that figure, because script memory is a fixed 3,584 byte block at stock engine settings.
- **SRAM:** not used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB fixed ROM bank shared by the GB Studio engine core, the
interrupt handlers and the GBDK runtime. Extra banked ROM is cheap to add,
bank 0 is not, so bank 0 is usually the first thing a project runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **+31** |
| Bank 0 free with this plugin installed | **1,420** of 16,384 (91% used) |

Everything else this plugin adds lives in banked ROM.

| Module | This plugin | Stock engine | Bank 0 cost |
|---|---|---|---|
| Text events | 709 | 678 | +31 |

A module that replaces a stock engine file costs only the *difference*, because
the stock version's bank 0 bytes were being spent anyway.

<details><summary>How this was measured</summary>

GB Studio 4.3.0-e1, default engine settings. Each module was compiled with the
toolchain and flags GB Studio itself uses, and the bank 0 size the compiler
recorded was read back. The stock column is the same compile of the engine file
the module replaces.

The "free" figure assumes a stock project with this plugin and nothing else.
Your own number will differ, because other plugins and any engine settings that
change what the core compiles move it too.

</details>
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version bumps, patch
regeneration, packaging fixes and documentation edits are omitted.

### 2026-06-28

- Added ContinuousScenePlugin compatibility.

### 2026-02-14

- Initial release.
