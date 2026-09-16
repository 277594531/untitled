My host simulator for horse racing. It plays as TCP server, follow ITSP protocol. Question, for this host simulator requirement, what tech stack should I choose to design and implement it? I am thinking of server and web ui instead of traditional desktop app

Yes. For a horse-racing host simulator that behaves as a TCP server and implements an ITSP protocol, I would strongly favor a server + web UI architecture over a traditional desktop application.

In fact, given the nature of the simulator, I would design it almost like a small distributed test platform rather than a GUI application.

My recommended stack

Layer	Recommendation	Why
TCP/ITSP server	Java 21 + Spring Boot	Excellent TCP/networking ecosystem, strong typing, mature enterprise tooling
TCP framework	Netty or Spring Integration TCP	Robust handling of persistent TCP connections and protocol framing
ITSP protocol	Custom Java protocol layer	Keep wire protocol completely separate from business simulation
Simulation engine	Java domain model + state machine	Deterministic, testable racing behavior
REST API	Spring Boot Web / WebFlux	Controls simulator from browser
WebSocket/SSE	WebSocket	Push live race/session/events to browser
Web UI	React + TypeScript	Good for real-time dashboards and control panels
UI build	Vite	Simple and fast
Database	SQLite initially, PostgreSQL if needed	Simulator usually doesn’t need a heavy DB
Configuration	YAML/JSON	Race courses, horses, timing, protocol scenarios
Testing	JUnit + Testcontainers + protocol integration tests	Especially important for TCP protocol simulator
Packaging	Docker + Docker Compose	Easy to run on developer/test machines

So conceptually:

                     ┌─────────────────────────┐
                     │       Web Browser       │
                     │                         │
                     │ React + TypeScript      │
                     │ Race Dashboard          │
                     │ Host Control             │
                     │ TCP Connections          │
                     │ Logs / Messages          │
                     └────────────┬────────────┘
                                  │
                         REST / WebSocket
                                  │
                                  ▼
┌───────────────────────────────────────────────────────────┐
│                    Host Simulator                         │
│                                                           │
│  Spring Boot                                              │
│                                                           │
│  ┌─────────────────┐       ┌──────────────────────────┐  │
│  │ REST Controller  │       │ WebSocket Event Stream   │  │
│  └────────┬────────┘       └────────────┬─────────────┘  │
│           │                             │                │
│           └──────────────┬──────────────┘                │
│                          ▼                               │
│                 Simulation Engine                        │
│                                                           │
│       Race State / Horse State / Timing / Events         │
│                          │                               │
│                          ▼                               │
│                  ITSP Protocol Layer                      │
│                          │                               │
│                          ▼                               │
│                  TCP Server / Netty                       │
└──────────────────────────┬────────────────────────────────┘
                           │
                           │ ITSP over TCP
                           ▼
                 ┌──────────────────────┐
                 │ Horse Racing System  │
                 │ / External Client    │
                 └──────────────────────┘

The most important architectural decision

I would not make the web UI directly responsible for the simulation.

Instead, think of three independent layers:

1. Protocol layer

This is the part that pretends to be the real horse-racing host.

TCP connection
     ↓
TCP framing
     ↓
ITSP message parser
     ↓
ITSP message model
     ↓
ITSP command dispatcher

For example:

Client
   │
   │ ITSP message
   ▼
TCP Server
   │
   ▼
Frame Decoder
   │
   ▼
ITSP Parser
   │
   ▼
StartRaceCommand

This layer should know nothing about React or the browser.

⸻

2. Simulation/domain layer

This is the actual “horse racing host simulator.”

For example:

Race
 ├── Race ID
 ├── Course
 ├── Distance
 ├── Status
 ├── Horses
 │    ├── Horse 1
 │    ├── Horse 2
 │    └── Horse 3
 └── Timeline

And:

RaceState
WAITING
   ↓
READY
   ↓
STARTED
   ↓
RUNNING
   ↓
FINISHED
   ↓
RESULT

This is where you implement things such as:

* race start
* horse positions
* timing
* race completion
* results
* simulated failures
* delayed responses
* invalid protocol scenarios
* connection behavior

The key point is that the simulation engine should be independently testable without TCP or a browser.

⸻

3. Web management layer

The browser becomes your operator console.

Something like:

┌──────────────────────────────────────────────┐
│ HOST SIMULATOR                               │
├──────────────────────────────────────────────┤
│                                              │
│ TCP Server                                   │
│ ● RUNNING       Port: 9000                   │
│                                              │
│ Connections                                  │
│ ┌──────────────────────────────────────────┐ │
│ │ 10.10.1.20:43122   CONNECTED             │ │
│ │ 10.10.1.21:43125   CONNECTED             │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│ Current Race                                 │
│                                              │
│ Race #20260916-001                           │
│ Status: RUNNING                              │
│                                              │
│ Horse       Position       Speed             │
│ ──────────────────────────────────────────   │
│ Horse 01       1            62 km/h          │
│ Horse 07       2            60 km/h          │
│ Horse 03       3            58 km/h          │
│                                              │
│ [Start Race] [Pause] [Finish] [Reset]        │
│                                              │
└──────────────────────────────────────────────┘

And importantly, the UI can show the actual ITSP messages:

22:14:31.120  RX
START_RACE ...
22:14:31.125  TX
ACK ...
22:14:32.100  TX
HORSE_POSITION ...
22:14:32.200  TX
HORSE_POSITION ...

That would make the simulator much more useful than a traditional desktop GUI.

⸻

Why I would choose Java for this particular project

Given your existing development environment, I think Java 21 + Spring Boot is particularly appropriate.

You already work with Spring Boot/Maven, Jenkins, SQL Server, OpenTelemetry, etc., so you don’t need to introduce an entirely different backend ecosystem.

More importantly, Java gives you a nice separation:

com.example.horsehost
│
├── protocol
│   └── itsp
│       ├── ItspMessage
│       ├── ItspDecoder
│       ├── ItspEncoder
│       └── ItspMessageHandler
│
├── tcp
│   ├── TcpServer
│   ├── ConnectionManager
│   └── Session
│
├── simulation
│   ├── Race
│   ├── Horse
│   ├── RaceEngine
│   ├── RaceState
│   └── RaceEvent
│
├── application
│   ├── RaceService
│   └── SimulatorService
│
├── api
│   ├── RaceController
│   └── ConnectionController
│
└── websocket
    └── SimulatorEventHandler

That is a very maintainable architecture for this type of system.

⸻

One thing I would avoid

I would not make the simulator a giant Spring @Service with TCP handling, simulation logic and UI logic mixed together.

For example, avoid this kind of design:

TcpController
    ↓
everything happens here
    ↓
update database
    ↓
run race
    ↓
send ITSP
    ↓
update UI

Instead:

              ┌──────────────┐
              │ Web API       │
              └──────┬───────┘
                     │
              ┌──────▼───────┐
              │ Application   │
              │ Service       │
              └──────┬───────┘
                     │
          ┌──────────▼──────────┐
          │ Simulation Engine   │
          └──────────┬──────────┘
                     │
             Domain Events
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
     ITSP Adapter          WebSocket
          │                     │
          ▼                     ▼
       TCP Client             Browser

This becomes particularly valuable when the simulator grows.

⸻

I would also make the simulator scenario-driven

This is probably one of the most valuable design decisions for a host simulator.

Instead of hard-coding every race:

if (raceId == 123) {
    ...
}

define scenarios as data.

For example:

scenario:
  id: normal-race
  race:
    distance: 1200
    horses: 8
  timing:
    startDelay: 2s
    raceDuration: 60s
  behavior:
    connectionTimeout: false
    malformedMessage: false
    delayedResponse: false

Then you could have:

normal-race.yaml
slow-host.yaml
connection-failure.yaml
invalid-message.yaml
timeout.yaml
multi-race.yaml

The web UI could simply let the operator choose:

Scenario
[ Normal Race           ▼ ]
[Load]
[Start]

This turns your application from merely a “horse racing mock” into a proper ITSP test simulator.

⸻

Another important feature: deterministic simulation

I would strongly recommend supporting a simulation clock.

Instead of relying completely on real wall-clock time:

real time
    ↓
race engine

use:

Simulation Clock
       ↓
Race Engine

Then you can run:

1x   real time
5x   faster
10x  faster
pause
step

For example:

Race time: 00:32.450
[▶ Run] [Ⅱ Pause] [⏭ Step] [↻ Reset]
Speed: [ 10x ▼ ]

That can be extremely useful when testing an external racing system.

⸻

Web UI technology

For the frontend I would use:

React + TypeScript + Vite

I would not introduce something overly elaborate initially.

Probably:

React
TypeScript
Vite

and a component library such as:

MUI

or another mature React UI library.

The UI talks to:

REST
GET  /api/races
POST /api/races
POST /api/races/{id}/start
POST /api/races/{id}/pause
POST /api/races/{id}/reset
GET  /api/connections

and:

WebSocket
/ws/events

for live events.

⸻

Database: don’t over-engineer it

I would initially use:

SQLite

for things like:

scenarios
race configurations
test runs
message logs
simulation results

You don’t necessarily need PostgreSQL just because this is a server application.

But structure the persistence layer so that switching to PostgreSQL later is easy.

⸻

Deployment becomes very nice

This architecture gives you something a desktop application cannot easily give you.

You can run:

horse-host-simulator
        │
        ├── TCP :9000
        ├── HTTP :8080
        └── WebSocket :8080/ws

Then from any machine:

http://host-machine:8080

and you get the simulator console.

You could even run it on a test server:

                    Test Network
        ┌───────────────────────────┐
        │ Horse Host Simulator      │
        │                           │
        │ TCP :9000                 │
        │ Web :8080                 │
        └─────────────┬─────────────┘
                      │
                      │ ITSP
                      ▼
              ┌───────────────┐
              │ System Under  │
              │ Test          │
              └───────────────┘
              Operator
                  │
                  │ Browser
                  ▼
           Web UI :8080

That is a much better fit for a test simulator than:

Windows Desktop EXE
    ↓
local GUI
    ↓
local TCP server

⸻

My final recommendation

If I were starting this project today, I’d choose:

Backend
──────────────
Java 21
Spring Boot
Netty
Maven
Protocol
──────────────
Custom ITSP encoder/decoder
Explicit protocol state machine
Simulation
──────────────
Java domain model
Deterministic simulation clock
Scenario-driven behavior
Event-driven architecture
API
──────────────
Spring REST
WebSocket
Frontend
──────────────
React
TypeScript
Vite
MUI
Persistence
──────────────
SQLite initially
PostgreSQL optional
Testing
──────────────
JUnit 5
Testcontainers
TCP integration tests
Protocol conformance tests
Deployment
──────────────
Docker
Docker Compose

And I would treat the ITSP protocol specification as the primary contract, with the simulator implementation built underneath it.

The resulting product is essentially:

ITSP Host Simulator = TCP protocol emulator + deterministic horse-racing simulation engine + browser-based operator/test console.

That separation is the part I think is most important. It also leaves room later for an automated test API, where Jenkins or another test framework can control the simulator without using the UI at all.



Good design. One more thing, what I need is more like a host simulator instances management platform instead of a single instance of host simulator. I need to start multiple instances, the TCP server listening on let’s say 9000, 9001,9002, etc. and manage this via the single web site with port 8080. Do you understand what I am talking about and what I need?


Yes — and that changes the architecture in an important but very manageable way.

What you’re describing is not primarily a “host simulator with a web UI.” It is a Host Simulator Instance Management Platform, where the web application is the central control plane and each simulator instance is an independently running ITSP TCP host.

I understand the requirement as:

                         Browser
                            │
                            │ HTTP / WebSocket
                            ▼
                  ┌─────────────────────┐
                  │   Management Web    │
                  │      :8080          │
                  │                     │
                  │ Instance Management  │
                  │ Configuration        │
                  │ Logs                 │
                  │ Monitoring           │
                  └──────────┬──────────┘
                             │
                    Instance Manager
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌────────────┐
       │ Simulator  │ │ Simulator  │ │ Simulator  │
       │ Instance 1 │ │ Instance 2 │ │ Instance 3 │
       │            │ │            │ │            │
       │ TCP :9000  │ │ TCP :9001  │ │ TCP :9002  │
       └──────┬─────┘ └──────┬─────┘ └──────┬─────┘
              │              │              │
              ▼              ▼              ▼
          ITSP Client    ITSP Client    ITSP Client

And potentially:

Instance 4 → :9003
Instance 5 → :9004
Instance 6 → :9005
...

The key distinction

I would make “instance” a first-class concept.

For example:

Instance
──────────────
ID:          simulator-001
Name:        SIT Host 01
TCP Port:    9000
Status:      RUNNING
Scenario:    Normal Race
Created:     ...

Another:

Instance
──────────────
ID:          simulator-002
Name:        SIT Host 02
TCP Port:    9001
Status:      RUNNING
Scenario:    Slow Host

The management application owns these instances.

⸻

I would slightly change my previous architecture

Instead of:

Spring Boot
 └── Simulator

I’d make it:

                    Host Simulator Platform
                    ────────────────────────
                         Web UI :8080
                              │
                              ▼
                    ┌──────────────────┐
                    │ Management API   │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Instance Manager │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         Instance 001   Instance 002   Instance 003
              │              │              │
           :9000          :9001          :9002
              │              │              │
           ITSP           ITSP           ITSP
              │              │              │
         Simulation      Simulation      Simulation

The Instance Manager becomes the central component.

⸻

There are actually two possible architectures

This is an important design decision.

Option A — One JVM, multiple simulator instances

You run one Spring Boot process:

Spring Boot :8080
     │
     ├── Instance 001 → TCP :9000
     ├── Instance 002 → TCP :9001
     ├── Instance 003 → TCP :9002
     └── Instance 004 → TCP :9003

Each instance has its own:

ITSP session manager
Race engine
State
Configuration
TCP listener
Event stream

This is probably what I would recommend initially.

It is simple and efficient.

⸻

Option B — Separate process/container per instance

The management server starts separate simulator processes:

Management Server :8080
       │
       ├── Process/Container 001 → :9000
       ├── Process/Container 002 → :9001
       ├── Process/Container 003 → :9002
       └── Process/Container 004 → :9003

This gives stronger isolation.

For example, if instance 002 crashes:

Instance 001    RUNNING
Instance 002    ❌ CRASHED
Instance 003    RUNNING

Instance 001 and 003 aren’t affected.

It also makes it easier eventually to distribute instances across machines.

⸻

I would NOT start with Option B

For your requirement, I’d start with Option A:

One management server / one JVM / multiple independent simulator instances.

Conceptually:

SimulatorInstance instance1 = manager.create(...);
SimulatorInstance instance2 = manager.create(...);
SimulatorInstance instance3 = manager.create(...);

Each instance owns its own TCP server.

For example:

InstanceManager
     │
     ├── SimulatorInstance #1
     │      ├── TcpServer :9000
     │      ├── RaceEngine
     │      ├── Scenario
     │      └── Connections
     │
     ├── SimulatorInstance #2
     │      ├── TcpServer :9001
     │      ├── RaceEngine
     │      ├── Scenario
     │      └── Connections
     │
     └── SimulatorInstance #3
            ├── TcpServer :9002
            ├── RaceEngine
            ├── Scenario
            └── Connections

The instances should be completely isolated at the domain level.

⸻

The web UI then becomes much more interesting

Your home page could be an instance dashboard:

HOST SIMULATOR PLATFORM
┌──────────────────────────────────────────────────────────┐
│ Instances                              [+ Create Instance]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│ ● SIT-HOST-01       :9000    RUNNING    2 connections    │
│   Scenario: Normal Race                                  │
│   [Open] [Stop] [Restart]                                │
│                                                          │
│ ● SIT-HOST-02       :9001    RUNNING    1 connection     │
│   Scenario: Slow Host                                    │
│   [Open] [Stop] [Restart]                                │
│                                                          │
│ ○ TEST-HOST-01      :9002    STOPPED    0 connections    │
│   Scenario: Connection Failure                           │
│   [Start] [Edit]                                         │
│                                                          │
└──────────────────────────────────────────────────────────┘

Clicking Open takes you to the individual instance dashboard:

SIT-HOST-01
TCP
────────────────────
Address: 0.0.0.0:9000
Status: RUNNING
Connections
────────────────────
10.10.1.20:51231
10.10.1.21:51232
Race
────────────────────
Race #12345
RUNNING
32.4 seconds
Horses
────────────────────
01    Position 1
03    Position 2
07    Position 3
...
ITSP Messages
────────────────────
RX  START_RACE
TX  ACK
TX  HORSE_POSITION
TX  HORSE_POSITION
...
[Pause] [Resume] [Reset] [Stop Instance]

This is much closer to what you’re describing.

⸻

And this gives you a very useful abstraction

I’d define the platform around:

Platform
   │
   ├── Instance
   │      │
   │      ├── Configuration
   │      ├── TCP Listener
   │      ├── Connections
   │      ├── Simulation Engine
   │      ├── Scenario
   │      ├── Events
   │      └── Logs
   │
   ├── Instance
   │      ...
   │
   └── Instance
          ...

The web application manages instances, while each instance behaves like an independent host.

That also means you can eventually expose a management REST API:

POST   /api/instances
GET    /api/instances
GET    /api/instances/{id}
POST   /api/instances/{id}/start
POST   /api/instances/{id}/stop
POST   /api/instances/{id}/restart
GET    /api/instances/{id}/connections
GET    /api/instances/{id}/races
POST   /api/instances/{id}/races
POST   /api/instances/{id}/races/{raceId}/start
POST   /api/instances/{id}/races/{raceId}/pause

And WebSocket events could include the instance ID:

{
  "instanceId": "sit-host-01",
  "type": "ITSP_MESSAGE",
  "direction": "TX",
  "timestamp": "...",
  "message": "..."
}

So the single browser application can monitor all simulator instances simultaneously.

⸻

One more thing I think is important

Don’t make the TCP port itself the identity of the instance.

Instead:

Instance ID
    ↓
Configuration
    ↓
TCP Port

For example:

instanceId: "horse-sit-01"
name: "SIT Horse Host 01"
port: 9000
scenario: "normal-race"

That way you can later change:

horse-sit-01
    :9000 → :9100

without changing the identity of the instance.

⸻

So yes, I understand the requirement now.

I’d describe your system as:

A web-based management and orchestration platform for multiple independent ITSP horse-racing host simulator instances, where each instance exposes its own TCP endpoint while all instances are centrally controlled and monitored through a single web application.

And with that clarification, I would slightly redesign the original architecture around an InstanceManager / SimulatorInstance model before writing any code. That will make the project much cleaner and also leave a path from one JVM with many instances → separate processes/containers → multiple simulator machines later without redesigning the core domain model.



