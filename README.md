# Orble

Orble is a tutor that lives in the MacBook's notch. Ask it about anything on your screen, by voice or by typing. It answers out loud, puts a mark on the exact control you need, and draws the explanation when words are not enough. It never clicks for you: it shows you, and you do it.

<!-- demo: the hero GIF and the video link go here -->
<p align="center">
  <img src="media/scene-render.jpg" alt="Blender's Rendering workspace on a Mac. Orble's island hangs from the notch and says: You're rendering on the CPU at 3,000 samples. Switch the device to the GPU, then lower the samples. Numbered brackets mark the Device menu, labelled Pick GPU Compute, and the Render samples field, labelled Then lower it to 512." width="100%">
  <br><sub>“My render takes forever. What should I change?”</sub>
</p>

It is a native macOS app written in Rust with Tauri 2. The source is private while it gets ready for release. This repository explains how Orble works and shows what it draws.

## What it does

**It points at the real control.** Ask where a setting is, and Orble puts brackets on the control itself, with a short label beside it. The model only decides *which* control you need. Where the app exposes its accessibility tree, the position comes from macOS, which knows each control's frame to the point.

<p align="center">
  <img src="media/scene-devmode.jpg" alt="Figma in Dev Mode. The island says: Dev Mode has already written the CSS for this frame. Copy it with the button beside Layout. Brackets mark the small copy button beside Layout, labelled Copy the layout CSS, with a magnified lens of the button below it." width="100%">
  <br><sub>“How do I copy the CSS for this?” A button too small to see gets a magnified lens beside it.</sub>
</p>

**It walks you through a task, in any app.** Give it a goal and it plans one step at a time from what is on screen now: it marks the next control, waits for your click, then looks again. There are no scripts written per app. If you go your own way, it re-plans from where you are.

<p align="center">
  <img src="media/scene-layers.jpg" alt="Figma's layers panel. The island says: Drag Follow onto Nav in the layers list, and it moves inside Nav. A ring circles the Follow layer and an arrow leads up to brackets on the Nav layer, labelled Drag Follow onto Nav." width="100%">
  <br><sub>A step that moves something gets a movement arrow: a ring on what moves, brackets where it goes.</sub>
</p>

**It draws the explanation.** Equations, matrices, charts, geometry, graphs, code, tables, timelines and trees appear on a board that drops out of the notch. Each panel appears on the sentence that introduces it.

<p align="center">
  <img src="media/scene-homework.jpg" alt="A finance homework question in Preview. The board hangs from the notch, titled Inventory days, with three numbered steps: a day's cost of goods sold, 560 divided by 365, is 1.534 million; inventory days, 90 divided by 1.534, is 58.7; the closest choice is b, 59 days. On the page, the cost of goods sold is underlined, the inventory figure of 90 is circled, and choice b is bracketed." width="100%">
  <br><sub>“How do I get the inventory days?” The board shows the working; the marks show where each number comes from.</sub>
</p>

**Circle to ask.** Hold <kbd>⌃</kbd><kbd>⌥</kbd> and move the pointer around anything. Your next question is about what you circled, and the model sees your circle.

**Scribe, for voice typing.** Hold <kbd>⌃</kbd><kbd>⇧</kbd><kbd>Space</kbd>, talk, let go. The text is shaped for where it lands: a list in Notes, a single line in a terminal. It never adds anything you did not say.

**Privacy Mode.** One switch keeps everything on the Mac: speech recognition with whisper.cpp or Apple's on-device recognizer, chat and pointing through local models in Ollama, a local decision model, and the system voice or a local speech server. Capture also pauses by itself for password managers, finance apps, meeting apps and screen sharing.

<details>
<summary>Two more examples, in Blender</summary>
<p align="center">
  <img src="media/scene-subdiv.jpg" alt="Blender's modifier panel. The island says: The Subdivision modifier keeps a separate level for the render. Raise Render, and leave Viewport as it is. Brackets mark the Render level field, labelled Raise Render to 3." width="100%">
  <br><sub>“Smoother in the final render, but keep the viewport fast?”</sub>
</p>
<p align="center">
  <img src="media/scene-tracking.jpg" alt="Blender's Motion Tracking workspace. The island says: Your tracks are in. Open the Solve tab, then press Solve Camera Motion. Brackets mark the vertical Solve tab, labelled Open the Solve tab." width="100%">
  <br><sub>“I've tracked my points. How do I solve the camera?”</sub>
</p>
</details>

<sub>How these pictures were made: real screenshots of Blender, Figma and a homework sheet (each © its owner) on a mock desktop, with the island, the marks and the board drawn by Orble's own overlay and board code. The targets were placed by hand for the picture. In the app they come from the model, snapped to the accessibility tree where the app exposes one.</sub>

## How pointing lands where it should

A model looking at a screenshot is accurate to about 1 to 2 percent of the frame. On a 14-inch MacBook that is 15 to 30 points: close to the right button, but not on it. Orble uses the model's answer to choose the control, then asks macOS where that control actually is.

```mermaid
flowchart TD
    F["Your question, with one screenshot<br/>labelled with its exact size"] --> M{"Chat model"}
    M -- "Claude: a box in pixels" --> G["The model's guess"]
    M -- "Gemini: a description" --> L["A grounding call finds it"]
    L --> G
    G --> H["Hit-test the guess<br/>in the accessibility tree"]
    H -- "overlaps, plausible size" --> S["Draw the control's exact frame"]
    H -- "nothing plausible" --> V["Draw the model's box"]
```

Three rules make this work:

1. **One screenshot per turn, sent at a size the API will not rescale.** The model's pixel coordinates then map one to one onto the screen. They are always resolved against the frame the model saw, never a newer one.
2. **Each model points in the format it was trained on.** Measured on synthetic screens with targets at known positions, Gemini was 100 to 200 pixels off when it gave coordinates through tool arguments. It is accurate only in its own `box_2d` text format. So Gemini describes the target, and a separate call finds it.
3. **The operating system has the last word.** A hit is accepted only if it overlaps the guess and is at most 2.5 times its size. A miss keeps the model's box, so this step can only improve a mark.

More in [docs/engineering.md](docs/engineering.md#pointing).

## How the board keeps time with the voice

The model sends an explanation as a list of panels, each with the sentence that goes with it. Panels carry content and structure only, never coordinates: the board does all the layout. A model that is weak at coordinates therefore has nothing positional to get wrong.

```mermaid
sequenceDiagram
    participant Model
    participant Core as Rust core
    participant Voice
    participant Queue as Playback queue
    participant Board
    Model->>Core: the explanation streams in
    Core->>Core: a panel closes and is checked on its own
    Core->>Voice: its sentence, with the panel attached
    Voice->>Queue: a marker, then the sentence's audio
    Queue-->>Board: playback reaches the marker
    Board->>Board: the panel appears as the sentence starts
```

Speech is synthesized ahead of playback, so a timer would drift. Instead, a zero-length marker is placed in the audio queue right before each sentence's samples, and the panel is shown when playback reaches it. If you interrupt, the queued markers are dropped with the audio.

<p align="center">
  <img src="media/board-matrix.png" alt="Board panel: a 4 by 4 upper-triangular matrix with its diagonal entries highlighted, captioned Multiply the highlighted diagonal" width="420">
  <img src="media/board-graph-cycle.png" alt="Board panel: a state machine graph from Idle through Listening and Thinking to Speaking, with links back" width="420">
</p>
<p align="center">
  <img src="media/board-figure.png" alt="Board panel: a 3-4-5 right triangle with labelled vertices and sides" width="420">
  <img src="media/board-chart-line.png" alt="Board panel: a line chart of capture latency against frame width for two series" width="420">
</p>

More panels and the rules behind them are in [docs/engineering.md](docs/engineering.md#the-board).

## Architecture

```mermaid
flowchart TB
    subgraph Windows["Webviews"]
        I["Island: hangs from the notch"]
        O["Overlay: one per display, click-through"]
        P["Settings"]
    end
    subgraph Core["Rust core"]
        C["Screen capture ring"]
        A["Microphone and speech to text"]
        R["Orchestrator: context, tools, turns"]
        X["Accessibility tree"]
        T["Voice"]
        D["Decision layer: Jev, or a local model"]
    end
    subgraph Models["Models"]
        CL["Claude, the default"]
        GE["Gemini"]
        OL["Ollama on the Mac"]
    end
    C --> R
    A --> R
    D --> R
    R <--> Models
    R --> X
    R --> T
    R -- "events" --> I
    R -- "events" --> O
    P -- "commands" --> R
```

The full tour, including the threads and what each window may call, is in [docs/architecture.md](docs/architecture.md).

## Why it's built this way

- **A Rust core, with Tauri windows.** Capture, listening and playback run all day beside your real work, so they must be cheap and must not pause. Rust calls the accessibility API, CoreAudio and ScreenCaptureKit directly. The windows are webviews in the system's own WebKit, because brackets, labels, equations and charts are what HTML and SVG do well, and the same code runs in headless Chrome for the snapshot tests.
- **It never clicks for you.** An autopilot existed and was removed. Orble's job is to teach you to do the thing.
- **The model chooses; macOS measures.** The model's box only decides which control you mean. The accessibility tree supplies its real frame.
- **A small model makes the small decisions.** Is that line said to Orble? Which of these listed controls is the next step? [Jev](https://typesafe.ai), a hosted decision model, answers in about half a second (481 ms at the median), picking only from options the code built. It races the full planner and can only remove a model call. A compatible local model can take its place, and it is the only kind used in Privacy Mode.
- **Local-first is a mode, not a fork.** Privacy Mode is the same app with every model, ear and voice at a loopback address.

Each decision, and what it costs, is in [docs/decisions.md](docs/decisions.md).

## Engineering

| | |
| --- | --- |
| Code | 124 Rust modules, about 129,000 lines including their tests; 50 UI scripts, no build step |
| Tests | 2,288 Rust tests, 467 UI tests in Node, 346 Python tests for the tooling |
| Snapshots | 170 checks over 70 board images rendered in headless Chrome; 18 prompt snapshot files, so any change to what a model is told fails a test until the diff is read and logged |
| Security | A threat model covering 48 outbound data flows and 15 threat classes, with 65 residual risks each tracked as an issue. A CI check fails if it cites a function, test or file that no longer exists |
| CI | Formatting, Clippy with warnings as errors, tests on Apple Silicon and on Intel under Rosetta, the minimum Rust version (1.88), cargo-deny for advisories and licences, the release bundle with and without Swift |
| Decisions | 30 written decision records, from "Orble teaches; it never drives the Mac" to "Nothing waits on the main thread under a lock the main thread takes". The main ones are summarised in [docs/decisions.md](docs/decisions.md) |
| Languages | English and French, for the interface and for what Orble says |

Some of the problems behind those numbers, in [docs/engineering.md](docs/engineering.md):

- [Pointing that lands on the control](docs/engineering.md#pointing), and why the accessibility walk never goes deep at answer time
- [Panels that land on their words](docs/engineering.md#the-board)
- [Drawing while the answer is still being written](docs/engineering.md#streaming)
- [A window that hangs from the notch on every Mac](docs/engineering.md#the-island)
- [Interrupting a voice the microphone can hear](docs/engineering.md#barge-in)
- [Not paying for silence](docs/engineering.md#cost)
- [Privacy that is checked](docs/engineering.md#privacy)
- [Testing an app no test can launch](docs/engineering.md#testing)

## How it was built

Orble has 1,091 commits made between 14 August and 4 November 2026. I designed it and directed the work. Most of the code was written by AI coding agents (Claude Code), several at a time, each in its own git worktree. My part was the product, the specifications, the order of work, the rules the code must keep, and deciding what ships. Every batch is read by independent review agents and must pass the full test suite before I merge it.

The project's instruction file keeps the hard-won rules, so the same mistake is not made twice. Two examples: "A transparent window still swallows clicks", and "Frames arriving don't prove Screen Recording".

## Status

Pre-release. Orble runs from source on macOS 13 and later. There is no signed public build yet. It uses Claude Sonnet 5.5 by default, with Claude Opus 5.5, Gemini and local Ollama models also supported. AssemblyAI and ElevenLabs are optional for streaming speech and voices, and TypeSafe's Jev for the decision layer. Each service uses your own key, and Privacy Mode needs none.

## Licence

Copyright © 2026 Erwan Dilliers. All rights reserved. This repository is published for review only; see [LICENSE](LICENSE).
