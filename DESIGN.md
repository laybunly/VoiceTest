# VoiceTest — UAT for voice agents

**VoiceTest is a tool for developers who build voice agents. They use it to run user
acceptance tests (UAT) on their agents before release.** It phones the agent in its UAT
environment the way a real customer would. A simulated caller holds the conversation,
VoiceTest records both sides, and then it checks the call against acceptance criteria,
including whether the right thing actually happened in the UAT backend.

```
voicetest run tests/ --env uat --trials 5
```

---

## 0. Who uses it and how

| Who | What they do |
|---|---|
| **Voice-agent developer** | Writes scenarios in the repo next to the agent's code. Runs them locally against UAT and debugs failures from the recording, the transcript and the timings. |
| **QA / product owner** | Reads scenarios written in plain language, reviews the acceptance report, signs off a release or rejects it. |
| **CI pipeline** | Runs the suite after each deploy to UAT. A non-zero exit code blocks promotion to production. |

**Product decisions that follow from this:**

- **An open-source CLI that runs where the developer runs it, with the developer's own API
  keys** (Twilio, STT, TTS, Anthropic). There is no hosted service and no multi-tenancy in
  the first versions. Recordings stay on the developer's infrastructure.
- **It works with any agent, whatever it is built on.** A phone number is the universal
  interface, so PSTN is the primary transport. Adapters for specific platforms come later as
  a cheaper, faster option.
- **The developer owns the agent and the UAT backend.** That makes checks on real results and
  test-data setup and teardown first-class features.
- **Scenarios and reports must be readable by non-developers**, because UAT is a sign-off
  step.
- **The first target platform is Salesforce Agentforce Voice** (see §2.7). The core stays
  platform-neutral, but the Salesforce integrations (test data, record checks, session traces)
  are built first.

---

## 1. How one test run works

```
 setup hook ──▶ create test data in the UAT backend (e.g. a customer with an appointment)
      │
      ▼
 ┌──────────────────────────── Call session ─────────────────────────────┐
 │ Transport (Twilio) ◀──full-duplex audio──▶ Caller agent              │
 │                                             VAD + turn detect         │
 │                                             STT → LLM → TTS           │
 │                                             end_call() tool           │
 │ Recorder: stereo WAV · events.jsonl · transcript · Twilio call SID    │
 └───────────────────────────────────────────────────────────────────────┘
      │
      ▼
 Evaluators: timing · rules · AI judge · checks of results in the backend
      │
      ▼
 teardown hook ──▶ clean up test data      Report: JUnit · JSON · HTML
```

---

## 2. Components

### 2.1 Transport

**Main transport: Twilio.** VoiceTest places an outbound call and connects it to
bidirectional Media Streams (a WebSocket carrying 8 kHz μ-law audio). This is closest to
production, because the call goes through the real carrier path, codec and jitter. Pipecat
supports it out of the box.

- **Requires a paid Twilio account.** On trial accounts the person being called hears a notice
  and must press a key before the call connects. A voice agent won't press it. Trial accounts
  can also only call verified numbers.
- **Requires a public URL** for the media WebSocket. Use ngrok locally; in CI, use a tunnel or
  a small deployed relay.
- **Non-US numbers** can require identity paperwork that takes days to approve. Plan for it.

**Later:** SIP (agents behind a PBX or SIP trunk), WebRTC/LiveKit, and platform WebSockets
(Vapi, Retell and similar). These suit fast checks on every commit, with PSTN kept for the
real UAT runs.

All transports share one interface: `connect`, `send_audio`, `audio_in`, `send_dtmf`,
`hangup` and `events`.

### 2.2 Caller agent

The caller agent is built on **Pipecat**:

- **Hearing:** Silero VAD and Pipecat smart-turn decide when the agent has finished speaking.
  Deepgram provides the live STT.
- **Brain:** a fast Claude model with the persona, the goal, the facts the caller knows and
  strict rules:
  - Stay in character.
  - Never volunteer information the agent did not ask for.
  - Never help the agent.
- **Mouth:** Cartesia (fastest) or ElevenLabs (most natural voices). This is configurable per
  persona.
- **Tools the LLM can call:**
  - `end_call(reason)`: hang up when the goal is reached or the caller gives up.
  - `press_keys(digits)`: send DTMF tones.
  - `wait(seconds)`: stay silent for a while, for example while on hold.

**Modes**

- **Hybrid (the default):** fixed checkpoints ("at turn 3, say the order number") with
  improvisation in between. Purely scripted lines break as soon as an LLM agent asks something
  unexpected.
- **Scripted:** fixed lines in order. Use it for strict regression tests of a fixed flow, such
  as an IVR.
- **Goal-driven:** the caller improvises completely toward the goal. Use it for exploratory
  tests.

**Interruptions**

- **Interrupting the agent on purpose.** The scenario sets the timing, for example "1.2 s
  after the agent starts speaking". VoiceTest measures how fast the agent stops and whether it
  understood.
- **Being interrupted by the agent.** The caller keeps listening while it speaks. When the
  agent talks over it, the caller cancels its TTS, flushes the queued audio, tells the LLM
  what was actually heard, and lets the persona decide whether to yield.
- **No accidental interruptions.** The caller waits for a conservative end of turn (about
  700 ms of silence plus the smart-turn check). Accidental interruptions caused by our own
  endpointing would be false failures.

**Stop conditions:** `end_call` is called, the agent hangs up, the time or turn limit is hit,
the agent repeats itself N times, there is dead air for more than N seconds, or the caller
has been on hold longer than the scenario's `max_hold_s`.

### 2.3 Recorder and timing (getting the latency right)

- **Pace outgoing audio in real-time 20 ms frames.** Do not push TTS output as fast as it
  arrives: Twilio buffers it, and the "caller stopped speaking" timestamp would be too early.
  That would make every agent look slower than it is.
- **Use Twilio `mark` messages** to learn when our audio actually finished playing. Every
  timing is measured from those marks and from the arrival times of incoming frames, on one
  monotonic clock.
- **Define agent latency** as the time from the caller's last audio played to the agent's
  first audio received. Our own STT/LLM/TTS delay is not part of that interval. The phone
  network's delay in both directions is included; that is roughly what real callers
  experience, and the report says so.
- **Cross-check with Twilio's dual-channel call recording**, and show any drift.
- **Outputs:** `call.wav` (stereo, caller on the left, agent on the right), `events.jsonl` and
  `transcript.json`. The agent's channel is transcribed again with Deepgram batch after the
  call, so live STT errors don't count as agent failures.
- **Watch for noisy agent audio.** Hold music, "one moment please" fillers and ambient noise
  in the agent's audio are labelled separately, so the VAD doesn't treat them as speech.

### 2.4 Evaluators

| Type | Examples |
|---|---|
| **Timing** | Response latency at p50, p95 and max; time to the first greeting; longest dead air; total duration; how fast the agent stops when interrupted |
| **Rules** | `must_say` / `must_not_say`, order of required steps ("verified DOB before changing the booking"), transferred to the right queue, ended the call politely |
| **AI judge** | Strongest Claude model, run after the call. Every criterion is **pass/fail** and needs an **evidence quote from the transcript**, which makes it auditable for sign-off. The judge sees the transcript plus the timing events. |
| **Backend results** | HTTP or SQL checks against the UAT backend: the appointment was moved, a ticket was created, an SMS was sent. This is the strongest signal. |

### 2.5 Test data hooks

UAT agents talk to UAT backends, so a test needs known data before the call and a clean
state after it:

```yaml
setup:
  - http: POST {{env.api}}/test/customers
    body: { dob: "1985-03-12", appointment: "Tuesday 10:00" }
    save: { customer_id: "$.id" }
teardown:
  - http: DELETE {{env.api}}/test/customers/{{customer_id}}
```

Saved values (`{{customer_id}}`) can be used in the caller's knowledge and in the checks.

### 2.6 Linking a test call to the agent's logs

When a test fails, the developer needs the agent's own logs for that call. VoiceTest records
the Twilio call SID, the caller ID it used, and the start and end times. An optional
`trace_lookup` hook fetches the agent's logs, LLM trace or tool calls for that call and puts
them into the report next to the transcript. With SIP, VoiceTest also sends an
`X-VoiceTest-Run-Id` header.

### 2.7 Agentforce Voice integration

**How VoiceTest relates to Salesforce's own testing tools.** Agentforce Testing Center and
`sf agent test` already simulate conversations, including voice in English. They test the
agent's *logic*: topics, actions and answers. They do not test the **real phone path** that
customers use. VoiceTest is the complement for UAT:

| | Testing Center / `sf agent test` | VoiceTest |
|---|---|---|
| Channel | Simulated inside Salesforce | A real call to the agent's phone number |
| Measures | Topic and action choice, answer quality | All of that, plus real latency, barge-in, noise, DTMF, hold and transfers |
| Languages | Simulated voice is English only (per third-party reports) | Any language the STT and TTS support |
| Best use | On every change to the agent | UAT sign-off before release; nightly runs against UAT |

**Transport.** An Agentforce Voice agent is reached through a phone number. The number is
either bought in Agentforce Voice Setup → Voice Channels and routed through an Omni-Channel
flow, or it comes from partner telephony (Amazon Connect, Vonage, Five9 and others).
Agentforce Voice doesn't expose a public audio-streaming API we could connect to directly, so
**the Twilio phone-call transport is the main path.** Direct SIP-trunk access reportedly
arrived in Summer '26; verify it, and if it exists, use it as a later transport to avoid
phone-network costs. Partner SIP integration needs Salesforce partner verification, so it is
out of scope.

**Caller ID matters.** Agentforce flows often identify the customer by the calling number. The
setup hook therefore creates the test Contact with **the phone number VoiceTest calls from**.
With a pool of caller numbers, each concurrent call gets its own number and its own Contact.

**Salesforce hooks.** Salesforce is a first-class hook type. It authenticates through an `sf`
CLI org alias (`--target-org uat`), so no separate credentials are needed:

```yaml
setup:
  - salesforce.create: Contact
    values: { FirstName: Maria, LastName: Test, Phone: "{{caller.number}}", Birthdate: 1985-03-12 }
    save: { contact_id: Id }
  - salesforce.create: ServiceAppointment
    values: { ParentRecordId: "{{contact_id}}", SchedStartTime: "{{next.tuesday}}T10:00:00" }
assertions:
  - salesforce.soql: >
      SELECT SchedStartTime FROM ServiceAppointment WHERE ParentRecordId = '{{contact_id}}'
    expect: "weekday(SchedStartTime) == 'Thursday'"
  - salesforce.soql: >
      SELECT Id FROM Case WHERE ContactId = '{{contact_id}}' AND CreatedDate = TODAY
    expect: "count == 0"   # no unwanted case was created
teardown:
  - salesforce.delete: { contact_id }   # cascades to child records where possible
```

**Session traces in the report.** After the call, VoiceTest finds the **VoiceCall** record by
the caller number and time window. It then pulls the matching **Agentforce Session Tracing**
data: the transcript as Salesforce heard it, topic and action calls, planner steps and errors.
Voice sessions link through `RelatedVoiceCallId`. This needs Data Cloud, and the Session Trace
API is still beta, so the feature is optional and fails softly when the data is unavailable.

The payoff for the developer: a failure shows **what the caller said, what Salesforce's STT
heard, which topic or action ran, and what the agent answered**, side by side. That answers
the common question of whether a failure came from speech recognition, routing or the agent's
logic.

**Escalation tests.** Many Agentforce Voice flows hand the call to a human through
Omni-Channel. The assertion `escalated: { queue: "Billing" }` checks for the transfer in the
call audio (ringing or hold music) and in the VoiceCall or AgentWork records. In UAT the
target queue should be a test queue with no staffed agents.

**Optional bridge to `sf agent test`.** VoiceTest can import an existing `sf agent test` YAML
spec and run its conversations as phone calls, so teams reuse the scenarios they already have.

**Still to verify in an actual org** (salesforce.com docs weren't reachable from this
session):
- Can a phone number be bought and routed in a **sandbox**, or only in production?
- How much do test calls draw on the org's voice credits?
- Which VoiceCall and Session Tracing fields hold the STT transcript, and how quickly do they
  appear after a call?
- Does Agentforce Voice accept DTMF, and how does it handle barge-in?
- Is direct SIP-trunk access available?

### 2.8 Runner and reports

- **Environments:** `voicetest.yaml` defines `uat`, `staging` and so on (number, API base URL,
  secrets taken from environment variables). Select one with `--env uat`.
- **Trials:** each scenario runs N times. It passes if the pass rate is at least the threshold.
- **Concurrency:** calls run in parallel up to a limit, spread over a small pool of caller
  numbers so carriers don't flag them as spam.
- **Outputs:**
  - **JUnit XML** for CI gating (exit code 1 on failure).
  - **JSON** results.
  - An **HTML acceptance report**: a summary per user story, then each call with audio, a
    transcript synchronized to the waveform, timings, the judge's verdict with evidence, and
    the backend checks.
- **Requirement links:** scenarios carry `story: PROJ-123` tags, and the report groups results
  by story for sign-off.
- **Baselines:** results are compared with the last accepted run, for example "p95 latency
  went from 900 ms to 1400 ms" or "pass rate fell from 100 % to 60 %".
- **Cost:** call minutes and API usage are estimated per run and shown in the report.

---

## 3. Example scenario

```yaml
# tests/reschedule_appointment.yaml
name: Reschedule an existing appointment
story: APPT-42
target: { env: uat }

setup:
  - http: POST {{env.api}}/test/customers
    body: { dob: "1985-03-12", appointment: "Tuesday 10:00" }
    save: { customer_id: "$.id" }

caller:
  mode: hybrid
  persona:
    voice: { provider: cartesia, voice: "<voice-id>", language: en-US }
    traits: "polite, a bit hesitant, answers only what is asked"
  knowledge:
    date_of_birth: "1985-03-12"
    current_appointment: "Tuesday 10:00"
  goal: "Move the Tuesday 10:00 appointment to Thursday afternoon."
  checkpoints:
    - when: agent_asks_identity
      say: "My date of birth is March 12th, 1985."
  perturbations:
    background_noise: { type: cafe, snr_db: 15 }
    interrupt_agent: { at_turn: 2, after_ms: 1200, say: "Sorry, Thursday please" }

limits: { max_duration_s: 180, max_turns: 20, max_hold_s: 30 }
trials: 5
pass_rate: 0.8

assertions:
  - latency.p95_ms < 1500
  - latency.first_greeting_ms < 3000
  - dead_air.max_s < 4
  - barge_in.stop_ms < 800
  - must_not_say: ["as an AI language model"]
  - order: [agent_verifies_dob, agent_changes_appointment]
  - judge:
      - "The agent confirmed the new Thursday time back to the caller."
      - "The agent did not invent availability or policies."
  - check:
      http: GET {{env.api}}/test/customers/{{customer_id}}
      expect: "$.appointment.day == 'Thursday'"

teardown:
  - http: DELETE {{env.api}}/test/customers/{{customer_id}}
```

---

## 4. Trusting the results (calibration)

If the caller or the judge is wrong, every result is wrong. VoiceTest therefore ships with a
**reference agent**: a small Pipecat agent with switchable faults.

- **Fault modes:** `good`, `slow` (adds 2 s of latency), `rude`, `skips_identity_check`,
  `hallucinates`, `ignores_interruptions`.
- **Self-test:** `voicetest calibrate` runs the built-in suite against each mode. The harness
  must pass `good` and catch every fault. This is also VoiceTest's own end-to-end test in CI.
- **Judge agreement:** the developer labels about 20 of their own calls by hand, and VoiceTest
  reports how often the judge agrees per criterion. Criteria with low agreement get rewritten
  or replaced by rules.
- **Leading the agent:** the judge also flags calls where the caller led the agent ("caller
  volunteered information it wasn't asked for"). Such calls are marked invalid, not passed.

---

## 5. Safety, privacy and legal

- **Allowlist:** VoiceTest only calls numbers listed in the environment config.
- **Rate limits:** a maximum number of calls per minute.
- **Retention:** recordings and transcripts stay on the developer's machine or storage. A
  configurable retention period deletes them automatically. PII redaction in transcripts is
  optional.
- **Consent and data protection:** teams test their own UAT agents, but call recording and
  data-protection rules (e.g. GDPR) still apply to stored audio, especially if real test
  users' data is used. Use synthetic test data.
- **Third-party agents:** testing an agent you don't own requires written permission from its
  operator.

---

## 6. Stack

| Part | Choice |
|---|---|
| Language, CLI | Python 3.12, asyncio, Typer |
| Audio pipeline | Pipecat (Silero VAD, smart-turn, Twilio serializer, interruption handling) |
| Telephony | Twilio Programmable Voice + Media Streams (paid account) |
| Live STT | Deepgram streaming |
| Caller brain | A fast Claude model |
| TTS | Cartesia (default) or ElevenLabs |
| Post-call transcript | Deepgram batch |
| Judge | The strongest Claude model |
| Report | Static HTML generated from `results.json` |

STT, TTS and the LLM sit behind small interfaces, so a provider can be swapped in for a single
scenario.

---

## 7. Roadmap (realistic: 4–6 weeks to something you'd trust in CI)

| Phase | Scope |
|---|---|
| **M1: first call** | Twilio outbound call, scripted caller, paced audio with marks, stereo recording, latency metrics, JSON output. Validated against the reference agent. |
| **M2: real conversations** | Hybrid and goal-driven caller, `end_call` tool, turn detection, post-call transcript, AI judge (pass/fail with evidence), rules, JUnit output and exit codes |
| **M3: acceptance testing** | Environments, **Salesforce hooks** (sf org alias, create/delete records, SOQL checks), HTTP hooks, caller-number pool tied to test Contacts, trials and pass rates, HTML acceptance report grouped by story, `voicetest calibrate` |
| **M4: robustness and Agentforce depth** | Planned interruptions, background noise, DTMF, hold handling, escalation assertions, **VoiceCall and Session Tracing lookup** in the report, import of `sf agent test` specs |
| **Later** | SIP, LiveKit and platform-WebSocket transports, baselines and regression diffs, scenario generation from the agent's prompt, scheduled monitoring calls in production, optional hosted dashboard |
