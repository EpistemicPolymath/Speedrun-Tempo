# OpenGOAL Speedrun Practice Mod & Rhythm Coach (`goal-tempo`)

An all-in-one, elite training environment for Jak and Daxter speedrunning, built on top of the community's **OG-Speedrun-Practice Mod** [OG-Speedrun-Practice/README.md at main · OpenGOAL-Mods/OG-Speedrun-Practice · GitHub] and expanded with a native, real-time input timing evaluator and auditory metronome [GOAL-Tempo Repository | GitHub].

This project operates deep within Naughty Dog's custom game kernel and virtual machine heap, leveraging native symbols, cooperative multitasking threads, and index-based font overrides [Process and State | OpenGOAL, Type system | OpenGOAL, Font Color Tables | OpenGOAL].

I built this ReadMe and tool by gathering all relevant sources, putting them in Gemini Notebook, and having back and forth discussions with the sources as shared contextual data. I hope to continue to expand this mod as I learn more about OpenGOAL and how it works. My dream was to create a metronome trainer and upon using the OG Speedrun Mod I realized they would work perfectly together. I will keep expanding this project as I continue to learn more.

---

## 🗺️ Table of Contents
- [OpenGOAL Speedrun Practice Mod \& Rhythm Coach (`goal-tempo`)](#opengoal-speedrun-practice-mod--rhythm-coach-goal-tempo)
  - [🗺️ Table of Contents](#️-table-of-contents)
  - [🎮 Section 1: Speedrun Practice Mod Controls](#-section-1-speedrun-practice-mod-controls)
  - [🚀 Section 2: Rhythm Coach \& Metronome Architecture (`goal-tempo`)](#-section-2-rhythm-coach--metronome-architecture-goal-tempo)
    - [1. Heap-Allocated Process State](#1-heap-allocated-process-state)
    - [2. Context-Aware Double \& Multi-Buffering](#2-context-aware-double--multi-buffering)
    - [3. Dual-Feedback Metronome](#3-dual-feedback-metronome)
  - [🎨 Section 3: Authentic HUD Controller Mapping](#-section-3-authentic-hud-controller-mapping)
  - [🏋️‍♂️ Section 4: Advanced Practice Preset Directory](#️️-section-4-advanced-practice-preset-directory)
    - [Preset A: Punch-Roll-Jump (PRJ) \[Default\]](#preset-a-punch-roll-jump-prj-default)
    - [Preset B: Ledge Boosted Jump (Ledge Boost)](#preset-b-ledge-boosted-jump-ledge-boost)
    - [Preset C: Ledge Extended Uppercut](#preset-c-ledge-extended-uppercut)
    - [Preset D: Ground Pound Jump](#preset-d-ground-pound-jump)
    - [Preset E: Rolljump High Jump (Bounce Jump)](#preset-e-rolljump-high-jump-bounce-jump)
  - [🏃‍♂️ Section 5: Standard Daily Boot \& REPL Setup](#️-section-5-standard-daily-boot--repl-setup)
    - [1. Configure Your Personal Profile (`goal_src/user/poly/`)](#1-configure-your-personal-profile-goal_srcuserpoly)
    - [2. Boot Protocol](#2-boot-protocol)
  - [📚 References \& Sources Index](#-references--sources-index)

---

## 🎮 Section 1: Speedrun Practice Mod Controls

The base codebase inherits the robust debugging and practice tools from the **OG-Speedrun-Practice** framework [OG-Speedrun-Practice/README.md at main · OpenGOAL-Mods/OG-Speedrun-Practice · GitHub]. Use these controller combinations in-game to manage your practice space:

| Button Combination | Keyboard Equivalent | Action / Effect |
|--------------------|---------------------|-----------------|
| **Hold L1 + R1 + X** and press **Start** | *Menu Trigger* | Brings up the on-screen speedrunner menu for fast resets and access to custom checkpoints [OG-Speedrun-Practice/README.md at main · OpenGOAL-Mods/OG-Speedrun-Practice · GitHub]. |
| **Hold L2 + dpad Down** | `Ctrl + C` | Set a custom checkpoint at Jak's current position (remembers if you are on FlutFlut or Zoomer) [OG-Speedrun-Practice/README.md at main · OpenGOAL-Mods/OG-Speedrun-Practice · GitHub]. |
| **Hold L2 + dpad Up** | `Ctrl + V` | Reset to last selected skip/trick, or to custom checkpoint, whichever was used most recently [OG-Speedrun-Practice/README.md at main · OpenGOAL-Mods/OG-Speedrun-Practice · GitHub]. |
| **Hold L2 + dpad Left** | `1 + Left Arrow` | Get on FlutFlut, or get off FlutFlut/Zoomer if already mounted [OG-Speedrun-Practice/README.md at main · OpenGOAL-Mods/OG-Speedrun-Practice · GitHub]. |
| **Hold L2 + dpad Right** | `1 + Right Arrow` | Get on Zoomer, or get off FlutFlut/Zoomer if already mounted [OG-Speedrun-Practice/README.md at main · OpenGOAL-Mods/OG-Speedrun-Practice · GitHub]. |

*Note: Ensure **Speedrunner Mode** is toggled to ON in the in-game Miscellaneous Options to enable custom inputs [In-game Settings | OpenGOAL].*

---

## 🚀 Section 2: Rhythm Coach & Metronome Architecture (`goal-tempo`)

The **Rhythm Coach** (`goal-tempo`) is a custom-engineered multitasking process that runs at **60 frames per second (300Hz internal engine ticks)** [Process and State | OpenGOAL, Standard library | OpenGOAL]. It evaluates timing accuracy between frame-perfect state transitions and triggers synchronized, spatialized audio cues to build auditory muscle memory [Process and State | OpenGOAL, Symbol Index | OpenGOAL].

```
  +-------------------------------------------------------------+
  |                   GOAL Kernel Scheduler                      |
  |  Ticks at 300Hz / Runs Process Loops at 60fps               |
  +------------------------------+------------------------------+
                                 | (Cooperative Yield)
                                 v
  +-------------------------------------------------------------+
  |              tempo-coach-process (LISP Heap)                |
  |  - Bypasses 256-Byte Thread Stack                           |
  |  - Tracks Target States via Pointer: (-> *target* state)     |
  +------------------------------+------------------------------+
                                 | (Visual & Audio Feeds)
                                 v
  +------------------------------+------------------------------+
  |           HUD Buffer         |         Audio Engine         |
  |   Prints to *stdcon* buffer  |   sound-play-by-name (300Hz) |
  +------------------------------+------------------------------+
```

### 1. Heap-Allocated Process State
The default OpenGOAL thread scheduler allocates a strict **256-byte stack** to running CPU threads [Process and State | OpenGOAL]. High-level variables declared in standard nested loops easily overflow this boundary [Process and State | OpenGOAL, Language basics | OpenGOAL]. To guarantee 100% stability, our coach is compiled as a custom type `tempo-coach-process` inheriting from the engine's base `process` class [Process and State | OpenGOAL, Type system | OpenGOAL]:
* All timing data, physical input captures, and session stats are stored as **heap fields** rather than stack registers [Process and State | OpenGOAL, Type system | OpenGOAL].
* Field updates use the pointer dereference operator `(-> self field-name)`, bypassing local stack frames completely [Process and State | OpenGOAL, Syntax and examples | OpenGOAL].

### 2. Context-Aware Double & Multi-Buffering
To handle the edge cases of high-speed speedrunning movement, the thread uses dynamic state-buffering:
* **Double-Buffered Uppercuts:** Evaluates both grounded (`target-attack-uppercut`) and aerial (`target-attack-uppercut-jump`) states during lunge-cancels to accommodate 1-tick grounded frames [Process and State | OpenGOAL, Symbol Index | OpenGOAL].
* **Multi-Buffered Landing Frames:** Detects when players carry Left Analog stick momentum upon landing, safely accepting `target-hit-ground`, `target-stance`, or `target-walk` to prevent dropped inputs [Process and State | OpenGOAL, Symbol Index | OpenGOAL].
* **Dynamic Error Gates:** Automatically disables accidental crouch-uppercut error handlers when executing drills that intentionally utilize uppercut transitions [Checkpoint Randomizer | OpenGOAL, Symbol Index | OpenGOAL].

### 3. Dual-Feedback Metronome
* **Visual Metronome:** Renders a real-time bouncing indicator (`*`) across an active HUD track on the connection frame buffer (`*stdcon*`) [Standard library | OpenGOAL, Symbol Index | OpenGOAL].
* **Acoustic Metronome:** Fires a high-pitched downbeat ping (`money-pickup` sound cue) [Symbol Index | OpenGOAL], followed by echoing sub-beat metronome ticks scheduled dynamically to match your personal best execution frame gaps [Standard library | OpenGOAL, Process and State | OpenGOAL].

---

## 🎨 Section 3: Authentic HUD Controller Mapping

To preserve the authentic look of the game, the rhythm coach renders color-coded PlayStation controller symbols directly onto the native text buffer [Font Color Tables | OpenGOAL, Standard library | OpenGOAL]. This is accomplished by leveraging the game engine's internal **Font Color Tables** defined in `font-h.gc` [Font Color Tables | OpenGOAL].

Using the escape command format `~[index]L`, text formatting is instantly mapped to PlayStation hardware color identities [Font Color Tables | OpenGOAL, Standard library | OpenGOAL]:

| Escape Code | Symbol Rendered | Associated Hardware Key | Color ID / Brand Match |
|-------------|-----------------|-------------------------|------------------------|
| `~24L`      | `[Square]`      | Lunge / Punch Attack    | Pink [Font Color Tables | OpenGOAL] |
| `~25L`      | `[Circle]`      | Spin Attack / Glide     | Red [Font Color Tables | OpenGOAL] |
| `~26L`      | `[Triangle]`    | Zoomer Mounting         | Green [Font Color Tables | OpenGOAL] |
| `~27L`      | `[X]`           | Ground Jump / Bouncer   | Blue [Font Color Tables | OpenGOAL] |
| `~22L`      | `[L1] / [R1]`   | Roll / Crouch Bumper    | Dark Gray [Font Color Tables | OpenGOAL] |
| `~0L`       | *Reset*         | Default Font Color      | White [Font Color Tables | OpenGOAL] |

*Example Implementation in GOAL:*
```goal
(format *stdcon* "Input: ~24L[Square]~0L -> ~27L[X]~0L" ) ;; [Standard library | OpenGOAL, Font Color Tables | OpenGOAL]
```

---

## 🏋️‍♂️ Section 4: Advanced Practice Preset Directory

The coach comes packed with five pre-configured training modules [GOAL-Tempo Repository | GitHub]. Each preset targets a specific speedrun movement mechanic by mapping its sequence directly to the internal player state machine [Process and State | OpenGOAL, Symbol Index | OpenGOAL].

### Preset A: Punch-Roll-Jump (PRJ) [Default]
* **Movement Tech:** Drills the timing of your flat-ground lunge cancel to maximize forward velocity [Appendix | OpenGOAL, Symbol Index | OpenGOAL].
* **Target State Sequence:** 
  $$\text{target-running-attack (Punch)} \rightarrow \text{target-wheel (Roll)} \rightarrow \text{target-wheel-flip (Jump)}$$
* **Target Inputs:** `Square` $\rightarrow$ `L1` $\rightarrow$ `X` [Appendix | OpenGOAL]
* **REPL Load Code:**
  ```goal
  (tempo-set-drill-presets "Punch-Roll-Jump (PRJ)" 'target-running-attack 'target-wheel 'target-wheel-flip 'square 'l1 'x)
  ```

### Preset B: Ledge Boosted Jump (Ledge Boost)
* **Movement Tech:** Slow-walk off a high ledge, executing a punch and jump cancel at the absolute last frame of ground contact to gain massive horizontal distance [Appendix | OpenGOAL, Checkpoint Randomizer | OpenGOAL].
* **Target State Sequence:** 
  $$\text{target-running-attack} \rightarrow \text{target-attack-uppercut-jump} \rightarrow \text{target-attack-air}$$
* **Target Inputs:** `Square` $\rightarrow$ `X` $\rightarrow$ `Circle` *(Gap 1: 1–2 frames, Gap 2: delayed glide)*
* **REPL Load Code:**
  ```goal
  (tempo-set-drill-presets "Ledge Boosted Jump" 'target-running-attack 'target-attack-uppercut-jump 'target-attack-air 'square 'x 'circle)
  ```

### Preset C: Ledge Extended Uppercut
* **Movement Tech:** Initiate a lunge punch on flat ground, carry your momentum past a cliff edge, and double-tap jump and spin in mid-air to clear massive gaps [Appendix | OpenGOAL, Symbol Index | OpenGOAL].
* **Target State Sequence:** 
  $$\text{target-running-attack} \rightarrow \text{target-attack-uppercut-jump} \rightarrow \text{target-attack-air}$$
* **Target Inputs:** `Square` $\rightarrow$ `X` $\rightarrow$ `Circle` *(Gap 1: extended slide, Gap 2: rapid double-tap)*
* **REPL Load Code:**
  ```goal
  (tempo-set-drill-presets "Extended Uppercut" 'target-running-attack 'target-attack-uppercut-jump 'target-attack-air 'square 'x 'circle)
  ```

### Preset D: Ground Pound Jump
* **Movement Tech:** High-jump, execute a dive, and tap jump the exact frame of landing to translate vertical slam momentum into a high bounce [Appendix | OpenGOAL, Symbol Index | OpenGOAL].
* **Target State Sequence:** 
  $$\text{target-jump} \rightarrow \text{target-flop} \rightarrow \text{target-flop-hit-ground}$$
* **Target Inputs:** `X` $\rightarrow$ `Square` $\rightarrow$ `X`
* **REPL Load Code:**
  ```goal
  (tempo-set-drill-presets "Ground Pound Jump" 'target-jump 'target-flop 'target-flop-hit-ground 'x 'square 'x)
  ```

### Preset E: Rolljump High Jump (Bounce Jump)
* **Movement Tech:** Execute a rolljump and tap jump on the exact frame you contact the ground to perform a frame-perfect bounce jump [Appendix | OpenGOAL, Symbol Index | OpenGOAL].
* **Target State Sequence:** 
  $$\text{target-wheel-flip} \rightarrow \text{target-hit-ground} \rightarrow \text{target-high-jump}$$
* **Target Inputs:** `L1` $\rightarrow$ `X` $\rightarrow$ `X`
* **REPL Load Code:**
  ```goal
  (tempo-set-drill-presets "Rolljump High Jump" 'target-wheel-flip 'target-hit-ground 'target-high-jump 'l1 'x 'x)
  ```

---

## 🏃‍♂️ Section 5: Standard Daily Boot & REPL Setup

Because OpenGOAL uses a stateful compiler, you must load the game's core type definitions and kernel symbols into memory before compiling your user mod [Language basics | OpenGOAL, Type system | OpenGOAL]. Follow this optimized daily routine to boot up and automatically connect your coach:

### 1. Configure Your Personal Profile (`goal_src/user/poly/`)
Isolate your files inside a custom user folder to keep the root directory clean [Language basics | OpenGOAL]:
*   **`goal_src/user/user.txt`** — Contains exactly: `poly`
*   **`goal_src/user/poly/user.gc`** — *(Keep empty to avoid offline load race conditions)* [Language basics | OpenGOAL]
*   **`goal_src/user/poly/startup.gc`** — Automates the compile and load order [Language basics | OpenGOAL]:
    ```goal
    (mi)                    ;; 1. Automatically build/reload the game on REPL startup
    (lt)                    ;; 2. Automatically listen/connect to the game client
    ;; og:run-below-on-listen
    (ml "goal_src/user/poly/goal-tempo.gc") ;; 3. Compile and load our coach only after connecting!
    (tempo-start)           ;; 4. Immediately boot the coach HUD!
    ```

### 2. Boot Protocol
1.  **Start the Stateful REPL (Terminal 1):**
    ```bash
    task repl
    ```
    *This automatically compiles the base game engine and starts listening on an open communication port [Language basics | OpenGOAL, GitHub - open-goal/jak-project: Reviving the language that brought us the Jak & Daxter Series · GitHub].*
2.  **Launch the Game Client (Terminal 2):**
    ```bash
    task boot-game
    ```
    *The split-second the window boots, Terminal 1 will connect, compile `goal-tempo.gc`, load the compiled bytes natively into the game's memory, and start rendering the Rhythm Metronome HUD automatically [Language basics | OpenGOAL, Process and State | OpenGOAL]!*

---

## 📚 References & Sources Index

All implementation details, native types, and low-level specifications utilized in this mod are strictly grounded in the official OpenGOAL documentation and project repositories:

1.  **[Appendix | OpenGOAL](https://opengoal.dev/docs/developing/modding_examples/appendix)** — Structural index mapping custom mod behaviors, pause menus, and character/target overrides.
2.  **[Blindfold Assist | OpenGOAL](https://opengoal.dev/docs/developing/modding_examples/blindfold_assist)** — Analog stick input configurations and native camera state events (`camera-master`).
3.  **[Checkpoint Randomizer | OpenGOAL](https://opengoal.dev/docs/developing/modding_examples/checkpoint_randomizer)** — In-depth guide on continuing, warp triggers, and resolving softlocks during level loads.
4.  **[Font Color Tables | OpenGOAL](https://opengoal.dev/docs/reference/color_table)** — Escape command index and color registers for native string formatting.
5.  **[GOOS | OpenGOAL](https://opengoal.dev/docs/reference/goos)** — Syntax and specifications of GOAL's compile-time macro language.
6.  **[GitHub - OpenGOAL-Mods/OG-Mod-Base: OG Mod Base · GitHub](https://github.com/OpenGOAL-Mods/OG-Mod-Base)** — Base template and automated release actions used for OpenGOAL mod packaging.
7.  **[GitHub - OpenGOAL-Mods/OG-Speedrun-Practice: OG Speedrun Practice Mod · GitHub](https://github.com/OpenGOAL-Mods/OG-Speedrun-Practice)** — Active repository containing state-resets, custom checkpoints, and practice bindings.
8.  **[GitHub - open-goal/jak-project: Reviving the language that brought us the Jak & Daxter Series · GitHub](https://github.com/open-goal/jak-project)** — Core compiler and decompiler source code.
9.  **[In-game Settings | OpenGOAL](https://opengoal.dev/docs/usage/settings/)** — Engine documentation outlining Speedrunner Mode limits and graphic settings.
10. **[Language basics | OpenGOAL](https://opengoal.dev/docs/reference/language_basics)** — Compiler states, top-level expressions, and S-expression structures.
11. **[Method System | OpenGOAL](https://opengoal.dev/docs/reference/method_system)** — Virtual method dispatching, vtable hierarchies, and special `_type_` overrides.
12. **[Package Index | OpenGOAL](https://opengoal.dev/docs/source-docs/jak1/package-index/)** — Physical file layout conventions for `goal_src/`.
13. **[Process and State | OpenGOAL](https://opengoal.dev/docs/reference/process_and_state)** — Low-level specifications of threads, stack limits, and state machines.
14. **[Reader | OpenGOAL](https://opengoal.dev/docs/reference/reader)** — Float, integer, symbol, and reader macro parsing.
15. **[Standard library | OpenGOAL](https://opengoal.dev/docs/reference/lib)** — Common execution blocks (`begin`, `block`), conditions, and `format` escape codes.
16. **[Symbol Index | OpenGOAL](https://opengoal.dev/docs/source-docs/jak1/symbol-index/)** — Comprehensive engine symbol directory.
17. **[Syntax and examples | OpenGOAL](https://opengoal.dev/docs/reference/syntax)** — Reinterpretation casts, memory arrays, and heap object construction (`new`).
18. **[Type system | OpenGOAL](https://opengoal.dev/docs/reference/type_system)** — Concrete specifications of the parent-child type hierarchy.
