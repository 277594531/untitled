


For monitoring my GitHub copilot cli working status and my CDP enabled browser working status with m365 copilot web chat on it, on my corporation dev vm, I am thinking of an app or service playing as a baby sitting role, what kind of such service or app I should develop? Do you understand what I am talking about and what I want?


Yes. I understand what you mean, and I think “babysitter” is actually a very good mental model for the service.

You are not really asking for another Copilot/AI client. You want a small supervisor/guardian service running on the corporate dev VM that watches the whole chain:

GitHub Copilot CLI → your local work → browser/CDP → M365 Copilot web chat → response → back to Copilot CLI

and notices when something gets stuck, dies, waits for human intervention, or otherwise stops making progress.

What I think you want

Something like a Coding Agent Babysitter / Supervisor:

                    Corporate Dev VM
┌───────────────────────────────────────────────────────────┐
│                                                           │
│   GitHub Copilot CLI                                      │
│          │                                                │
│          │ stdin/stdout / process state                   │
│          ▼                                                │
│   ┌──────────────────────┐                                │
│   │   Babysitter Agent   │◄──────────────┐                │
│   │     Supervisor       │               │                │
│   └──────────┬───────────┘               │                │
│              │                           │                │
│       monitors / diagnoses               │                │
│              │                           │                │
│              ▼                           │                │
│   ┌──────────────────────┐               │                │
│   │ Edge + CDP :9222     │               │                │
│   │ M365 Copilot Web     │               │                │
│   └──────────────────────┘               │                │
│              │                           │                │
│              ▼                           │                │
│        actual web chat                   │                │
│                                          │                │
│   heartbeat / events / logs ─────────────┘                │
│                                                           │
└───────────────────────────────────────────────────────────┘

The important distinction is:

The babysitter doesn’t do the coding. It watches the worker doing the coding.

And eventually it could become somewhat intelligent.

⸻

I would design it as a “Supervisor”, not just a monitor

There are probably four levels of functionality.

Level 1 — Health monitoring

The easiest first version.

It continuously knows:

* Is Copilot CLI process alive?
* Is Copilot CLI producing output?
* When was the last output?
* Is the process consuming CPU?
* Is the process waiting for input?
* Is Edge alive?
* Is CDP 9222 reachable?
* Is the M365 Copilot tab present?
* Is the M365 session still authenticated?
* Is the page responsive?
* Is M365 Copilot generating an answer?
* Is the browser stuck?
* Is there an error message?

Then expose something like:

COPILOT        RUNNING       03:27:18
COPILOT OUTPUT ACTIVE        12 sec ago
EDGE           RUNNING
CDP            CONNECTED     :9222
M365 COPILOT   CONNECTED
SESSION        AUTHENTICATED
CHAT           GENERATING
OVERALL        🟢 HEALTHY

That alone would already be extremely useful.

⸻

Level 2 — Detect “stuck”

This is where the babysitter metaphor becomes useful.

For example:

20:01:12  Copilot started
20:01:15  Copilot sent request
20:01:17  Browser request detected
20:01:20  M365 Copilot generating
20:01:35  M365 response detected
20:01:40  Copilot processing
20:01:55  No activity
20:02:25  ⚠ Possible stall

The supervisor doesn’t immediately interfere.

Instead it could classify:

STATE: POSSIBLY_STUCK
Reason:
  No Copilot output for 30 seconds
  M365 page is responsive
  Browser session authenticated
  CPU activity normal
Confidence: 72%

Then perhaps after another threshold:

STATE: STUCK
Likely cause:
  Copilot waiting for response from backend

⸻

Level 3 — Recovery

This is where it gets much more interesting.

The babysitter could perform safe, predefined recovery actions.

For example:

Browser problem

CDP disconnected
       ↓
try reconnect
       ↓
Edge still alive?
       ↓
yes
       ↓
re-discover M365 Copilot page

M365 session expired

M365 page says "Sign in"
       ↓
STATE = AUTH_REQUIRED
       ↓
notify user

Rather than blindly attempting authentication.

Browser page frozen

Potentially:

reload page
       ↓
wait
       ↓
verify Copilot UI
       ↓
resume monitoring

Copilot process crashed

Potentially:

Copilot exited unexpectedly
       ↓
capture final logs
       ↓
notify
       ↓
optionally restart

I would make automatic recovery conservative initially.

⸻

Level 4 — Semantic babysitting

This is the really interesting future direction.

Instead of only asking:

“Is the process alive?”

the supervisor asks:

“Is the work actually progressing?”

For example, your Copilot CLI might be alive for 15 minutes but actually be stuck because the M365 backend stopped responding.

The babysitter could correlate multiple signals:

Copilot CLI
   │
   ├── process alive
   ├── CPU activity
   ├── stdout activity
   └── waiting state
           │
           ▼
      Supervisor
           │
           ├── CDP
           │
           ├── Edge
           │
           ├── M365 page
           │
           ├── chat state
           │
           └── request/response timestamps

And derive:

WORK PROGRESS = STALLED
Copilot: alive
Edge: alive
CDP: connected
M365: authenticated
M365: no response for 94 sec
Copilot: waiting
Probable bottleneck:
M365 Copilot web response

That is much more valuable than a conventional process monitor.

⸻

I would also give it an event timeline

This could become one of the best parts of the application.

For example:

20:14:02  COPILOT_STARTED
20:14:04  REQUEST_DETECTED
20:14:05  CDP_CONNECTED
20:14:06  M365_CHAT_ACTIVE
20:14:07  PROMPT_SUBMITTED
20:14:09  M365_GENERATING
20:14:28  M365_RESPONSE_COMPLETE
20:14:29  RESPONSE_CAPTURED
20:14:31  COPILOT_PROCESSING
20:15:04  COPILOT_NO_ACTIVITY
20:15:34  POSSIBLE_STALL
20:16:04  STALL_CONFIRMED

Then when something goes wrong, you don’t have to reconstruct what happened from several logs.

⸻

And I think there’s an even better name for the architecture

I’d call the overall thing something like:

Agent Supervisor

or

Coding Agent Supervisor

with the “babysitter” being the conceptual model.

Because eventually your architecture could supervise more than this one setup:

                    Agent Supervisor
                          │
          ┌───────────────┼───────────────┐
          │               │               │
      Copilot CLI     Browser/CDP     Gateway
          │               │               │
          │          M365 Copilot       │
          │          Gemini             │
          │          DeepSeek           │
          │          Meta               │
          │                             │
          └────────── Work State ────────┘

That fits very naturally with your webai-gateway project.

Your gateway is essentially the worker/backend communication layer.

The new service would sit one level above it as the supervisor/orchestrator.

⸻

I would NOT start by making it an AI agent

This is important.

For the first version, I’d make it almost boring:

             Supervisor
                 │
       ┌─────────┴─────────┐
       │                   │
    Sensors             State Machine
       │                   │
       ├─ Process          │
       ├─ stdout           │
       ├─ CDP              │
       ├─ Edge             │
       ├─ DOM              │
       └─ timestamps       │
                           ▼
                    HEALTH / STALL
                           │
                           ▼
                      Recovery

No LLM required.

Then later, an LLM can become the diagnostician, not the fundamental monitoring mechanism:

deterministic sensors
        ↓
deterministic state
        ↓
"something is wrong"
        ↓
LLM diagnosis
        ↓
"probably M365 response stalled"
        ↓
deterministic recovery policy

That is much safer and much easier to debug.

⸻

And it could eventually become a “watchdog for AI coding”

I think the larger idea you’re circling around is actually bigger than monitoring Copilot CLI.

It’s something like:

A persistent local service that watches an autonomous software-engineering agent and the infrastructure it depends on, determines whether the agent is making progress, and intervenes or alerts when it isn’t.

That is a legitimate architectural concept.

You could have states such as:

IDLE
STARTING
WORKING
WAITING_FOR_BACKEND
WAITING_FOR_USER
BROWSER_BUSY
BROWSER_STALLED
AUTH_REQUIRED
COPILOT_STALLED
BACKEND_ERROR
RECOVERING
FAILED
COMPLETED

And the most important concept would be:

Progress, not merely liveness.

A process can be alive but deadlocked.

A browser can be open but unusable.

CDP can be connected but the relevant page can be broken.

M365 Copilot can be displaying normally but not producing a response.

Copilot CLI can be running but waiting forever.

The babysitter’s job is to understand those distinctions.

⸻

Given your particular environment, I think this is actually a very good next project to build alongside webai-gateway, and I’d probably make it a separate small service rather than adding it directly into the gateway.

A good first milestone would be a “Mission Control” dashboard + deterministic state machine, where you can see in one place Copilot CLI + Edge/CDP + M365 Copilot + gateway + current request + elapsed time + last activity + detected problem. That gives you the foundation for the much smarter babysitter later.


You already know that I have a m365 wrapper service, I don’t want this new supervisor to break my working pattern of GitHub copilot cli and my m365 wrapper service and the m365 web chat. Actually I can put this new supervisor into my wrapper service, but I am thinking of keeping the wrapper service focusing on wrapping m365 web chat to OpenAI compatible API, don’t want too many other things to it. The new supervisor will handle things like, detecting the m365 web chat re authentication required via cdp, and performing m365 copilot web chat login, etc






Yes — and with that clarification, I think your separation is better.

You essentially have two different responsibilities:

                 GitHub Copilot CLI
                         │
                         ▼
              ┌─────────────────────┐
              │   M365 Wrapper      │
              │                     │
              │ OpenAI-compatible   │
              │ API adapter         │
              │                     │
              │ CDP → M365 Web Chat │
              └──────────┬──────────┘
                         │
                         ▼
                 M365 Copilot Web
              ┌─────────────────────┐
              │   New Supervisor    │
              │                     │
              │ Watch everything    │
              │ Detect problems     │
              │ Recover problems    │
              │                     │
              │ Auth / login        │
              │ Browser health      │
              │ CDP health          │
              │ Wrapper health      │
              │ Copilot health      │
              └─────────────────────┘

The key architectural principle I’d use is:

The wrapper owns the conversation. The supervisor owns the environment.

That keeps your existing working path untouched.

⸻

I would make the supervisor an independent sidecar

I would not put its logic into the M365 wrapper.

Instead:

┌────────────────────────────────────────────────────────────┐
│                    Corporate Dev VM                        │
│                                                            │
│  ┌───────────────────┐        ┌─────────────────────────┐  │
│  │ GitHub Copilot CLI│───────►│ M365 Wrapper             │  │
│  └───────────────────┘        │ :8787                    │  │
│                               │                           │  │
│                               │ OpenAI API                │  │
│                               │ CDP → M365 Copilot        │  │
│                               └────────────┬──────────────┘  │
│                                            │                 │
│                                            ▼                 │
│                               ┌─────────────────────────┐  │
│                               │ Edge / M365 Copilot     │  │
│                               │ CDP :9222               │  │
│                               └─────────────────────────┘  │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              AI Coding Supervisor                    │  │
│  │                                                      │  │
│  │  Process monitor       Browser monitor               │  │
│  │  CDP monitor           M365 session monitor           │  │
│  │  State machine         Recovery manager               │  │
│  │  Event/log collector   Notification                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
└────────────────────────────────────────────────────────────┘

The supervisor can observe the wrapper without becoming part of the wrapper’s request/response path.

That’s an important property.

If the supervisor crashes:

Copilot CLI → wrapper → M365 should continue working.

If the supervisor has a bug:

It should not be capable of corrupting a normal wrapper request.

If you later replace the supervisor entirely:

Your existing wrapper remains unchanged.

⸻

The supervisor should be allowed to be more powerful than the wrapper

This is where your idea becomes particularly interesting.

The wrapper should be deliberately stupid:

“I receive an OpenAI-compatible request and turn it into an M365 web-chat interaction.”

The supervisor can be smart:

“I own the health and lifecycle of the environment required to make that interaction possible.”

For example, the supervisor can watch:

1. Copilot CLI

process exists?
process responding?
last stdout?
CPU?
stuck?
exited?

2. Wrapper

127.0.0.1:8787 reachable?
process alive?
last request?
current request?
current provider?
request duration?
errors?

3. Edge

Edge process alive?
correct profile?
CDP :9222 alive?
browser pages?

4. M365 Copilot

correct tab?
page responsive?
authenticated?
login required?
chat composer available?
generating?
response completed?

⸻

Authentication/re-login should definitely belong to the supervisor

I agree with your example completely.

Suppose M365 expires the session.

The wrapper should not suddenly gain a bunch of code like:

if authentication expired:
    launch browser
    find login page
    click...
    wait...
    ...

That pollutes the wrapper’s job.

Instead:

                 Supervisor
                     │
                     ▼
             inspect M365 page
                     │
              authentication?
                /          \
             YES             NO
              │               │
              ▼               ▼
          healthy        login required
                              │
                              ▼
                       perform recovery
                              │
                              ▼
                       verify session
                              │
                              ▼
                           READY

Then the wrapper simply assumes:

“The environment I’m connecting to is managed and ready.”

⸻

But I would make authentication recovery a separate subsystem

I wouldn’t make “login” just another generic recovery action.

I’d model it explicitly:

AuthManager
    detectAuthState()
          │
          ├── AUTHENTICATED
          ├── AUTH_REQUIRED
          ├── AUTH_IN_PROGRESS
          ├── AUTH_FAILED
          └── UNKNOWN

Then:

AUTH_REQUIRED
      │
      ▼
Can automatic login be performed?
      │
   ┌──┴──┐
   │     │
  yes    no
   │     │
   ▼     ▼
login   notify user
   │
   ▼
verify
   │
 ┌─┴──────────┐
 │            │
success      failure
 │            │
 ▼            ▼
READY       AUTH_FAILED

This gives you a clean boundary.

⸻

There is one important question: what does “performing login” mean?

This is the part I’d design carefully.

There are actually several authentication situations:

A. Session still valid
       ↓
   nothing needed
B. Session expired but browser can silently recover
       ↓
   refresh/retry
C. Microsoft login page appears
       ↓
   supervisor navigates login flow
D. MFA / interactive authentication required
       ↓
   supervisor detects it
       ↓
   notify you / wait for human
E. Certificate-based authentication
       ↓
   potentially automate if your environment permits it
F. Completely broken profile/session
       ↓
   don't blindly destroy/recreate profile

Especially in your corporate environment, I would make the supervisor very conservative about authentication.

It should never say:

“Login failed, therefore delete cookies and start over.”

That’s exactly the kind of “helpful” automation that could destroy a perfectly recoverable work session.

⸻

I would give the supervisor a concept called “Intervention Level”

This could make the system much safer.

Level 0 — Observe

read-only

No actions.

Level 1 — Soft recovery

reconnect CDP
rediscover page
refresh page
retry connection

Level 2 — Controlled recovery

restart wrapper
restart browser automation connection
navigate back to M365

Level 3 — Authentication recovery

perform known login procedure

Level 4 — Human intervention

"Please complete authentication"

The supervisor should never automatically escalate indefinitely.

⸻

Another thing I strongly recommend: don’t let the supervisor control Copilot CLI initially

Monitor it, yes.

Control it, eventually perhaps.

But initially:

Copilot CLI
     │
     │
     ▼
Wrapper
     │
     ▼
M365
     
Supervisor
     │
     ├── observes Copilot
     ├── observes Wrapper
     ├── observes Edge
     └── observes M365

Rather than:

Supervisor
     │
     ├── starts Copilot
     ├── kills Copilot
     ├── sends keystrokes
     ├── restarts Edge
     ├── logs in
     └── modifies everything

That second version becomes dangerous and difficult to reason about very quickly.

⸻

I think the real architecture is a “Guardian”

Your M365 wrapper is a protocol adapter.

Your new service is more like an environment guardian.

Something like:

                  ┌──────────────────────┐
                  │    GitHub Copilot    │
                  │         CLI          │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │    M365 Wrapper      │
                  │                      │
                  │ OpenAI-compatible    │
                  │ API                  │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │    M365 Web Chat     │
                  └──────────────────────┘
       ┌─────────────────────────────────────────┐
       │            Environment Guardian         │
       │                                         │
       │  Process       Browser       Network    │
       │  Health        CDP           Session    │
       │                                         │
       │  M365 Auth     Progress      Recovery   │
       │                                         │
       │            Human Intervention            │
       └─────────────────────────────────────────┘

And I would deliberately make the Guardian provider-agnostic eventually.

Today:

M365 Copilot

Tomorrow:

M365 Copilot
Gemini
DeepSeek
Meta
Qwen
Doubao

But your wrapper remains responsible for the actual provider protocol.

⸻

One particularly powerful feature I’d add later

The Guardian should maintain a machine-readable environment state.

Something like:

{
  "copilot": "WORKING",
  "wrapper": "HEALTHY",
  "edge": "RUNNING",
  "cdp": "CONNECTED",
  "m365": "AUTHENTICATED",
  "m365_chat": "GENERATING",
  "overall": "WORKING"
}

Then your future dashboard can show:

╔══════════════════════════════════════╗
║       AI CODING ENVIRONMENT          ║
╠══════════════════════════════════════╣
║ Copilot CLI       🟢 WORKING          ║
║ M365 Wrapper      🟢 HEALTHY          ║
║ Edge              🟢 RUNNING          ║
║ CDP               🟢 CONNECTED        ║
║ M365 Session      🟢 AUTHENTICATED    ║
║ M365 Chat         🟢 GENERATING       ║
╠══════════════════════════════════════╣
║ Overall            🟢 WORKING         ║
║ Runtime            2h 17m             ║
╚══════════════════════════════════════╝

And when something happens:

🔴 M365 AUTHENTICATION REQUIRED
Detected: 20:31:14
Affected: M365 Web Chat
Wrapper: still healthy
Copilot: waiting
Action:
  Supervisor attempting session recovery...

That is exactly the kind of “babysitter” experience I think you’re describing.

And importantly, your existing Copilot CLI → M365 wrapper → M365 Web Chat workflow remains the primary system and doesn’t need to know that the Guardian exists. The Guardian is simply standing beside it, watching over it and taking responsibility for the environment when something goes wrong.


So what is the form of such a supervisor will be? A windows desktop app or some kind of a service? What’s the proper technology stack for designing and developing it?


Yes. Given your setup, I would not make it primarily a Windows desktop app. I would make it a Windows background service/daemon with a small local web UI.

That gives you the best separation between the thing that babysits the environment and the thing you use to look at the babysitter.

My recommended form

                Windows Dev VM
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   GitHub Copilot CLI                                    │
│          │                                              │
│          ▼                                              │
│   M365 Wrapper :8787                                    │
│          │                                              │
│          ▼                                              │
│   Edge / M365 Copilot                                  │
│          │                                              │
│         CDP :9222                                      │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │           AI Coding Supervisor                    │  │
│  │                                                   │  │
│  │  Process Monitor                                 │  │
│  │  CDP Monitor                                     │  │
│  │  Browser Monitor                                 │  │
│  │  M365 Session Monitor                            │  │
│  │  Progress Detector                               │  │
│  │  Recovery Manager                                │  │
│  │  Event/State Store                               │  │
│  └──────────────────────┬────────────────────────────┘  │
│                         │                                │
│                         ▼                                │
│                  localhost HTTP API                      │
│                         │                                │
│                         ▼                                │
│              ┌─────────────────────┐                    │
│              │ Web Dashboard        │                    │
│              │ http://127.0.0.1:... │                    │
│              └─────────────────────┘                    │
│                                                         │
└─────────────────────────────────────────────────────────┘

So conceptually it is:

Windows service + local control plane + browser-based dashboard

rather than:

Windows GUI application that happens to run in the background.

⸻

Why I prefer a service

Your babysitter has to keep working even when you’re not looking at it.

Imagine:

You RDP into VM
       ↓
start Copilot CLI
       ↓
start your normal work
       ↓
disconnect RDP
       ↓
Copilot continues working
       ↓
M365 authentication expires
       ↓
Supervisor notices
       ↓
Supervisor recovers it
       ↓
Copilot continues

A desktop application’s UI lifecycle is irrelevant to that.

The supervisor itself should have no dependency on:

* an open RDP session
* a visible desktop
* a particular user interface
* you actively watching it

The UI is just a view into the supervisor.

⸻

But I would NOT necessarily use a traditional Windows Service initially

There is an important Windows complication in your case.

Your supervisor needs to interact with:

* Edge
* an existing browser profile
* CDP
* M365 web pages
* possibly interactive authentication
* potentially Windows certificate authentication
* the user’s desktop session

A traditional Windows Service runs in Session 0, while Edge and your interactive browser live in your user’s Windows session.

That makes GUI/browser interaction awkward.

So I would initially make it a:

per-user background agent

that starts automatically when your development Windows account logs in.

Later, if necessary, split it into:

Windows Service
      │
      │ system-level monitoring
      ▼
User-session Supervisor Agent
      │
      ├── Edge
      ├── CDP
      └── M365

But don’t introduce that complexity in V1.

⸻

Technology stack

For you, my strongest recommendation would be:

TypeScript + Node.js

Specifically:

Runtime:       Node.js
Language:      TypeScript
Browser:       Playwright
HTTP:          Fastify
UI:            React + Vite
State:         in-memory initially
Persistence:   SQLite later
Logging:       Pino
Configuration: YAML
Packaging:     Windows executable / service wrapper

And there is a very good reason for this choice:

you already have webai-gateway written in TypeScript/Node/Playwright.

You don’t need to introduce another programming ecosystem merely to build the supervisor.

⸻

I would structure the project like this

Something along these lines:

ai-coding-supervisor/
│
├── src/
│   ├── app/
│   │   ├── main.ts
│   │   ├── server.ts
│   │   └── config.ts
│   │
│   ├── core/
│   │   ├── supervisor.ts
│   │   ├── state-machine.ts
│   │   ├── events.ts
│   │   └── health.ts
│   │
│   ├── monitors/
│   │   ├── copilot-cli/
│   │   ├── wrapper/
│   │   ├── edge/
│   │   ├── cdp/
│   │   └── m365/
│   │
│   ├── recovery/
│   │   ├── recovery-manager.ts
│   │   ├── browser-recovery.ts
│   │   └── auth-recovery.ts
│   │
│   ├── integrations/
│   │   ├── copilot.ts
│   │   ├── edge.ts
│   │   └── m365.ts
│   │
│   ├── api/
│   │   ├── routes.ts
│   │   └── websocket.ts
│   │
│   └── storage/
│       └── ...
│
├── web/
│   ├── src/
│   └── ...
│
├── config/
│   └── supervisor.yaml
│
├── data/
│   ├── logs/
│   └── events/
│
└── package.json

Notice that there is no M365 wrapper code in here.

It communicates with your existing wrapper rather than absorbing it.

⸻

The most important component is the state machine

I wouldn’t start with the dashboard.

I’d start with this:

                 Supervisor
                     │
                     ▼
              State Machine
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
     COPILOT       BROWSER       M365
     state         state         state
        │            │            │
        └────────────┼────────────┘
                     ▼
               Overall State

For example:

IDLE
STARTING
WORKING
WAITING_FOR_RESPONSE
BROWSER_UNAVAILABLE
CDP_DISCONNECTED
M365_AUTH_REQUIRED
M365_GENERATING
M365_STALLED
WRAPPER_UNAVAILABLE
COPILOT_EXITED
RECOVERING
WAITING_FOR_USER
FAILED
COMPLETED

This becomes the brain of the babysitter.

⸻

Use an event-driven architecture

Instead of having one giant loop:

while (true) {
   checkEverything();
   sleep();
}

I’d eventually build it around events:

CopilotOutputReceived
CDPConnected
CDPDisconnected
M365PageFound
M365AuthRequired
M365AuthRestored
M365GenerationStarted
M365GenerationCompleted
WrapperRequestStarted
WrapperRequestCompleted
WrapperRequestTimedOut
CopilotExited

Then:

event
  ↓
state machine
  ↓
new state
  ↓
policy evaluation
  ↓
optional recovery

This will make the project dramatically easier to extend.

⸻

Playwright is particularly appropriate

You already use Playwright in the wrapper.

The supervisor can connect to the existing Edge instance through CDP rather than launching another browser:

Supervisor
    │
    │ CDP
    ▼
Edge :9222
    │
    ├── M365 Copilot tab
    ├── other tabs
    └── ...

It can inspect things such as:

Is the M365 page there?
Is the composer present?
Is the user logged in?
Is a login page displayed?
Is the response generation indicator present?
Is the page responsive?
Has the page navigated unexpectedly?

That is much better than trying to infer browser health from process existence alone.

⸻

Fastify for the local control plane

I’d expose something like:

GET /api/status
GET /api/state
GET /api/events
GET /api/health
GET /api/copilot
GET /api/browser
GET /api/m365
GET /api/recovery

And eventually:

POST /api/actions/reconnect-cdp
POST /api/actions/recover-m365
POST /api/actions/restart-wrapper
POST /api/actions/acknowledge

But I’d initially make most of the action APIs read-only or manually triggered.

⸻

WebSocket/SSE for the dashboard

You don’t want the dashboard constantly polling.

Instead:

Supervisor
     │
     │ events
     ▼
WebSocket / SSE
     │
     ▼
Dashboard

Then your dashboard can update immediately:

20:42:01  🟢 Copilot working
20:42:14  🟢 M365 response generating
20:43:02  ⚠ No response activity
20:43:32  🟠 Possible M365 stall
20:44:02  🔴 Authentication required
20:44:03  🔧 Starting authentication recovery
20:44:17  🟢 Authentication restored

⸻

SQLite is enough for persistence

Don’t introduce PostgreSQL, Redis, MongoDB, etc.

This is a single-machine supervisor.

SQLite is ideal.

You could store:

sessions
events
state_transitions
recovery_attempts
errors

For example:

2026-09-18 20:42:01
COPILOT_WORKING
2026-09-18 20:43:32
M365_STALLED
2026-09-18 20:44:02
AUTH_REQUIRED
2026-09-18 20:44:17
AUTH_RESTORED

That gives you historical diagnosis without creating infrastructure.

⸻

One thing I would deliberately NOT build

Don’t make the supervisor another AI coding agent.

Avoid:

Supervisor
   ↓
LLM
   ↓
"Let's inspect the system"
   ↓
LLM executes arbitrary commands

at least initially.

Instead:

Sensors
   ↓
Deterministic state
   ↓
Rules
   ↓
Safe recovery

Later you can add:

                    ┌─────────────┐
                    │ Deterministic│
                    │ Supervisor  │
                    └──────┬──────┘
                           │
                     problem detected
                           │
                           ▼
                    ┌─────────────┐
                    │ Diagnostic  │
                    │ LLM         │
                    └─────────────┘

The LLM can answer:

“Why does this look stuck?”

but shouldn’t be responsible for deciding whether to kill your browser.

⸻

And I would give it a CLI too

The web UI is for watching.

A CLI is useful for administration:

aicoding status
aicoding events
aicoding watch
aicoding recover m365
aicoding browser status
aicoding copilot status

So the architecture becomes:

                   Supervisor Core
                         │
             ┌───────────┼───────────┐
             │           │           │
          Web API       CLI        Logs
             │
             ▼
        Web Dashboard

The core doesn’t care which interface is being used.

⸻

My recommended V1 technology stack

Component	Choice
Language	TypeScript
Runtime	Node.js
Browser/CDP	Playwright
HTTP server	Fastify
Live events	WebSocket or SSE
Dashboard	React + Vite
Persistence	SQLite
Logging	Pino
Configuration	YAML
Process monitoring	Node + Windows APIs/commands
Packaging	Windows executable / startup mechanism
Deployment	Per-user background agent initially

And architecturally:

                 ┌───────────────────────────┐
                 │     Supervisor Core       │
                 │                           │
                 │ State Machine             │
                 │ Event Bus                 │
                 │ Policy Engine             │
                 │ Recovery Manager           │
                 └─────────────┬─────────────┘
                               │
          ┌────────────────────┼───────────────────┐
          │                    │                   │
          ▼                    ▼                   ▼
    Process Monitors      Browser/CDP        M365 Monitor
          │                    │                   │
          ▼                    ▼                   ▼
    Copilot CLI             Edge :9222       M365 Web Chat
          │
          ▼
    M365 Wrapper :8787
               ┌─────────────────────────┐
               │     Control Plane       │
               │                         │
               │ REST + WebSocket/SSE    │
               └────────────┬────────────┘
                            │
                     ┌──────┴──────┐
                     ▼             ▼
                  Browser         CLI
                  Dashboard       Admin

The most important design decision

I would establish this boundary from day one:

webai-gateway

“Make M365 Web Chat behave like an API.”

ai-coding-supervisor

“Make the environment in which the AI coding agent operates stay healthy.”

That means your existing working path stays untouched:

Copilot CLI → webai-gateway → Edge/CDP → M365 Copilot

while the new program simply watches beside it and, when explicitly permitted, repairs the environment.

I think that is the cleanest architecture for what you’re trying to build.










end