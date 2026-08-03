---
name: agent-harness
description: Building the loop that gives a language model tools — file access, commands, long-running servers — and keeping the model's picture of the world true. Use when writing or debugging a tool-calling loop, deciding what goes in a system message, replaying conversation history, or reporting the state of things the model started. Complements the `model-client` skill, which covers the chat surface; this covers the agentic layer underneath it.
---

# Giving a model hands without lying to it

Almost every defect in the agentic layer of `~/workspace/kimi` had the same
shape, and it was never the shape anyone expected. It was not a tool doing the
wrong thing. It was **the model being told something false about its own
situation, and then being blamed for acting on it.**

That failure has a signature worth learning to recognise: *the model insists on
something the user can see is untrue.* It reads exactly like confabulation. It
is usually the harness.

> A model reporting accurately on a stale context is indistinguishable, from the
> outside, from a model making things up. The fix belongs to whoever is doing the
> showing.

## The central asymmetry

**The model's own prose is replayed. The world is not.**

An assistant message saying *"your server is running at http://…"* is in the
history for the rest of the conversation. The tool result that started it is
not — most APIs never replay tool output. An event that happened *outside any
turn* — the user stopping that server from your own UI — certainly is not.

So the only evidence in the model's context says the server is up, and it
faithfully reports that. Everything below follows from this one fact.

- **Anything that changes the world outside a turn must be injected into the
  next turn explicitly.** Nothing else will carry it.
- **Say it even when there is nothing to say.** A note that lists running
  servers and is omitted when none are running teaches nothing: absence is not a
  message. "No servers are running" is a fact; a missing paragraph is an
  invitation to keep believing whatever was last said.
- **Tell the model its prose has a half-life.** One standing line, in the stable
  part of the system message: *earlier replies record what was said at the time
  and are not a statement of what is true now; only the current-state note is
  current; for anything else, look again.* From inside there is no way to
  distinguish "fact I verified" from "fact that was true when written".
- **Record why something ended, not just that it did.** "It crashed", "the user
  stopped it" and "it hit a time limit" need opposite responses and are
  otherwise indistinguishable. Carry the exit status and the last of the
  process output.
- **Do not delete the record when the thing ends.** `_processes.pop(pid)` on
  stop is the specific bug: it erased the only evidence that a server had ever
  existed, so nothing downstream could report the stop. Keep the entry, prune it
  on a timer.

## Alive is not reachable

A liveness check on the process is not a check that the thing works.

An app that hardcodes its own port ignores the one you assigned, binds somewhere
unpublished, and keeps running happily. `poll()` says up. If you then compose a
URL from *the port you allocated* and tell the model "you have this running, do
not start a second copy", the model will correctly decline to restart a server
the user cannot open — and look like it is refusing to work.

- **Decide "running" by connecting to the port**, from inside the sandbox,
  against the port that was assigned.
- **Wait for a bind, not for the absence of a crash.** A start should wait
  several seconds for something to listen, and distinguish three outcomes:
  exited immediately, alive but nothing listening, actually serving.
- **Never build a success artefact from a field that is present on failure.** A
  URL composed from "the port we allocated" rather than "it is listening" is a
  link to a corpse. This generalises past ports: any time you construct
  something that *asserts* success, key it on the evidence of success.
- **Carry alive-but-unreachable as its own state** with its own instruction, and
  quote the process's own output back — it usually names the port it really
  took.

## History

**Filter on empty, not on failed.** It is natural to exclude failed turns from
replayed history, and the reason is sound: an empty assistant message teaches
the model that empty replies are acceptable. Do not let that become "exclude
anything with an error set". A turn that did twenty minutes of real work and
then hit a limit carries both an error *and* the only account of what it
changed. Dropping it leaves the model with no record of its own actions, and a
model with no record guesses. In one conversation this filter had erased 19,335
characters of the model's own work log.

**A round cap should buy a handover, not an error.** When the tool-round budget
runs out mid-task, spend one more completion with no tools offered, asking for:
what changed and was verified, what is still wrong, the exact next step. The
work happened and was paid for; the only thing missing is somebody saying what
state things were left in. Ending on "stopped after N rounds" throws that away.

**Size the budget for the work.** A cap chosen before the sandbox could run
commands will be spent on inspection alone — read, edit, compile, start, curl,
read again. Bound it, because a round is a request and requests are money, but
bound it somewhere the work fits.

## The system message is a cache boundary

Prompt caching works on prefixes. One character that changes between turns —
an uptime in minutes, a timestamp, a count — misses the cache for the **entire**
conversation behind it, and the damage grows as the conversation does.

Measured on one real conversation: 36,096 of 36,202 prompt tokens cached while
the note was static; 1,280 of 59,134 once it carried a running server's uptime.
At a tenth the price for a hit, that is most of the cost of a turn.

**Split it.** Standing rules first, byte-identical every turn. Live state in a
short, timestamped note inserted immediately before the newest user message,
where it costs one uncached tail instead of one uncached conversation.

## Prose and tool calls are not coupled

Nothing in the interface forces a model's description of an action to be
accompanied by the action. A turn can end with *"let me write that file now"*
and `finish_reason: "stop"`, having requested nothing. The description then
stands in the history as though it happened.

- **Verify the effect, never the account of the effect.** Read the file back
  after writing it; that converts a claim into a transcript.
- **Instrument it.** Store `finish_reason` alongside the tool calls you actually
  executed. The pair distinguishes "the model never asked" from "the model asked
  and we dropped it", and you cannot diagnose anything without that distinction.

## Containment, briefly

Covered in the `model-client` skill; the essentials, because they belong to this
layer:

- **Layer it independently.** The API checks the path, a separate service
  re-checks against its own root whatever the API claimed, and that service's
  container has only the permitted tree mounted.
- **No shell.** Path checking does not contain arbitrary commands. `shell=False`
  with an argv array, and no metacharacter category at all.
- **Approvals live server-side**, so a turn paused on a laptop can be answered
  from a phone.
- **Scope to one project folder per conversation**, with autonomy per
  conversation, and never inherit an auto-approving level into a new one.
- Once the model can read files, everything it reads is untrusted input. That is
  what the approval gate is for.

## The habit that catches all of this

Every one of these defects was locally reasonable — clean up a record when a
process ends, build a URL from the port you assigned, don't replay failed turns,
bound the loop, put standing context in the system message. None was careless.
All were invisible from inside the system.

They were found by querying the database and the sandbox directly, and by a
human running `ls`. **Check your own instruments against something outside the
system, periodically and on purpose.** A verifier with no external calibration
is indistinguishable from an oracle, and behaves like one right up until it is
wrong.
