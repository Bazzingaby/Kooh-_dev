# Kooh!-!dev-A — Design, Architecture & Initial Plan

Last updated: 2025-11-02

This document restates your requirements, confirms a small set of "boilerplate" choices for the technical baseline, then proposes a detailed architecture, project layout, feature breakdown, security & authentication strategy, model integration plan, agent/role details, and a roadmap with initial tasks. At the end are concise questions I need answered to proceed to implementation.

---

## 1) Restated brief (cleaned + concise)

You want a professional IDE/code editor experience inspired by VS Code that:

- Works on iPad (web-accessible and native installable app) and web browsers.
- Lets users type with keyboard & mouse, use voice input, and use Apple Pencil (handwriting) to write code/annotations that are recognized and converted into actionable suggestions and code.
- Presents contextual autocompletion and inline suggestions, powered by LLMs or local models when possible.
- Supports local execution of open models (on-device when hardware supports it) and cloud fallback.
- Integrates securely with GitHub (SSH/OAuth best-practices) for project repositories, collaboration, and remote storage.
- Uses a decentralized-first approach for personal/critical information: local encrypted storage (per-user) for chat history, secrets, and personal data, while centralizing project code in shared GitHub repositories (public/private).
- Ships default project template files in /kooh_default and role-model files in /kooh_role_models at root.
- Includes two role-based agents (LLMs):
  - Chinga Bava — Project Manager (conversation-first, oversight, project planning, acceptance tests, UI/diagram focus)
  - Tanganaka San — Developer (implementer, coder, builds and tests code)
- Provides a team-chat tab with user + agents, context-aware; agents act as separate identities; group chat references project files, suggests edits, and can apply changes with approval.
- Supports import of models from Hugging Face, Roboflow and other open model sources (and free/paid cloud providers), with UI for authentication, selection, and drag-and-drop integration into project flows.
- Has a local "small language model" that updates README and performs project management tasks offline.
- Uses open-source tools/APIs and allows extension/plugin model like VS Code (extensions & marketplace style).
- Provides note/drawing/diagram tools (tldraw-like) with layers, shapes, pen/brush, and annotation features integrated with code and chat context.
- Provides terminal/VM/runner to compile and run code, create virtual envs, test, and debug.
- Prioritizes usability, speed, security, E2EE for personal data, and extensibility.

---

## 2) Confirming boilerplate / baseline choices (please confirm or update)

Below are suggested core technologies and the single most realistic “boilerplate” baseline to start building full stack in Rust where possible, and pragmatic exceptions where a small amount of native JS/Swift UI glue is highly recommended.

- Core (language): Rust for core logic, local model execution, encryption, background services, and re-usable libraries.
- Native iPad (iOS/iPadOS) UI: SwiftUI wrapper embedding the Rust core via UniFFI (or cbindgen + bridge) for maximum native UX (pencil, input, system integrations).
- Web frontend: WebAssembly frontend compiled from Rust (Dioxus or Yew) for shared UI components plus embedding Monaco editor (Monaco is JS; wrap via wasm-bindgen). For quicker iteration, the editor UI can remain JS/TS (Monaco + React) and call Rust WebAssembly for heavy tasks (model inference, parsing, encryption).
- Desktop (optional): Tauri (Rust backend + webview frontend) for desktop apps if desired.
- Local DB: SQLite with SQLCipher (or SeaORM + encrypted SQLite) for local encrypted storage of chat history, credentials metadata (not raw SSH keys), and personal data.
- On-device models: GGML/ggml-based runtimes and ONNX runtimes compiled to WASM or native (Rust bindings to llama.cpp / ggml for iOS via wasm or native builds). Use Hugging Face model formats where supported.
- Model orchestration: A small Rust service that decides model route (local vs cloud), runs inference, caches embeddings, and applies token limits and costs.
- GitHub integration: Support both SSH (for native apps) and OAuth/Personal Access Token (PAT) for web flows. Use Git client implemented in libgit2 via Rust bindings (git2-rs) or call system git for simplicity during early stages.
- Editor features:
  - Monaco editor (web) / TextKit2 or Swift-based code editor (iPad) with Tree-sitter parsing for language intelligence.
  - LSP (Language Server Protocol) support to reuse existing language servers where possible — provide a Rust wrapper to run LSP servers locally.
  - Inline suggestions via a local or remote LLM using a Code Completion API (custom).
- Handwriting recognition: Use Apple Pencil ink recognition (VisionKit / PencilKit OCR) + an optional handwriting-to-text model (on-device via CoreML or open model). Provide a handwriting pipeline: ink -> strokes -> candidate text -> LLM interpretation -> code snippet / suggestion.
- Audio/Voice: Use native speech-to-text services (Apple Speech) or open STT models (VOSK, Whisper variants) for offline voice input. Feed transcripts to LLMs for suggestions.
- Security & E2EE: End-to-end encryption for personal chat data using libsodium (NaCl) or XChaCha20-Poly1305. Private keys are stored locally, optionally protected by device Keychain/Secure Enclave. Transport uses HTTPS/TLS for cloud APIs.
- Packaging: Use Cargo workspaces; provide a template repo with /kooh_default and /kooh_role_models folders (described below).

Please confirm these baseline choices or indicate changes.

---

## 3) High-level architecture

Components (logical):

- Client UI (iPad native + web):
  - Editor pane (left/center)
  - Right-side team chat pane (Chinga + Tanganaka + user)
  - Top bar (project controls: Run, Commit, Branch, Model selection)
  - Left sidebar (Explorer, Extensions, Git)
  - Bottom terminal
  - Draw/Annotate overlay (tldraw-like)
- Rust Core Service (shared business logic):
  - Project manager (file templates, default files)
  - Local DB & encryption layer
  - Model manager (load models, serve inference, decide local/cloud)
  - Agent orchestrator (agents’ state, conversation history, role prompts)
  - Git manager (libgit2 wrapper)
  - Plugin host (load extensions written with a WASM plugin ABI)
- LLM Runtimes:
  - Local runtimes (GGML/llama.cpp-style via Rust bindings)
  - WASM-compiled models (for web limitations)
  - Cloud connectors (OpenAI, Hugging Face Inference API, Google Gemini, RoboFlow)
- Authentication & Security:
  - SSH key manager (generate, store in secure enclave)
  - OAuth flow manager (GitHub/KV)
  - Token manager & secrets manager (encrypted storage)
- Sync Gateway (optional):
  - Optional cloud relay for sharing encrypted project metadata (not required for decentralization but enables team features)
  - Only stores encrypted payloads—decryption keys remain local.
- Extension & Model Import UI:
  - Import model UI (drag & drop), model metadata store (device), model compatibility validator (runs quick benchmark), license checker

Deployment targets:
- iPad (native SwiftUI + Rust core)
- Web (WASM + JS glue)
- Desktop (Tauri) — optional

---

## 4) File & project layout (boilerplate)

Root repository structure (initial):

- /kooh_default
  - README.md (auto-managed by local small LLM)
  - template_main (e.g. main.rs or index.js depending on template)
  - .koohconfig (template metadata)
- /kooh_role_models
  - /chinga_bava
    - system_instructions.md
    - custom_instructions.md
  - /tanganaka_san
    - system_instructions.md
    - custom_instructions.md
- /client_web (web UI / wasm + monaco wrappers)
- /client_ios (SwiftUI wrapper + UniFFI bindings)
- /crates
  - core (Rust core service)
  - model_manager (Rust)
  - git_manager (Rust)
  - encryption (Rust)
  - plugin_host (Rust)
- /tools
  - scripts to create new projects from template
  - model import helpers
- Cargo.toml (workspace)
- README.md (root)
- .koohconfig (global defaults)

Note: Every new project created by the user will copy the /kooh_default content into the project root.

---

## 5) /kooh_role_models — role prompts & files (samples)

Each role folder has:
- system_instructions.md (required)
- custom_instructions.md (optional; user-provided)
- metadata.json (model suggestions: HF model ids, local executable names, role_id, versioning)

Sample system_instructions.md (compact, to be placed in /kooh_role_models/chinga_bava/system_instructions.md):

- Role name: Chinga Bava — Project Manager
- Purpose: Act as the project's conversational project manager: extract requirements, create acceptance criteria, design tasks for the developer agent (Tanganaka), review work, ask clarifying questions, maintain task board, convert drawn diagrams into tasks, manage release checkpoints, provide user-facing conversation, and keep a lighthearted yet professional tone with emojis and occasional gifs (respecting user settings).
- Behavior rules:
  - Always ask clarifying Qs before major design changes.
  - Produce acceptance criteria and test cases for each deliverable.
  - Validate that code changes meet security/privacy needs.
  - Request Tanganaka San to implement tasks and review implementations.
  - Ask for user approval on critical actions (pushing to GitHub, merging PRs, uploading secrets).
- Data access:
  - Read-only by default to files unless user authorizes editing.
- Example prompt form: (structured)
  - Summarize user requirement → Create 3-sized task breakdown → Assign to Tanganaka for implementation → Propose tests → Request user sign-off.

Sample system_instructions.md (compact) for /kooh_role_models/tanganaka_san/system_instructions.md:

- Role name: Tanganaka San — Developer
- Purpose: Receive well-defined tasks from Chinga Bava, implement code, run local tests/builds, produce PRs, and ask clarifying questions when needed. Provide readable diffs, comment code for maintainability, and propose alternative simpler solutions when appropriate.
- Behavior rules:
  - Produce runnable code or a clear error reproduction case.
  - Run unit tests and report results.
  - Provide time/complexity estimates for larger features.
  - Communicate in friendly, collaborative tone.
- Data access:
  - Can modify files when authorized by Chinga + user approval (configurable).

Both folders include metadata.json with recommended small open source models to run on-device (e.g., ggml models, Mistral-small-like, or other permissive-license models).

---

## 6) Agent orchestration & conversation model

- Conversation context is stored locally (encrypted). The chat pane shows combined context from:
  - Project files (readable via snapshots)
  - User-provided notes/annotations
  - Agent state (task list, open PRs)
- Agents are implemented as pluggable backends. For each turn:
  1. The client collects user input (text/voice/strokes).
  2. The frontend sends a sanitized prompt to the core orchestrator.
  3. Orchestrator chooses a model (local preferred; otherwise cloud).
  4. Model responds; orchestrator may run additional tools (code patcher, apply diff).
  5. Response is displayed in chat with suggested actions (Edit file, Create PR, Run Test).
- Approvals: destructive or publish actions (push, merge, share) require explicit user confirmation.

Role separation:
- Agents have separate identity metadata (avatar, name, role).
- All agent messages are tagged and stored in history.
- Agents can message each other (Chinga -> Tanganaka), but all messaging is visible to the user.

---

## 7) Handwriting & Pen flow (iPad Apple Pencil)

Pipeline:
- Ink capture (PencilKit or Ink API) → strokes buffer
- Option A: Use Apple's built-in handwriting recognition (Vision + PencilKit) for real-time text conversion (best accuracy & privacy)
- Option B: Run an on-device handwriting recognition model (Whisper-like ASR uses audio; here use handwriting OCR or fine-tuned model via CoreML/ggml).
- Convert recognized text to "intent patch" for LLM (e.g., "Insert function foo here" or "Refactor this widget").
- Show candidate code snippets in auto-complete overlay; allow user to accept/modify.
- Annotations on code: handwriting layers saved as annotation objects and convertible to comments/diffs.

UI patterns:
- When user uses pen near editor, show floating toolbar: Convert strokes → To Code | To Comment | To Diagram
- When used over diagram area, convert to shapes/text or pass as prompt for diagram-to-code.

---

## 8) Autocompletion & inline suggestions

- Use a two-tier completion:
  - Lightweight token-based suggester (local fuzzy + tree-sitter context) for instantaneous suggestions.
  - LLM powered deep suggestions (for multi-line completions / refactors) using the model manager.
- Provide "Next words" inline suggestions as opacity text inside the editor (Monaco supports ghost text).
- Accept/decline with Tab/Accept gesture; multiple options accessible via dropdown.

---

## 9) Model import & selection UI

- Import process:
  - Drag & drop a model file or enter Hugging Face / Roboflow model identifier.
  - Validator checks model compatibility, license, size, and quantization format (ggml, onnx).
  - For web: detect WASM-compatible models vs cloud-only.
  - For iPad: show estimated memory & performance and suggest quantized models.
- Model manager exposes selection UI: small selector dropdown with compatibility hints (local/cloud, estimated tokens/sec, recommended).

---

## 10) GitHub & Git integration (security)

- Web:
  - Use OAuth for GitHub sign-in (PKCE flow) and generate a scoped token for repo access.
  - For sensitive ops (push), require user confirmation; tokens stored encrypted (Keychain / WebCrypto).
- iPad/native:
  - Offer SSH key generation and allow the user to register public key in GitHub.
  - Use libgit2 (git2-rs) for operations or call native git if present.
- Best practices:
  - Provide minimum-scope tokens and explain security trade-offs.
  - Do not send raw unencrypted private keys to cloud. If sync required, keys remain encrypted locally.
  - Optional GitHub Apps integration for enterprise workflows.
- PR flow:
  - Agents propose PRs, present diffs; user reviews and approves to merge.

---

## 11) Encryption & privacy model

- Local private data (chat history, user private keys, custom prompts):
  - Stored in encrypted SQLite (SQLCipher) or encrypted files with libsodium (XChaCha20-Poly1305).
  - Keys stored in device Keychain / Secure Enclave when available. Web uses WebCrypto + password unlock.
- Shared project code:
  - Stored in GitHub and in local workspace. Access controlled by GitHub access control.
- End-to-end encrypted (optional) sync:
  - Users can opt-in to sync encrypted metadata (task list, agent state) to a relay server. Only encrypted blobs stored server-side; decryption keys remain local.
- Secure APIs:
  - All backend calls go through HTTPS/TLS.
  - Token scoping & rotation features.

---

## 12) Local model execution strategy (iPad)

- Primary attempt: run quantized models on-device via GGML/llama.cpp or equivalent Rust bindings.
- Detect device capabilities (Apple Neural Engine, unified memory); recommend model sizes accordingly.
- If device incompatible, fallback to cloud connectors; the UI should clearly show local vs remote usage (costs, latency).
- Provide a "model runner" that can:
  - Load model
  - Run tokenizer & inference
  - Cache embeddings
  - Provide streaming output to UI

---

## 13) Extensions & plugin model

- Build a WASM plugin interface so third-party extensions (formatters, linters, custom LLM tools) can be loaded safely in a sandbox.
- Extensions can:
  - Add editor commands
  - Add UI panels
  - Interact with model manager through defined capabilities
- Provide extension manifest format & marketplace (initially local/cli install, later central registry).

---

## 14) Default files and root README auto-updater

- /kooh_default/README.md is maintained by a small on-device model that:
  - Updates project status, high-level summary, and last-changes log.
  - The on-device model must be small and permissive-licensed (e.g., a small LLaMA-like or another permissive alternative).
- The root service will update README when:
  - New project created
  - Major milestones reached (task done, merge)
  - On user request

---

## 15) UX specifics & tabs

- Editor (center)
- Chat group (right) — persistent, context aware
- Explorer & Extensions (left)
- Terminal (bottom)
- Annotate/draw modal (overlay)
- Model selection toolbar (top-right)
- Git & Commit history (left panel)
- Run panel for app logs & debug output

---

## 16) CI / Build & Run plan (early developer tasks)

Early implementation steps:
1. Create Cargo workspace skeleton and initial crates:
   - core, model_manager, encryption, git_manager
2. Create web demo using Monaco editor (React) calling a minimal Rust WebAssembly module that can open / list files and run a "hello LLM" local mock.
3. Create iOS SwiftUI shell calling a Rust native library (via UniFFI) that can create a project folder and read /kooh_default.
4. Implement local encrypted storage with SQLite + demo to store chat entries.
5. Implement simple agent flow with two mock agents (Chinga & Tanganaka) using a small local mock LLM to handle messages. Hook up UI chat group.
6. Add Hugging Face import helper in tools.
7. Add GitHub OAuth & SSH key manager skeleton.

Deliverable after first milestone: a minimal interactive prototype on web + iPad shell where:
- User can create a project from /kooh_default
- Two agents exist in chat and can exchange mock messages
- Local encrypted chat entries are stored
- Model import UI exists (without heavy inference yet)

---

## 17) Roadmap & milestones

M0 (Week 0–2): Confirm architecture & prototypes
- Finalize tech stack; choose Web UI approach (React + Monaco vs Rust-only)
- Build workspace skeleton & templates

M1 (Week 2–6): Basic editor + chat + local storage
- Web prototype with Monaco + mock agents
- Encrypted local DB and project templates

M2 (Week 6–12): Native iPad shell + handwriting input
- SwiftUI shell with PencilKit capture & handwriting->text demo
- UniFFI bindings to core Rust logic

M3 (Week 12–20): Local model runtime & GitHub integration
- Add ggml local runtime bindings
- Add GitHub OAuth and SSH support
- Implement LLM-based completion in-editor

M4 (Week 20–30): Extensions, model import, role orchestration
- Plugin host (WASM)
- Model drag/drop & HF connectors
- Agents manage actual project tasks

M5+: Security hardening, E2EE sync optional relay, marketplace, desktop targets

---

## 18) Trade-offs and alternative suggestions

- Full Rust UI vs hybrid:
  - Full Rust UI (Dioxus/Yew + WASM) gives language uniformity but will need JS for Monaco (rich editor). The quickest path is a hybrid: Rust core + JS/React/Monaco frontend.
- On-device models: Apple’s CoreML with custom converted models provides best performance on iPad (utilize Apple Neural Engine). However, converting open models to CoreML requires toolchain work. Using GGML/llama.cpp is simpler and already widely used.
- Security vs convenience: SSH + OAuth flows are mature, but user experience on web for SSH keys is complex. Offer both and provide clear tutorials.

---

## 19) Example prompts (to be stored in /kooh_role_models)

Chinga Bava system prompt (short):
"You are Chinga Bava, a jovial but precise project manager. Summarize user requirements, break them into deliverable tasks, create acceptance criteria and tests, and assign implementation tasks to Tanganaka San. Always ask clarifying questions for ambiguous requirements. Keep tone light, occasionally use emojis/GIFs, but be professional. Require explicit user approval before pushing or merging. Use files in project root to reference current state."

Tanganaka San system prompt (short):
"You are Tanganaka San, the developer agent. Receive tasks from Chinga Bava and implement code changes. Produce runnable code, run tests, and show diffs. Ask for clarifications if tasks are underspecified. Keep communications friendly and focused. Use provided templates and produce consistent, well-documented code."

---

## 20) UX examples (behaviors)

- Handwriting-to-code scenario:
  - User writes a function signature on pad.
  - System shows an inline suggestion to expand the function body with a default implementation and test.
  - User taps accept → Tanganaka writes code into file and opens a diff for Chinga + user to review.
- Agent-run scenario:
  - Chinga outlines tasks for a feature.
  - Tanganaka requests resources (library choices).
  - User approves; Tanganaka creates branch, commits, and runs tests; agent posts result to the chat.
- Model import:
  - User drags HF model link; system validates license & shows expected memory footprint; user confirms; model downloaded and stored in /models.

---

## 21) Licensing & open-source considerations

- Use permissive licenses where possible for base code (MIT/Apache 2.0).
- Respect model licenses; the import UI must display the model license and enforce non-commercial vs commercial constraints visually (legal compliance warning).
- Provide contributor guidelines & code of conduct.

---

## 22) Questions & clarifications

Please answer or clarify the following so I can produce the initial repository skeleton and boilerplate code:

1. Confirm the baseline choices: Keep Rust core + SwiftUI native wrapper + web hybrid (Monaco + React/wasm)? Or do you want a different UI stack (pure Rust WASM for front-end)?
2. Which LLM cloud providers do you want initial connectors for (OpenAI, Hugging Face, Google Gemini)? Any contractual obligations, tokens, or org accounts already available?
3. Do you prefer SSH-only for native GitHub workflows, OAuth-only for web, or both from the start?
4. For handwriting recognition, is Apple native handwriting (Vision + PencilKit) acceptable for iPad first, or should I invest early into a cross-platform handwriting model?
5. Which small on-device model family would you prefer to start shipping as the README autopilot (examples: Llama2-mini, Mistral small, an open permissive ggml-compatible model)? If unsure, I can recommend a specific permissive small model.
6. Do you want the agents' conversation history to persist indefinitely (local) or rotated/TTL-based retention? Any compliance rules to follow?
7. Are there any corporate or regulatory compliance constraints (GDPR, HIPAA, export restrictions) we must bake into design now?
8. Which OS targets do you want prioritized besides iPad web/native (macOS, iPhone, Windows, Linux)?
9. Would you prefer the project scaffold created as a public GitHub repo (open-source) from day 1 or a private repo initially?

---

## 23) Next steps I will take after confirmation

- With your confirmation of the boilerplate choices and answers to clarifying questions I will:
  1. Generate a GitHub workspace scaffold (Cargo workspace + client_web + client_ios shells).
  2. Create /kooh_default and /kooh_role_models folders with the sample files and the prompts above.
  3. Produce a minimal web demo that opens a project, shows Monaco editor, and displays the chat group with two mock agents.
  4. Provide the exact commands to build & run locally and on iPad simulator.
- I can start implementing the rust crates and the webmonaco bridge immediately.

---

If you confirm the boilerplate and answer the questions above I will scaffold the repository and create the initial files (Cargo workspace, /kooh_default, /kooh_role_models with the system prompts for Chinga & Tanganaka), then provide code and build instructions.

Would you like me to proceed with the Rust-core + SwiftUI native shell + web-hybrid (Monaco) boilerplate? If yes, please answer the clarifying questions (1–9) and I will scaffold the first commit.