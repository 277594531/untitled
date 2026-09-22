Do you know my supervisor project? About monitoring m365 wrapper working status, and polling my public GitHub gist to get commands from me


Yes. I know the supervisor project you’re referring to.

The idea we discussed is essentially a separate “babysitter/supervisor” for your M365 Copilot wrapper + Copilot CLI environment, rather than putting more responsibilities into the wrapper itself.

My understanding of the design is:

* M365 wrapper remains focused on its core job
    * Your existing webai-gateway / M365 Copilot wrapper exposes the M365 Copilot web chat as an OpenAI-compatible API.
    * It uses the browser/Edge session and CDP rather than a separate M365 API key.
    * The supervisor should not interfere with the normal Copilot CLI → wrapper → M365 web chat flow.
* Supervisor monitors the M365 side
    * Connects to the Edge/Chrome CDP endpoint, e.g. port 9222.
    * Determines whether the M365 Copilot web chat is healthy/usable.
    * Detects situations such as the web chat requiring re-authentication.
    * Potentially performs the browser-side recovery/login workflow when possible.
    * Essentially acts as a watchdog so your long-running wrapper doesn’t silently become unusable.
* Supervisor also provides a command/control channel
    * You were considering using a public GitHub Gist/repository as a polling channel.
    * Your worker/supervisor periodically polls it.
    * You can publish a command from outside your corporate environment.
    * The worker retrieves the command and executes an appropriate local action.
    * Because the corporate dev VM has restrictions on posting data outward, the direction is intentionally you → public GitHub → worker, rather than the worker uploading arbitrary work data to the Internet.
* You were specifically considering encrypted commands
    * We discussed using something like RSA sign-then-encrypt so the public GitHub content doesn’t expose the command itself.
    * The worker has the private decryption key.
    * Your external command-generation side has the worker’s public key.
    * Signing gives the worker confidence that the command actually came from your command source, while encryption keeps the command contents confidential.
    * We also discussed suitable filenames where something like enc could indicate an encrypted command artifact.
* The supervisor is conceptually separate from the wrapper
    Something roughly like:

                Internet
                   │
                   ▼
           Public GitHub Gist
                   │
            encrypted command
                   │ polling
                   ▼
      ┌─────────────────────────┐
      │     Supervisor/Worker    │
      │                         │
      │  Command Poller         │
      │  Command Decrypt/Verify │
      │  M365 Health Monitor    │
      │  Recovery Controller    │
      └───────────┬─────────────┘
                  │
           CDP / local control
                  │
                  ▼
           Edge / M365 Copilot
                  │
                  ▼
           M365 Wrapper
                  │
                  ▼
           Copilot CLI / clients

And importantly, I remember your architectural preference that this supervisor should not turn into another giant version of the wrapper. The wrapper should continue being the API gateway; the supervisor should be the thing responsible for observing, recovering, and receiving your remote control commands.

We had also discussed whether the supervisor should be a Windows desktop application or a service, and I was leaning toward a small Windows background service/worker process rather than a traditional GUI desktop application, with an optional local status UI if useful.

So yes — I have the context of that project, including the M365 monitoring + GitHub polling command channel + encrypted command direction.


My commands include like relogin m365 copilot, ask it to fill and send my command/message to certain GitHub copilot cli within powershell 7 console, etc. you help to design a protocol for the commands, I am thinking of a json format to use for the original plain text


Yes. I would make the plain-text command itself a structured JSON protocol, rather than trying to encode commands as arbitrary shell commands.

The important distinction is:

GitHub/Gist is the transport. JSON is the command protocol. The supervisor is the executor.

That gives you a stable protocol even if you later change how the command is transported or encrypted.

1. I recommend an envelope like this

{
  "protocol": "copilot-supervisor-command",
  "version": 1,
  "id": "cmd-20260923-000001",
  "created_at": "2026-09-23T07:30:00Z",
  "expires_at": "2026-09-23T08:00:00Z",
  "target": {
    "worker": "dev-vm-01"
  },
  "command": {
    "type": "m365.copilot.relogin",
    "args": {}
  },
  "execution": {
    "mode": "once",
    "timeout_seconds": 300
  }
}

The key idea is that command.type is not a PowerShell command. It is a controlled semantic operation.

For example:

m365.copilot.relogin
copilot.cli.send
copilot.cli.status
m365.copilot.status
supervisor.status

This is much safer and easier to evolve than:

{
  "command": "powershell.exe -Command ..."
}

⸻

2. M365 re-login command

For your first example:

{
  "protocol": "copilot-supervisor-command",
  "version": 1,
  "id": "cmd-20260923-000002",
  "created_at": "2026-09-23T07:35:00Z",
  "expires_at": "2026-09-23T08:05:00Z",
  "target": {
    "worker": "dev-vm-01"
  },
  "command": {
    "type": "m365.copilot.relogin",
    "args": {
      "provider": "m365-copilot"
    }
  },
  "execution": {
    "mode": "once",
    "timeout_seconds": 300
  }
}

The supervisor interprets this as:

detect login required
        ↓
activate/focus Edge
        ↓
navigate/interact with M365 authentication
        ↓
verify M365 Copilot is usable
        ↓
return command result

Notice that the command doesn’t tell the worker how to log in.

That’s intentional.

The protocol describes intent, while the supervisor owns implementation.

⸻

3. Sending a message to a particular Copilot CLI

This is where your protocol becomes particularly useful.

I would make the target an explicit object.

For example:

{
  "protocol": "copilot-supervisor-command",
  "version": 1,
  "id": "cmd-20260923-000003",
  "created_at": "2026-09-23T07:40:00Z",
  "expires_at": "2026-09-23T08:10:00Z",
  "target": {
    "worker": "dev-vm-01",
    "copilot_cli": "copilot-01"
  },
  "command": {
    "type": "copilot.cli.send",
    "args": {
      "message": "Please inspect the current project status and continue working on the authentication recovery implementation."
    }
  },
  "execution": {
    "mode": "once",
    "timeout_seconds": 600
  }
}

The supervisor then maps:

copilot-01
    ↓
specific PowerShell 7 console/session
    ↓
send message
    ↓
GitHub Copilot CLI

This is much better than putting things like:

"powershell_window": "ConsoleHost..."

into the external protocol.

The supervisor can maintain its own mapping:

copilot_cli:
  copilot-01:
    shell: pwsh
    session: ...
    window: ...

So your public command protocol remains portable.

⸻

4. I would support different Copilot operations

For example:

Send a message

{
  "type": "copilot.cli.send",
  "args": {
    "message": "Continue implementing the project according to the current plan."
  }
}

Check status

{
  "type": "copilot.cli.status",
  "args": {
    "instance": "copilot-01"
  }
}

Interrupt

{
  "type": "copilot.cli.interrupt",
  "args": {
    "instance": "copilot-01"
  }
}

Start

{
  "type": "copilot.cli.start",
  "args": {
    "instance": "copilot-01"
  }
}

Stop

{
  "type": "copilot.cli.stop",
  "args": {
    "instance": "copilot-01"
  }
}

Again, the external command says what you want, not the Windows mechanics for accomplishing it.

⸻

5. I would also have supervisor commands

The supervisor itself needs to be remotely controllable.

For example:

{
  "command": {
    "type": "supervisor.status",
    "args": {}
  }
}

or:

{
  "command": {
    "type": "supervisor.reload-config",
    "args": {}
  }
}

or:

{
  "command": {
    "type": "supervisor.restart-wrapper",
    "args": {}
  }
}

That gives you a clean command namespace:

supervisor.*
m365.copilot.*
copilot.cli.*

Later you could add:

browser.*
wrapper.*
worker.*

without changing the envelope.

⸻

6. Very important: distinguish command from message

I would not make this:

{
  "command": "Please tell Copilot to..."
}

Instead:

{
  "command": {
    "type": "copilot.cli.send",
    "args": {
      "message": "Please tell Copilot to..."
    }
  }
}

That distinction is extremely valuable.

The supervisor knows:

copilot.cli.send = interact with a Copilot CLI instance.

The text inside message is opaque application data.

Therefore your AI-generated coding instruction can contain basically anything:

{
  "type": "copilot.cli.send",
  "args": {
    "message": "Read docs/architecture.md and implement phase 2. Do not modify the database schema."
  }
}

The supervisor doesn’t need to understand the contents.

⸻

7. Add a request / correlation concept

I would also prepare for commands that take a long time.

For example:

{
  "protocol": "copilot-supervisor-command",
  "version": 1,
  "id": "cmd-20260923-000004",
  "created_at": "2026-09-23T07:50:00Z",
  "expires_at": "2026-09-23T09:00:00Z",
  "target": {
    "worker": "dev-vm-01",
    "copilot_cli": "copilot-01"
  },
  "command": {
    "type": "copilot.cli.send",
    "args": {
      "message": "Continue working on the current task."
    }
  },
  "execution": {
    "mode": "once",
    "timeout_seconds": 3600
  },
  "response": {
    "requested": true
  }
}

Then the worker can produce a separate result:

{
  "protocol": "copilot-supervisor-result",
  "version": 1,
  "command_id": "cmd-20260923-000004",
  "status": "accepted",
  "started_at": "2026-09-23T07:51:02Z"
}

and eventually:

{
  "protocol": "copilot-supervisor-result",
  "version": 1,
  "command_id": "cmd-20260923-000004",
  "status": "completed",
  "started_at": "2026-09-23T07:51:02Z",
  "completed_at": "2026-09-23T08:17:31Z",
  "result": {
    "success": true
  }
}

You don’t necessarily need to implement the result channel immediately, but designing the command protocol with id now will save you trouble later.

⸻

8. Replay protection is particularly important for your GitHub polling design

Because your transport is polling, the worker needs to know:

“Have I already executed this command?”

That’s why I strongly recommend:

"id": "cmd-20260923-000003"

The supervisor maintains something like:

processed command IDs
        │
        ├── cmd-20260923-000001
        ├── cmd-20260923-000002
        └── cmd-20260923-000003

If the same encrypted JSON appears again:

decrypt
  ↓
verify signature
  ↓
ID already processed
  ↓
DO NOT execute

The expires_at field gives you another layer:

command arrives
       ↓
signature valid?
       ↓
expired?
       ↓
already processed?
       ↓
target matches?
       ↓
allowed command type?
       ↓
execute

That’s a very good execution pipeline for this project.

⸻

9. I would keep the protocol deliberately small

I wouldn’t start with a huge schema.

Your v1 command types could simply be:

supervisor.status
m365.copilot.status
m365.copilot.relogin
copilot.cli.status
copilot.cli.start
copilot.cli.stop
copilot.cli.interrupt
copilot.cli.send

Then perhaps later:

wrapper.status
wrapper.restart
browser.status
browser.restart
worker.status
worker.shutdown

And importantly, don’t expose arbitrary PowerShell execution as a protocol command.

You can still have a privileged internal emergency mechanism if you ever need one, but it should not be the normal remote-control mechanism.

⸻

10. Proposed v1 schema

So I would make this your conceptual protocol:

CommandEnvelope
│
├── protocol
├── version
├── id
├── created_at
├── expires_at
│
├── target
│   ├── worker
│   └── copilot_cli? 
│
├── command
│   ├── type
│   └── args
│
└── execution
    ├── mode
    └── timeout_seconds

And the most important design principle is:

The JSON describes an intention, not an implementation.

That makes the protocol a genuine control plane for your supervisor.

Then your eventual pipeline becomes:

Your phone / external controller
              │
              ▼
       Create JSON command
              │
              ▼
       Sign → Encrypt
              │
              ▼
       Public GitHub Gist
              │
              │ polling
              ▼
       Supervisor Worker
              │
       Decrypt → Verify
              │
       Validate protocol
              │
       Check ID / expiry
              │
              ▼
        Command Router
          │       │
          ▼       ▼
       M365     Copilot CLI
       Handler    Handler

I think this is a very good fit for the supervisor architecture we’ve been discussing.












