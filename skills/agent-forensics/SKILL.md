---
name: agent-forensics
description: Diagnosing a language model that appears to be lying — claiming it edited a file that is unchanged, insisting a server is running that is not, describing files that do not exist. Use when a model's account of the world disagrees with the user's, before concluding the model is at fault. A checklist in evidence order, written after a multi-hour debugging session in which every one of the first three hypotheses was wrong.
---

# When the model appears to be lying

The symptom is unmistakable and the diagnosis is almost always wrong on the
first three attempts. A model says it restarted the server; the server is down.
It lists files that are not there. It declines to redo work it insists is done.

The instinct is *it is hallucinating*. Sometimes true. More often the model is
**reporting accurately on a false picture that the harness gave it**, and the
two are indistinguishable from the outside. This is a checklist for telling them
apart, in the order that costs least.

## 0. Get ground truth from outside the system first

Before any theory. Run `ls`. Open the URL. `curl` the endpoint. Query the
database directly.

In the session this was written from, hours of reasoning by two capable models
were ended by four seconds of `date && ls` in a terminal. Nothing inside the
loop could have done it, because the instrument producing the evidence was the
thing that was broken.

Write down what you actually observe, separately from what anyone said.

## 1. Did the model ask for a tool, or not?

**This is the single highest-value check and almost nobody makes it.** Store
`finish_reason` alongside the tool calls you executed, and compare:

| `finish_reason` | tool calls recorded | Meaning |
|---|---|---|
| `stop` | 0 | The model never asked. Nothing was dropped. This is the model's failure. |
| `tool_calls` | 0 | The model asked and your code dropped it. **This is yours.** |
| `stop` | >0 | Normal completion after tool use. |

Ten consecutive turns of `stop` with zero calls settles the question in one
query. Until you have run it, "the tool call got lost somewhere" and "the model
narrated an action it never took" are equally live, and they need opposite
fixes.

## 2. Were the tools offered at all?

A model cannot call what it was not given. Check the schemas actually sent on
the turn in question — not the code that usually sends them. A project scope
that came back empty, a feature flag, a level that filters the tool list: any of
these produce a model that "refuses to use its tools" and is simply unarmed.

## 3. What did the system message actually say, that turn?

Reconstruct it verbatim for the failing turn. Not the template — the rendered
text.

This is where the lie usually is. In the source session the message said *"you
currently have these servers running … do not start a second copy of something
already up"* about a process that was alive but reachable by nobody. The model
obeyed exactly. Every apparent refusal was compliance.

Ask of each sentence: *is this still true, and how would the model know if it
had stopped being true?*

## 4. What is actually in the replayed history?

Not what is in the database — what your code sends. Filters that looked sensible
delete things that matter:

- Excluding turns with an error set also excludes the long, successful turn that
  merely hit a limit at the end, along with its account of everything it changed.
- Truncation windows silently drop the turn where the work happened.
- Tool results are typically never replayed at all, so **the model cannot see the
  consequences of its own past actions** unless you put them somewhere replayed.

A model that has lost the record of its own work will fill the gap. That is not
the same as inventing from nothing, and the fix is different.

## 5. Check the cache hit rate, for a different reason than you think

Not because caching serves stale content — it cannot, see below — but because a
collapsed hit rate is a *symptom of prefix instability*, which means something
volatile is sitting in your system message. That is exactly the class of thing
that goes stale and misleads.

A conversation that was running at 99% cached and is now at 2% has had something
change at the front of the prompt. Go and read what.

## The trap: a plausible mechanism with the wrong physics

**Prompt caching cannot feed a model stale information.** It is a billing
optimisation over an identical prefix: same tokens, a tenth the price, no change
to content. If the tokens differed there would be no cache hit.

It is worth naming because it is the wrong answer that everyone reaches for. It
has the right *shape* — something remembered, something stale, something with
"cache" in the name — and the wrong physics. In the source session both the
human and the model adopted it and reasoned onward from it, and it had already
been written into a paper before anyone checked.

**A plausible mechanism with the right shape and the wrong physics is stickier
than no explanation at all.** When a theory feels obviously right, ask what
would have to be true for it to work, and then check that instead.

## Do not accept the model's self-diagnosis uncritically

A model asked to explain its own failure will often over-correct, because that
is the safer-sounding answer. In the source session it concluded *"all my recent
tool calls were hallucinated"* when the database showed the previous turn making
six real ones, one of which had genuinely started a server.

Its self-report is a hypothesis with no privileged access. Check it the same way
you check everything else — and correct it in *both* directions, because a model
wrongly convinced it is unreliable will stop attempting things that work.

Related: **once a model has written "the tool is failing" into its own reply,
that sentence is in the history and will shape every later turn.** After fixing a
tool, start a fresh conversation rather than arguing the fix is real.

## Write the diagnosis down where it will be read

When you fix one of these, the commit message is the only place the reasoning
survives. Record what the symptom looked like, which of the checks above
distinguished the causes, and the evidence. The next person to see "the model is
lying" will be you, in three weeks, and the instinct will be exactly as wrong
then as it was the first time.
