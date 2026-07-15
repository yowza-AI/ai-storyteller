# AI Storyteller: Personalized Bedtime Stories with Persistent Characters

> **Problem**: Generic bedtime stories are boring. Kids want *their* story, with *their* characters, told in *their* voice—every night.

**AI Storyteller solves this** with a local multi-agent writing system that generates personalized stories on-demand, maintains a consistent cast of characters across stories, and narrates them with per-character voices.

Press a button. Get a custom bedtime story in 5 minutes. Same characters sound the same every night.

---

## The Vision

Bedtime should be magical, not scripted. A parent can say:
- "Tell me a Fantasy story about Emma"
- "Make it spooky but not scary (PG)"
- "Use the characters we met last week"

The system writes it, narrates it with Emma's own voice, and saves the story for later.

---

## How It Works

```
User Input
├─ Child's name
├─ Genre (Fluffy / Fantasy / Sci-Fi / Suspense / Horror)
├─ Content rating (G / PG / PG-13 / R)
└─ Cast preference
    ↓
[Writers' Room] — Multi-Agent System
├─ Writer Agent
│  └─ Generates initial story (~1500 words, ~5 min read)
│     Respects genre style and rating ceiling
│  ↓
├─ QA Agent (Iterative)
│  └─ Checks story for:
│     • Age-appropriateness (respects ceiling)
│     • Engagement (flow, pacing, cliffhangers)
│     • Continuity (existing characters consistent)
│     ↓ (up to 2 revision rounds)
│
├─ Manager Agent (Final Gate)
│  └─ Approves story + confirms rating
│     Decides: ready to cast or needs rewrite
│  ↓
└─ Casting Agent (Character Designer)
   └─ Analyzes new characters introduced
      Designs each: persona, backstory, voice profile
      Maps to Qwen's 9-voice preset palette
      Persists characters in database (for reuse)
    ↓
[Scriptify] — Playwright Format
├─ Split story into per-line dialogue
├─ Attach character, emotion, resolved voice
├─ Save as ordered script
    ↓
[Narration] — Local TTS
├─ Per-line clip: character voice + emotion + text
├─ Concatenate clips in story order
├─ Output: single audio file (character-consistent throughout)
    ↓
[Output]
├─ Playable story (audio + text transcript)
├─ Saved to library (for continuity cues in future stories)
└─ Character roster updated (same voice every appearance)
```

---

## Key Features

### 1. **Multi-Agent Writers' Room**

Not just "write a story." Multiple agents with distinct roles:
- **Writer**: Crafts narrative
- **QA/QC**: Ensures age-appropriateness, iterates
- **Manager**: Final approval gate (quality + safety)
- **Casting**: Character design + voice assignment

Why multiple agents?
- Quality: QA catches tone/pacing issues before narration
- Safety: Rating enforcement has human-in-the-loop (Manager)
- Consistency: Casting agent ensures characters are described before voicing

### 2. **Persistent Characters**

Stories are not one-offs. Characters recur with the **same voice** every appearance.

Database tracks:
- `character_name`
- `voice_preset` (mapped to Qwen's 9 presets)
- `persona` + `backstory` (fed to Writer for continuity)
- `ref_audio_path` (optional: for voice cloning)

Example:
- Story 1: "Princess Luna" → voice = `Aria` (preset), backstory = "kind but clumsy"
- Story 2: "Luna returns as a guide" → system pulls same voice + backstory
- Kids hear the *same voice* for Luna in both stories (voice consistency is the magic)

### 3. **Genre + Rating Control**

Not just "write a story for a 5-year-old."

**Genres** (style templates):
- **Fluffy**: whimsical, silly, light-hearted
- **Fantasy**: magical worlds, quests, wonder
- **Sci-Fi**: future tech, space exploration, discovery
- **Suspense**: mystery, mild tension, "what happens next?"
- **Horror**: spooky but age-gated (G=charming-creepy, PG-13=genuinely eerie)

**Ratings** (content ceiling):
- **G**: no violence, no scary stuff, jokes + wonder
- **PG**: minor peril (lost, small scary moment), resolved happily
- **PG-13**: genuine tension, mild danger, stakes matter
- **R**: intense, real danger, mature themes (rare for bedtime, but available)

Example: "Horror+G" = charming-spooky (friendly ghost in a silly house), not terrifying.

### 4. **Local-First Approach**

- **No cloud**: runs on local hardware
- **No surveillance**: stories generated + stored locally
- **Fast**: Ollama (local LLM) + Qwen3-TTS (local TTS)
- **Offline-capable**: once models are downloaded, no internet needed

### 5. **Character Voice Cloning (Future)**

For characters that don't map to Qwen's 9 presets (e.g., warm female English), voice cloning is built into the schema but not yet wired up. Future enhancement: record a short voice sample → system clones that voice for future stories.

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **LLM (Story Writing)** | Ollama + Qwen 2.5 7B | Local, fast, good narrative quality |
| **TTS (Narration)** | Qwen3-TTS 1.7B | Local TTS, per-emotion voice control |
| **Backend** | Python + FastAPI | Simple, fast, single-file-runnable |
| **Database** | SQLite | Local, no setup, full-text search for continuity |
| **Frontend** | Vanilla HTML/JS | No build step, instant UI |
| **Voices** | Qwen's 9 presets | Diverse + consistent (same preset = same character) |
| **Audio** | Soundfile (WAV) | Simple, no transcoding overhead |

---

## Architecture Highlights

### Multi-Agent Reasoning with Iteration

The QA agent can iterate up to 2 rounds, talking *back* to the Writer:

```
Writer: [generates story v1]
  ↓
QA: "Pacing is slow in Act 2. Also, rating ceiling is PG but you have a scary scene. Revise."
  ↓
Writer: [generates story v2, faster pacing, toned-down scary]
  ↓
QA: "Better. Approve for Manager."
  ↓
Manager: [confirms age-appropriate, quality OK]
  ↓
Casting: [designs new chars]
  ↓
[Script + TTS]
```

Why iterate?
- First-draft stories often miss tone or pacing
- QA loop catches issues before expensive TTS synthesis
- User experience: waits 5 min for a *good* story, not 2 min for a mediocre one

### Character Persistence as Design Pattern

The system treats characters as *persistent entities*, not generated strings:

```
characters table:
  id: 1
  name: "Princess Luna"
  first_appearance_story_id: 1
  voice_preset: "Aria"
  persona: "kind but clumsy, learning to be brave"
  backstory: "youngest princess of the Crystal Kingdom"
  voice_instruct: "warm, slightly playful, magical"

stories table:
  ...
  character_roster_ids: [1, 3, 5]  ← reused from prior stories
```

When Writer generates a new story:
1. Fetch existing character roster
2. Inject personas + backstories into prompt
3. Writer can reuse characters (same voice, consistent personality)
4. New characters created via Casting agent

This is non-trivial architecture: requires linking characters → voices → audio clips in order.

### Per-Line Emotion Control

Each line of dialogue has an explicit `emotion` field:

```
story_lines:
  line_id: 1
  character: "Luna"
  emotion: "excited"
  text: "What's that magical glow in the forest?"
  speaker: "Aria"
  voice_instruct: "warm, slightly playful, magical + excited"
  audio_path: "data/audio/story_1_line_1.wav"
```

When TTS is invoked:
```
for line in story_lines:
  full_instruction = line.voice_instruct + " + " + line.emotion
  audio = tts(text=line.text, instruction=full_instruction, speaker=line.speaker)
  save(audio, path=line.audio_path)

concatenate(all_audio_paths) → final_story.wav
```

Emotions shape voice inflection: "excited" = higher pitch, faster; "sad" = lower, slower.

---

## For Technical Interviews

**What to look for in code review:**

1. **Multi-agent orchestration** (Writer → QA → Manager → Casting)
   - How does iteration work? (QA feedback loop)
   - How does state flow between agents?
   - How are failures handled (QA rejection, Manager disapproval)?

2. **Character persistence** (voice continuity across stories)
   - How does the system map characters → voices → audio?
   - What happens if a character is re-introduced? (same voice, same tone)
   - Edge case: character appears in story 1 and story 3, skips story 2 (OK? needs reminder?)

3. **Per-emotion TTS** (voice modulation)
   - How is emotion embedded in the TTS instruction?
   - Does emotion modify pitch/speed/tone meaningfully? (or just cosmetic in prompt?)
   - What if emotion is impossible (e.g., "shocked + sleepy")?

4. **Rating enforcement** (safety gate)
   - Who validates the rating? (QA? Manager? Both?)
   - What happens if a story violates the ceiling? (rewrite or reject?)
   - How are edge cases handled (e.g., "mildly scary scene but rated G")?

5. **Offline-first design** (no cloud, models cached)
   - What breaks if Ollama is down? (graceful degradation? silent fail?)
   - TTS model is 4.5GB. How is download handled? (progress indicator? resumable?)
   - GPU memory constraints (Turing cards): how are models placed across GPUs?

**Interview questions:**

- "Why have both a Writer and a QA agent? Why not just have Writer output the final story?"
- "A character's voice preset is `Aria`, but a future story needs a male voice for that character. How does the system handle it?"
- "Rating enforcement: what's the difference between QA validating the rating vs. Manager? Why two gates?"
- "The TTS model is 4.5GB. First story generation takes 2 min. What's the user experience like?"
- "A kid says 'I don't like Luna's voice.' Can you change it? How?"

---

## Full Code

**This repository contains architecture and design philosophy.** Full source code is private.

For technical interviews, I can grant read-only access:
- Complete multi-agent writers' room (Writer, QA, Manager, Casting agents)
- Character persistence layer (database schema + queries)
- Per-line narration pipeline (emotion + voice mapping)
- TTS integration (Qwen3-TTS 1.7B setup + GPU placement)
- FastAPI backend with SQLite
- Frontend UI (vanilla JS)
- Genre/rating templates

**Contact**: yow.stephend@gmail.com

---

## Development Approach

Architected and designed by me. Implementation accelerated with Claude as an AI design/coding partner.

This demonstrates: building a *multi-agent system where agents have distinct roles and iterate toward quality*. Most AI products are "call the LLM once and hope." This one is "call multiple agents, let them argue, then produce a polished output."

---

**Status**: MVP complete (local LLM + TTS working, character persistence implemented)  
**Hardware**: Tested on dual RTX 2080 SUPER (Turing), 32GB RAM  
**Next**: Voice cloning for custom character voices (schema ready, TTS wiring pending)
