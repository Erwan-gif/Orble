# Orble

Orble is a tutor that lives in the MacBook's notch. Ask it about anything on your screen, by voice or by typing. It answers out loud, puts a mark on the exact control you need, and draws the explanation when words are not enough. It never clicks for you: it shows you, and you do it.

<p align="center">
  <img src="media/buddy.svg" width="128" alt="Orble's buddy at rest: a small black bun with two soft ears, an ice-blue rim, ice-blue eyes with black pupils looking at you, and two small stubs for arms.">
  <br>
  <img src="media/buddy-going-home.svg" width="100%" alt="Eight frames of the buddy going home: he rises from the screen towards the notch, his top flattens along its edge and widens into a line, his body narrows into a U and his eyes slide up into two D shapes, until he is Orble's mark.">
  <br><sub>Orble's buddy, a black bun with soft ears, lives under the notch: he listens while you talk, thinks while Orble thinks and goes to each mark it draws, and at home he becomes Orble's mark.</sub>
</p>

<!-- demo: the video link goes here -->
<p align="center">
  <img src="media/ask.gif" alt="A question asked out loud in Blender's Rendering workspace. Orble's island grows out of the notch and listens while the words appear: My render takes forever. What should I change? It thinks, then says: You're rendering on the CPU at 3,000 samples, with caustics on. Three numbered marks land one after another: the Device menu, labelled Pick GPU Compute; the Render samples field, labelled Lower it to 512; and the two caustics checkboxes, labelled Turn both caustics off." width="100%">
  <br><sub>Asked out loud in Blender: “My render takes forever. What should I change?”</sub>
</p>

It is a native macOS app written in Rust with Tauri 2. The source is private while it gets ready for release. This repository explains how Orble works and shows what it draws.

## What it does

**It points at the real control.** Ask where a setting is, and Orble puts brackets on the control itself, with a short label beside it. The model only decides *which* control you need. Where the app exposes its accessibility tree, the position comes from macOS, which knows each control's frame to the point.

<p align="center">
  <img src="media/scene-devmode.jpg" alt="Figma in Dev Mode. The island says: Dev Mode has already written this frame's CSS. Copy it with the button beside Layout. Brackets mark the small copy button beside Layout, labelled Copy the layout CSS, with a magnified lens of the button below it." width="100%">
  <br><sub>“How do I copy the CSS for this?” A button too small to see gets a magnified lens beside it.</sub>
</p>

**It walks you through a task, in any app.** Give it a goal and it plans one step at a time from what is on screen now: it marks the next control, waits for your click, then looks again. There are no scripts written per app. If you go your own way, it re-plans from where you are. A step that moves something, a layer into a group or a piece to its square, gets a movement arrow instead of brackets.

<p align="center">
  <img src="media/scene-tracking.jpg" alt="Blender's Motion Tracking workspace during a live guide. The island shows only its lip. Brackets mark the vertical Solve tab at the window's left edge, labelled Open the Solve tab." width="100%">
  <br><sub>A live guide toward “solve the camera”. The step's words sit beside its control, and a click on it moves the guide on.</sub>
</p>

**It draws the explanation.** Equations, matrices, charts, geometry, graphs, code, tables, timelines and trees appear on a board that drops out of the notch. Each panel appears on the sentence that introduces it, and equations are typeset.

<p align="center">
  <img src="media/explain.gif" alt="A finance homework question in Preview. Asked: How do I get the inventory days? The board drops out of the notch, titled Inventory days, with a typeset equation: inventory over cost of goods sold divided by 365 equals 90 over 560 divided by 365, about 58.7 days, captioned So the closest choice is b, 59 days. On the page, the cost of goods sold is underlined and the inventory figure of 90 is circled, then choice b is bracketed." width="100%">
  <br><sub>“How do I get the inventory days?” The board shows the working; the marks show where each number comes from.</sub>
</p>

**Hold to talk, circle to ask.** Hold the right <kbd>⌘</kbd>, ask, let go: Orble hears you only while you hold it. Move the pointer around anything as you hold it, and your question is about what you circled; the model sees your circle.

**Scribe, for voice typing.** Hold <kbd>⌃</kbd><kbd>⇧</kbd><kbd>Space</kbd>, talk, let go. The text is shaped for where it lands: a list in Notes, a single line in a terminal. It never adds anything you did not say.

**Capture pauses by itself.** Screen capture stops while a password manager, a finance app or a meeting app is in front, and while another app is sharing your screen, and the island says why. Screen frames stay in memory and never touch the disk.

<details>
<summary>Two more examples, in Figma and Blender</summary>
<p align="center">
  <img src="media/scene-export.jpg" alt="A Figma design file with three phone screens. The island says: Select the screen, then add an export at the bottom of the Design panel. Numbered brackets mark the Florian screen, labelled Select the Florian screen, and the Export row of the Design panel, labelled Add an export, then pick 3x, joined by a thin line." width="100%">
  <br><sub>“How do I export this screen at 3x?” Steps are numbered, and a line joins them in order.</sub>
</p>
<p align="center">
  <img src="media/scene-subdiv.jpg" alt="Blender's modifier panel. The board hangs from the notch, titled Subdivision levels, with the equation F n equals F 0 times 4 to the n, captioned: Each level splits every face into four, so Render at 3 has four times the faces of Viewport at 2. Brackets mark the Render level field, labelled Raise Render to 3." width="100%">
  <br><sub>“Smoother in the final render, but keep the viewport fast?” A mark and the board in one answer.</sub>
</p>
</details>

<sub>How these pictures were made: real screenshots of Blender, Figma and a homework sheet (each © its owner), shown at one point per pixel on a mock desktop. The island, the marks and the board are drawn by Orble's own overlay and board code, and the animations are that code captured frame by frame. The targets were placed by hand for the picture. In the app they come from the model, snapped to the accessibility tree where the app exposes one.</sub>

## How pointing lands where it should

A model looking at a screenshot is accurate to about 1 to 2 percent of the frame. On a 14-inch MacBook that is 15 to 30 points: close to the right button, but not on it. Orble uses the model's answer to choose the control, then asks macOS where that control actually is.

```mermaid
flowchart TD
    F["Your question, with one screenshot<br/>labelled with its exact size"] --> M{"Claude"}
    M -- "a box in that frame's pixels" --> G["The model's guess"]
    M -- "only a description,<br/>such as a guide step's target" --> L["A grounding call boxes it in pixels<br/>and checks it zoomed in"]
    L --> G
    G --> H["Hit-test the guess<br/>in the accessibility tree"]
    H -- "overlaps, plausible size" --> S["Draw the control's exact frame"]
    H -- "nothing plausible" --> V["Draw the model's box"]
```

Three rules make this work:

1. **One screenshot per turn, sent at a size the API will not rescale.** The model's pixel coordinates then map one to one onto the screen. They are always resolved against the frame the model saw, never a newer one.
2. **The model points in the frame's own pixels.** A model points best in the format it was trained on, and Claude's is pixel numbers passed as tool arguments, so every pointing tool takes pixels in the frame Claude was shown. A target known only by its description, such as a guide step's, is boxed in pixels by a second call and checked zoomed in.
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
        D["Decision layer"]
    end
    subgraph Mac["On the Mac"]
        LS["Apple's recognizer or whisper.cpp"]
        MV["The macOS voice"]
        LD["A compatible decision model"]
    end
    subgraph Service["Through Orble's service"]
        CL["Claude: answers, pointing, planning"]
        AS["AssemblyAI: streaming speech"]
        JV["Jev: decisions"]
    end
    C --> R
    A --> R
    D --> R
    R <--> CL
    A <--> AS
    A <--> LS
    D <--> JV
    D <--> LD
    R --> X
    R --> T
    T --> MV
    R -- "events" --> I
    R -- "events" --> O
    P -- "commands" --> R
```

The full tour, including the threads and what each window may call, is in [docs/architecture.md](docs/architecture.md).

## Why it's built this way

- **A Rust core, with Tauri windows.** Capture, listening and playback run all day beside your real work, so they must be cheap and must not pause. Rust calls the accessibility API, CoreAudio and ScreenCaptureKit directly. The windows are webviews in the system's own WebKit, because brackets, labels, equations and charts are what HTML and SVG do well, and the same code runs in headless Chrome for the snapshot tests.
- **It never clicks for you.** An autopilot existed and was removed. Orble's job is to teach you to do the thing.
- **The model chooses; macOS measures.** The model's box only decides which control you mean. The accessibility tree supplies its real frame.
- **A small model makes the small decisions.** Is that line said to Orble? Which of these listed controls is the next step? How hard does this question need thinking about? [Jev](https://typesafe.ai), a hosted decision model, answers in under half a second (481 ms at the median, measured in October), picking only from options the code built. On a guide step it races the full planner, so it can only remove a model call. A compatible model running on the Mac can take its place.
- **No keys, and no model to pick.** People should not have to choose a model or bring their own keys, so the Orble you download reads none. A grown-up connects the Mac to an Orble account once, and Orble's own service carries every model, speech, voice and decision call.

Each decision, and what it costs, is in [docs/decisions.md](docs/decisions.md).

## Engineering

| | |
| --- | --- |
| Code | 180 Rust modules, about 220,000 lines including their tests; about 100 UI scripts, no build step |
| Tests | About 3,700 Rust tests, 1,900 UI tests in Node and 800 Python tests for the tooling |
| Snapshots | 227 board images drawn in headless Chrome and checked against their pixels, and the 3D models drawn live in software WebGL; 18 prompt snapshot files, so any change to what a model is told fails a test until the diff is read and logged |
| Security | A threat model covering 54 outbound data flows and 16 threat classes, with 139 numbered residual risks. A CI check fails if it cites a function, test or file that no longer exists |
| CI | Formatting, Clippy with warnings as errors, tests on Apple Silicon and on Intel under Rosetta, the minimum Rust version (1.88), cargo-deny for advisories and licences, the release bundle with and without Swift, and a copy signed with a throwaway identity |
| Decisions | 33 written decision records, from "Orble teaches; it never drives the Mac" to "Nothing waits on the main thread under a lock the main thread takes". The main ones are summarised in [docs/decisions.md](docs/decisions.md) |
| Languages | Nine for the interface and for what Orble says: English, French, German, Spanish, Italian, Brazilian Portuguese, Japanese, Simplified Chinese and Korean. It also answers in Dutch and Russian |

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

Orble has 2,258 commits made between 14 August and 23 November 2026. I designed it and directed the work. Most of the code was written by AI coding agents (Claude Code), several at a time, each in its own git worktree. My part was the product, the specifications, the order of work, the rules the code must keep, and deciding what ships. Every batch is read by independent review agents and must pass the full test suite before I merge it.

The project's instruction file keeps the hard-won rules, so the same mistake is not made twice. Two examples: "A transparent window still swallows clicks", and "Frames arriving don't prove Screen Recording".

## Status

Pre-release: a free pilot opens around 1 December 2026. Orble runs on macOS 13 and later, on Apple silicon and Intel. Releases are signed with Orble's own identity but not notarised by Apple, so the first time you open Orble, macOS asks you to choose Open Anyway; updates then arrive by themselves and keep its permissions.

There is no key to bring and no model to pick. A grown-up connects each Mac to its Orble account by approving a short code, or scanning its QR code, on their phone. From then on every model, speech, voice and decision call goes through Orble's own service.

- **Answers:** Claude Opus 5.5, in fast mode. The decision layer judges how much thought each question needs (quick, standard or deep) and sets Claude's effort to match. Claude Haiku 5.5 does background work, such as shaping Scribe's text and planning guide steps, and never answers a question.
- **Listening:** AssemblyAI's streaming recognizer, through the service. Apple's on-device recognizer, or whisper.cpp where it is installed, listens on the Mac instead when the person's language needs it or the service is unavailable.
- **Voice:** the Mac's own voice. The service can also carry an ElevenLabs voice, for a plan that includes one.
- **Decisions:** TypeSafe's Jev, through the service, or a compatible model running on the Mac.

## Licence

Copyright © 2026 Erwan Dilliers. All rights reserved. This repository is published for review only; see [LICENSE](LICENSE).
