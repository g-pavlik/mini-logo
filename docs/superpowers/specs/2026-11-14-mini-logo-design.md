# mini-logo — design spec (V1)

Date: 2026-11-14
Status: approved

## Intent

A small program for learning to program in LOGO. Write a few turtle
commands on the left, watch the turtle draw on the right, and step through
the program one line at a time to see what each line did.

Success means: a person can type eight lines, press Step eight times, and
see exactly which line moved the turtle each time.

### Constraints the design must respect

- Runs on Linux and on Windows.
- Looks fairly modern.
- Works offline.

### Decisions taken during brainstorming

- **Shell: a single self-contained HTML file.** Tauri, Electron, a Go
  binary, an AppImage, a nix flake and a Windows `.exe` were all considered
  and dropped. The app is pure client-side, so the file *is* the program —
  double-click it and it runs on both operating systems, with no build step
  and no toolchain. This is the cheapest thing that satisfies "Linux and
  Windows".
- **Syntax: classic LOGO** (`fd`/`bk`/`rt`/`lt`/`pu`/`pd` with long-form
  aliases), so any LOGO tutorial reads correctly against it.
- **Speed control: delay between lines**, not a gliding turtle. Each
  instruction executes atomically and completely; speed only sets the pause
  before the next one.

## Scope

### In

Six commands, a two-pane window, four controls (Reset, Step, Run/Stop,
speed), current-line highlighting while stepping, and a parse-error report.

### Out (V1)

- `repeat N [ ... ]` and any other loop. **Deferred to V2 by agreement** —
  expected to be the first thing wanted, because drawing a square otherwise
  costs eight lines. It is out of V1 because a block structure means the
  instruction pointer is no longer a flat line index, which complicates
  both the interpreter and the highlight.
- Procedures (`to square ... end`), variables, arithmetic, colours, turtle
  wrap, save/load, undo.
- Packaging as AppImage, nix flake or `.exe`.

## Architecture

One file: `mini-logo.html`, with inline `<style>` and `<script>`. No CDN
references, no npm, no build step. It must run from a `file://` URL.

```
┌────────────────────────────────────────────┐
│ [Reset] [Step] [Run ▸] [slow│med│fast ▾]   │
│ line 3/8                                   │  ← status
├─────────────────────┬──────────────────────┤
│  textarea (edit)    │                      │
│    ↕ swaps ↕        │   canvas × 2         │
│  line list (run)    │                      │
└─────────────────────┴──────────────────────┘
```

### Components

**Parser** — `parse(source) → { program, errors }`. Pure, DOM-free.

**Interpreter** — `step(state, instr) → { state, segment }`. Pure,
DOM-free. `state` is `{ x, y, heading, pen }`; `segment` is `{x1,y1,x2,y2}`
or `null`. The interpreter never touches the canvas; the caller draws what
it returns. This separation exists so the logic can be tested without a
DOM, and is the only structural decision in the file.

**Renderer** — two `<canvas>` elements stacked by absolute positioning.
The lower one holds the drawn path and is cleared only on Reset. The upper
one holds the turtle triangle and is cleared before each redraw. There is
no redraw loop and no stored list of segments to replay.

**Controller** — owns `pc`, the timer handle, and the edit/run mode flag;
wires the four controls to the above.

## The language

One instruction per line: `cmd [number]`. Case-insensitive. A `;` begins a
comment that runs to end of line.

| Canonical | Aliases          | Arity |
|-----------|------------------|-------|
| `fd`      | `forward`        | 1     |
| `bk`      | `back`, `backward` | 1   |
| `rt`      | `right`          | 1     |
| `lt`      | `left`           | 1     |
| `pu`      | `penup`          | 0     |
| `pd`      | `pendown`        | 0     |

Blank lines and comment-only lines are retained in the program array as
no-op instructions, so a program index is always equal to its source line
number. The current-line highlight depends on this and must not be
optimised away.

### Semantics

Heading 0 points north (up the screen); `rt` increases the heading. Canvas
y grows downward, so a `fd n` moves to
`x + sin(heading)·n`, `y − cos(heading)·n`.

- `fd n` — move; if the pen is down, emit the segment travelled.
- `bk n` — identical to `fd −n`.
- `rt n` / `lt n` — `heading ± n`.
- `pu` / `pd` — set pen false / true. Emit no segment.

The turtle starts at the centre of the canvas, heading 0, pen down.

## Modes

**Edit mode** — the textarea is visible and editable. The initial mode.

**Run mode** — the textarea is hidden and replaced by a read-only list of
`<div class="line">`, one per source line, with `.cur` on the line at `pc`.
The source cannot be edited; Reset returns to edit mode.

Entering run mode compiles the source. This happens on the first Step or
Run after an edit.

## Controls

- **Reset** — stop the timer, clear both canvases, reset the turtle to
  centre / heading 0 / pen down, set `pc = 0`, return to edit mode.
- **Step** — enter run mode if not already in it, execute the instruction
  at `pc`, advance `pc`, move the highlight. At the end of the program the
  status reads `done` and further presses do nothing.
- **Run / Stop** — enter run mode, then advance on a `setTimeout` at the
  current delay. The button reads **Stop** while running and halts the
  timer, leaving `pc` where it is so Step can continue from there.
  Pressing Run once the program is `done` does nothing; Reset first.
- **Speed** — `slow` 600 ms, `medium` 200 ms, `fast` 0 ms. The delay is
  read at each tick, so changing speed mid-run takes effect immediately. At
  `fast` the runner executes the remaining program in a single synchronous
  loop rather than scheduling a timer.

## Errors

A line whose command is unknown, or whose argument is missing or not a
number, is a parse error. On entering run mode:

- every bad line is given an `.err` class,
- the status bar shows the first one, as `line 4: unknown command "fwd"`,
- neither Step nor Run starts; the program stays in edit mode until
  the source is corrected.

Parse errors are reported for the whole program at once, not one per run
attempt.

## Testing

Loading `mini-logo.html?test` runs the assertions and writes pass/fail into
the page instead of starting the app. No framework, no DOM dependency —
the assertions call `parse` and `step` directly.

Cases:

- `fwd 10` is rejected with an unknown-command error.
- `fd 10` with no argument is rejected.
- `fd 100 / rt 90 / fd 100` leaves the turtle at the expected x, y and
  heading.
- `bk 50` produces the same state as `fd -50`.
- An instruction executed with the pen up returns a `null` segment.
- A blank line and a comment line each occupy an index, so a three-line
  source with a blank middle line yields a three-entry program.
