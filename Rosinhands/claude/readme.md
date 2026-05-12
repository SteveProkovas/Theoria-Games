# 🎻 ROSINHANDS: A Year of First Notes

_A narrative browser game about adult learning, grief, and a 1937 Mittenwald violin._

---

## Table of Contents

- [Quick Start](#quick-start)
- [State Machine](#state-machine)
- [Class & Data Architecture](#class--data-architecture)
- [Practice Mode Deep Dive](#practice-mode-deep-dive)
- [Soundscape Platform System](#soundscape-platform-system)
- [Audio Engine](#audio-engine)
- [Particle System](#particle-system)
- [Input Handling](#input-handling)
- [Screen Fade System](#screen-fade-system)
- [Story Content Flow](#story-content-flow)
- [Scoring & Resonance](#scoring--resonance)

---

## Quick Start

Open `index.html` in any modern browser. No build step, no dependencies.
All assets (audio, visuals) are generated at runtime via Canvas 2D API and Web Audio API.

```
rosinhands/
├── index.html      ← The entire game (single file)
└── README.md       ← You are here
```

---

## State Machine

The game flows through 7 distinct states. Transitions use a crossfade helper (`goTo()` → `tickFade()`) that fills the screen with black at adjustable alpha.

```mermaid
stateDiagram-v2
    [*] --> TITLE
    TITLE --> STORY : click (after frame > 80)
    STORY --> STORY : click (4 panels, then auto-advance)
    STORY --> ATTIC : auto after panel 4
    ATTIC --> JOURNAL : find all 3 items → click button
    JOURNAL --> PRACTICE : click "Begin Practice Session"
    PRACTICE --> SOUND : complete session → click "Enter the Soundscape"
    SOUND --> JOURNAL : collect 3 memories + spirit → "Continue"
    SOUND --> FINALE : if lessonIdx >= 3
    FINALE --> TITLE : "Play Again"
```

**State enum and transition map:**

```mermaid
flowchart LR
    subgraph States
        T["TITLE (0)"] --> S["STORY (1)"]
        S --> A["ATTIC (2)"]
        A --> J["JOURNAL (3)"]
        J --> P["PRACTICE (4)"]
        P --> W["SOUND (5)"]
        W --> |lessonIdx < 3| J
        W --> |lessonIdx >= 3| F["FINALE (6)"]
        F --> T
    end
```

**Key variables:**
| Variable | Role |
|----------|------|
| `GST` | Current state enum value |
| `fadeDst` / `fadeDir` / `fadeA` | Crossfade destination, direction (-1/0/1), alpha |
| `goTo(next)` | Sets fade destination, triggers fade-in |
| `enterState(s)` | Resets all state-specific variables |

---

## Class & Data Architecture

```mermaid
classDiagram
    class Dot {
        +Number x, y, vx, vy
        +String col
        +Number life, ml, r
        +tick()
        +draw()
        +Boolean dead
    }
    class NoteGlyph {
        +String glyph
        +Number rot, rv
        +tick()
        +draw()
    }
    class FallNote {
        +Number lane, delay, dt, pulse
        +Boolean alive, hit, missed
        +String qual
        +tick()
        +inZone()
        +draw()
        +Boolean dead
    }
    class whisperParticle {
        +String msg
        +Number life
        +tick()
        +draw()
        +Boolean dead
    }

    Dot <|-- NoteGlyph : extends
    Dot <|-- FallNote : structural sibling

    class PTCLS ["Particle Pool (Array)"] {
        +push()
        +filter(dead)
        +forEach(tick/draw)
    }
    Dot <-- PTCLS : contains
    NoteGlyph <-- PTCLS : contains
    whisperParticle <-- PTCLS : contains (anonymous)

    class StoryScreen {
        +String text, sub
    }
    class JournalEntry {
        +String week, title, body
    }
    class LessonDef {
        +String title, sub, hints[]
        +Number bpm
        +Array seq (lane indices 0-3)
    }
    class WorldDef {
        +String title, spirit
        +Array frags
    }
    class LaneDef {
        +String name
        +Number x
        +String col
    }
```

**Content arrays** are indexed by `lessonIdx` (0–2, wrapping via `Math.min`):

| Array | Length | Drives |
|-------|--------|--------|
| `STORY_D` | 4 | Story panels (always fixed) |
| `JOURNAL_D` | 3 | Journal entries per lesson |
| `LESSON_D` | 3 | Practice session config per lesson |
| `WORLD_D` | 3 | Soundscape worlds per lesson |
| `LANES` | 4 | Fixed G/D/A/E lane positions |
| `FINALE_L` | 7 | Finale poem lines |

---

## Practice Mode Deep Dive

This is the core gameplay. Players align a horizontal bow with falling notes across 4 string lanes.

```mermaid
sequenceDiagram
    participant Player
    participant Input as Input Handler
    participant Practice as Practice Loop
    participant Notes as FallNote Pool
    participant Audio as Audio Engine
    participant UI as HUD / Critic

    loop Every frame (~60fps)
        Practice->>Notes: tick() all notes
        Notes-->>Notes: advance y, check bounds
        Notes-->>Notes: if y > BOW_BOT+28 → missed
        Practice->>Practice: update bowPhase (pressure sine wave)
    end

    Player->>Input: A/S/D/F or click lane
    Input->>Notes: find closest in-zone note
    Notes-->>Practice: best note + distance from bow

    Practice->>Practice: score quality (perfect/good/poor)
    Practice->>Audio: playString(lane, quality)
    Practice->>UI: update pScore, pCombo, pResonance
    Practice->>UI: update criticSz (± based on quality)
    Practice->>Notes: mark note.hit = true

    Notes-->>Practice: all notes dead?
    Practice->>UI: show results screen
```

**Note quality matrix:**

| Condition | Quality | Points | Resonance | Critic Effect |
|-----------|---------|--------|-----------|---------------|
| `bd < 10` AND `\|pressure-0.5\| < 0.13` | perfect | +10 | +10 | criticSz -15 |
| `bd < 24` AND `\|pressure-0.5\| < 0.27` | good | +6 | +5 | criticSz -7 |
| otherwise hit | poor | +2 | -2 | criticSz +3 |
| missed (fell past zone) | miss | -5 | -7 | criticSz +12 |

**Bow pressure system:**
```
bowPressure = (sin(bowPhase) + 1) / 2   → range [0, 1]

Zone interpretation:
  0.00 – 0.20  → "too light"   (blue)
  0.20 – 0.38  → "light/good"  (green)
  0.38 – 0.62  → "perfect"     (gold)    ← target zone
  0.62 – 0.80  → "heavy"       (orange)
  0.80 – 1.00  → "too heavy"   (red)
```

**Key variables for extending practice:**

| Variable | Meaning |
|----------|---------|
| `bowPhase` | Incremented by `0.024 + lessonIdx * 0.0042` per frame |
| `BOW_TOP` / `BOW_BOT` | Y=442 to Y=498 — the hit zone |
| `FallNote.vy` | `1.88 + lessonIdx * 0.48` — increases with difficulty |
| `criticSz` | Starts at 64, clamped [18, 138], drives critic entity size |
| `pResonance` | 0–100%, affects finale tone |

---

## Soundscape Platform System

A 2D side-scrolling world where the player (Elena) collects memory fragments across musical platforms.

```mermaid
flowchart TB
    subgraph World["Soundscape World (WORLD_D[lessonIdx])"]
        PLAT["Platforms Array (sPLATS)"] --> |type g| Ground["Solid ground blocks"]
        PLAT --> |type s| Staff["Staff-line bridges (♩ decorated)"]
        PLAT --> |type h| Hover["Hovering platforms (glowing, oscillating)"]

        FRAG["Memory Fragments (sFRAGS)"] --> |3 per world| Collect["♬ pickup (r=30)"]
        SPIRIT["Father's Spirit (sSPIRIT)"] --> |activates when sCOLL >= 3| Dialog["Shows spirit text"]
    end

    subgraph Player["Player (SP object)"]
        MOVE["← → / A/D"] --> VX["vx = ±3.25"]
        JUMP["↑ / Space / W"] --> |on ground| VY["vy = -10.4"]
        GRAVITY["vy += 0.4/frame"]
    end

    Collect --> |distance < 30| BURST[burst() + playChime()]
    BURST --> DIALOG[show sDLG for 235 frames]
    SPIRIT --> |distance < 40 + sCOLL >= 3| END[sDONE = true]
    END --> CONTINUE[Show Continue button]
```

**Platform scrolling logic:**
```
if SP.x > 548:  sSCR += SP.x - 548;  SP.x = 548   (push scroll right)
if SP.x < 92:   sSCR = max(0, sSCR - (92 - SP.x))  (push scroll left)
```

All platform X coordinates are in **world space** (`pl.x - sSCR` converts to screen space). The world extends from x=0 to approximately x=2500.

---

## Audio Engine

Lazily initialized Web Audio API. All sounds are synthesized — no audio files.

```mermaid
flowchart LR
    subgraph playString["playString(laneIdx, quality)"]
        FREQ["freqs = [196, 294, 440, 659]<br/>G3 D4 A4 E5"]
        DUR["dur = 0.85 / 0.5 / 0.3"]
        VOL["vol = 0.17 / 0.11 / 0.06"]
        FC["filter cutoff = 1800 / 1100 / 700"]

        FREQ --> H1[Harmonic 1]
        FREQ --> H2[Harmonic 2]
        FREQ --> H3[Harmonic 3]

        H1 --> FLT[Lowpass Filter]
        H2 --> FLT2[Lowpass Filter /2]
        H3 --> FLT3[Lowpass Filter /3]

        FLT --> GAIN[Gain → exp fade]
        FLT2 --> GAIN2[Gain → exp fade]
        FLT3 --> GAIN3[Gain → exp fade]

        GAIN --> OUT[ac.destination]
        GAIN2 --> OUT
        GAIN3 --> OUT
    end
```

**Note:** Each `playString()` call creates 3 simultaneous sawtooth oscillators with harmonic spacing (1×, 2×, 3× the fundamental). Quality affects duration, volume, and filter cutoff simultaneously — creating a brighter, longer, louder tone for "perfect" hits.

`playChime(freq, dur)` is a simpler single sine oscillator used for UI feedback and special moments.

---

## Particle System

All particles live in the global `PTCLS` array and are ticked/drawn every frame.

```mermaid
classDiagram
    direction LR
    class PTCLS["Global Particle Array"] {
        <<Array>>
        filter(not dead)
        forEach: tick()
        forEach: draw()
    }

    class Dot {
        x, y, vx, vy
        col, life, ml, r
        tick(): update position, apply gravity, shrink
        draw(): arc with globalAlpha
        dead: life <= 0 or r < 0.2
    }

    class NoteGlyph {
        extends Dot
        glyph: ♩♪♫♬ (random)
        rot, rv (rotation + velocity)
        tick(): add rotation drift
        draw(): rotated text glyph
    }

    class whisperParticle {
        <<anonymous, created in drawCritic>>
        msg: string
        x, y, vx, vy, life
        tick(): drift upward
        draw(): red italic text
    }

    Dot <|-- NoteGlyph
    PTCLS --> Dot
    PTCLS --> NoteGlyph
    PTCLS --> whisperParticle
```

**Spawn functions:**
| Function | Creates | Used in |
|----------|---------|---------|
| `burst(x, y, col, n, type)` | n Dots or NoteGlyphs | Note hits, attic discovery, spirit encounters |
| `NoteGlyph constructor` | 1 musical glyph | Title screen background (every 15 frames) |

---

## Input Handling

Three input paths converge on gameplay:

```mermaid
flowchart TD
    subgraph Mouse["Mouse Events"]
        MM[mousemove] --> |stores| MOUSE["mouse.x, mouse.y"]
        MD[mousedown] --> PCLICK[pendingClick = true]
        MU[mouseup] --> |clears| mouse.down
        CLICK[click] --> LANECHECK{In PRACTICE?}
        LANECHECK --> |yes| FIND["find closest lane<br/>within 58px"]
        FIND --> STRIKE["strikeNote(laneIdx)"]
    end

    subgraph Keys["Keyboard Events"]
        KD[keydown] --> |stores| KEYS["keys[code] = true"]
        KU[keyup] --> |clears| KEYS["keys[code] = false"]
        KD --> PRACTICE_KEYS{A/S/D/F in PRACTICE?}
        PRACTICE_KEYS --> |yes| STRIKE
        KD --> SOUND_KEYS{Arrow/Space in SOUND?}
        SOUND_KEYS --> |yes| PREVENT[preventDefault]
    end

    subgraph Click["Click Processing"]
        PCLICK --> NEXT_FRAME["jClick = pendingClick<br/>pendingClick = false"]
        NEXT_FRAME --> BTN["btn() helper checks<br/>hover + jClick"]
        NEXT_FRAME --> STATE_CLICK["state-specific click handlers"]
    end
```

**Multi-frame click pattern:** `pendingClick` captures the raw event; `jClick` is set at the start of the next frame and consumed immediately. This prevents double-processing and allows the `btn()` helper to check both hover state and click in the same frame.

---

## Screen Fade System

All state transitions use a 2-phase crossfade:

```
Frame N:   goTo(ST.STORY) → fadeDst = ST.STORY, fadeDir = 1
Phase 1:   fadeA increases (+0.055/frame) until >= 1.0
           → Screen fully black
           → GST = fadeDst assigned
           → enterState(fadeDst) called
           → fadeDir = -1 (start fading back in)
Phase 2:   fadeA decreases (-0.04/frame) until <= 0
           → fadeDir = 0 (idle)
```

The fade rectangle is drawn **after** all game content each frame, ensuring it overlays everything.

---

## Story Content Flow

```mermaid
flowchart TD
    TITLE["Title Screen<br/>+ violin silhouette<br/>+ floating ♩ notes"]

    TITLE --> S1["Story 1: Elena's call from Seville"]
    S1 --> S2["Story 2: Miguel's secret violin, La Voz"]
    S2 --> S3["Story 3: Life had other plans"]
    S3 --> S4["Story 4: She pressed Enter"]

    S4 --> ATTIC["Attic: Find 3 items<br/>• Violin Case<br/>• Papá's Journal<br/>• Old Photograph"]

    ATTIC --> J1["Journal: Week 1 — First Contact"]
    J1 --> P1["Practice: Open Strings (bpm 54)"]
    P1 --> W1["Soundscape: The Open String Sea"]

    W1 --> J2["Journal: Week 4 — The Inner Critic"]
    J2 --> P2["Practice: String Crossings (bpm 65)"]
    P2 --> W2["Soundscape: The Crossing Bridges"]

    W2 --> J3["Journal: Week 9 — Something Opened"]
    J3 --> P3["Practice: Father's Melody (bpm 74)"]
    P3 --> W3["Soundscape: The Memory Garden"]

    W3 --> FINALE["Finale: 7-line poem<br/>→ 'Play Again' returns to Title"]
```

---

## Scoring & Resonance

The Resonance meter (0–100%) is the emotional core of practice mode:

```mermaid
flowchart LR
    PERFECT[Perfect hit] --> |+10| RES["Resonance"]
    GOOD[Good hit] --> |+5| RES
    POOR[Poor hit] --> |-2| RES
    MISS[Missed note] --> |-7| RES
    RES --> AFFECTS["Affects:<br/>• Critic size<br/>• Sofia's hints<br/>• Finale tone<br/>• Burst particle colors"]
```

Resonance never drops below 0 or exceeds 100. The `pct` shown on the results screen is `score / maxScore` (not resonance), determining the 1–3 star rating:

| Stars | pct range | Sofia's quote |
|-------|-----------|---------------|
| ★☆☆ | ≤ 0.5 | "Every scratch is progress." |
| ★★☆ | 0.5–0.78 | "Finding the sound." |
| ★★★ | > 0.78 | "Pure and resonant." |

---

## Extending the Game

**To add a new lesson:**
1. Add an entry to `LESSON_D` with bpm, sequence array, and hints
2. Add matching entries to `JOURNAL_D` and `WORLD_D`
3. The loop will auto-advance through the new lesson

**To add new note judgment tiers:**
Modify the `strikeNote()` function — the `bd` (distance from bow center) and `pq` (pressure deviation) thresholds are the tuning knobs.

**To add new platform types in Soundscape:**
Add entries to `sPLATS` with a new `t` character, then add a rendering branch in `drawSoundscape()`.

**To add new particle types:**
Create a new class following the `Dot` interface (`tick()`, `draw()`, `dead` getter) and push instances to `PTCLS`.

---

## Color Palette Reference

| Token | Hex | Usage |
|-------|-----|-------|
| `P.void` | `#04020c` | Deep background |
| `P.cream` | `#f0dcb4` | Primary text |
| `P.amber` | `#c47828` | UI accents, borders |
| `P.gold` | `#f0a820` | Highlights, perfect hits |
| `P.G` | `#b83820` | G string lane |
| `P.D` | `#c87028` | D string lane |
| `P.A` | `#d89828` | A string lane |
| `P.E` | `#f0be30` | E string lane |
| `P.spirit` | `#c8b8f2` | Father's spirit glow |
| `P.critic` | `#8a1818` | Inner critic entity |

---

_This README was generated for maintainability. The diagrams use Mermaid syntax — they render natively on GitHub, GitLab, and most Markdown viewers._
```

This README gives you:

1. **7 Mermaid diagrams** covering state flow, class hierarchy, practice sequencer, soundscape platformer, audio synthesis chain, input routing, and content progression
2. **Decision tables** for note scoring and pressure zones
3. **Variable reference tables** so you can find the right tuning knob quickly
4. **Extension guide** with concrete steps for adding lessons, note tiers, platform types, and particles
5. **Palette reference** tying every color token to its role

The diagrams will render automatically on GitHub because Mermaid is built into their Markdown renderer. If you need ASCII-only versions for a terminal or plain-text viewer, I can produce those as well.
