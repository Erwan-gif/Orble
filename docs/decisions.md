# Why it's built this way

Orble's decisions are written down as they are made. The private repository holds 30 decision records, each with its context, the decision and what it costs. This page collects the ones that shape the product. The first four, the stack, are older than the records, so they are explained here from what the code asks of them.

- [The stack](#the-stack): [a Rust core](#a-rust-core), [Tauri 2 windows](#tauri-2-not-electron-not-a-swiftui-app), [Swift for one thing](#swift-for-one-thing), [plain JavaScript](#plain-javascript-no-framework-no-build-step)
- [What Orble does, and won't](#what-orble-does-and-wont): [it never drives](#it-teaches-it-never-drives), [it lives in the notch](#it-lives-in-the-notch)
- [Pointing and drawing](#pointing-and-drawing): [macOS measures](#the-model-picks-the-control-macos-says-where-it-is), [the board](#the-board-carries-content-never-coordinates), [live guides](#one-kind-of-guide-planned-live)
- [Models](#models): [Claude by default](#claude-by-default), [a small model decides](#a-small-model-chooses-it-never-generates), [local-first](#local-first-is-a-mode-not-a-fork)
- [Cost and evidence](#cost-and-evidence): [paying for speech](#pay-for-speech-not-silence), [measuring first](#measure-before-adding), [keys in a file](#keys-in-a-file-for-now)

## The stack

### A Rust core

Most of Orble is work nobody sees. A capture loop keeps a rolling buffer of the screen. A microphone thread feeds the listeners, a playback thread plants markers in the audio queue, and a watchdog restarts any of them that stalls. All of it runs all day, next to the apps the person is actually using.

- **It is cheap to keep running.** No garbage collector pauses the audio or the capture loop. When the screen is still and the person is idle, capture slows to one frame every 2 seconds, every 3 on battery, every 5 in Low Power Mode. A frame identical to a recent one is stamped again rather than encoded again.
- **It talks to macOS directly.** The accessibility API, the screen's notch geometry, CoreAudio, ScreenCaptureKit, the power sources and the notification centre are called through Rust's Objective-C bindings, in the same process. There is no bridge to keep in sync.
- **Its parsers are memory-safe.** Orble reads bytes it does not control: model streams, speech services, dropped files, web pages, other apps' accessibility trees. Each parser is also run on a few thousand damaged and hostile inputs by a seeded generator in its tests.
- **Threads share state under the compiler's rules.** One deadlock stayed possible: a window getter waiting on the main thread while holding a lock the main thread takes. The lock's own type now refuses that wait, and a test fails on any window getter called outside the one module allowed to call them.
- **The core can travel.** Control geometry comes from the accessibility layer, behind one module, written so that Windows UI Automation and Linux AT-SPI can stand in for it. Nothing in the updater is specific to a platform.

**What it costs.** Builds are slow and the compiler is strict, which slows down the first draft of anything. Calling AppKit from Rust takes about 120 `unsafe` blocks and some ceremony that Swift would not need.

### Tauri 2, not Electron, not a SwiftUI app

What Orble shows is drawn: brackets, arrows, labels that keep off each other, equations, charts, graphs. HTML, CSS and SVG are good at that, so the windows are webviews. Tauri hosts them in the system's own WebKit and leaves everything else to Rust.

- **No browser in the bundle.** A release build in October was 15 MB.
- **Native windows where it matters.** Tauri hands over the real window, so the island floats at the status bar's level, above the menu bar, and the overlay floats higher still. The overlay is click-through, on every Space and beside full-screen apps.
- **Each window gets only what it calls.** Commands are granted per window, a test checks the grants against what each page invokes, and the content security policy allows no inline script or style.
- **The same pages run in a test.** The overlay and board code runs unchanged in headless Chrome, where 70 board images are compared pixel by pixel. The pictures in this repository were made the same way.

**What it costs.** Two languages, with a boundary between them. To keep both sides honest, every event from the core to a page is defined once, in one Rust table, with its target window as a type and its payload. The page-side list and the documentation are generated from that table. A SwiftUI app would have made the AppKit calls simpler, and the board much harder to lay out and to test.

### Swift, for one thing

Apple's on-device speech recognizer is wrapped by one small Swift program instead of being called from the app. Orble runs it the way it runs whisper.cpp: one clip at a time, or live audio on its input with partial and final lines coming back. It is its own process with on-device recognition forced on, so nothing it hears can reach the network, and a crash in it cannot take the app down. The app still builds on a machine without Swift, and CI builds the bundle both ways.

### Plain JavaScript, no framework, no build step

The island, the overlay, the board and Settings are plain scripts loaded by the webview. Nothing is compiled or bundled, an edit shows up as soon as it is saved, and the pages ship no third-party code.

Every decision a page makes lives in a model file with no DOM. These files decide which panel is open and what closes it, which sentence the caption shows, where a label goes so it covers nothing, and how an arrow bends round another mark. They are tested in Node, 467 tests in all, and the page only wires them to the screen.

**What it costs.** No components and no reactivity: structure is held by convention, lint and tests. One rule came out of that. A panel never manages its own visibility with classes and timers. The first island did, with about eight independent flags, and past answers never went away.

## What Orble does, and won't

### It teaches; it never drives

Orble once had an autopilot that clicked and typed in the person's apps. It was off by default, it carried a large surface (synthesized input, an approval flow, a second way for every app to fail), and it worked against the point: teaching the person to do the thing. It was removed.

Orble points, explains and guides, and the person does every action. Two bounded exceptions remain, and no third will be added. Scribe pastes the person's own dictated words, once, after they let go of its key. The optional pointer tint moves the pointer 1 point and back, so that macOS redraws it in the new colour.

A guide still has to see the person act. It reads the system's click counter and the pointer position, so it needs no event tap and no Input Monitoring permission.

**What it costs.** Doing it for the person would be faster. Teaching them is the point.

### It lives in the notch

Nothing opens in the middle of the screen. The island is one black shape hanging from the camera housing: a stem exactly the notch's width, then a panel that grows downward only when it has something to hold, one panel at a time.

The first version was a fixed 236 × 38 capsule at the top centre. On a real MacBook most of it sat behind the camera housing, invisible and unclickable. The notch's width and the menu bar's height differ between the 14-inch and 16-inch models, and display scaling changes every number. So every number now comes from macOS itself. Without a notch (an external display, an older Mac, the lid closed), the island draws its own.

**What it costs.** The window must be exactly the size of what is drawn, because a transparent window still swallows clicks. macOS also moves windows in the menu bar strip when the Space changes, so the island re-docks itself on every move.

## Pointing and drawing

### The model picks the control; macOS says where it is

A model looking at a screenshot is accurate to about 1 to 2 percent of the frame. On a 14-inch display that is 15 to 30 points: near the right button, not on it. So the model's answer only decides *which* control. Orble hit-tests that point in the accessibility tree, the same way a click would. It takes the control's real frame only if the frame overlaps the guess and is not several times larger, so a panel never swallows a button.

Three rules hold the chain together. Each turn sends exactly one screenshot, labelled with its pixel size and capped so the provider never rescales it. Coordinates are always read against the frame the model saw, never a newer one. And each model points in the format it was trained on. Measured on synthetic screens, Gemini was 100 to 200 pixels off when it gave coordinates as tool arguments. It is accurate only in its own text format, so Gemini describes the target and a separate call finds it.

The accessibility tree is never walked in full while an answer waits. A full walk of an Electron app is about 10,000 round trips and 300 to 500 ms, and under any budget a hint can afford, it returns the menu bar and none of the window.

**What it costs.** Two pointing paths to maintain. Apps with no accessibility tree, such as games, canvases and some pro apps, get only the model's accuracy. To catch a wrong mapping, as opposed to a wrong model, pointing drift is measured per display from every accepted snap.

### The board carries content, never coordinates

An explanation is a title and a list of panels: an equation, a matrix, a chart, a graph, code, a table, a timeline, a tree, and more. Each panel carries the sentence that introduces it. The model sends content and structure only, and the board does all the layout. A model that is weak at coordinates therefore has nothing positional to get wrong. Each panel appears when playback reaches its sentence, through a marker in the audio queue.

The board first floated on the overlay, across the bottom of the screen, and could not be dismissed. The one complaint about it was "the extra window that opens in the middle of my screen", so it now drops out of the notch like everything else.

**What it costs.** A new kind of drawing is a new panel kind in the board and in the tool schema, never a free-form canvas. Anything the model wants to draw has to fit a kind.

### One kind of guide, planned live

A fixed script of steps breaks as soon as the person's screen differs from the script, or the person takes a step their own way. So a guide has one goal and no preset steps. Each step is planned from the screen as it is now and drawn as one mark. A click on the target, a settled change on screen, or the person saying "continue" ends that step, and then Orble looks again. A miss never ends the guide; it becomes a note for the next plan.

**What it costs.** Every step is a model call, which is why a small model races the planner on each step (below). A step is only as good as the model's knowledge of the app.

## Models

### Claude by default

Gemini was the default at first, because it costs about a tenth as much. But it points only through text detection, so accurate pointing took a second call to Claude whenever a Claude key was set. Claude points in the frame's own pixels, so that second call goes away. Claude Sonnet answers and Claude Haiku plans guide steps, and a Gemini key on its own still works.

**What it costs.** Every chat turn is billed at Claude's rate. Accuracy of a mark was judged worth more than the difference.

### A small model chooses; it never generates

Many of Orble's questions are closed ones: is this line said to Orble, does "okay, and then?" mean the next step, which of these 20 listed controls is the step's target. A full model turn for those is slow and expensive. The decision layer sends them to [Jev](https://typesafe.ai), TypeSafe's hosted decision model, which answers typed questions over text in a few hundred milliseconds. It returns a probability for a yes/no question, or one of the options Orble offered, with a confidence.

It never produces coordinates or text the user sees. Every option is built by code first, from the accessibility tree's controls or a closed set of intents. An answer that is not on the list is refused. The position of a chosen control comes from its own accessibility frame.

It can only ever remove a model turn. In a guide, it races the full planner on the same frame. A confident pick draws the step in about 300 ms and the planner's call is dropped. A low confidence, a timeout or an app with no accessibility tree leaves the step to the planner, which is already running, so the race adds no wait.

Measured against the hosted model over 41 calls, after the first cold call: 481 ms at the median and 649 ms at the 95th percentile, no off-list answers, 15 of 17 guide steps and 24 of 24 routing cases right. The threshold for dropping chatter, a probability of 0.38, sits between the measured scores: chatter at 0.36 or below, the lowest real follow-up at 0.43.

The protocol is open to a local model at a loopback address, and that is the only way the layer runs in Privacy Mode. We measured Von, which runs on the Mac and speaks the same protocol. Its guide intents were right at about 230 ms. As a step chooser it was rejected: it leaned towards the last option in a list and took 3 to 5 seconds a request on an M1 Pro. With a local decision model, guides still run in Privacy Mode on accessibility-tree steps. A step that would need a look at the screen is handed to the person: do it your way, then say "next".

An endpoint that is down costs nothing. It is probed in the background, and while it is down, nothing waits for it. Its readiness is judged against each caller's own time budget, because an endpoint that answers in 1.5 seconds is up, and useless to an 800 ms router.

Images were measured before any were trusted. Yes/no questions about the screen were right 16 times out of 16. But 4 of 24 confident boxes missed small targets by 19 to 52 points, so the decision model never places a mark. It is only asked whether a step done by hand looks finished.

**What it costs.** A second model, a second key and a second failure mode. That is why it stands aside on its own, and why every path it sits on works without it.

### Local-first is a mode, not a fork

One switch, Privacy Mode, keeps everything on the Mac. Every model, every ear and every voice must then be at a loopback address. Speech is heard by whisper.cpp or Apple's on-device recognizer. Chat runs on a local model through Ollama, with a vision model that points through the same accessibility snap. The voice is macOS's own or a local speech server. Decisions go to a local model, or to none.

It is the same app with the same rules, not a second product. Orble bundles no speech or language engine: the person installs what they want. A whisper model is downloaded only on the person's click, from a pinned revision, and checked against its published checksum. There is no wake-word model either. The usable pre-trained ones were licensed for non-commercial use only or validated a key online, so the wake word is spotted by the local recognizer, primed with the wake words.

Loopback has to mean loopback. The HTTP library's default once sent `127.0.0.1` through the system proxy, which carried Privacy Mode's local model traffic off the Mac. Every client is now built by one function that exempts loopback.

**What it costs.** Local models are slower and point less well, and some features stand down: the vision planner, web search, cloud voices. Speed was measured instead of assumed. On an M1 Pro, the base English whisper model transcribes a second of speech in 0.17 seconds, model load included, with no errors on clean synthetic English. For French, the small model is the first that hears it well.

## Cost and evidence

### Pay for speech, not silence

The streaming speech service bills for the time its socket is open, not for the speech in it. A socket held open while listening is on bills all day for silence. So a small state machine opens the socket when speech starts, with a short pre-roll so the first word is not lost. It holds the socket through brief pauses, closes it after 60 seconds of quiet, and closes it at once when a call starts. Optionally, the wake word is heard on the Mac first, and only then does anything go to the cloud.

**What it costs.** The first words after a quiet minute wait for the socket's handshake. The pre-roll keeps them, so they arrive a little late rather than not at all.

### Measure before adding

Noise suppression looked like an easy win. Offline, RNNoise cost under 1 percent of a core and silenced steady noise between words, but gained only about 1 dB inside speech. That says nothing about how the recognizers read its output, and Apple's voice processing cannot be measured offline at all. So nothing was added: each microphone's calibrated noise floor handles steady noise, and the live measurement still owed is written down with the design for whichever option wins.

The same rule set the thresholds above. The decision layer's numbers come from replayed fixtures, and barge-in's energy rules write a log line on every decision so they can be tuned on real rooms.

**What it costs.** Some improvements wait for a measurement, and are written down as owed rather than assumed.

### Keys in a file, for now

API keys were kept in the macOS Keychain. An unsigned build gets a new code identity on every rebuild, so macOS asked for the login password at every launch and every settings save. Keys now live in a separate file with owner-only permissions. Settings never hands a stored key back to a page, exports and feedback reports never read the file, and environment variables work as a fallback.

**What it costs.** Any program running as the same user can read that file. The threat model and the privacy notice say so, and the Keychain comes back once releases are signed with a stable identity.
