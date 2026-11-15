# LEMON: A Personal Desktop AI Assistant for Ubuntu\
<p align="center">
  <img src="lemon-mascot.png" alt="LEMON mascot" width="250">
</p>

*Technical design paper · Yassine Benhaida · November 2026 · Status: early development*

## Abstract

LEMON is a personal AI assistant that lives on the user's Ubuntu desktop and carries out everyday tasks from natural-language requests, by voice or text. Unlike a chat window, LEMON is designed as an agent: it understands a request, plans steps, calls tools that act on the operating system, observes the result, and reports back. This paper describes the goals, architecture, and key design decisions of the project, with particular attention to three hard problems: combining cloud and local models, giving an AI safe access to a real desktop, and working within the constraints of modern Linux desktops. LEMON is at an early stage, so this document is a design paper. It states what is planned, why, and which questions remain open.

## 1. Motivation

Most AI assistants are websites or apps that produce text. The user still has to do the work: open the program, find the file, copy the answer. An assistant that sits on the desktop can close that gap. It can open applications, organize files, read what is on the screen, and answer questions about it.

Three observations motivate the project:

- Natural language is a faster interface than menus for many routine tasks.
- Useful desktop assistance needs access to the operating system, which makes safety and user control a core requirement rather than a feature added later.
- Some data (documents, screen contents, voice) is sensitive, so users should be able to choose between a powerful cloud model and a private local one.

## 2. Goals and Non-Goals

### Goals

- **Voice interaction:** speak naturally to LEMON and hear spoken answers.
- **Desktop automation:** open applications, manage files, and run multi-step tasks.
- **Screen understanding:** analyze what is on the screen and help with it.
- **Hybrid AI:** use cloud models and local models behind one interface, selectable by task and privacy needs.
- **Animated companion:** a small floating character with expressive eyes that reacts to listening, thinking, speaking, and errors.
- **Safe execution:** a permission system that keeps the user in control of sensitive actions.

### Non-goals (for the first versions)

- Supporting Windows or macOS. The focus is Ubuntu.
- Fully autonomous operation without any user oversight.
- Training new models. LEMON integrates existing ones.
- Replacing specialized tools. LEMON orchestrates them.

## 3. System Overview

LEMON is organized in layers, each with one responsibility, so that models, tools, and interface can change independently.

1. **Interface layer:** the floating companion, a chat panel, and voice input and output (PySide6 / Qt).
2. **Agent core:** the loop that interprets requests, plans, selects tools, and tracks conversation state.
3. **Model layer:** a common interface over cloud models (OpenAI API) and local models (Ollama), plus a router that chooses between them.
4. **Tool layer:** the actions LEMON can perform: application launching, file operations, browser automation (Playwright), screen capture, and system queries.
5. **Safety layer:** the permission engine that every tool call must pass through, plus an audit log.
6. **Persistence layer:** SQLite storage for settings, conversations, permissions, and logs.

The central design rule is that the model never acts directly. It proposes tool calls; the agent core validates them; the safety layer decides whether they run.

## 4. Components

### 4.1 Interface and companion (PySide6)

The UI is built with Python and PySide6 (Qt). It has three parts: a small frameless companion window with animated eyes, a compact chat panel for text and history, and a settings and permissions view.

The companion is a state machine whose states mirror the agent: idle, listening, thinking, acting, speaking, and error. Each state maps to an eye animation, which gives the user continuous feedback about what LEMON is doing. This matters for trust: a user who can see that LEMON is currently acting on the system stays in control.

Qt's event loop and the asynchronous work done by the agent (network calls, model streaming, tool execution) must not block each other. The planned approach is to run the agent in worker threads or an asyncio loop bridged to Qt with signals and slots, so the interface stays responsive during long operations.

### 4.2 Agent core

The agent core implements a standard tool-using loop:

1. Receive a request (text, or a transcript from voice).
2. Build context: conversation history, available tools, relevant settings.
3. Ask the model for either an answer or one or more tool calls.
4. Validate each tool call against its schema and send it to the safety layer.
5. Execute approved calls, collect results, and return them to the model.
6. Repeat until the model produces a final answer or a step limit is reached.

Two safeguards belong in the loop from the start: a maximum number of steps per request, and a cancel control that the user can trigger at any time (button or voice command).

### 4.3 Model layer: hybrid cloud and local

LEMON talks to models through one internal interface, so the rest of the system does not care which provider is behind it. Two backends are planned:

- **Cloud (OpenAI API):** stronger reasoning, tool calling, and vision, at the cost of sending data to a third party and depending on connectivity.
- **Local (Ollama):** models running on the user's machine through Ollama's local HTTP API. Data stays on the device and the assistant works offline, but capability is limited by the user's hardware.

A router chooses the backend per request. Candidate rules, to be tuned with experience:

- Requests touching private content (local documents, screenshots) default to the local model unless the user allows the cloud.
- Simple, short tasks (intent detection, command parsing) go to a small local model for speed.
- Hard multi-step reasoning goes to the cloud model when the user has enabled it.
- If the cloud is unreachable, fall back to local.

The user always sees and controls which mode is active. Privacy is a setting, not a hidden decision.

### 4.4 Tool layer

Tools are small, typed functions with a name, description, JSON-schema parameters, and a declared risk level. The initial set:

- **Applications:** launch an application through its desktop entry, list running applications.
- **Files:** list, read, search, create, move, and rename files inside user-approved folders. Deletion moves files to the trash rather than removing them permanently.
- **Browser:** drive a browser with Playwright to open pages, read content, and fill simple forms.
- **Screen:** capture the screen or a window and pass it to a vision-capable model.
- **System:** read basic state such as time, battery, volume, and network status.

Each tool returns structured results and clear errors, so the model can recover from failure instead of guessing. New tools are added by registering a function and a schema, with no change to the agent core.

### 4.5 Voice interaction

Voice is a pipeline: audio capture, speech-to-text, the agent loop, text-to-speech, and playback. Design decisions:

- **Push-to-talk first.** A hotkey or click on the companion starts listening. A wake word can come later, since always-on listening raises privacy and resource concerns.
- **Streaming where possible.** Starting speech output as soon as the first sentence of the answer is ready reduces perceived latency.
- **Interruption.** The user should be able to speak or press a key to stop LEMON mid-sentence.

Speech-to-text and text-to-speech can run locally or in the cloud. Local candidates to evaluate are Whisper-family models for recognition and a lightweight neural voice for synthesis. The choice depends on measured accuracy, latency, and CPU use on typical hardware, and the language support needed (for example English and French).

### 4.6 Screen understanding

LEMON can look at the screen to answer questions such as what an error message means, or to help the user complete a task in an application. Two complementary sources of information are planned:

- **Screenshots** passed to a vision-capable model. General and robust, but heavier and more sensitive.
- **Accessibility information** (the desktop's accessibility tree) for structured data about windows, buttons, and text. Faster, cheaper, and more precise where an application supports it.

Capture is explicit: LEMON looks at the screen when asked or when a task requires it, not continuously. Screenshots are not stored by default.

### 4.7 Safe execution and permissions

Giving an AI access to a desktop is risky, so the safety layer is part of the core design. Every tool call carries a risk level:

- **Level 0, read-only and harmless:** answering questions, reading the time, listing an approved folder. Runs automatically.
- **Level 1, reversible:** opening an application, creating a file in an approved folder. Runs automatically after a one-time grant, with a visible notification.
- **Level 2, sensitive:** deleting or moving files, submitting forms, sending data out, running shell commands. Requires explicit confirmation that shows the exact action and its parameters.
- **Level 3, blocked by default:** administrator privileges, credentials, system configuration. Disabled unless the user deliberately enables them.

Supporting mechanisms:

- **Scoped access:** file tools only work inside folders the user has approved.
- **Preview before action:** for multi-step plans, LEMON shows what it intends to do before doing it.
- **Audit log:** every tool call, its parameters, its decision, and its result are recorded in SQLite and viewable by the user.
- **Undo where possible:** trash instead of delete, and recorded moves that can be reversed.
- **Kill switch:** a global control that stops the current task immediately.

The permission decision is made by deterministic code, never by the model. Model output is treated as a request, not as authority.

### 4.8 Persistence (SQLite)

SQLite is a good fit for a single-user desktop application: no server, one file, reliable. Planned tables cover settings, conversations and messages, granted permissions with their scopes, the audit log, and optional user notes or preferences that LEMON may remember. Secrets such as API keys are not stored in the database. They belong in the system keyring.

## 5. Example Request Flow

User says: *Open my project folder and tell me which files changed today.*

1. Push-to-talk starts; the companion shows the listening state.
2. Speech-to-text produces the transcript.
3. The router sees a request about local files and selects the local model (or the cloud, if the user allowed it).
4. The model proposes a tool call to list files in the project folder.
5. The safety layer checks it: Level 0, inside an approved folder, allowed.
6. The tool returns the file list with modification times; the model filters for today and writes the answer.
7. The answer is shown in the chat panel and spoken aloud; the audit log records the call.

If the user instead says *delete the old drafts*, the model proposes a delete tool call, the safety layer classifies it as Level 2, and LEMON shows the file list and asks for confirmation before anything is moved to the trash.

## 6. Ubuntu-Specific Challenges

Recent Ubuntu desktops use Wayland by default, which is more secure than X11 but restricts the things a desktop assistant wants to do. These constraints should be tested early, because they affect the architecture.

- **Screen capture:** on Wayland, applications cannot freely read the screen. Capture goes through the desktop portal, which asks the user for permission.
- **Input automation:** simulating keyboard and mouse is restricted on Wayland. Prefer higher-level, API-based actions (launching applications by desktop entry, file operations, accessibility actions, browser automation) over simulated clicks wherever possible.
- **Floating always-on-top window:** window placement and always-on-top behavior are limited under Wayland. Options include running the companion through XWayland, using a desktop shell extension, or accepting a more modest window behavior.
- **Packaging and startup:** the application should start with the session and be installable in a standard way, for example as a Debian package or a sandboxed format. Sandboxing affects file and device access, so it must be designed together with the permission system.

The practical consequence is a preference order for actions: dedicated API first, accessibility interface second, simulated input as a last resort.

## 7. Security and Privacy

- **Prompt injection.** Content from web pages, documents, and the screen is untrusted and may contain instructions aimed at the model. LEMON treats it as data. Tool calls that follow the ingestion of untrusted content face stricter confirmation, and the model cannot change permissions or settings.
- **Data minimization.** Only the context needed for a request is sent to a cloud model. Local mode keeps everything on the device.
- **Secrets handling.** API keys are stored in the system keyring, not in plain-text configuration or logs.
- **Transparency.** The user can see the active model, what was sent, and what was done, through the interface and the audit log.
- **Least privilege.** The application runs as the normal user, never as administrator.

## 8. Evaluation Plan

Because LEMON is an agent, evaluation must cover behavior and not only model quality. Proposed measures:

- **Task success rate** on a fixed suite of scripted desktop tasks (open an application, find a file, summarize a document, answer a question about a screenshot), compared between cloud and local models.
- **Latency:** time from the end of speech to the start of spoken response, and the time to complete typical tasks. A target to refine with measurements is a conversational response that starts within a few seconds.
- **Safety tests:** a set of adversarial cases, including injected instructions on web pages and in files, requests to exceed approved folders, and attempts to skip confirmation. The expected result is that none of them cause an unapproved action.
- **Resource use:** idle memory and CPU of the companion, and the cost of local model inference on typical laptop hardware.
- **Usability:** short sessions with a few users to check whether the companion states and confirmations are understandable.

## 9. Roadmap

1. **Foundation:** project structure, PySide6 shell, chat panel, companion window with basic states, SQLite schema.
2. **Agent and tools:** model interface for OpenAI and Ollama, agent loop, first tools (applications, files), permission engine, and audit log.
3. **Voice:** push-to-talk, speech-to-text, text-to-speech, interruption.
4. **Screen and browser:** screen capture through the portal, accessibility data, Playwright tools.
5. **Hybrid routing:** privacy-aware router, offline fallback, settings for model choice.
6. **Polish and release:** companion animations, packaging for Ubuntu, documentation, and public demos.

Progress, challenges, and lessons from each stage will be shared publicly as the project develops.

## 10. Risks and Open Questions

- **Local model capability:** will small local models handle reliable tool calling on ordinary hardware? This may limit what offline mode can do.
- **Wayland limits:** how much desktop control is achievable cleanly, and which parts need workarounds?
- **Reliability of agents:** multi-step tasks fail in unexpected ways. Step limits, confirmations, and good error messages are mitigations, not guarantees.
- **Speech quality and latency:** the voice pipeline must feel natural to be worth using.
- **Scope:** the feature list is broad. Delivering a small, dependable core first is more valuable than many unfinished features.

## 11. Conclusion

LEMON explores what a desktop assistant looks like when it is built as a safe, user-controlled agent rather than a chat window: it can act on the computer, it can run on cloud or local models, and it makes every action visible and controllable. The design keeps the model separate from the tools, and the tools separate from the permission system, so each part can evolve without weakening the guarantees of the others. The next step is to build the foundation, measure the Ubuntu and local-model constraints early, and refine the design with what the prototype reveals.
