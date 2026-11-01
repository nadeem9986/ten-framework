# TEN Framework, Reverse-Engineered

A learning map for someone who knows frontend and has never built voice AI.
Read top to bottom. Every file path is real - open them as you read.

---

## 1. What this repo actually is

TEN is Agora's open-source framework for building real-time voice AI agents.
"Agent" here means: a program you talk to with your voice, that talks back.

The repo is not one program. It is 3 layers stacked on top of each other:

```
┌─────────────────────────────────────────────────────────┐
│  LAYER 1: The app you actually run (ai_agents/)         │
│  A ready-made voice assistant + web UI + control server │
├─────────────────────────────────────────────────────────┤
│  LAYER 2: The runtime (core/)                           │
│  The engine that moves audio/text between plugins       │
├─────────────────────────────────────────────────────────┤
│  LAYER 3: Plugins, called "extensions"                  │
│  (packages/ and ai_agents/agents/ten_packages/)         │
│  One extension = one job: STT, LLM, TTS, a tool, etc.   │
└─────────────────────────────────────────────────────────┘
```

**You will spend 95% of your time in Layer 1.** Layers 2 and 3 are the
machinery underneath. Read them only to understand, not to modify.

---

## 2. The voice loop, end to end

This is the single most important thing in the repo. When you talk to the
agent, this happens, in this order:

```
You speak in the browser
  │
  │  (1) Your mic audio flies over Agora's network into a "channel"
  ▼
agora_rtc extension (the agent's ears on the wire)
  │  raw PCM audio frames
  ▼
streamid_adapter (tiny pass-through that tags which user is speaking)
  │
  ▼
stt extension (Deepgram) - Speech To Text. Audio in, text out.
  │  data message named "asr_result"
  ▼
main_control extension (THE BRAIN OF THE APP - Python)
  │  decides what to do, keeps the conversation memory
  │  cmd named "chat_completion"
  ▼
llm extension (OpenAI) - reads the text, streams the reply word by word
  │  reply text, sentence by sentence
  ▼
main_control again - splits the reply into sentences as it streams
  │  data message named "tts_text_input"
  ▼
tts extension (ElevenLabs) - Text To Speech. Text in, audio out.
  │  PCM audio frames
  ▼
agora_rtc extension - publishes the audio back into the channel
  │
  ▼
Your browser plays the voice
```

Two side flows happen at the same time:

- **Transcripts.** Every piece of text (what you said, what the AI said)
  goes from `main_control` into `message_collector`, which ships it to the
  browser over Agora RTM (a messaging channel) so the chat UI can show it.
- **Interruptions (barge-in).** If you start talking while the AI is
  talking, the STT result hits `main_control`, which fires a "flush"
  command everywhere: the LLM stops generating, the TTS dumps its queue,
  the audio buffer clears. This is why the agent shuts up instantly when
  you interrupt it.

---

## 3. The graph: how the parts get wired together

No code connects these extensions. A JSON file does.

Open: `ai_agents/agents/examples/voice-assistant/tenapp/property.json`

It has two halves:

**Nodes** - which extensions exist in this app and their settings:

```json
{ "name": "stt",  "addon": "deepgram_asr_python",   "property": {...API key, model...} }
{ "name": "llm",  "addon": "openai_llm2_python",    "property": {...model, prompt...} }
{ "name": "tts",  "addon": "elevenlabs_tts2_python","property": {...voice id...} }
```

**Connections** - who sends what to whom. Example (simplified from the
real file): "pcm_frame audio from agora_rtc goes to streamid_adapter,
which passes it to stt". "asr_result data from stt goes to main_control".

That's the whole architecture: **extensions are LEGO bricks, the graph
JSON is the instruction sheet, and the runtime (Layer 2) carries messages
along the wires you declared.** Want to swap Deepgram for another STT?
Change one line in the graph and add your API key. No other code changes.

The same file (`property.json`, around line 1-1971) also holds several
ready-made alternative graphs (`voice_assistant_deepgram_tts`,
`voice_assistant_anthropic`, etc.) so you can see how swapping vendors
is just a different wiring.

---

## 4. Extensions: the plugin system

One extension = one folder with this shape:

```
my_extension/
├── manifest.json    # name, version, what it depends on
├── property.json    # default settings (API keys, model names)
├── extension.py     # the actual logic (if Python)
└── addon.py         # how the runtime loads it
```

Three kinds of extensions matter to you:

| Kind | Examples | What it does |
|---|---|---|
| Transport | `agora_rtc`, `websocket_server` | Moves audio in/out of the real world |
| AI service | `deepgram_asr_python` (STT), `openai_llm2_python` (LLM), `elevenlabs_tts2_python` (TTS) | Wraps one vendor's API in the standard TEN message format |
| Tool | `weatherapi_tool_python`, `vision_tool_python`, `mcp_client_python` | Gives the LLM abilities (function calling) |

There are ~100 vendor extensions in
`ai_agents/agents/ten_packages/extension/`. Every STT/TTS/LLM vendor you
have heard of is already wrapped. You don't write these; you pick them.

An extension can be written in **Python, Go, C++, or TypeScript/Node** -
the runtime speaks to all of them the same way. Almost all AI-service
extensions are Python. `agora_rtc` is C++ (downloaded pre-built from the
TEN store - you'll see it in `manifest.json` as a versioned dependency,
not a folder).

---

## 5. main_python: read this one folder twice

Path: `ai_agents/agents/examples/voice-assistant/tenapp/ten_packages/extension/main_python/`

This is the orchestrator - the only place "app logic" lives. 4 small files:

| File | What it teaches you |
|---|---|
| `extension.py` | The event handlers: user joined -> say greeting; STT result arrived -> interrupt + forward to LLM; LLM sentence arrived -> forward to TTS. ~200 lines, reads like a script. |
| `agent/agent.py` | A small event bus: queues, callback registry, how commands and data get turned into events. |
| `agent/llm_exec.py` | How conversation memory works (a list of messages appended each turn) and HOW TOOL CALLS WORK. Read this for your hotel project. |
| `helper.py` | `parse_sentences()` - how streaming LLM text gets cut into speakable sentences. |

If you understand these ~900 lines, you understand the agent.

---

## 6. Tool calling (this is where YOUR hotel idea plugs in)

The flow, traced from real code:

1. At startup, `weatherapi_tool_python` sends a `tool_register` command to
   `main_control`, describing its functions in OpenAI format
   ("get_current_weather", parameters, descriptions).
   See `ai_agents/agents/ten_packages/extension/weatherapi_tool_python/extension.py`.
2. `main_control` (in `agent/llm_exec.py`, `register_tool`) stores the
   tool and remembers which extension owns it.
3. Every LLM request includes the tool list
   (`LLMRequest(..., tools=self.available_tools)`).
4. When the LLM decides to call a tool, its reply is a
   `LLMResponseToolCall` instead of normal text.
5. `llm_exec.py` sends a `tool_call` command to the tool extension,
   waits for the result, appends it to the conversation, and re-asks the
   LLM - which now speaks the answer out loud.

**For the hotel receptionist, you write ONE new extension like
`weatherapi_tool_python`** - call it `hotel_dispatch_tool`. Its tools
would be things like `create_housekeeping_task(room, item, priority)`.
Instead of calling a weather API, it would hit your backend / send the
WhatsApp message to staff. The LLM decides when to call it based on your
system prompt. Everything else in the pipeline stays untouched.

---

## 7. The web frontend (playground/)

Path: `ai_agents/playground/` - a **Next.js + React + Redux + Tailwind**
app. This is your home turf.

How it works:

- It joins the same Agora channel as the agent using the official web SDK
  (`agora-rtc-sdk-ng`, see `package.json`). Your mic audio goes up, the
  agent's voice comes down. Agora's cloud carries it - the browser never
  talks to the agent directly.
- It also logs into Agora RTM (`src/manager/rtm/index.ts`) to receive the
  live transcript messages that `message_collector` sends.
- UI components live in `src/components/` (`Agent/`, `Chat/`). Mic
  button, camera, audio visualizer, message list - normal React.

**For the hotel project, the dashboard is a new page you build here (or
in your own app):** it listens to the same RTM/websocket messages, or
reads from your own backend where your dispatch tool wrote the tasks.

---

## 8. The control server (ai_agents/server/)

A small **Go** HTTP server (`main.go` + `internal/http_server.go`). Its
jobs:

- `POST /token/generate` - mints Agora tokens so browsers are allowed to
  join a channel (uses Agora's `rtctokenbuilder`).
- `POST /start` / `POST /stop` - spins an agent process up/down when
  someone opens the demo.
- `GET /graphs` - lists the available graphs from `property.json`.

That's it. It's plumbing, not intelligence.

---

## 9. How it connects to Agora's hosted side

- The agent and the browser are **both clients of Agora's cloud network**
  (SD-RTN). They meet inside a named channel, like two people joining the
  same call. Agora's servers carry the audio; nothing audio-related runs
  on your machine except encode/decode.
- Trust is by **token**: the Go server holds your `AGORA_APP_ID` +
  `AGORA_APP_CERTIFICATE` (in `.env`) and signs short-lived tokens for
  browsers and the agent.
- The AI vendors (Deepgram, OpenAI, ElevenLabs) are called directly from
  the agent process over normal HTTPS - Agora is not in that path.

So "Agora's hosted engine" = the real-time audio network + token service.
The agent brain itself runs wherever you run it (Docker locally, or any
server).

---

## 10. Folder tour: what matters vs what to skip

```
ten-framework/
├── ai_agents/                      ★ START HERE. The runnable agent.
│   ├── agents/
│   │   ├── examples/
│   │   │   ├── voice-assistant/    ★ THE example to study. One folder = one agent app.
│   │   │   │   └── tenapp/
│   │   │   │       ├── property.json     ★ the graph + settings. Read first.
│   │   │   │       ├── manifest.json     dependency list (extensions it uses)
│   │   │   │       └── ten_packages/extension/main_python/  ★ the brain
│   │   │   ├── websocket-example/  same idea without Agora RTC (plain websocket)
│   │   │   ├── voice-assistant-with-ten-vad/  + voice activity detection
│   │   │   └── voice-assistant-with-turn-detection/  smarter end-of-speech
│   │   └── ten_packages/extension/ ★ ~100 vendor plugins. Browse as a menu.
│   ├── playground/                 ★ Next.js web UI. Your comfort zone.
│   ├── server/                     Go control server (tokens, start/stop)
│   └── esp32-client/               skip (hardware demo)
├── core/                           runtime engine (C, C++, Rust, Go bindings). SKIP for now.
├── packages/
│   ├── core_extensions/            skip - empty templates for each language
│   ├── example_extensions/         skip - toy examples
│   └── ...                         runtime plumbing. skip.
├── docs/                           official docs mirror; getting-started/ is decent
├── third_party/, build/, tools/    skip - build machinery
└── tests/                          skip
```

---

## 11. Suggested reading order (an evening's worth)

1. This file.
2. `ai_agents/agents/examples/voice-assistant/tenapp/property.json` -
   find the `voice_assistant` graph. Read the nodes, then the
   connections. Draw the boxes yourself once.
3. `main_python/extension.py` - the whole file. It reads top to bottom
   like a story.
4. `main_python/agent/llm_exec.py` - stop when you reach the tool-call
   `match` block; that's the part you'll imitate later.
5. `weatherapi_tool_python/extension.py` - skim; notice how little code a
   "tool" needs.
6. `playground/src/manager/rtm/index.ts` and
   `playground/src/components/Agent/Microphone.tsx` - see your own world
   on the other end of the wire.
7. Only if curious: `docs/getting-started/`, and the `websocket-example`
   to see the same agent without Agora in the middle.

Do NOT start in `core/`. That's the engine room; you drive the car first.

---

## 12. Mapping this to your hotel receptionist project

Your hackathon idea, translated into TEN parts:

| Your feature | TEN piece you touch | How hard |
|---|---|---|
| "You are a polite hotel receptionist..." personality | `llm` node property in `property.json` (the `prompt` field) | 10 minutes. Pure config. |
| Greeting when a guest calls | `main_control` property `greeting` | 1 line of config. |
| Guest speaks -> task for staff | **New extension** `hotel_dispatch_tool`, cloned from `weatherapi_tool_python`. Its `tool_call` handler writes the task to your backend and/or triggers the WhatsApp message | The real work, but small: one Python file, one API call out |
| Multilingual guests (Kannada-English) | Pick STT/TTS extensions with Indian-language support (Sarvam is already wrapped: `sarvam_asr_python`, `sarvam_http_tts`) and set `language` in the graph | Config swap |
| Live task dashboard | New page in `playground/` (or your own Next.js app) fed by your dispatch tool's backend | Frontend - your strongest skill |
| Phone calls instead of browser | `voice-assistant-sip-twilio` example already wires PSTN calling in | Reuse, later |

What you do NOT need to build: audio capture, network transport,
interruption handling, STT/LLM/TTS plumbing, transcripts. All already
here. That's the whole point of the framework.

---

## 13. Glossary (words you'll see everywhere)

- **Extension** - a plugin. One job, one folder.
- **Graph** - the JSON wiring that says who talks to whom.
- **RTC** - Real-Time Communication. The audio call plumbing.
- **RTM** - Real-Time Messaging. Text/data messages over the same network (used for transcripts).
- **STT / ASR** - Speech To Text (the ears).
- **LLM** - the brain.
- **TTS** - Text To Speech (the mouth).
- **VAD** - Voice Activity Detection: knowing when a human is speaking.
- **Turn detection** - knowing when the human has FINISHED speaking.
- **PCM** - raw uncompressed audio. The currency extensions trade in.
- **Channel** - a named "room" in Agora that browser and agent both join.
- **Barge-in / flush** - interrupting the AI mid-sentence.

---

*Written by studying the actual code at the pinned commit of this fork,
not from docs. If the repo updates, trust the code over this file.*
