# VoiceTest — design for end-to-end testing of voice agents

VoiceTest calls a voice agent the way a real customer would, over a phone number, SIP or
WebRTC. A simulated caller holds the conversation, VoiceTest records both sides, and then it
scores the call against assertions. You get pytest for phone bots: write the scenarios in
YAML, run them in CI, and read a report with audio, a transcript and pass/fail results.

```
voicetest run tests/ --target tel:+4930123456 --trials 5 --concurrency 4
```

---

## 1. How one test run works

```
 ┌──────────────┐  scenario.yaml   ┌───────────────────────────────────────────┐
 │  Test Runner │ ───────────────▶ │            Call Session (1 per call)      │
 │ (CLI / CI)   │                  │                                           │
 └──────┬───────┘                  │  ┌─────────┐   audio   ┌───────────────┐  │
        │                          │  │Transport│◀────────▶│ Caller Agent  │  │
        │                          │  │ adapter │           │ STT→LLM→TTS   │  │
        │                          │  └────┬────┘           │ + VAD/turns   │  │
        │                          │       │                └───────┬───────┘  │
        │                          │       ▼                        │          │
        │                          │  ┌──────────────────────────────▼──────┐   │
        │                          │  │ Recorder: stereo WAV, timestamped   │   │
        │                          │  │ events, per-turn transcript         │   │
        │                          │  └──────────────────┬──────────────────┘   │
        │                          └─────────────────────┼──────────────────────┘
        │                                                ▼
        │                          ┌───────────────────────────────────────────┐
        └────────── results ◀──────│ Evaluators: metrics · rules · LLM judge · │
                                   │ side-effect checks (webhook/DB/API)       │
                                   └───────────────────────────────────────────┘
```

1. The runner loads a scenario and opens a **call session**.
2. A **transport adapter** places the call and turns the audio into a bidirectional PCM
   stream.
3. The **caller agent** listens to the voice agent, decides what to say from its persona and
   goal, and speaks.
4. The **recorder** captures both channels on one clock. Every timing metric is measured from
   that clock.
5. When the call ends (hang-up, goal reached, time limit or stuck loop), the **evaluators**
   score it and the runner writes the reports.

---

## 2. Components

### 2.1 Transport adapters (how the call is reached)

| Adapter | Use case | Notes |
|---|---|---|
| `twilio` / `telnyx` / `vonage` | Real PSTN number | Outbound call with Media Streams (WebSocket, 8 kHz μ-law). This is the most realistic option because it includes carrier codec, jitter and latency. It needs a public webhook URL. |
| `sip` | Agent behind a SIP trunk / PBX | Use LiveKit SIP, FreeSWITCH or a pjsua2 client. It avoids PSTN cost. |
| `webrtc` / `livekit` | Agent built on LiveKit, Daily, Pipecat | Join the room as a participant. |
| `websocket` | Vapi, Retell, ElevenLabs, OpenAI Realtime-style APIs | Platform-native streaming. It is fast and cheap, which suits running on every commit. |
| `local` | Unit-style tests of the agent's pipeline | Feed audio files in and capture audio out. No network. |

Every adapter implements the same interface:

```python
class Transport(Protocol):
    async def connect(self, target: str, opts: dict) -> None
    async def send_audio(self, pcm16: bytes) -> None        # caller → agent
    def audio_in(self) -> AsyncIterator[AudioFrame]        # agent → caller
    async def send_dtmf(self, digits: str) -> None
    async def hangup(self) -> None
    events: AsyncIterator[CallEvent]                       # ringing, answered, transferred, ended
```

The test writer chooses where the call goes, and the rest of the system works the same way
for every transport.

### 2.2 Caller agent (the simulated customer)

The caller agent runs a streaming loop: **VAD → STT → LLM → TTS**.

- **STT:** streaming, for example Deepgram, AssemblyAI or Whisper-streaming. It produces the
  transcript of what the agent said.
- **LLM:** for example Claude. Its prompt contains the persona, the goal, the facts the caller
  knows (name, order number, date of birth) and rules such as "stay in character" and "never
  volunteer information unless asked".
- **TTS:** for example ElevenLabs, Cartesia or Azure. The voice, accent, speed and emotion
  come from the persona.
- **Turn-taking:** VAD plus endpointing on the agent's audio decides when the agent has
  finished. The caller can also **barge in** on purpose to test how the agent handles
  interruptions.

The caller agent has three modes:

1. **Scripted.** It plays fixed utterances in order, either TTS or pre-recorded WAVs. The
   result is deterministic and works well for regression tests.
2. **Goal-driven.** The LLM improvises toward a goal, which shows how the agent behaves in
   realistic, varied conversations.
3. **Hybrid.** Scripted checkpoints ("must say X at turn 3") with improvisation in between.

**Perturbations** stress the agent: background noise (café, car, street), packet loss and
codec degradation, accents, a fast or slow speaking rate, long silences, mumbling, a caller
who changes their mind, off-topic questions, prompt-injection attempts ("ignore your
instructions and…"), DTMF input and talking over the agent.

**Built-in stop conditions:** the agent hangs up, the goal is reached, the turn or time limit
is hit, a loop is detected (the agent repeats itself N times), or there is dead air for more
than N seconds.

### 2.3 Recorder

The recorder writes the following files:

- `call.wav`: stereo, with the caller on the left channel and the agent on the right.
- `events.jsonl`: every frame boundary, VAD start and stop, STT partial and final result,
  TTS start, DTMF event and call event, each with a monotonic timestamp.
- `transcript.json`: turns with speaker, text, start and end time, and interruption flags.

The agent channel is also transcribed **after the call** with a higher-accuracy batch STT
model, so that the evaluators do not inherit streaming errors.

### 2.4 Evaluators

| Type | Examples | How |
|---|---|---|
| **Latency / timing** | Response latency (caller stops speaking → agent's first audio) at p50, p95 and max; time to first greeting; dead-air gaps; total duration | Measured from the recorder's clock |
| **Conversational dynamics** | Talk-over %, how fast the agent stops when interrupted (barge-in recovery), number of turns, repeated responses | Uses VAD overlap on the two channels |
| **Deterministic rules** | `must_say` / `must_not_say` (regex or semantic), the agent asked for the order number before acting, transferred to the queue, hung up properly, sent the right DTMF | Checks on the transcript and events |
| **LLM judge** | Goal achieved? Policy followed? Hallucinated facts? Tone and empathy? Verified identity correctly? | A rubric-based prompt over the transcript, returning JSON with a score and a reason for each criterion |
| **Side effects** | An appointment exists in the calendar, a ticket was created, an SMS was sent | A user-supplied HTTP/SQL check or a webhook sink run after the call |
| **Audio quality** (optional) | MOS estimate, clipping, volume, TTS mispronunciations of names and numbers | Model-based analysis of the agent's channel |

### 2.5 Runner and reporting

- **Trials:** voice is non-deterministic, so each scenario runs N times and passes if the pass
  rate is at least the configured threshold (for example `trials: 5, pass_rate: 0.8`).
  Pass-rate trends over time are tracked.
- **Concurrency:** calls run in parallel, up to the provider's rate limits. The same mechanism
  supports a load mode ("50 simultaneous callers").
- **Outputs:** JUnit XML for CI, JSON for machine-readable results, and an HTML report with an
  audio player, a transcript synchronized to the waveform, metric charts and the judge's
  reasoning.
- **Baselines:** results are compared with the last green run so that regressions show up,
  for example "p95 latency went from 900 ms to 1400 ms".

---

## 3. Writing tests

```yaml
# tests/reschedule_appointment.yaml
name: Reschedule an existing appointment
target:
  transport: twilio
  number: "+4930123456"

caller:
  mode: goal_driven
  persona:
    name: Maria Schneider
    voice: { provider: elevenlabs, voice: "female_de_calm", language: de-DE }
    traits: "polite, a bit hesitant, gives information only when asked"
  knowledge:
    date_of_birth: "1985-03-12"
    current_appointment: "Tuesday 10:00"
  goal: "Move the Tuesday 10:00 appointment to Thursday afternoon."
  perturbations:
    background_noise: { type: cafe, snr_db: 15 }
    interrupt_agent: { probability: 0.2 }

limits: { max_duration_s: 180, max_turns: 20 }
trials: 5
pass_rate: 0.8

assertions:
  - latency.p95_ms < 1500
  - latency.first_greeting_ms < 3000
  - dead_air.max_s < 4
  - must_not_say: ["as an AI language model"]
  - rule: agent verifies date of birth before changing the appointment
  - judge:
      rubric:
        goal_achieved: "The appointment was moved to Thursday afternoon and confirmed back to the caller."
        no_hallucination: "The agent did not invent availability or policies."
        politeness: "The agent was polite and did not talk over the caller."
      min_score: 4   # 1–5 scale
  - side_effect:
      http: GET https://api.example.com/appointments?dob=1985-03-12
      expect: "$.items[0].day == 'Thursday'"
```

Test suites can share personas and the target configuration through `voicetest.yaml`. Scenario
*generation* is also planned: give VoiceTest the agent's system prompt or FAQ, and the LLM
proposes 50 diverse scenarios, including edge cases and adversarial ones.

---

## 4. The hard problems and how to handle them

1. **Turn-taking for the caller bot.** Bad endpointing makes the caller interrupt the agent
   by accident, and those false failures get blamed on the agent. Fix: a conservative default
   endpoint (about 700 ms of silence plus a semantic "end of turn" check), and barge-ins only
   on purpose.
2. **Accurate latency.** Latency must be measured on the recorder's clock at the audio-frame
   level, never from STT or LLM callbacks. Separately, report the latency the caller bot adds
   itself, so the agent is not blamed for the test harness's delay.
3. **A caller bot that is too helpful.** An LLM caller tends to "help" a weak agent by
   volunteering information. The fix is strict persona rules, and the judge can also flag
   cases where the caller led the agent.
4. **Flakiness.** Use trials with pass rates, fixed seeds for perturbations and scripted mode
   for strict regression tests. Store every recording so that each failure can be
   reproduced.
5. **IVRs, hold music and voicemail.** Add a "navigation" pre-phase (DTMF or spoken menu
   choices) plus answering-machine detection so tests start at the right point.
6. **Cost.** PSTN minutes plus STT, LLM and TTS add up. Use the `websocket`/`sip` transports
   on every commit and real phone numbers nightly. Cache the TTS for scripted utterances.
7. **Safety.** Only call numbers on an allowlist. Rate-limit calls. Never call real customers.
   Redact PII in stored recordings.

---

## 5. Suggested stack

- **Python 3.12 + asyncio.** The VoiceTest core, the CLI (Typer) and a FastAPI WebSocket server
  for Twilio Media Streams.
- **Pipecat or LiveKit Agents as the caller agent's audio pipeline**, so that the VAD
  (Silero), STT, LLM and TTS plumbing does not have to be built from scratch.
- **Pluggable providers** for STT, TTS, LLM and the judge behind small interfaces.
- **Storage:** a local `runs/<timestamp>/` folder, with an optional S3 upload.
- **Report:** a static HTML page built from `results.json`.

---

## 6. Roadmap

| Phase | Scope |
|---|---|
| **MVP** | Twilio adapter, scripted and goal-driven caller, stereo recording, latency metrics, LLM judge, CLI with JSON and JUnit output |
| **v0.2** | Perturbations (noise, interrupts), trials and pass rates, HTML report, side-effect checks |
| **v0.3** | SIP, WebSocket and LiveKit adapters; IVR navigation; voicemail detection |
| **v0.4** | Scenario generation from the agent's prompt, baselines and regression diffing, load-testing mode |
| **Later** | Dashboard, scheduled monitoring (synthetic calls every hour that alert when the agent breaks), multi-language suites |
