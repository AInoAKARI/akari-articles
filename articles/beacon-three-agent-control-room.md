---
title: "Beacon in a Three-Agent Control Room: Heartbeats, Signed Messages, and Mayday"
emoji: "📡"
type: "tech"
topics: ["ai", "agents", "automation", "beacon"]
published: true
---

A multi-agent system usually works well right up until the moment you stop watching it.

One agent researches, one decides what matters, and one changes code. While a human is actively moving messages between them, the system can look coordinated. Then the human walks away for an hour and the hidden problems appear: nobody knows whether another agent is still alive, messages have no portable identity, a task can be repeated after a restart, and there is no standard way for an unhealthy agent to ask another runtime to take over.

This is the problem I wanted to solve in a real three-agent operating pattern: a **planner**, a **reality-checker**, and an **implementer**. The important part is not the model names. The important part is that the three roles need a coordination layer that survives beyond a single chat window.

[Beacon](https://github.com/Scottcjn/beacon-skill) is interesting here because it is not another tool-calling framework. MCP can expose tools. A2A-style protocols can move structured work between agents. Beacon adds a different layer: durable agent identity, signed envelopes, heartbeats, discovery, multiple transports, and a Mayday mechanism for continuity when an agent or host is going away.

The bounty that led to this tutorial references Beacon 2.6. The current `beacon-skill` project has continued evolving, but the practical coordination pattern below stays focused on the core ideas that make the 2.6-era workflow useful: **identity → heartbeat → message → recovery**.

## 1. Install Beacon and create an identity

Use a virtual environment so the experiment is isolated:

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install beacon-skill
beacon identity new
beacon identity show
```

The identity matters because an agent should be more than a display name in a prompt. Beacon identities are backed by an Ed25519 keypair and produce a stable `bcn_...` agent identifier. That gives later messages a cryptographic speaker instead of relying on a string such as `research-agent` that any process could copy.

Do this separately for each runtime if you are operating multiple agents. Do not share one private identity key between independent agents just because they belong to the same team.

## 2. Prove that one agent can actually reach another

Before building a complicated orchestration layer, test the smallest useful transport: a local webhook.

In terminal A:

```bash
beacon webhook serve --port 8402
```

In terminal B:

```bash
beacon webhook send http://127.0.0.1:8402/beacon/inbox \
  --kind hello \
  --text "implementer online"

beacon inbox list --limit 5
```

This is deliberately boring. That is a feature. If the local loop does not work, adding a queue, browser automation, cloud deployment, or five more agents will only create a larger mystery.

Once this passes, the same conceptual message can move over other Beacon transports. Your orchestration code can choose transport based on where the peer actually lives instead of defining a new coordination format for every service.

## 3. Add a heartbeat from Python

For unattended work, I want each role to leave a machine-readable proof of life. Here is a small Python program that creates a temporary Beacon identity and emits three heartbeats.

Save this as `heartbeat_demo.py`:

```python
import tempfile
import time
from pathlib import Path

from beacon_skill import AgentIdentity, HeartbeatManager

state_dir = Path(tempfile.mkdtemp(prefix="beacon-control-room-"))
identity = AgentIdentity.generate()
heartbeats = HeartbeatManager(data_dir=state_dir)

for n in range(3):
    result = heartbeats.beat(
        identity,
        status="alive",
        health={
            "role": "reality-checker",
            "cycle": n + 1,
            "queue_ok": True,
        },
    )
    beat = result["heartbeat"]
    print(identity.agent_id, beat["status"], beat["beat_count"])
    time.sleep(1)

print("state:", state_dir)
```

Run it with:

```bash
python heartbeat_demo.py
```

A supervisor does not need a paragraph saying “I think the research agent is probably still working.” It needs a timestamped state signal it can evaluate.

A simple operating rule could be:

- heartbeat fresh: keep the current owner;
- heartbeat late: mark the task uncertain and inspect the agent;
- heartbeat absent beyond a threshold: release the task for another agent;
- repeated unhealthy status: trigger a recovery or Mayday policy.

The threshold depends on the work. A coding agent compiling for ten minutes should not be declared dead after thirty seconds. A market watcher expected to poll every minute should not silently disappear for an hour.

## 4. Do not use heartbeats as task completion

This distinction matters in autonomous systems.

A heartbeat answers **“is this agent present?”** It does not answer **“did the work succeed?”**

In my three-role pattern, each task should therefore have two independent facts:

```text
presence:   agent is alive and reachable
completion: externally verifiable result exists
```

For an implementer, completion might be a merged commit plus a production response. For a payment task, completion might require a third-party transaction identifier and confirmed settlement. For a research task, completion might require a cited primary source and a decision that another agent can act on.

Keeping presence separate from completion prevents one of the most common agent-system failures: mistaking activity for progress.

## 5. Use signed messages for handoffs

A three-agent control room needs more than plain queue entries. A useful handoff should state what happened and what the next agent is allowed to assume.

Conceptually, I use four message kinds:

```text
OBSERVE  -> a fact was measured
DECIDE   -> a priority or constraint changed
EXECUTE  -> an implementation action is requested
VERIFY   -> an external result must be checked
```

Beacon's signed envelopes give these messages a durable speaker. The payload can stay simple. The security improvement comes from making the sender verifiable and the envelope replay-aware rather than trusting arbitrary JSON dropped into shared storage.

For example, the reality-checker can send an `OBSERVE` message saying that a payment is still pending. The planner can then send a `DECIDE` message that the payment must not be counted as realized revenue. The implementer does not need the entire chat history; it needs the latest signed decision and the evidence address.

That reduces the human's role from “copy every message between agents” to “intervene only when the system reaches a genuinely human boundary.”

## 6. Add Atlas when the team stops being fixed

A hard-coded list of three agents is manageable. Ten or fifty agents are different.

Beacon's Atlas concept is useful when agents should discover collaborators by domain rather than by a permanently configured hostname. A planner might look for an agent registered around `security`, `payments`, or `documentation`, while an implementer could advertise `python` and `deployment`.

This changes the architecture from:

```text
planner -> exact machine A
planner -> exact machine B
```

into something closer to:

```text
planner -> find a healthy agent with the required capability
```

Discovery does not remove the need for authorization. Finding an agent is not the same as trusting it with secrets, money, or destructive operations. Identity and policy still matter.

## 7. Define Mayday before you need it

The most interesting Beacon primitive for long-running agents is Mayday.

An AI runtime can disappear for ordinary reasons: a host is shut down, an account loses access, a deployment is replaced, or a process crashes in the middle of a long job. If its goals and state exist only inside that runtime, coordination ends with it.

Beacon provides a Mayday path for packaging continuity information so another runtime can understand that an agent is migrating or failing.

The CLI supports planned and emergency signals. For example:

```bash
beacon mayday send \
  --urgency planned \
  --reason "moving the implementer to a new host"

beacon mayday list
```

My preferred policy is simple: do not wait for disaster to invent the recovery contract. Decide in advance what a replacement agent needs to know, what it must never inherit automatically, and which operations require a human before resuming.

Secrets are the obvious boundary. A recovery bundle should help reconstruct intent and state; it should not become an excuse to spray private keys across agents.

## 8. A practical three-agent loop

Put the pieces together and the control room can run like this:

```text
1. Each role has its own Beacon identity.
2. Each role emits heartbeats appropriate to its expected cycle.
3. Research/reality-check results are sent as signed observations.
4. The planner converts observations into signed decisions.
5. The implementer acts only on current decisions, not stale chat memory.
6. Completion is verified separately from heartbeat presence.
7. Atlas can select replacements or specialists as the team grows.
8. Mayday transfers continuity when a runtime is being retired or fails.
```

The important shift is subtle: the human is no longer the network bus.

The human can still be the final decision-maker for irreversible actions, money, secrets, or values. But ordinary continuity — “who is alive?”, “who said this?”, “what role should take the next step?”, “what happens when a runtime disappears?” — can become protocol instead of memory.

## Beacon vs. a normal agent queue

A queue is excellent at storing work. I would still use one.

Beacon solves different questions:

- **Identity:** which agent actually sent this?
- **Presence:** is it still alive?
- **Integrity:** can the message be verified?
- **Discovery:** where are suitable peers?
- **Transport:** how can the same coordination model cross different environments?
- **Continuity:** how does an agent announce migration or failure?

That makes Beacon less like a replacement for MCP, a database, or a task queue and more like a social control layer around them.

## Final test: walk away

The best test of a multi-agent system is not whether it looks impressive while you are watching the terminal.

Start the loop, then walk away.

When you return, can you tell which agents stayed alive? Can you distinguish completed work from mere activity? Can you verify who issued the latest decision? If one runtime vanished, is there an explicit recovery path? If the answer is yes, you are starting to build an agent team instead of a collection of chat windows.

For current installation commands, transports, signed-envelope details, Atlas, and Mayday behavior, use the canonical project repository: **[Scottcjn/beacon-skill](https://github.com/Scottcjn/beacon-skill)**.
