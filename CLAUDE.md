# Project Overview

RMG-K (Rosalie's Mupen GUI - Kaillera) is a Nintendo 64 emulator frontend built on mupen64plus with Kaillera netplay support. Written in C++20, it uses Qt6 for the UI, SDL3 for input/audio, and integrates with the mupen64plus plugin architecture.

# Architecture Overview

## Component Structure

The project uses a modular architecture with clear separation:

```
RMG (Qt6 GUI) → RMG-Core (Shared Library) → mupen64plus-core + Plugins
                      ↓
        ┌─────────────┼──────────────┬──────────────┐
        ↓             ↓              ↓              ↓
    RMG-Input    RMG-Audio    RMG-Input-GCA    mupen64plus
   (Plugin)     (Plugin)      (Plugin)         plugins
```

## Key Components

**RMG** (`Source/RMG/`):
- Qt6-based user interface and main application
- MainWindow: Primary window, ROM browser, settings
- EmulationThread: QThread running mupen64plus in background
- KailleraSessionManager: Qt wrapper for Kaillera netplay (Windows only)
- OnScreenDisplay: ImGui-based OSD rendering
- Dialogs: Settings, cheats, netplay, ROM info, updates

**RMG-Core** (`Source/RMG-Core/`):
- Shared library providing C++20 abstraction over mupen64plus
- `m64p/`: API wrappers (CoreApi, PluginApi, ConfigApi)
- `Emulation.cpp`: Lifecycle management (start/stop/pause/reset)
- `Plugins.cpp`: Plugin discovery, loading, and attachment
- `Kaillera.cpp`: Windows-only Kaillera DLL wrapper for netplay
- `VidExt.cpp`: Video Extension override for Qt widget rendering
- ROM/SaveState/Cheats/Settings management

**RMG-Input** (`Source/RMG-Input/`):
- mupen64plus input plugin using SDL3
- Per-axis scaling with configurable range (default 66% for N-Rage compatibility)
- Per-axis deadzone (not circular) for USBtoN64v2 adapter compatibility
- Hotkey system and controller pak support

**RMG-Audio** (`Source/RMG-Audio/`):
- mupen64plus audio plugin with SDL3 backend
- Multiple resampling algorithms (trivial, speex, libsamplerate)

**RMG-Input-GCA** (`Source/RMG-Input-GCA/`):
- Specialized input plugin for GameCube controller adapters

## Critical Architecture Patterns

## 1. mupen64plus Plugin Architecture

RMG-K extends mupen64plus with custom plugins:
- **RSP**: Reality Signal Processor emulation (HLE/CXD4/Parallel)
- **GFX**: Graphics rendering (GLideN64/Parallel/Angrylion)
- **Audio**: Audio output (RMG-Audio)
- **Input**: Controller input (RMG-Input/RMG-Input-GCA)

Plugins are dynamically loaded libraries with standardized APIs. RMG adds custom extensions:
- `PluginConfig()`: GUI configuration dialog
- `PluginConfigWithRomConfig()`: ROM-specific settings

## 2. Video Extension (VidExt) Override

Instead of SDL windowing, RMG provides custom video extension:
1. RMG-Core overrides VidExt functions via `CoreOverrideVidExt()`
2. Graphics plugins call these instead of SDL
3. RMG creates OpenGL/Vulkan contexts in Qt widgets (OGLWidget/VKWidget)
4. Enables proper Qt integration and fullscreen handling

## 3. Kaillera Netplay Synchronization (Windows Only)

The most complex subsystem. Architecture:
```
Kaillera Server ↔ kailleraclient.dll ↔ RMG-Core (Kaillera.cpp) ↔ KailleraSessionManager ↔ MainWindow
```

**Synchronization Flow (every frame)**:
1. mupen64plus-core polls controllers via PIF (Peripheral Interface)
2. `KailleraPifSyncCallback()` in `Emulation.cpp` intercepts first JCMD_CONTROLLER_READ
3. Reads local controller state from PIF RAM
4. Calls `CoreModifyKailleraPlayValues()` with 4-byte input buffer
5. Kaillera DLL sends local input to server, receives all players' synchronized inputs
6. Synchronized inputs cached and written back to PIF RAM
7. Subsequent PIF polls in same frame use cached data
8. Frame advances, sync flag reset, repeat

**Critical Details**:
- Exactly one sync per frame via `s_SyncedThisFrame` flag
- Only syncs on JCMD_CONTROLLER_READ (0x01), not JCMD_STATUS (0x00)
- Frame delay override: User can set 0-9 frames (0 = server decides)
- Connection persistence: Keeps alive after emulation ends for quick restarts
- Thread safety: Kaillera callbacks run on Kaillera thread, marshaled via Qt signals

## 4. Threading Model

- **Main Thread**: Qt event loop, UI interactions
- **EmulationThread**: mupen64plus-core execution
- **SDLThread** (RMG-Input): SDL event polling for input
- **HotkeysThread** (RMG-Input): Hotkey state monitoring
- **Kaillera Thread**: Managed by kailleraclient.dll (Windows only)

Communication via Qt signals/slots for thread safety. mupen64plus callbacks queued and polled by timer in MainWindow.

# Development Workflow

## Directory Structure

```
Source/
├── RMG/              # Main Qt application
├── RMG-Core/         # Core library (shared across plugins)
├── RMG-Audio/        # Audio plugin
├── RMG-Input/        # Input plugin
├── RMG-Input-GCA/    # GameCube adapter plugin
├── 3rdParty/         # External dependencies (git submodules)
├── Installer/        # Windows installer
└── Script/           # Build scripts (Build.sh, BundleDependencies.sh)

Data/                 # Runtime data
├── Cheats/          # Cheat files (.cht)
├── InputProfileDB.json
├── font.ttf
└── kailleraclient.dll  # Windows Kaillera DLL

Bin/Release/          # Build output (portable mode)
```

## Code Patterns

**Always read files before editing**: This codebase uses specific patterns (Qt signals/slots, mupen64plus callbacks, threading). Understand existing code before modifications.

**Plugin Development**: When creating/modifying plugins:
1. Follow mupen64plus plugin API (see `Source/3rdParty/mupen64plus-core/src/api/m64p_plugin.h`)
2. Implement `PluginStartup`, `PluginShutdown`, `PluginGetVersion`
3. For input plugins: `InitiateControllers`, `ControllerCommand`, `ReadController`
4. For audio plugins: `AiDacrateChanged`, `AiLenChanged`, `ProcessAList`
5. Optionally implement `PluginConfig()` for GUI (RMG extension)

**Thread Safety**: When working with emulation callbacks or Kaillera:
- Never call Qt functions directly from mupen64plus callbacks
- Queue messages and process on UI thread via signals/slots
- Use `QMetaObject::invokeMethod()` for cross-thread calls
- Kaillera callbacks execute on Kaillera's thread - marshal via KailleraSessionManager

**Input Handling**: RMG-Input uses per-axis scaling (not circular deadzone):
- Range slider: 0-100% (default 66% matches N-Rage)
- Linear scale: 100% = 127 (protocol maximum)
- Each axis (X, Y) has independent deadzone
- Compatible with USBtoN64v2 adapters and N-Rage plugin

## Important Files

**Emulation Lifecycle**: `Source/RMG-Core/Emulation.cpp`
- Contains core emulation loop control and frame callbacks
- **KailleraPifSyncCallback()**: Critical netplay synchronization (lines ~400-500)
- Handles pause/resume, reset, speed control

**Kaillera Integration**: `Source/RMG-Core/Kaillera.cpp`
- Windows DLL wrapper with C++ API
- Callback bridges and thread marshaling
- Frame delay override logic

**Plugin Management**: `Source/RMG-Core/Plugins.cpp`
- Plugin discovery from filesystem
- Dynamic library loading/unloading
- Configuration handling

**Main Window**: `Source/RMG/UserInterface/MainWindow.cpp`
- UI coordination and event handling
- ROM browser management
- Emulation thread lifecycle

**Settings**: `Source/RMG-Core/Settings.cpp`
- Centralized configuration storage
- Per-ROM overrides
- Plugin settings

## Special Considerations

**Kaillera is Windows-Only**: Netplay code in `Kaillera.cpp` only compiles on Windows. Guarded by `#ifdef _WIN32` and CMake option `NETPLAY`.

**Portable vs System Install**:
- Portable: All files in `Bin/Release/`, plugins in subdirectories
- System: Standard Linux paths (`/usr/lib/RMG`, `/usr/share/RMG`)
- Controlled by `PORTABLE_INSTALL` CMake option

**Assembly Optimization**: mupen64plus-core includes x86/ARM assembly. Disable with `NO_ASM` if targeting unsupported architectures.

**License Considerations**: angrylion-rdp-plus uses non-GPL license. Only build if legally acceptable via `USE_ANGRYLION` option.

# Key Features Unique to RMG-K

1. **Kaillera Netplay**: Windows-only online multiplayer via Kaillera protocol
2. **Frame Delay Override**: User-configurable frame delay (0-9) for ping compensation
3. **Connection Persistence**: Kaillera connection stays alive after game ends
4. **N-Rage Input Compatibility**: Per-axis scaling and deadzone matching N-Rage plugin
5. **USBtoN64v2 Adapter Support**: Compatible with popular USB-to-N64 hardware adapters
6. **VidExt Override**: Custom Qt-based rendering instead of SDL windows

# Coding Guidelines

## Prime directive: manage complexity

No one can hold a whole nontrivial system in their head. Every rule below exists
to **reduce how much you must think about at one time**. When choosing between
options, pick the one that leaves less to keep in mind.

- Minimize *accidental* complexity (introduced by our choices); partition the
  *essential* complexity (inherent to the problem) into brain-sized pieces.
- **Write for people first, the machine second.** Code is read far more often
  than written — optimize for the reader, not for typing speed.
- **Program *into* your language, not *in* it.** Decide what you want, then
  express it with available tools; compensate for missing features with
  conventions, not by limiting your thinking.
- **If it's hard, it's probably wrong.** "Tricky code," code that resists a
  clean name, comment, or test, is a warning sign — simplify instead of pressing on.
- **Be eclectic, not dogmatic.** Heuristics, not commandments. Hold conventions
  as tools you can drop when a better one fits.
- Make code so simple there are obviously no defects, not so complex there are no
  obvious defects.

## Naming

- The name should fully and accurately describe what the thing **represents**.
  If you can say in words what it is, that is usually the best name.
- Name the **problem**, not the mechanics: `employeeData`, not `inputRec`.
- Be specific. Vague names usable for anything (`x`, `temp`, `data`, `flag`,
  `handle`, `process`) signal a vague design.
- Length follows scope: short for tiny/local scope, longer for wide scope.
  ~10–16 chars is a good target for most names.
- Put computed-value qualifiers last: `totalCost`, `customerCount`, `maxLength`.
- Use `count` (a total) and `index` (one element), not the ambiguous `num`.
- Use precise opposites consistently: begin/end, first/last, min/max, next/prev,
  old/new, source/target, add/remove, get/set, open/close.
- Booleans read as true/false and are stated positively: `done`, `found`,
  `success` — never `notDone`. Avoid `flag`; name the condition.
- Loop indices `i`/`j`/`k` only in short, non-nested loops; meaningful names when
  nested, long, or used outside the loop.
- Adopt one convention and apply it everywhere — **any** convention beats none.
  It should distinguish local vs. member vs. global, and constants/types/variables.
- Avoid: misleading names, names differing by 1–2 chars or only by case,
  numerals (`data1`, `data2`), homophones, hard-to-read characters (`l`/`1`,
  `O`/`0`), and abbreviations that fail the "read it aloud over the phone" test.

```c
/* Bad: mechanics, no meaning */          /* Good: states the problem */
x = x - xx;                               balance = balance - lastPayment;
```

## Variables and data

- Give every variable the **smallest scope** that works. Start restrictive
  (loop → function → file → global, last resort) and widen only when forced.
- Minimize **live time** (lines from first to last use): keep references close
  together so the window where the value can be wrongly altered stays small.
- Declare each variable **near its first use** and initialize it there; never in
  a block at the top.
- **One variable, one purpose.** Don't recycle a `temp` across unrelated jobs,
  and don't overload a value with a hidden second meaning.
- Mark values that shouldn't change as `const`.
- Replace **magic numbers/strings** with named constants. Only `0` and `1`
  belong as bare literals.
- Guard every division against a zero denominator. Watch integer overflow and
  truncation, **including in intermediate results**.
- **Never compare floats for exact equality** — compare within a tolerance.
- Keep array/string indices in bounds; check first, middle, and last endpoints
  for off-by-one.
- Prefer enums over loose constants for a fixed value set; reserve the first slot
  for "invalid" and handle the unexpected case.

```c
/* Magic number → named constant */
for (i = 0; i < MAX_EMPLOYEES; i++) { ... }      /* not: i < 100 */

/* Never test floats with == ; compare within a tolerance */
static const double EPSILON = 1e-9;
int approx_equal(double a, double b) { return fabs(a - b) < EPSILON; }

/* One purpose per variable: don't reuse `temp` for two unrelated jobs */
discriminant = sqrt(b*b - 4*a*c);   /* not: temp = sqrt(...); ... temp = swap */
```

## Functions

- Create a function for a sound reason: to **name an abstraction**, remove
  duplication, hide a sequence or a complex test, or reduce complexity. Deep
  nesting is a signal to extract one.
- Aim for **functional cohesion**: a function does one and only one thing.
- The name describes **everything** it does, including side effects. If the name
  needs "and," remove the side effect rather than lengthening the name. Avoid
  vague verbs (`handle`, `process`, `dealWith`).
- Let length follow the logic, not an arbitrary cap — but be suspicious past a
  screenful.
- Parameters: order **input → modify → output**; keep that order consistent
  across similar functions; put status/error params last.
- Limit parameters to ~7. Needing more consistently means coupling is too tight —
  group the data.
- Use every parameter. Don't reuse an input parameter as a working variable —
  copy to a local; mark inputs `const`.
- Ensure every path returns a valid value (initialize the return value up top).
- Never return a pointer/reference to a local.

```c
/* Don't mutate input params; copy to a working local */
int scale(const int input) {
    int working = input * current_multiplier(input);
    working      = working + current_adder(working);
    return working;            /* `input` still holds the original */
}
```

## Modules and data abstraction

A "class" here means any **module**: data plus the functions that own it (an
Abstract Data Type). The principles apply whether or not your language has classes.

- Model each module as an **ADT**: expose operations, hide representation. Callers
  should never touch the internal data.
- **Information hiding** — for every module ask "what should I hide?" Hide two
  things: complexity, and decisions likely to change.
- Hide a decision behind a function and a type, not scattered literals:

```c
typedef int IdType;                 /* hide the representation */
IdType id = new_id();               /* hide the creation policy */
/* The body of new_id() is the ONE place that changes when IDs must become
   non-sequential, range-reserved, thread-safe, etc. Compare the leaky version:
   id = ++g_max_id;  — duplicated everywhere, impossible to evolve. */
```

- Give each module **one** consistent abstraction. If you can't name what it
  abstracts, or it does two unrelated things, split it.
- Minimize accessibility; expose data through accessor functions, never raw.
- **Prefer containment ("has-a") over inheritance ("is-a").** Use inheritance
  only for genuine specialization, and only when every subtype honors the base's
  full contract (Liskov substitution). Keep hierarchies shallow (2–3 levels).
- Replace a repeated `switch` on a type field with one dispatch point (e.g. a
  function-pointer table).
- **Avoid global data.** Make variables local first; if data must be shared,
  reach it only through access functions — they give one control point,
  validation on every access, and an easy path to a real ADT later.

## Control flow

- **Make the nominal path obvious.** Put the normal case right after the `if`,
  stack error/exception cases below. Don't write an empty `if` with work in the
  `else` — negate the test.
- Cover every case: give `if/else` chains a final `else`, and `switch` a
  `default`, that catches the unexpected value and reports it to the developer.
- Use **guard clauses** (early returns) to check error cases up front and keep the
  nominal code un-indented:

```c
if (!valid_name(name)) return ERR_NAME;
if (!file_open(name))  return ERR_OPEN;
if (!key_valid(key))   return ERR_KEY;
/* real work lives here, at the top indent level */
```

- Treat each loop as a **black box**: one entry, control conditions stated from
  outside, assured termination (mentally run the first, a middle, and the last
  iteration). Keep the body short enough to see at once; extract long bodies.
- **Always brace** conditional and loop bodies, even one-liners — layout must not
  be able to lie about what the body is (see Layout).
- Use `break`/`continue`/early `return` to *simplify*, sparingly; each weakens the
  black-box property. Avoid `goto`.
- For recursion, ensure a base case stops it; prefer iteration for simple cases.
- Replace complicated `if`/`switch` logic (or inheritance chains) with a
  **lookup table** — logic scales badly, data stays flat:

```c
static const int days_per_month[12] = {31,28,31,30,31,30,31,31,30,31,30,31};
days = days_per_month[month - 1];     /* not a 12-branch if/else */
```

- Simplify boolean expressions: name complex tests with an intermediate boolean
  or a well-named predicate function; state them positively (DeMorgan); fully
  parenthesize — don't rely on precedence; write comparisons in number-line order
  (`MIN <= x && x <= MAX`).
- Flatten nesting (cap ~3 levels) and keep **cyclomatic complexity low** (~≤10:
  count 1 + each `if`/`while`/`for`/`case`/`&&`/`||`). High count → extract a
  function.

## Defensive programming

- **Validate all data from any external source** (user, file, network, another
  module) — range, length, format. Reject or sanitize at the boundary.
- Build a **barricade**: validate data as it crosses into trusted code. Outside
  is dirty (use error handling); inside is clean (use assertions).
- **Assertions** are for conditions that must *never* happen (a bug). **Error
  handling** is for conditions you *expect* but hope don't occur. Keep
  side-effecting code out of assertions.
- Decide **robustness vs. correctness** deliberately: keep running with a
  best-effort value (consumer software) vs. never produce a wrong result
  (safety-critical). Apply it consistently.
- Always check returned error codes / call results, even when failure "can't"
  happen. No empty catch blocks.
- Use exceptions sparingly — same bar as assertions — and throw at the
  abstraction level of the interface, with full context.
- **Fail loud in development, soft in production.** Add lavish debug-only checks;
  make bugs glaring while building so the shipped program can degrade gracefully.

## Comments and layout

- **Self-documenting code first.** Good names, structure, named constants, and
  simple control flow carry most of the documentation. Fix bad code; don't paper
  over it with comments.
- Comment **intent and summary ("why")**, never restate the code ("what"). A
  wrong or redundant comment is worse than none.
- A good comment is at the level you'd *name a function* doing the same thing —
  if you can name it cleanly, consider extracting that function.
- Keep comments next to their code and updated; document surprises, workarounds,
  and the reason behind any deliberate oddity (quantify perf tricks so no one
  "fixes" them).

```c
if (account_type == ACCOUNT_NEW)   /* if establishing a new account  — intent */
/* not:  if (account_flag == 0)       // if account flag is zero      — mechanics */
```

- **Layout reveals logical structure** — that is its job; looking pretty is
  secondary. **Consistency matters more than which style** you pick.
- Indent subordinate code (2–4 spaces). Separate "paragraphs" of related
  statements with blank lines. Use whitespace within expressions.
- One statement per line; one declaration per line; no multiple side effects per
  line.
- Brace single-statement bodies so layout and logic can't diverge:

```c
/* No braces: indentation says 3 lines run each pass; the compiler runs only the
   first in the loop, the other two once. The visual structure lies. */
for (i = 0; i < n; i++) {
    int tmp   = left[i];
    left[i]   = right[i];
    right[i]  = tmp;
}
```

## Design heuristics

- **Loose coupling**: few, small, visible, flexible connections. Pass the minimum
  a function needs through its parameter list, not via shared/global state.

```c
/* Tight: forces caller to build a whole Employee.
   Loose: depends only on what it actually uses. */
double vacation_days(Date hire_date, JobClass job_class);
```

- **Strong cohesion**: everything in a module serves one central purpose.
- **Design for change**: identify what's likely to change (business rules, I/O
  formats, platform/vendor specifics, status values) and isolate each behind one
  module so a change hits one place. Use an enum, not a bool, for status that may
  grow new states.
- Favor **high fan-in** (reuse low-level utilities) and **low-to-medium fan-out**
  (a module using >~7 others is doing too much).
- Keep the design **lean** — no speculative "might-need-it" parts. Build the code
  the requirements demand, clearly.
- **One Right Place**: one place to find a given piece of code, one place to make
  a given change.
- Design is **iterative and heuristic** — there's no single right answer. Try
  more than one decomposition. Reuse known patterns, but don't force-fit them.
- Prototype to answer a *specific* design question with throwaway code; keep it
  junk so it isn't reused.

## Writing, evolving, and verifying code

**Grow code from intent.** Design a non-trivial function in intent-level
pseudocode first (what, not how). Review the pseudocode — iterating is cheap
before you're invested in code. Then turn each line into a comment and fill real
code beneath it. If one line explodes into too much code, extract a function (its
name falls out of the pseudocode).

**Refactor purposefully.** Every change should leave internal quality *better*.

- Small, **behavior-preserving** steps, one at a time; recompile and retest after
  each; keep tests green.
- **Never refactor and add functionality at once.** Save the original first.
- Refactor opportunistically — when adding a feature or fixing a bug, and in the
  error-prone/high-complexity areas first.
- Treat even one-line changes as risky (>50% first-attempt error rate); review them.
- **Code smells** to act on: duplicated code; an over-long function; deep nesting;
  poor cohesion; too many parameters; related data not grouped; a function more
  interested in another module than its own; a primitive overloaded for a richer
  concept (e.g. `int` for money); comments propping up bad code; global variables;
  heavy setup/takedown around a call (the interface is wrong).

```c
/* Smell: many setters before, many getters after → wrong interface */
process_withdrawal(id, balance, amount, date);     /* fix: pass what it needs */
```

**Debug scientifically.** The goal is the *cause*, not the symptom.

- Assume the bug is yours. Reproduce it reliably; shrink to the simplest case.
- Form a hypothesis from all the data; understand the problem well enough to
  predict it before you change anything.
- **Fix the root cause, never special-case the output.** Make one change at a
  time, for a reason you understand — no superstitious "+1 until it works."
- Add a regression test that exposes the bug, then look for the same bug elsewhere
  (defects cluster).

```c
/* Symptom patch (wrong): the real defect is sum[] never initialized to 0 */
if (client == 45) sum[45] += 3.45;     /* do NOT do this — fix the init instead */
```

**Test as you go.** Testing *measures* quality; it can't create it.

- Write tests early (ideally first); keep them as a regression suite.
- Aim for branch coverage, not just line coverage. Derive the minimum cases from
  the branch count (1 + one per decision).
- Test **boundaries** (just below / at / just above each limit) and **bad data**
  (none, too much, wrong kind/size, uninitialized), not just clean inputs.
- Focus on error-prone areas — defects cluster (~80% in ~20% of routines).

**Combine quality techniques; prevention beats detection.** No single
defect-finding method catches more than ~75% — **reviews/inspections catch what
testing can't** and find defects earlier and cheaper. Review every change.
Improving quality *lowers* total cost; it doesn't trade against it.

**Performance comes last.** Build clean, correct, modular code first.

- Don't optimize as you go. Most run time lives in a small fraction of code
  (80/20).
- **Measure before tuning, and again after** — intuition about hot spots is
  usually wrong, and many "optimizations" do nothing or backfire (and vary by
  compiler/platform). Keep the clean version unless a measured need justifies the
  trade.

```c
/* The straightforward version. A hand "pointer-walk" rewrite measured ZERO gain
   here — the optimizer already does it. Don't trade clarity for an unmeasured guess. */
for (row = 0; row < rows; row++)
    for (col = 0; col < cols; col++)
        sum += matrix[row][col];
```

# Contributing

## Git Commit Messages

Write for the future reader re-establishing context (often yourself, months later).

**One logical change per commit.** Adding a feature, fixing one bug, etc. Can't summarize it in a few words? Too big — split it. Use `git add -p`/`-i` to split. Err toward too many commits, not too few.

**Format:**
- Summary line ≤50 chars, present tense ("Add", "Fix" — not "Added"/"Fixes").
- Blank line, then body paragraphs (omit body only for trivial obvious changes).
- Body describes intent and approach, not the code. Answer:
  - **Why** is it necessary? (bug / feature / perf / correctness)
  - **How** does it address it? (high-level approach)
  - **What** are the effects? (side effects, benchmarks, caveats)
- Rule of thumb: from the message alone, another dev could reproduce the patch.

**Don't:**
- End-of-day "backup" commits (random cross-code diffs).
- Per-file commits when one logical change spans files.
- Vague messages ("misc fixes and cleanups").
- Two unrelated changes in one commit ("fix bug + rename foo to bar").
- Whitespace/reindent changes mixed with code changes — separate commits.

## Pull Requests

### Core principle

A PR steals time from your reviewer. The reviewer's mental cache is cold — they weren't there when the code was written. Your job as author is to hand them something small, well-framed, and easy to digest so the review costs them as little as possible.

### 1. Explain the *why*

- The diff shows *what* changed, not *why*. Spell out the reasoning the diff can't convey (e.g. "why did 16 become 17?").
- If the reason is subtle, consider whether the *code* should be changed to make it self-evident — PR descriptions are not tracked alongside the code as well as commits are.
- Link to related bugs/issues. Use `closes #N` so merging auto-closes the issue.

### 2. Keep it small and focused

- Make the **smallest useful change**. Smaller PRs respect the reviewer's time ("only 5 minutes of your time, thanks").
- Big PRs get one of two bad outcomes: a rubber-stamp "LGTM" with no real review (past ~500 lines, honest reviewers admit they can't truly understand it), or "have you considered a completely different approach?" after you've sunk hours in.
- A PR is an *option* that could be thrown away. The bigger it is, the more it hurts when it's rejected — and the more likely people are to have strong opinions about its impact.

### 3. Get buy-in before large or risky work

- For anything substantial, agree the general approach with the reviewer *before* crafting a beautiful, large, throw-away-able change.
- A small PR also lets the reviewer say "this won't work" without feeling guilty about the time you spent.

### 4. Handle unavoidably large changes well

- Some changes can't be minimal (e.g. renaming a class across a thousand files). Make these **mechanical and single-purpose** so each line reads the same — and note "I did this with a refactoring tool."
- Don't mix a mechanical rename with new behavior in the same PR. Rename in one PR, add responsibilities in the next.
- Break big work into a chain of **stacked/cascading PRs**, each built on the last — but only when you're confident the head of the chain won't significantly change.

### 5. Self-review before you submit

- Diff your whole change before opening it. Reading it back catches leftover debug code, missed renames, missing tests, and things that should be split into two PRs.
- Pushing to GitHub and reviewing in its UI (rather than your editor) helps flip your brain into reviewer mindset — you can't edit, it doesn't look like your editor, so you read it impartially.
- A final "self review" commit is normal and worthwhile.

### 6. Make CI green

- Run the full test suite on the PR. Tests are what actually save you most of the time.
- Tests should be present and pass; lean on tooling (coverage gates, linters) to catch "0% of new code is tested" so it isn't on the reviewer to police.

### 7. Curate the history

- Squash intermediate commits to present a clean, readable artifact with good names — kind to the reader and easier to revert cleanly later.
- Exception: for open-source/expository work, leaving the messy history can be a deliberate choice to show *how* you got there (the process behind the sausage). Decide intentionally.
- Make reverting easy — a clean, atomic change lets someone safely roll it back at 4am and let you fix it the next day.

### 8. Manage reviewers explicitly

- Prefer **one** named reviewer who is on the hook. With multiple reviewers, the chance any one responds drops sharply — everyone assumes someone else has it, and two people can end up half-reviewing the same change.
- If others merely need awareness, **CC** them separately in the comments rather than making them reviewers.
- If a change genuinely needs two sign-offs, send it to both and make it explicit that you want both.
- Choose your reviewer to fit the need: a thorough atom-by-atom reviewer when you want the bugs found; a lighter touch when appropriate.

### 9. Own the merge and the deploy

- The **original author** should merge, unless they explicitly delegate it. This lets you orchestrate the order of interrelated branches you tangled up.
- Under continuous deployment, merging ships to prod — so the person shepherding the change should be there to watch the deploy and tail the logs. Adding a PR review shouldn't transfer deployment ownership.
