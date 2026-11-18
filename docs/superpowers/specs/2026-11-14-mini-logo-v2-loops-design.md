# mini-logo V2 — loops

Date: 2026-11-14
Status: approved
Supersedes: the "The language", "Architecture" and "Controls" sections of
`2026-11-14-mini-logo-design.md`. Everything else in V1 stands.

## Intent

Add `repeat` so a square costs one loop instead of eight lines, and keep
stepping legible while it runs — you should be able to watch the body
re-execute and always know which iteration you are on.

Multi-line loop bodies are the preferred form, because this is a tool for
reading a program one line at a time.

## Scope

### In

`repeat N [ ... ]`, nested to any depth, written across lines or on one
line. A **Step Out** control that finishes the enclosing loop. Iteration
counters in the status bar.

### Out

- **Step Over** (run a whole loop as one step from its header). Raised
  during design, deliberately not taken: it is nearly free once Step Out
  exists — the same depth-pumping loop — so it costs nothing to add later
  and nothing to leave out now.
- `repcount`, `forever`, `while`, `until`, `for`.
- Procedures (`to … end`), variables, arithmetic. These bring a call stack
  and argument binding; they get their own spec.

`forever` in particular stays out until the step budget below is proven,
since it cannot be allowed near `fast` without one.

## Why this is not a small change

Multi-line bodies do not merely break `pc == line index`. They break
**line as the unit of execution**: a `[` body spans lines, so the source
can no longer be parsed one line at a time, and a single line can hold
several instructions.

So V2 replaces V1's line-splitting `parse()` with a tokenizer and a
recursive-descent parser. Two consequences follow, both intended:

1. One-line `repeat 4 [fd 100 rt 90]` comes free. Forbidding it would cost
   extra code, and it is the canonical LOGO form.
2. **V1's "too many words" error disappears.** `fd 100 rt 90` on one line
   becomes legal everywhere, not only inside brackets.

A line may therefore be highlighted several times in a row — once per
instruction on it, once per iteration of a loop containing it. The status
bar counters are what keep that readable, which is why they are in scope
and not a nicety.

## Syntax

```logo
repeat 4 [
  fd 100
  rt 90
]
```

and equivalently `repeat 4 [fd 100 rt 90]`. Brackets are not loop syntax:
in LOGO `[...]` is a list literal and `repeat` takes a number and a list.
V2 does not implement lists as values — only `repeat` accepts a bracketed
block — but the syntax is chosen to stay compatible with that reading.

Comments (`;` to end of line) and blank lines may appear anywhere,
including inside a block.

The count must be a non-negative integer. `repeat 0 [...]` is legal and
runs the body no times; `repeat 2.5` and `repeat -1` are parse errors
rather than silently rounded or skipped.

## Architecture

### Tokenizer

`tokenize(source) → [{ text, line }]`

Strips comments, pads `[` and `]` with spaces so they separate from
adjacent words, splits on whitespace, lowercases. Every token keeps the
line it came from; that is the only thing the highlight needs.

### Parser

`parseProgram(tokens) → { body, errors }`

Recursive descent. Two node shapes:

- `{ op, arg?, line }` — the six V1 primitives, unchanged.
- `{ op:'repeat', count, body, line }` — `body` is a node array.

A block is parsed by the same routine that parses the top level, differing
only in whether `]` closes it or is an error.

### Execution

A generator walks the tree. This is the whole execution model:

```js
function* walk(nodes, trail = []) {
  for (const n of nodes) {
    if (n.op === 'repeat')
      for (let i = 1; i <= n.count; i++) yield* walk(n.body, [...trail, { i, of: n.count }]);
    else
      yield { instr: n, trail };
  }
}
```

Nesting costs nothing — `yield*` recurses. `trail` carries one
`{ i, of }` per enclosing loop, so `trail.length` is the depth and the
entries are the iteration counters. `[...walk(ast)]` is the full execution
trace, which is what the tests assert against.

**`step(state, instr)` from V1 is unchanged.** The generator yields
instructions; the driver applies the same pure function and draws whatever
segment comes back. The split between "what runs next" and "what that does
to the turtle" is what makes both halves testable alone.

### Driver

The driver keeps the generator plus one instruction of lookahead:

- `iter` — the generator, created on entering run mode, discarded on Reset.
- `pending` — the next instruction, already pulled, not yet executed.
  `null` once the program is exhausted.

Lookahead is what makes the controls honest: `remaining` is
`pending !== null`, and the current loop depth is `pending.trail.length`.
Both are known before the user presses anything.

A step executes `pending`, highlights its line, reports its trail, then
pulls the next one.

**What V1 loses:** the flat `program` array, and `remaining()` scanning it.
The read-only listing, `highlight(line)`, both canvases, `drawSegment`,
`drawTurtle` and `step` are untouched.

**What a generator gives up** is rewinding, so there is no cheap Step Back.
Programs are tiny and execution is deterministic, so if Step Back is ever
wanted it is "replay from zero, stop one instruction earlier" rather than
anything structural.

## Editing

Added after the loops work landed, because multi-line blocks are what make
an unmatched bracket easy to write.

A `<textarea>` cannot style a range of its own text, so bracket matching
uses an overlay: a `#mirror` div renders the same text *behind* a textarea
whose text is transparent and whose caret is not.

- `#src` and `#mirror` share **one** CSS rule for font, padding and
  `white-space: pre`. If they ever disagree on a metric the overlay drifts,
  so they must not be styled apart.
- `wrap="off"` removes line wrapping, the overlay's usual failure mode.
  LOGO lines are short, so nothing is lost.
- The textarea's scrollbars are hidden and `#mirror` is scrolled
  programmatically to follow it.
- `#src::selection` is translucent. An opaque selection would hide the
  mirror text underneath it.

`matchPair(text, caret)` returns the pair touching the caret as
`[open, close]`, a single index when the bracket has no partner, or `null`
when the caret is not on a bracket. It runs against `maskComments(text)`,
which blanks comments while preserving length so indices still line up —
otherwise a `[` inside a `;` comment matches and the real pair does not.

An unmatched bracket is marked in the error colour rather than left plain:
it is the case most worth seeing, and it costs two lines.

### Tidy

A **Tidy** button re-indents the whole program — two spaces per bracket
depth, a line beginning with `]` dedenting itself, blank lines left blank.

It is line-based rather than token-based, because the tokenizer discards
comments and Tidy must keep them. It does not validate: unbalanced brackets
clamp the depth at zero instead of erroring, since Run already reports them
properly.

Tidy replaces the text through `document.execCommand('insertText')`. That
API is deprecated, but assigning to `textarea.value` destroys the native
undo stack, and being able to undo a Tidy matters more than avoiding a
deprecated call.

## Step Out

Finishes the enclosing loop — all its remaining iterations — and stops at
the first instruction outside it. It does not finish just the current
iteration.

```
const depth = pending.trail.length;
while (pending && pending.trail.length >= depth) { execute(pending); advance(); }
```

From a nested loop this lands in the enclosing loop, not at the top level,
which is the useful behaviour.

Step Out runs synchronously regardless of the speed setting. The drawing
still appears; only the pacing is skipped.

## Status bar

- Outside any loop: `line 4/12`
- Inside one: `line 4/12 · repeat 2/4`
- Nested: `line 5/12 · repeat 2/4 › 3/4`
- Exhausted: `done`

The counters come from the trail of the instruction just executed, so the
status describes where you are, consistent with the highlight showing the
line you just ran.

Note that the status and the Step Out button read different things on
purpose: the status reports the instruction just **executed**, while Step
Out is enabled from the depth of the one **pending**. On the last
instruction of a loop's last iteration the status still reads
`repeat 4/4` while Step Out is already greyed — correctly, because there
is nothing left in the loop to finish.

## Step budget

`repeat 1000000 [fd 1 rt 1]` is a plausible thing for a beginner to type,
and at `fast` it would hang the page. The runner stops after
`MAX_STEPS = 100000` instructions in a single synchronous burst (`fast`,
or Step Out) and reports `stopped after 100000 steps` in the error style.

Hitting the budget stops the run where it is and leaves the program
Paused, not Finished: `pending` still points at the next instruction, so
Step and Run continue from there.

This is a ceiling, not a limit anyone should hit; the constant is a knob.
It applies only to synchronous bursts — a timed run is interruptible by
Stop and needs no budget.

## Controls

A fourth button, **Step Out**, between Step and Run.

`buttonState({ editing, running, remaining, depth })`, with
`finished = !editing && !remaining`:

|              | Editing | Paused         | Playing        | Finished |
|--------------|---------|----------------|----------------|----------|
| **Step**     | on      | on             | off            | off      |
| **Step Out** | off     | on if depth>0  | off            | off      |
| **Run**      | on      | on             | on, as *Stop*  | off      |
| **Reset**    | off     | on             | on             | on       |
| **Tidy**     | on      | off            | off            | off      |

- `step: !running && !finished`
- `stepOut: !editing && !running && depth > 0`
- `run: !finished`
- `reset: !editing`
- `tidy: editing`

`depth` is 0 while Editing and when Finished, so those columns need no
special case. Step Out stays off in Editing deliberately: you must take at
least one Step to be inside a loop, and that keeps the control from having
to compile the source itself.

### Reset

Reset stops the timer, discards the generator, returns the turtle to the
centre at heading 0 with the pen down, and goes back to edit mode.

It does **not** clear the drawing. What the last run produced stays on
screen at 35% opacity, with the turtle's final position stamped onto the
path layer just before the fade, so both the trace and where the program
got to remain visible. The next Step or Run clears it.

Dimming is a class on the path canvas, not the canvas composited onto
itself at reduced alpha. A class cannot dim twice, so pressing Reset
repeatedly leaves the ghost at exactly one depth, with no flag to get
wrong. The live turtle lives on the other canvas and so stays at full
strength at home.

The ghost is cleared inside `enterRun`, after the parse succeeds. A program
that fails to parse has not started, and must not wipe what you were
looking at.

## Errors

Existing V1 messages stand, minus "too many words", which is no longer an
error. New ones:

| condition | message | reported at |
|---|---|---|
| `repeat` with no count | `repeat needs a number` | the `repeat` |
| `repeat 4 fd 100` | `repeat needs [ after the count` | the `repeat` |
| `repeat 2.5` / `repeat -1` | `repeat needs a whole number of times` | the `repeat` |
| block never closed | `unclosed [` | the line the `[` **opened** on |
| `]` with nothing open | `unexpected ]` | the `]` |

Reporting `unclosed [` at the opening bracket rather than at end-of-file is
the point of keeping a line number on every token.

Error handling is otherwise unchanged from V1: the first error goes to the
status bar, the editor stays open, and the offending line is selected.

### Recovery

One mistake produces one message. A token that fails takes its likely
argument or block with it, rather than leaving it to be reported a second
time on its own:

- an unknown command swallows a following number,
- a `repeat` that fails for any reason still consumes its `[ ... ]`,
- a stray `[` consumes the block it opens.

Found while implementing. Without it `fwd 10` reports both
`unknown command "fwd"` and `unknown command "10"`, and `repeat 2.5 [fd 10]`
reports the bad count and then the unexpected bracket. The tests assert
exactly one error for each of these.

## Sample program

The default program becomes the loop form, since it is now the point:

```logo
; draw a square
repeat 4 [
  fd 100
  rt 90
]

; hop away without drawing
pu
rt 45
fd 60
pd
fd 40
```

## Testing

V1's assertions stay (minus "too many words"). Added, all against the pure
tokenizer, parser and `walk`:

- `[fd` tokenizes as `[` then `fd`.
- `repeat 4 [fd 100 rt 90]` and its multi-line form produce the same AST
  once line numbers are stripped — they necessarily differ on those.
- A `repeat` square ends in the same state as V1's unrolled square.
- `repeat 2 [repeat 2 [fd 10]]` yields four instructions.
- The third instruction of `repeat 4 [fd 1 rt 1]` carries
  `trail = [{ i:2, of:4 }]`.
- Depth in a nested trace rises to 2 and returns to 0.
- Step Out from inside a loop, computed over the trace, lands on the first
  instruction whose depth is lower.
- `repeat 2 [fd 10` reports `unclosed [` on line 1.
- A stray `]`, and `repeat` without a count, are each one error.
- `repeat 0 [fd 10]` parses and yields no instructions.
- `repeat 2.5 [fd 10]` is an error.
- `buttonState` returns the table above, including a Paused row at depth 0
  and one at depth 1.
- `matchPair` finds a partner across lines and picks the right one when
  nested.
- A bracket inside a comment is ignored, and the same text unmasked fails
  to match — so the test would catch the mask being dropped.
- An unmatched bracket returns one index; a caret off any bracket returns
  `null`.
- `tidy` re-indents a mangled program, is idempotent, and keeps comments.
