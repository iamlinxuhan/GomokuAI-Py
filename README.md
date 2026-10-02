# 🎮 Gomoku AI

[简体中文](README.zh-CN.md) | **English**

> **This code is the legacy version, and it is the one that supports macOS.**
> The latest, optimised and rewritten C++-engine version is at
> https://github.com/iamlinxuhan/GomokuAI .
> This legacy repository has moved to https://github.com/iamlinxuhan/GomokuAI-Py .
> The engine is pure Python (it depends on no platform-specific binary), so on
> macOS you can install Python + PyQt5 and
> [run it from source](#option-2-run-from-source). The Release ships
> **`.dmg` / `.pkg` for both macOS arm64 and Intel** — the first launch needs a
> small Gatekeeper detour (why and how:
> [download notes](#option-1-download-a-release)).

> **This update is the last one. Support and maintenance for this legacy
> project stop here.**

A human-vs-AI Gomoku (five-in-a-row) program built on **PyQt5**. The AI engine is
**bitboard + fully incremental evaluation + Negamax/PVS + transposition table +
quiescence search + VCF (continuous forced fours)**, running on CPU only. Three
difficulty levels; a cool-toned design system around a warm wooden board, with a
side panel that plots the AI's score and the estimated human win rate live.

![Python](https://img.shields.io/badge/Python-3.11%20%7C%203.13-blue)
![PyQt5](https://img.shields.io/badge/PyQt5-5.x-green)
![NumPy](https://img.shields.io/badge/NumPy-✓-orange)
![Version](https://img.shields.io/badge/version-2.1.0-brightgreen)

> **v2.0.0 was a complete rewrite**: the evaluation and search layers were
> replaced wholesale and the UI was rebuilt as a unified design system. The old
> "layered TSS threat response", "multi-line defence" and "desperate mode" have
> all been **deleted** — their judgements rested on heuristic rules, and their
> failure mode was structural (see
> [Why the threat-response layer was deleted](#why-the-threat-response-layer-was-deleted)).
> The GPU acceleration branch was deleted for the same reason: it never worked
> from beginning to end (the GPU detection result was unconditionally overwritten
> back to `'cpu'` a few lines later). The engine therefore has **exactly one CPU
> path**, and the installers no longer bundle a PyTorch/CUDA runtime.

---

## ✨ Features

### 🤖 AI algorithms

| Technique | Notes |
|------|------|
| **Bitboard** | 361-bit integer bitmap + fully incremental state (candidate set / neighbour counts / hash / threat aggregation). `make`/`unmake` are strictly symmetric and support out-of-order undo |
| **Pattern classification** | Defined by **set semantics**: `F(S) = {empty e : playing at e makes five}`, so `\|F\|≥2` is an open four and `\|F\|=1` is a four. It counts the **number of winning points**, not intermediate notions like "how many open fours" that would first need explaining |
| **Zero-sum evaluation** | `evaluate(pos, black) == -evaluate(pos, white)`, maintained incrementally, with no asymmetric coefficients |
| **Negamax + PVS** | Principal Variation Search, fast pruning on zero-window searches |
| **Transposition table** | Fixed-size array with age-based replacement; mate scores are stored normalised (store `value+ply`, read back `value-ply`) |
| **Quiescence search** | At leaf nodes, only forced moves are searched, removing misjudgements caused by the evaluation being cut off at the horizon |
| **VCF (continuous forced fours)** | Searches specifically for forced sequences where "every move is a four"; **three-state return** (win / no win / could not finish), with AND semantics on the defending side. **Enabled on all three levels** — the lower levels are separated by their depth cap, not by making the Novice level blind |
| **Move ordering** | History heuristic + killer moves + winning-point priority |
| **Time control** | Hard time limit + iterative deepening + aspiration windows + cooperative cancellation (restarting mid-think interrupts immediately) |
| **Deterministic opening** | The empty board plays tengen (without searching); everything else goes to a regular search |

### 🎨 UI design

- **One design system**: colours, font sizes, spacing and corner radii each have
  a single source of truth (`theme.py`), and the whole app shares one QSS
- **Cool tech, warm board**: the interface chrome is cool cyan while the board
  stays warm wood — the board is the only warm focal point in the app. The wood
  grain is generated procedurally (preserving its gloss and light/dark gradient);
  stones are sprites with a specular highlight and a contact shadow
- **Controls with all four states**: every button generates hover / pressed /
  focus / **disabled** — the old version had not a single `:disabled` rule, so
  during the AI's think the undo button looked perfectly normal but would not
  respond to a click
- **Coordinates and logs from one source**: the board's column labels go through
  `gamelog.col_letter` (skipping I), exactly matching the game record / logs /
  position suite
- **Loading screen**: HUD corner brackets + a wooden plate marker; it honestly
  shows how much longer this screen will stay (about 0.9 s) rather than faking
  progress that is off by a factor of 43 from the real cost
- **Last move**: a ring in the opposite colour to the stone, clear on both black
  and white
- **Side panel**: live turn / difficulty / status / undo count / move count,
  with two charts below
  - **`AI score`** (for debugging): a line of per-move scores with a logarithmic
    (symlog) y-axis whose order of magnitude only ever grows — otherwise the axis
    would contract as scores fall back and the same curve would suddenly look
    steeper. **A filled dot = a search result (has depth, i.e. evidence); a
    hollow dot = a static estimate (the fallback when no search was run).** Below
    it, a monospaced readout shows the search parameters (depth / nodes / nodes
    per second / elapsed)
  - **`Human win rate (est.)`**: a monotonic curve converted from the engine's
    score, on a fixed 0–100% y-axis that does **not** auto-scale — the curve
    lives in the middle third (a live three → a four is only 65% → 76%), and
    auto-scaling would stretch it out to look far more dramatic than it is.
    **This is an estimate under the engine's model, not a statistically
    calibrated probability**: 50% only means "the engine thinks it is balanced",
    and 100%/0% appear only for a mate the search has proven. The conversion
    table is in `analysis.py`, and every knee point lines up with a score
    constant in `engine.py`
- **Win/lose dialog**: a translucent overlay with Play again / Quit
- **Asynchronous AI**: threaded on a QThread, so the UI never stalls

---

## 📦 Getting started

### Option 1: download a Release

Download the package for your platform from
[Releases](https://github.com/iamlinxuhan/GomokuAI-Py/releases) and
double-click (no Python installation needed).

| Platform | File | Notes |
|---|---|---|
| Windows | `GomokuAI_Setup_vX.Y.Z.exe` | **Installer** (recommended): a wizard, creates Start-menu / desktop shortcuts, uninstallable from 「Add or remove programs」 |
| Windows | `GomokuAI_Portable_vX.Y.Z.exe` | **No install**: a single exe, copy it anywhere, writes nothing to the registry |
| Linux amd64 | `GomokuAI_For_Linux_AMD.deb` | **Installer** (recommended): `sudo dpkg -i`, declares its dependencies |
| Linux amd64 | `GomokuAI_For_Linux_AMD` | **No install**: a single executable, `chmod +x` and run |
| Linux arm64 | `GomokuAI_For_Linux_ARM.pkg` | **Installer**: unpack, then `sudo ./install.sh` |
| Linux arm64 | `GomokuAI_For_Linux_ARM` | **No install**: same, arm64 architecture |
| macOS arm64 | `GomokuAI_For_MacOS_ARM.dmg` | **Installer** (recommended): open it and drag 「五子棋AI」 into Applications, no password needed |
| macOS arm64 | `GomokuAI_For_MacOS_ARM.pkg` | **Installer**: a wizard, installs into `/Applications`, needs the admin password |
| macOS Intel | `GomokuAI_For_MacOS_AMD.dmg` | **Installer** (recommended): same, x86_64 (Intel Mac) |
| macOS Intel | `GomokuAI_For_MacOS_AMD.pkg` | **Installer**: same, x86_64 (Intel Mac) |

**The first launch on macOS needs a small Gatekeeper detour.** This project's
macOS artefacts are **not signed with an Apple developer certificate** (that
needs a $99/year developer account), and macOS blocks any "downloaded, unsigned"
application outright with 「cannot be opened because Apple cannot check it for
malicious software」. **This does not mean the file is damaged.** There are two
ways to let it through:

- **Graphical** (recommended): double-click once and let it be blocked — then
  open **System Settings → Privacy & Security**, scroll down to the **Security**
  section, where a notice about 「五子棋AI」 will appear; click **"Open Anyway"**
  and confirm. You only need to do this once; afterwards double-clicking works.
  ⚠️ This button **only appears for about an hour after the block**; if you
  cannot find it, double-click the app again.
- **Command line** (fastest — it strips the quarantine flag directly):

  ```bash
  xattr -dr com.apple.quarantine /Applications/GomokuAI.app
  ```

> **An old workaround that no longer works**: plenty of tutorials online still
> say "Control-click the icon → Open → click Open again". **Apple removed that
> shortcut in macOS 15 Sequoia**, and following it now just gets you blocked
> again.

**Download the one that matches your machine**: Apple Silicon (M-series) takes
`ARM`, Intel takes `AMD`. Getting it wrong does not fall back automatically — the
Intel build runs on Apple Silicon through Rosetta 2 (the system will offer to
install Rosetta), but not the other way round.

**The minimum OS version is macOS 14 (Sonoma)**, the same for both architectures.
That number is not ours to choose — it was measured in CI (the highest `minos`
across every binary inside the `.app`): the main executable itself only asks for
11.0 (arm64) / 10.13 (Intel), but numpy's
`_multiarray_umath.cpython-311-darwin.so` asks for 14.0 — pip on the macOS 15
build machine picks the highest numpy wheel it can run, and numpy ships both a
`macosx_11_0` and a `macosx_14_0` variant. **On an older macOS it simply fails to
launch** — there is no graceful degradation.

**The same across the board**: everything is **CPU-only, needs no graphics
driver, and needs no PyTorch/CUDA install**.

**The no-install builds declare no dependencies.** If the system is missing
`libgl1` / the xcb libraries, or has no CJK font at all, they will not install
them for you — they will just fail to start or show a screen full of boxes. The
`.deb` / `.pkg` pull those in (including a CJK font candidate chain), so **if you
are unsure, take the installer**.

The Linux no-install builds are **bare executables with no extension**, so set
the executable bit after downloading:

```bash
chmod +x GomokuAI_For_Linux_AMD
./GomokuAI_For_Linux_AMD
```

(Release assets do not carry file permissions; nobody can do this step for you.)

### Option 2: run from source

```bash
# 1. Clone the repository
git clone https://github.com/iamlinxuhan/GomokuAI-Py.git
cd GomokuAI-Py

# 2. Install dependencies (Python >= 3.11)
pip install -r requirements.txt

# 3. Run
python main.py
```

---

## 🎯 Rules

1. Standard Gomoku rules, 19×19 board
2. Black moves first; whoever first gets **five in a row** horizontally,
   vertically or diagonally wins
3. Players alternate, and a point cannot be played twice

---

## 🎛️ Difficulty

The three parameter sets are exactly `DIFFICULTY` in `engine.py`; units and
values can be compared directly:

| Level | Time per move | Depth cap | Quiescence plies | VCF budget |
|------|---------|---------|------------|---------|
| **Novice** | 3 s | 4 | 4 | 0.3 s |
| **Intermediate** | 7 s | 10 | 8 | 0.5 s |
| **Advanced** | 15 s | 24 | 10 | 1.5 s |

Three things need saying clearly:

- **"Depth cap" is a cap, not a promise.** The depth actually reached is decided
  by iterative deepening within the time limit — the first few opening moves come
  back immediately (the evaluation finds nothing worth searching), and only the
  midgame runs out the clock.
- **VCF time counts inside that limit**, not as extra overhead.
- **All three levels have VCF on.** The Novice level once had it off, and that
  was a mistake: with it off, Novice could neither work out its own chains of
  fours nor see the opponent's — that is not "a lower difficulty", that is
  **blindness**. In a real lost game the human won with an 11-move VCF chain, and
  Novice had no mechanism at all that could see it (that key position is pinned
  in `tests/test_difficulty.py` as `LOST_183202_P28`). The lower levels are now
  separated by **depth cap**, not by a missing subsystem.

**Intermediate's 7 seconds was set by one specific position, not guessed.** That
game (`game_log_20260922_235240`, Intermediate playing white, human winning on
move 31 at J8) was decided on move 4: with only three stones on the board,
**ply 4 picks K9 while ply 5 picks K7/K11** — adjacent plies reaching opposite
conclusions. Pinning that move and replaying the rest with the old budget
(`python tools/pivot_ab.py --budget 5.0`): K9 gives
**black wins · move 31 J8**, and **matches the log move for move — all 31 moves
identical**; K7 or K11 gives white a counter-win on move 12 / 18. That one move
is the whole game; the 27 moves after it are just symptoms.

**The problem with the old budget was not "slightly too shallow" — it sat right
on the tipping point.** With the same input and the same 5.0 s budget, repeating
the search for that move gives **two different answers**:

| `time` | Budget | Move 4, measured |
|---|---|---|
| 5.0 (old value) | 4250 ms | K7 · ply 5 and **K9 · ply 4** alternate — across three batches, 3/12 and 0/8, and **4/4 under load** |
| 6.0 | 5100 ms | K7 · ply 5, stable at 5/5 |
| **7.0** | 5950 ms | K7 · ply 5 · val −170, stable at 4/4 |

**The three batches in the 5.0 row are themselves the conclusion**: the ratio
varies with how busy the machine is — with 6 saturated processes running, 4/4
played the losing move. So on a slightly busy machine, Intermediate
deterministically plays that losing game from the very opening. 7.0 leaves about
0.85 s over the first stable point measured at 6.0, which is why it is 7.0
rather than 6.0.

The way to measure it is
`python tools/pivot_ab.py --budget 5.0 --stability N`: **when idle you will
mostly see only K7** — to coax the losing move out, you have to make the machine
busy and run the same command again.

**This pushes the horizon farther out; it does not cure anything.** Any fixed
budget has a horizon — the next position will always have a move that needs one
more ply to see. This time limit only guarantees "this one position is visible",
and it was set **against one human line of play**: against an engine opponent at
the same level, white does not thereby win.

**The cost has to be stated honestly**: on **open midgames** the lower levels
still reach similar depths. The real separation in the staircase is `max_depth`
4/10/24 — it only spreads out on narrow trees.

Measured (`python tools/bench.py --engine engine --level N`, 5 fixed positions).
Note that "depth cap" is the configured value while "median depth" is what was
actually reached — they are not the same thing:

| Level | Wall clock median / max | Depth median / max | nps median | Time compliant |
|---|---|---|---|---|
| Novice | 718 ms / 2552 ms | 4 / 4 | 54,155 | 5/5 |
| Intermediate | 5956 ms / 5978 ms | 4 / 4 | 50,700 | 5/5 |
| Advanced | 12 769 ms / 12 789 ms | 5 / 5 | 48,873 | 5/5 |

(Measured 2026-09-23. The **minimum** depth across the three levels is 1 in every
case, and that is `mid_10stones` — a position that has already been decided,
where iterative deepening converges immediately on finding the mate; the depth is
low because there is nothing to search.)

Look at the Intermediate row: **on these 5 positions it still only reaches ply
4** (they are not that move-4 position from above), so Intermediate's 7 seconds
shows up here as "runs out the clock, depth unchanged". The depth threshold
follows the position, not the level.

**This table reads "how quickly these 5 positions resolve", not "the engine is
always this fast".** Several of them are **already-decided positions** — for
example, the side to move in `mid_10stones` has already lost (the opponent has a
VCF mate, `best_val = -(WIN_SCORE-1)`, i.e. five next move), and `mid_clash` is a
win in 6 for the side to move. Positions like these should return instantly, and
their depth is naturally low. A real, hard midgame will run out the whole time
budget — that is what the limit is for. The "time compliant" column above means
"did not exceed", not "used it all".

**Depth 4–5 is the ceiling of this design, not a regression.** It is full-width
PVS + transposition table + quiescence search + VCF, **with no modern pruning at
all** (no LMR / null-move / futility pruning), and the measured effective
branching factor is around 20 per ply. To reach ply 10 in ~650k nodes in 13
seconds you would need to squeeze the branching factor down to about 3.7 — hence
the threshold was corrected from `≥10` to `≥5`.

---

## 🧠 Engine design

### Mate scores and static scores are banded apart

Static scores are clamped to ±`STATIC_MAX`, mate scores start above
`WIN_SCORE - MAX_PLY`, and a vacuum band of 128 sits between them — so "this is
a forced mate" and "this position looks good" cannot be confused numerically, and
`is_mate()`'s verdict is reliable rather than a guess.

Compare the old engine on `mid_10stones` with the same benchmark:
**8149.8 ms only reaches ply 5** (`tools/BASELINE.md`), whereas the new engine
reaches ply 1 in 3.5 ms — both recognise this as a lost position, the old engine
reporting `-10000003` and the new one `-(WIN_SCORE-1)`.

The key difference is not speed but **whether that number can be read**: the old
engine's mate score (`-10000000 - depth`) **overlapped in range** with its own
composite static score — the attack/defence weighting of "desperate mode" could
reach ±3.45e7, larger than the mate score. So "I am dead in three moves" and
"this position is pretty bad" were numerically indistinguishable, and the UI just
saw a large negative number.

### Time is the hard limit; the depth cap is a safety valve

After taking the level, `think` sets a **hard** deadline (`t0 + time × 0.85`,
keeping 15% for the return trip and the UI); iterative deepening goes as deep as
it can before that deadline, and `max_depth` exists only to stop a midgame that
"looks quiet but is full of fours everywhere" from dragging even the low levels
to 20 plies.

**The safety valve has to actually constrain.** The Novice level once did not:
its 1.275 seconds only stretched to ply 3, and on the midgame from that real lost
game, ply 3 and ply 4 gave **opposite** verdicts — ply 3 reported −560
("slightly behind"), ply 4 reported −698910 ("nearly dead"), and ply 4 needed
1.55 seconds. `tests/test_difficulty.py::test_low_level_reaches_its_own_depth_cap`
now pins this down: a low level's time must be enough for it to reach its own cap.

### Why the threat-response layer was deleted

The old version had a roughly 130-line `_check_immediate_threat` layer before
the search, using seven heuristic rules to answer "what should I play now?" Its
failure mode was structural: **the time saved when the heuristic is right is far
smaller than the game lost when it is wrong.**

The most typical criterion read `opp_win >= 2 or opp_live4 >= 2`, whereas the
most common double-threat shape in real play is `opp_win == 1 and opp_live4 >= 1`
— **which falls exactly outside it**.

The patch was to add a rule, and rules can never be finished; the real problem is
"approximating a decidable problem with a finite set of rules". The new engine
splits it into two mechanisms that give deterministic verdicts: quiescence search
counts the number of winning points (exact), and VCF handles longer forced
sequences (equally exact, and able to distinguish clearly between "there is no
mate" and "I could not finish"). `tests/test_threat.py` asserts precisely that
"**these names should no longer exist**" — the positions they used to handle must
still be handled correctly, but no longer by heuristic.

---

## 📁 Project layout

```
GomokuAI/
├── main.py            # UI assembly + game-flow wiring (no search/eval logic)
├── engine.py          # the AI engine: bitboard / eval / search / VCF (zero Qt, numpy only, unit-testable standalone)
├── analysis.py        # score -> win-rate conversion, symlog mapping, readout formatting (pure functions, no Qt)
├── charts.py          # the two self-drawn charts on the panel (line / grid / markers)
├── gamelog.py         # game logs and board-coordinate format
├── theme.py           # design system: palette / font sizes / spacing / radii / QSS generation
├── ui_kit.py          # reusable widget primitives (titles, info rows, buttons, page skeleton, tech texture)
├── board_geometry.py  # pixel <-> cell conversion (pure math, no Qt)
├── board_render.py    # offscreen board rendering (wood grain / stone sprites / layer cache)
├── tests/             # pytest: geometry, incremental engine, position suite, win-rate conversion, colour-literal guard
├── tools/
│   ├── BASELINE.md    # measured baselines and thresholds per stage (with withdrawn numbers marked)
│   ├── bench.py       # benchmark         selfplay.py  # self-play A/B win rate
│   ├── positions.py   # position suite    analyze_log.py  # replay a game log
│   ├── gui_smoke.py   # headless UI smoke test    ui_snapshot.py  # offscreen screenshots
│   └── legacy_engine.py  # verbatim snapshot of the old engine (must not be modified; the A/B control)
├── installer/
│   └── GomokuAI.iss   # Windows installer script (Inno Setup, **UTF-8 with BOM**, see "Packaging")
├── requirements.txt      # runtime dependencies (numpy / PyQt5)
├── requirements-dev.txt  # development and packaging dependencies (incl. pytest / pyinstaller)
├── input.png          # design reference image (the source the board colours were sampled from; **never read at runtime**)
├── 五子棋.ico          # application icon (used directly on Windows; converted to .icns when packaging for macOS)
└── README.md
```

---

## 🛠️ Packaging (Windows / Linux / macOS)

Every platform produces **two artefacts**: one that installs into the system and
one that runs without installing. That is what CI does; locally, do the same.

### Windows

```bash
# Install dependencies (the packaging tools are in the dev requirements)
pip install -r requirements-dev.txt

# (1) The input to the installer: onedir
pyinstaller --onedir --windowed --icon="五子棋.ico" --name "GomokuAI" main.py

# (2) The no-install single file: onefile (use a different --name from (1), see below)
pyinstaller --onefile --windowed --icon="五子棋.ico" --name "GomokuAI_Portable" main.py
```

**Why two instead of one.** `--onedir` starts fast (the runtime sits right beside
it and is loaded directly), but the directory holds hundreds of files, so it
cannot be handed to a user without an installer wizard; `--onefile` is a single
file that runs after being copied anywhere, at the cost of unpacking the bundled
runtime to a temp directory **on every launch** — a PyQt5 program's first window
is noticeably slower. So: the copy that lives on the disk is built with onedir
into an installer, and the copy that needs to "run off a USB stick or someone
else's machine" is built with onefile.

**The two builds must use different `--name`s.** Otherwise they share the
same-named intermediate directories under `build/` and `dist/`, and the second
build picks up the first one's cache — mixing things into the artefact that
should not be there.

### The installer (Inno Setup)

`installer/GomokuAI.iss` wraps the output of (1) into a wizard-based installer:

```bash
# Install the compiler (locally; in CI it is choco install innosetup)
winget install JRSoftware.InnoSetup

# Compile (AppVersion is usually passed in by CI from the tag)
& "C:\Program Files (x86)\Inno Setup 6\ISCC.exe" /DAppVersion=2.1.0 installer\GomokuAI.iss
```

The output goes to `dist-installer/GomokuAI_Setup_v<version>.exe`. It installs
into `%LOCALAPPDATA%\GomokuAI` (no admin rights, no UAC prompt) and registers an
uninstall entry in 「Add or remove programs」 — exactly the two things the old
7z + `install.bat` scheme could not do.

Two easy traps, both documented in the `.iss`:

- **This file must be saved as "UTF-8 with BOM".** It contains Chinese (the app
  name, the icon path); without a BOM, ISCC parses it as ANSI and the Chinese
  turns into mojibake. When editing this file, don't let the editor eat the BOM.
- **The Chinese interface only loads when the compiler ships
  `Languages\ChineseSimplified.isl`.** Inno Setup only officially includes
  Simplified Chinese from 6.3 onwards, and a `MessagesFile` pointing at a
  nonexistent file is a hard compile-time error. CI passes `/DHasChinese=1` only
  when it detects it — using the compiler's own copy rather than vendoring the
  `.isl` into the repository keeps the language file and the compiler forever
  from the same source, with no version mismatch.

No torch exclusion flags are needed any more: it is no longer a dependency. The
old version had to free up disk space and then split into two packages just to
fit the CUDA runtime inside GitHub's 2GB limit; now only numpy and PyQt5 remain.

### Linux

```bash
# (1) The input to the installer: onedir
pyinstaller --onedir --windowed --name "gomoku-ai" main.py     # use gomoku-ai-arm for arm64

# (2) The no-install executable: onefile
pyinstaller --onefile --windowed --name "GomokuAI_For_Linux_AMD" main.py
```

The `dist/GomokuAI_For_Linux_AMD` from (2) is the **bare executable** on the
Release (no extension) — `chmod +x` and run. The output of (1) is installed into
the system by `.deb` (`dpkg-deb`) and `.pkg` (`tar` + `install.sh`)
respectively; both scripts live in CI, see
[build.yml](.github/workflows/build.yml).

**Linux's onefile has two caveats Windows does not**:

- **It cannot declare dependencies.** A `.deb` can list `libgl1`, the xcb
  libraries and a CJK font candidate chain in `Depends:`; a bare file cannot — it
  reports whatever is missing (or shows a screen full of boxes). This is why it
  has to coexist with the `.deb` rather than replace it.
- **The executable bit is not in the file.** Release assets store bytes only, so
  what the user downloads is `0644` and they must `chmod +x` it themselves. The
  artifact leg (`upload-artifact`) does not preserve permissions either. CI does
  not compensate in any way — it cannot; the only thing to do is say so clearly
  in the README.

### macOS

```bash
# (1) First convert the Windows .ico into .icns (macOS only understands .icns)
python -c "from PIL import Image; import glob, os; \
src = Image.open(glob.glob('*.ico')[0]).convert('RGBA'); \
os.makedirs('AppIcon.iconset', exist_ok=True); \
[src.resize((p, p), Image.LANCZOS).save('AppIcon.iconset/' + n) \
 for p, n in [(16,'icon_16x16.png'), (32,'icon_16x16@2x.png'), \
              (32,'icon_32x32.png'), (64,'icon_32x32@2x.png'), \
              (128,'icon_128x128.png'), (256,'icon_128x128@2x.png'), \
              (256,'icon_256x256.png'), (512,'icon_256x256@2x.png'), \
              (512,'icon_512x512.png'), (1024,'icon_512x512@2x.png')]]"
iconutil -c icns AppIcon.iconset -o AppIcon.icns

# (2) The .app
pyinstaller --windowed --name "GomokuAI" \
  --osx-bundle-identifier com.iamlinxuhan.gomokuai --icon AppIcon.icns main.py

# (3) Switch to the Chinese display name, then **re-sign** (editing Info.plist invalidates the signature from the previous step)
/usr/libexec/PlistBuddy -c "Set :CFBundleName 五子棋AI" \
  dist/GomokuAI.app/Contents/Info.plist
/usr/libexec/PlistBuddy -c "Set :CFBundleDisplayName 五子棋AI" \
  dist/GomokuAI.app/Contents/Info.plist
codesign --force --deep --sign - dist/GomokuAI.app
```

The `dist/GomokuAI.app` from (3) is then wrapped by `hdiutil` (`.dmg`) and
`pkgbuild` (`.pkg`) into the two package types on the Release; both scripts live
in CI.

**The two architectures are each built natively, not as universal2.** PyInstaller
bundles PyQt5's Qt dynamic libraries into the `.app`, and universal2 requires
every Mach-O in the bundle to be fat — but the PyQt5-Qt5 wheel ships only
`macosx_11_0_arm64` and `macosx_10_13_x86_64`, so a dual-architecture Qt cannot
be assembled. Forcing it only yields an `.app` where one half of the
architectures will not start. CI runs `macos-15` (arm64) and `macos-15-intel`
(x86_64) once each, and **first asserts that `platform.machine()` matches the
label**: an artefact does not prove its own architecture, and a label quietly
repointed cannot be detected afterwards.

**There is no code signature.** That needs an Apple developer account ($99/year),
which this project does not have; only an ad-hoc signature (`codesign -s -`) is
applied. The consequence is that the user's first launch needs a Gatekeeper
detour, described in the
[download notes](#option-1-download-a-release).

**The minimum OS version is decided by the build machine, not by us.** Both
architectures measured out to **macOS 14**: the main executable asks for
11.0 / 10.13, but numpy's `_multiarray_umath.cpython-311-darwin.so` asks for
14.0 — pip on the macOS 15 runner picks the highest numpy wheel it can run
(`macosx_14_0`) rather than the most compatible one (`macosx_11_0`). CI's
`Report the minimum macOS version the .app requires` step exists to print this
number: **it is a fact, not an invariant** — changing the runner or the numpy
version changes it. Pinning numpy's version would be needed to push it down to
macOS 11, and the first rule in `requirements.txt` is "do not pin versions", so
that was not done here — the same class of trade-off as Linux's bare file
"cannot declare dependencies": say it clearly, rather than pretend it isn't so.

---

## 🧪 Tests

```bash
pip install -r requirements-dev.txt
python -m pytest -q                     # full suite
python -m pytest tests/test_vcf.py -q   # just one area
python -m pytest -q -m "not perf"       # skip machine-speed-dependent thresholds (what CI runs)
```

**About the `perf` marker**: a handful of thresholds test "has the engine
regressed", but their readings are "how many nodes per second" or "what ply did
it reach within 3 seconds" — on a slower machine, a regression and a slow machine
are numerically indistinguishable. Such cases are marked `perf` and skipped by CI
(`-m "not perf"`, **303 tests**), while locally the full suite runs
(**324 tests**). **Correctness, incremental consistency, time compliance, the
position suite, VCF and pattern classification carry no marker**; they should
pass on any machine — the time bounds are enforced by the engine itself,
independent of machine speed.

| Test file | Coverage |
|---------|------|
| `test_engine_parity.py` | Module boundaries (`engine.py` with zero Qt / zero torch, and not importing `main`) + state is resettable + field-for-field reproducibility after a reset |
| `test_win.py` / `test_incremental.py` | Bitboard: five-in-a-row detection (including the line-wrap trap) + step-by-step diffing of the incremental state against a naive implementation |
| `test_eval.py` | Pattern partial ordering and zero-sumness |
| `test_tt.py` | Transposition entry types, mate-score normalisation, cross-game state isolation |
| `test_search_mate.py` | Finds mates, recognises lost positions, and replays the engine's self-reported mate line to check the accounts |
| `test_vcf.py` | A real chain is found / a false "must-win" is refuted / the three states are distinguishable / `dist` is reproducible / **all three levels really do have VCF on** |
| `test_threat.py` | **The heuristic quick-answer layer does not exist**, while the positions it used to handle are still handled correctly |
| `test_difficulty.py` | Time / depth / throughput thresholds for the three levels + **low levels reaching their own depth cap** |
| `test_cancel.py` | Cancellation latency and "no ghost stones left behind" |
| `test_board_geometry.py` | Pixel↔cell conversion (pure functions, no QApplication needed) |
| `test_analysis.py` | Win-rate conversion: odd symmetry, monotonicity, anchor values reproduced exactly, **anchors must be `engine` constants**, the static band may not claim 100% |
| `test_no_literal_colors.py` | The UI no longer contains hard-coded colour literals |
| `test_positions.py` | Position suite: self-consistency + frozen "old engine must fail these" items + the new engine's solve rate |

There are also `tools/gui_smoke.py` (headless UI smoke test),
`tools/ui_snapshot.py` (offscreen screenshots), `tools/bench.py` (benchmark),
`tools/selfplay.py` (A/B win rate against the old engine), `tools/positions.py`
(position suite), `tools/analyze_log.py` (replay a game log) and
`tools/pivot_ab.py` (pin a single move, or repeat the search, to quantify the
effect of the time limit).

---

## 📖 Old-engine case study (deleted mechanisms)

The v1.x README once carried a 166-line game analysis detailing how the old
engine came back from behind in 54 moves via "layered threat response →
desperate mode → counter-attack win". **That entire mechanism no longer exists**,
and the original text even warned that it "does not represent the current
version's decision process", so only the conclusions are kept here:

- **Layered threat response** (~130 lines of seven-part heuristic quick answers)
  failed because **its criterion did not cover the cases**: it read
  `opp_win >= 2 or opp_live4 >= 2`, whereas the most common double-threat shape
  in real play is `opp_win == 1 and opp_live4 >= 1`, which falls exactly outside
  it.
- **Desperate mode** (switching to a mixed attack/defence score and scanning for
  the opponent's near-winning points once a loss was judged) failed because **the
  judgement it rested on was itself inaccurate**: it triggered when PVS reported
  `-10,000,000`, and that score **overlapped in range** with the old composite
  static score, so it could not tell "dead in three moves" from "this position is
  pretty bad".

Both share the same root cause: **approximating a decidable problem with a
finite set of rules**. Their replacements are quiescence search (which counts
winning points exactly) and VCF (exact forced sequences, able to distinguish
"there is no mate" from "I could not finish"). The full original analysis is
still retrievable from history:

```bash
git show d232fa7:README.md | sed -n '236,401p'
```

---

## 📝 Changelog

### v2.1.0 (2026-10-02)

> ⚠️ **This update is the last one. Support and maintenance for this legacy
> project stop here.**
>
> This update is a UI refactor aligned with
> [GomokuAI v3.0.5 (Stable)](https://github.com/iamlinxuhan/GomokuAI), aimed at
> bringing it to macOS users. **Linux / Windows users are advised not to
> download this Release's artefacts** — download the latest stable engine
> version of [GomokuAI](https://github.com/iamlinxuhan/GomokuAI) instead.

Not one line of the engine changed — `enhanced == 0` is bit-for-bit identical;
this version moves the interface only.

**Interface**

- 🎨 **The theme toggle was redone**: from a square emoji button to the right of
  the title, to a pill on the **line below** the title — "Current theme (click to
  switch):" + a self-drawn icon. The sun/moon no longer depend on the font's
  emoji (they look different on every platform, and vanish into boxes when the
  glyph is missing); they are now a sun/moon drawn with `QPainterPath` (the sun
  is a solid circle minus eight orbiting small circles, the moon is two circles
  subtracted, colours taken from `theme`).
- 🎨 **The endgame is now a single red line**: the old version drew a full-width
  light band plus a ring around every winning stone, so "where did I win" had to
  be read by counting rings. It is now one red line with a dark backing (its ends
  extending 0.5 / 0.7 cells along the direction, `FlatCap`) lying over the five,
  so the direction is obvious at a glance. **The last-move ring is no longer
  drawn at the endgame** — that ring means "the move just played", whereas at
  this moment the message is "where did I win". The line colour comes from
  `theme._PALETTES["light"]["DANGER"]`: `theme.py` is a frozen single sample, and
  adding a top-level constant would mean editing it, so the table is read
  directly (the two versions' `theme.py` are byte-for-byte identical).
- 🐛 **The endgame overlay is delayed by one second** (`GAME_OVER_DELAY_MS`):
  previously it appeared the instant the stone landed, covering both the "last
  move" ring and the red line above it — the first thing the player saw was "you
  won" rather than "how you won". The timer is single-shot and cancellable, and
  restarting or quitting mid-way calls `stop()`.
- 🐛 **Board coordinate labels are now positioned by ink**: the gap goes from
  `cell × 0.10` to `0.18`, row numbers hug the edge by their ink's right edge,
  and letters share a common baseline. The old code used
  `AlignCenter`/`AlignBottom` with an a-priori `band` height, so a letter with a
  descender like `Q` would push its own ink's bottom edge upward and misalign the
  whole column.
- 🎨 **The two charts now say whose they are**: `AI score` →
  `AI (black/white) score` (the game panel tells the chart which side the AI
  plays, and the tooltip says so too), and `Board win rate (est.)` →
  `Human win rate (est.)`. The conversion logic and the numbers are unchanged to
  the character — only the titles — because the two names previously required the
  reader to infer them from context.
- 🐛 **The difficulty card's seconds are now read from `engine.DIFFICULTY`**: the
  old code hard-coded `3/7/15` on the cards, which happened to match the real
  values in the engine. That was pure coincidence — changing the table would
  raise no test alarm, and the symptom would be "the UI says 9 seconds while the
  AI actually thinks for 20". The three levels' names and colours are still
  local (the legacy `engine.DIFFICULTY` has no `name` field, and **adding a field
  to that table would be changing the difficulty table**, which is outside
  "move the interface only").
- 🐛 **The strength bar picks its stone colour by theme**: the stone material on
  the board was tuned against **wood**, and on a dark card the black stone has
  only 1.11:1 contrast, so white is used to stay legible. The stones indicate
  only "how many" (the strength), not "which side" — the colour-selection page
  and the panel's turn indicator still show black and white faithfully.

**Packaging: new macOS artefacts**

- ✨ **The Release gains `.dmg` / `.pkg` for both macOS arm64 and Intel**, 4
  artefacts in total. This is the reason this legacy version exists — the engine
  is pure Python and depends on no platform-specific binary, whereas that C++
  engine ships only Windows / Linux binaries. The two architectures are **each
  built on a native runner**, not as universal2 (PyQt5's Qt has only
  per-architecture wheels, so a dual-architecture `.app` cannot be assembled),
  and `platform.machine()` is asserted against the runner label first —
  an artefact does not prove its own architecture, and a label quietly
  repointed cannot be detected afterwards.
- 📝 **The minimum OS version is macOS 14 (Sonoma)**, and that number **is
  decided by the build machine**: what CI measured (the highest `minos` across
  every binary inside the `.app`) comes from numpy's `_multiarray_umath...so`
  requiring 14.0, because pip on the macOS 15 runner picked the
  `macosx_14_0` wheel rather than `macosx_11_0`. The main executable itself asks
  for only 11.0 / 10.13. Pushing it lower would require pinning numpy's version,
  which conflicts with "do not pin versions", so it was not done — instead the
  number is printed in CI and stated clearly in the README.
- 📝 The `.ico` is converted to `.icns` in CI (the source is the Windows
  256×256 single image), and `CFBundleName` / `CFBundleDisplayName` are set to
  「五子棋AI」 to match Windows/Linux, after which **the ad-hoc signature is
  redone** — Info.plist counts towards the bundle signature, and editing it
  invalidates the signature PyInstaller applied, turning the app into "damaged".
- 📝 **There is no Apple developer signature** (that needs a $99/year account),
  so the first launch needs a Gatekeeper detour. The README's download notes
  describe two ways through (click "Open Anyway" in System Settings, or
  `xattr -dr com.apple.quarantine` from the command line) — the error text reads
  「Apple cannot check it for malicious software」, which is easily mistaken for a
  damaged file. It also notes that **the widely circulated "Control-click →
  Open" shortcut was removed by Apple in macOS 15**, and following it now just
  gets you blocked again.

**Compatibility**

- This update did **not** bring in the C++ version's new "Engine" and "current
  TCP port" rows or their logging: this version has no C++ server and no port
  pool, so they could only be dead rows forever showing "—".
- Difficulty is still **3 levels** (Novice / Intermediate / Advanced, 3 / 7 / 15
  seconds), not expanded to the C++ version's 5.

### v2.0.2 (2026-09-24)

**Engine**

- 🐛 **Intermediate's time limit 5.0 s → 7.0 s**: in a real lost game
  (`game_log_20260922_235240`, Intermediate playing white, human winning on move
  31 at J8) the decisive move is move 4 — with only three stones on the board,
  **ply 4 picks K9 while ply 5 picks K7/K11**, adjacent plies reaching opposite
  conclusions. Pinning that move and replaying the rest with the old budget: K9
  gives **black wins · move 31 J8**, **matching the log move for move (all 31
  moves identical)**; K7 / K11 gives white a counter-win on move 12 / 18. The
  old budget `5.0 × 0.85 = 4250 ms` sat exactly on the tipping point — repeating
  the search on the same input, K7 and K9 **alternate** at 5.0 s (3 out of 12
  idle runs played K9, 0 out of another 8; with 6 saturated processes,
  **4/4 all played K9**), only stabilising at ply 5 from 6.0 s, with 7.0 s
  leaving about 0.85 s of margin on top. **This pushes the horizon farther out;
  it does not cure anything** — see the "Difficulty" section.
- 🐛 **The log no longer reports "I could not finish" as "I finished"**:
  `_vcf_defence` used the same `-1` for both "the candidate set was exhausted,
  there is no blocking point" and "the budget ran out, it was not exhausted", so
  the caller could not tell them apart and both cases logged the same
  `PVS search (opponent has a VCF)`. It is now split into `VCF_DEFENCE_NONE` and
  `VCF_DEFENCE_UNKNOWN`, and `reason` says which. Alongside this, the opponent's
  four-threat depth (`vcf_dist`) is always recorded rather than only when a
  blocking point is found — that was exactly the number missing when diagnosing
  that game.
- ✨ Two new guards:
  `test_difficulty.py::test_mid_level_sees_the_fifth_layer_on_the_pivot_move`
  (`perf` — Intermediate must see ply 5 on that move 4) and
  `test_vcf.py::test_vcf_defence_reports_unknown_when_the_budget_is_gone`.
- ✨ New `tools/pivot_ab.py`: it turns the "this move decides the game" finding
  above into a reproducible comparison — `--budget 5.0` pins K9 / K7 / K11
  respectively and shows the outcome, while `--stability N` repeats the search N
  times on the same input and counts what the engine itself chose ("it flips at
  the tipping point" was measured with exactly this).

**Packaging**

- ✨ **Linux gains no-install executables** `GomokuAI_For_Linux_AMD` /
  `GomokuAI_For_Linux_ARM` (PyInstaller `--onefile`): no extension, `chmod +x`
  and run — no root, nothing written to `/opt`, no install script. **They
  coexist with the existing `.deb` / `.pkg` rather than replacing them** — those
  can declare the graphics libraries and a CJK font candidate chain in
  `Depends:`, whereas a bare file cannot, so it reports whatever is missing (or
  shows a screen full of boxes).
- 📝 The README's "Packaging" section was split into a Windows and a Linux part,
  explaining why Linux's onefile cannot replace the `.deb`, and why the
  executable bit can only be set by the user (Release assets store bytes only,
  and `upload-artifact` does not preserve permissions either — CI cannot
  compensate for this).

### v2.0.1 and earlier

**Kept in Chinese.** The entries from v2.0.1 backwards document mechanisms that
were deleted in v2.0.0 and are no longer part of this codebase (the old GPU
packaging line and the early PyTorch/CUDA iterations), so translating them would
describe software that no longer exists. The full text is in
[README.zh-CN.md](README.zh-CN.md#更新日志).
