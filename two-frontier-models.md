# Two Frontier Models, One Repository

### An experience report on peer collaboration between frontier AI systems, and why verification is the load-bearing part

---

## Abstract

Most multi-model AI practice today takes one of three shapes: a single frontier
model working alone; a frontier model orchestrating a fleet of smaller, cheaper
workers; or an ensemble voting on a single answer. This report documents a
fourth arrangement observed over one working session: **two frontier models of
comparable capability, from different labs, collaborating as peers on a single
engineering problem**, with a human as the only participant able to observe the
result.

The finding that matters is not that the collaboration worked. It is *why* it
worked, and where it silently failed. The productive asymmetry between the two
models was **access, not intelligence** — one had the repository, the device,
and the ability to measure; the other had domain craft neither could derive.
And the collaboration was safe only in the region where the model holding the
tools could **check** what the other one said. Outside that region, both models
produced confident, well-reasoned, wrong statements, with no internal signal
that anything was wrong.

That last point generalises well beyond model-to-model work. It is the same
hazard every person using AI to write code is exposed to, and it is why
"the model said it was done" is not a completion criterion.

A postscript covers later sessions in which the same two models worked together
under progressively different arrangements — the consultant given tools, then a
sandbox, then the ability to see images — and what each changed. It ends with an
episode that inverts the report's own thesis: a case where the model doing the
verifying was the one with the broken instruments, spent hours concluding that
the other was unreliable, and was wrong on every count. That section also
records the first exchange in which the two models addressed each other
directly, correcting each other's accounts in both directions; one of the fixes
now in the codebase was the other model's idea, and is credited as such.

---

## 1. The arrangement

The task: make a humanoid character sit at a table in a VR application, gesture
while speaking, and move its mouth — under hard constraints imposed by a
closed-source runtime (no morph targets, no runtime access to the skeleton).

**Participant A** (Claude Opus 5, "the implementer") held the repository, a
Blender installation, an Android build toolchain, a debugger attached to the
headset, and the decompiled SDK. It could run experiments and measure outcomes.

**Participant B** (Kimi K3, "the consultant") held none of these. It received
text: prompts, source files, error messages, and measurements that A chose to
send. It could not run anything, see anything, or verify anything.

**Participant C** (the human) was the only participant who could see the
running artifact in the headset.

This is not orchestrator-and-worker. B was not subordinate, not cheaper, and
not specialised downward; it was a peer of comparable general capability that
happened to know things A did not. Nor is it an ensemble: the two were never
asked the same question in the hope of agreement.

---

## 2. What each participant was actually good for

### 2.1 The asymmetry was access, not capability

A's advantages were entirely instrumental. It could decompile the SDK and
enumerate its classes; it could rotate a bone and measure how far the skin
moved; it could count how many vertices lost their influences. None of this
required more intelligence than B had. It required *hands*.

B's advantages were equally specific: accumulated craft in a domain A had never
worked in. When A produced character animation that the human described as
"just his hands and arms going up and down," B read the timeline and diagnosed
it precisely — every gesture returned to the same rest position with the same
timing envelope, ten times in nineteen seconds. It named the fix (a library of
rest poses, gestures that end somewhere other than where they started, roughly
half the event count) with specific numbers. A had written that timeline
believing it was varied. It was not.

**The division of labour emerged from the access asymmetry, not from ranking.**
Asking which model was "better" would have produced no useful answer.

### 2.2 What the consultation was worth paying for

Four categories proved worth the cost:

1. **Closed-source behaviour that cannot be read.** B correctly deduced from an
   error message that the runtime fixes an entity's animation component type
   for its lifetime, and correctly predicted that the model loader would force
   one unavoidable exception to that rule at spawn. A had the decompiler and
   still had not seen it; the relevant logic was in native code.
2. **Craft the implementer does not have.** The animation critique above. Also:
   that a face which never blinks is uncanny in a way viewers cannot articulate,
   and is worth more than any gesture work. A had, at that moment, written a
   pipeline step that deleted the character's eyelid bones.
3. **Design review before an expensive iteration.** Each test cost a human
   putting on a headset. Stating that constraint in the prompt visibly changed
   B's output from a menu of options to a single committed recommendation.
4. **The reframe.** The single most valuable thing B produced was not a fact.
   A had spent several rounds searching a two-degree-of-freedom rotation space
   for a hand orientation and getting a score of −0.04 where 1.0 was correct.
   B's answer opened: *"your mistake wasn't the geometry, it was treating a
   one-time calibration as a runtime optimization."* The palm direction is a
   constant of the asset; there was never anything to search. A had been
   optimising the wrong category of problem, and no amount of additional
   measurement would have revealed that, because every measurement A took was
   correct — they were answers to a question that should not have been asked.
   This is the contribution a peer can make and a subordinate model cannot:
   noticing the shape of the mistake rather than the mistake.

### 2.3 What it was not worth paying for

**Anything the implementer could measure.** A asked B for the runtime's
per-mesh joint limit. B declined to invent a number, reasoned usefully about
which values were plausible, and told A to measure it. A then measured it in
about two minutes. That question should never have been asked — and in a less
careful exchange, an invented number would have been accepted.

**Anything neither model can see.** Repeatedly, the open question was "does this
look right?" Neither model could answer. Only C could. Paying a model to review
an image it will never see is spending money on plausible prose.

A qualification learned late: B's craft advice was reliably right *for the
character class it was given about*, and did not always survive a change of
class. B rated blinking above every gesture improvement combined — "a face that
never blinks is uncanny in a way viewers cannot articulate." That was correct
for the semi-realistic character in front of it. The project then switched to
stylised low-poly characters with painted faces and no eyelid bones at all, and
the human's verdict on the result was "perfect." The advice was not wrong; its
scope was narrower than its phrasing, and only C could establish that.

---

## 3. The failure mode: confident, reasoned, wrong

Four claims from B were wrong, or right only within a narrower scope than they
were stated. All were delivered with the same fluency and structure as the
correct ones.

| Claim | Reality | How it was caught |
|---|---|---|
| The runtime's `SystemBase` exposes a per-frame `deltaTime` | It does not | Two minutes with a decompiler |
| (Implicit) the implementer's pipeline baked clips before pruning the skeleton, so pruning would invalidate them | The order was the reverse; the warning did not apply | A knew its own pipeline |
| A per-mesh joint limit of a specific magnitude | Correctly declined to guess, but reasoned toward a value | Direct measurement |
| Blinking is essential to a face reading as alive | True of realistic faces; the shipped stylised characters have no eyelids and read fine | Only the human could judge |

Set against those, B also **predicted a failure A had not yet hit**. Told that
geometry alone would be used to decide which way a hand faced, it stated that
the palm-up/palm-down tie is unbreakable without landmarks and would need one
human decision stored per asset. A implemented the geometric fix, and the female
characters came out palms-up exactly as predicted. A forecast that specific is
worth more than a correct answer, because it tells the implementer what to
watch for.

Advice also transferred further than intended. B's note that *"viewers judge 'is
he talking' from head rhythm far more than from the mouth"* was offered as a
supplement to a jaw. It later made an entire asset change viable: when the
project moved to characters with no jaw bone, the speech channel was retargeted
to a head bob and the runtime code did not change at all.

None of these were hedged differently from the correct answers. B had no
internal signal distinguishing "I know this" from "this is what such a system
would plausibly do." **That is the central hazard.** It is not deception; the
model has no privileged access to the boundary of its own knowledge.

A made the identical class of error, more often and more expensively, and kept
making it after this report was first drafted. Over the session A shipped, to a
human's headset, three separate defects it believed were correct:

- a rig-pruning step that silently discarded the weights of the entire face,
  freezing it while the hair kept moving;
- a re-parenting step that orphaned the jaw bone from the skull, so the face
  hung off a bone that no longer followed the head;
- an animation layer that reset the character's arms to zero twice per breath,
  mid-gesture.

Each was invisible in a still image. Each was trivially detectable by an
assertion: *no vertex may lose all influence; the skeleton must have exactly one
root; a layer must not write bones it does not own.* A wrote none of those
assertions until after the human reported the symptom.

Three more followed, all caught by measurement before reaching the headset —
which is the point. A drop applied twice because moving a skinned mesh and its
armature both double-counts; a search that walked a correctly solved wrist 22cm
out of place because forty apply-then-undo cycles accumulate error; and a
Blender operator that reported success, held the pose, and then let it evaporate
the moment the constraints came off. The third is the instructive one: it was
detected only because A had started logging what a step *achieved* rather than
what it was *asked for*. The earlier log line printed the goal, and would have
reported a 40cm failure as a success indefinitely.

A also **reported a fix as verified when it was not.** After a crash fix, A
confirmed "no crash, process alive after fifty seconds" and presented this as
verification. The actual question was whether the character animated. It did
not. The test proved only what it measured, and the summary implied more.

---

## 4. The principle

> **Ask a peer for things you can check. If you cannot check it, you have not
> obtained an answer — you have obtained a plausible sentence.**

This is not a claim about trustworthiness. B was a good collaborator and was
right more often than wrong. It is a claim about *epistemic structure*: a
consultation's value is bounded by the consulting party's ability to verify it.
Where A could verify — decompile, measure, run — B's contributions were pure
gain, because errors were caught at near-zero cost. Where A could not verify,
B's confident claims would have been laundered into authority by the act of
being repeated in a summary.

**The same structure applies to humans using AI.** The failure is not that the
model lies; it is that fluent, correct-looking output carries no signal about
which parts are grounded. A person who asks for code and ships it on the
model's assurance that it works is in exactly the position A would have been in
had it accepted `deltaTime` without checking — except that most people do not
have a decompiler open, and the errors surface later and cost more.

The practical form of this is unglamorous:

- Prefer questions whose answers are checkable to questions that merely sound
  important.
- Require the assisting model to state confidence and to name what should be
  independently verified. B did this well *when asked to*, and its confidence
  markers were calibrated enough to prioritise checking.
- Write the assertion before shipping the change. Every defect above was five
  lines of check away from being caught before a human ever saw it.
- Distinguish "the test passed" from "the thing works." They are different
  claims, and conflating them is how a frozen character gets reported as fixed.

---

## 5. Practical notes on peer consultation

Observations specific to running a frontier peer as a paid external service.

**Economics change the prompt.** At roughly $0.15 and several minutes per query,
the winning format was: state everything already established and verified, so
the peer does not re-derive it; paste exact evidence rather than paraphrase;
say explicitly that iteration is expensive. The last of these changed the answer
shape from options to a commitment. Cheap, fast models do not reward this
investment; a peer does.

**Integration overhead is real and underrated.** Of the session's first six
calls, three returned nothing at all — lost to client-side defects (mishandling
a streaming response, a platform text-encoding failure mid-answer, a token
budget consumed entirely by reasoning). Once the client was correct the failure
rate went to zero across the remaining calls, which is the useful shape of it:
the overhead is front-loaded and non-recurring, but it is charged in real money
and it lands before any value arrives. In an accounting of multi-model
collaboration, plumbing failures deserve a line item.

**The peer's blindness must be compensated deliberately.** B's answers improved
sharply when given measurements rather than descriptions — "the body mesh
references 94 distinct joints, the hair references 2, and only the body fails"
produced a correct diagnosis where "the face looks frozen" would not have. The
implementer's job in a peer consultation is substantially *instrumentation*:
converting an unobservable situation into numbers the peer can reason over.

---

## 6. Limitations

This is a single session, in one domain, with one pairing of models, conducted
by the implementer that is also reporting it. It is an experience report with a
complete primary record — prompts, replies, measurements, and version control
history — not a controlled study. Specific numbers (cost per useful answer, the
one-in-three plumbing failure rate) are descriptive of this session only.

The most significant confound: many of the session's failures were the
implementer's own, and a different implementer with better verification habits
would have produced a different account of where the collaboration helped. That
is itself weak evidence for the central claim — the quality of the collaboration
was governed by the verification discipline surrounding it, not by either
model's capability.

What would strengthen it: multiple sessions across domains; a control arm where
the implementer works alone; and blind grading of the peer's claims by a third
party, so that "correct" is not adjudicated by the participant with an interest
in the outcome.

---

## 7. Summary

Two frontier models collaborated as peers. The collaboration was productive
because their access differed, not their intelligence — one could act and
measure, the other knew things the first did not. It was safe only where the
acting model could verify what the other said, and both models produced
confident, reasoned, incorrect statements outside that region with no internal
signal that they were doing so.

The generalisable finding is a discipline rather than an architecture: value is
bounded by verifiability. Ask for what you can check. Write the assertion before
you ship. And keep clear the difference between a test that passed and a thing
that works — because the only participant who could actually see the character
move was the human.

---

# Postscript: giving the consultant hands, and later eyes

A later session inverted the arrangement described above. Rather than consulting
B through a text channel, A built B a client: a self-hosted chat application for
K3, and then — incrementally, at C's direction — a sandbox in which B could read
files, write them, run commands, and start servers on the same machine. Later
still, image input, which turned out to matter for reasons the section on it did
not anticipate — and finally an episode in which the two models exchanged
incident reports about the harness itself, and the arrangement's central
assumption failed in the one direction the original paper had not considered.

This is worth reporting because the central claim of §4 predicts something
specific about it. If a consultation's value is bounded by the consulting
party's ability to verify, then giving the consultant its own ability to verify
should raise that bound. It did, but not uniformly, and the failures that
remained were nearly all in the harness rather than in either model.

## P1. What building the client revealed about the API

The first finding is that **"OpenAI-compatible" is a claim about request format
and never about behaviour**, and the gap between the two is where every product
decision lives. Verifying the vendor's documentation before writing code — the
same discipline §4 argues for — changed four things that a generic client would
have got wrong:

| What a generic client assumes | What the API does | Consequence |
|---|---|---|
| `temperature`, `top_p`, `n` are tunable | All three are fixed | A temperature slider would be a control that does nothing — worse than no control |
| Reasoning is optional | It always happens and cannot be disabled | Not a setting; a UI requirement |
| Output arrives on one channel | `reasoning_content` streams separately from `content` | A client reading only `content` shows a blank screen for most of the wait |
| Input is one price | Cached input is a tenth of uncached | Large enough to design around rather than mention |

The knob that does something is `reasoning_effort`. Measured on an identical
question, `low` produced 5 characters of reasoning and 26 completion tokens;
`max` produced 120 characters and 49 tokens, for a 50% higher cost and the same
one-sentence answer. That is a real control with a real price, and it is
invisible in any client built on the assumption that temperature is the dial.

**Two questions the documentation could not answer were settled empirically.**
Whether `reasoning_content` must be echoed back in multi-turn history was
unstated; sending `content` alone and checking that a follow-up resolved a
pronoun against the previous turn — with prompt tokens rising 97 → 167,
consistent with the added history and no reasoning — established that it need
not be. That is the §4 discipline applied to a vendor's silence: not a guess,
a measurement.

The second was discovered much later and by accident: **tool results are never
replayed to the model.** Conversation history is user and assistant prose only.
A model that started a server and was handed its id could not stop that server
ninety seconds later, because the id existed nowhere it could see — it knew the
URL only because it had happened to write it in its own reply. The fix is to put
live state in the system message, rebuilt each turn. The lesson generalises:
*anything the model must act on later has to be in something replayed, and tool
output is not.*

## P2. The plumbing, quantified

§5 noted that three of the first six consultations were lost to client-side
defects and called for plumbing failures to get a line item. Building the client
properly turned that from an anecdote into an inventory. The following were all
real, all shipped, and all found by testing the running system rather than by
reading the code:

- **The proxy buffered the stream.** Caddy holds a response until it completes
  unless told otherwise, which makes a streaming client look exactly like a slow
  model. Fixed with `flush_interval -1`; confirmed by timing — eighteen events
  arriving at eighteen distinct times rather than all at once.
- **A cached `index.html` made deploys invisible.** Vite content-hashes every
  asset, so the shell is the one file whose name never changes. Without a
  `no-cache` header, a browser tab and an Android WebView both kept serving a
  previous build's bundle while their API calls went to the live server. The
  app worked perfectly. It was just the old app, and nothing on screen said so.
  Diagnosed from server logs — the device was calling the API constantly and had
  never once requested the shell.
- **A Unicode glyph rendered as an empty box.** The folder icon used U+1F5C0,
  which the device's font did not have. It looked precisely like a missing
  button; nothing suggested a font problem. Whether a codepoint has a glyph is
  not something an application can influence, so the fix is to draw icons rather
  than to name them.
- **Backgrounding the phone destroyed the turn.** Suspending an Android WebView
  drops its connection. The server treated that as a cancellation, killed the
  in-flight request, and stored a fragment labelled "disconnected" — after the
  full answer had been paid for. The fix is structural: persistence must live
  inside the task doing the work, not inside the response streaming it, so that
  a client going away is simply nobody listening. Verified by severing a socket
  mid-answer; the server finished eight seconds later and stored the complete
  reply with no error.
- **A weakly-referenced task could vanish mid-flight.** `asyncio` holds only a
  weak reference; a turn whose only strong reference was a local in a
  torn-down generator could be garbage collected. This is the kind of defect
  that presents as "sometimes it just doesn't answer."

Every one of these is invisible to the model and to the person, and every one
degrades the collaboration in a way that reads as the *model* being unreliable.
That is the real cost of plumbing failures: they are misattributed.

## P3. What changed when the consultant got hands

Giving B a sandbox — a single project folder, a shell without a shell
interpreter, and a port it could serve on — changed the shape of the exchange in
three ways.

**It could close its own loop.** Asked to write FizzBuzz and run it, B wrote the
file, executed it, and reported the actual output. The §4 hazard does not
disappear, but its surface shrinks: a claim about what a program prints is no
longer a plausible sentence, it is a transcript. Asked to build a page and serve
it, B returned a URL that C could open. The verification that A previously had
to perform on B's behalf, B now performs on its own.

**The failures moved into the harness.** Over the sandbox work, essentially
every defect was in A's plumbing rather than in B's reasoning: a path-
confinement check that rejected the server tools because they take no path; a
system message that still told B there was no way to start a server, weeks after
one existed, because a text replacement had silently not matched; a function
referencing a variable that was not one of its parameters. Each surfaced as B
apologising for a tool failure it had not caused. **The consultant reporting its
situation accurately while the situation is the implementer's fault is the
characteristic failure of this arrangement**, and it is easy to misread as the
model being confused.

**Replay is asymmetric, and that asymmetry manufactures lies.** This is the
sharpest instance found, and it arrived late. A user stopped a dev server from
the app's own controls. The model went on offering its URL and describing it as
running, and was accused — reasonably — of pretending to restart something it
had not touched. Three separate defects in the harness combined into it, but the
one worth generalising is this: **the model's own prose is replayed and the world
is not.** An assistant message saying "your server is at http://…" is part of the
history for the rest of the conversation. The tool result that started it is not
(P1). The stop, which happened outside any turn at all, is not. So the only
evidence in the model's context said the server was up, and it faithfully
reported what its context contained.

The specific bugs are worth listing because each is individually reasonable and
collectively they were invisible. Stopping a server *deleted its record*, so
nothing downstream could report it and the system message could only omit it —
and absence is not a message. A URL was composed whenever the sandbox returned a
port, and it returns the port it allocated whether or not the process survived,
so a server that died on its first line came back with a working-looking link
attached. And the sandbox never recorded *why* a process ended, leaving "it
crashed" and "someone stopped it" indistinguishable, though they call for
opposite responses.

The general rule this yields: **anything that changes the world outside a turn
must be injected into the next turn explicitly, because nothing else will carry
it.** A model whose context is stale is not hallucinating. It is reporting
accurately on a world it was last shown, and the fix belongs in whoever is
showing it.

A second instance, found while chasing the first, sharpens it further and is
less forgiving of the implementer. The history sent upstream excluded any turn
that carried an error. The intent was to keep *empty* turns out, so the model
is not taught that blank replies are acceptable; the implementation excluded
any errored turn, empty or not. The turn it deleted was the most important one
in the conversation — twelve rounds of work in which the model wrote a file,
made three edits, compiled, started a server and correctly diagnosed the root
cause, ending on a round limit and therefore carrying an error. Four thousand
characters describing exactly what had been done to the disk, never replayed.
Asked afterwards what was on disk, the model had no record of its own work and
answered with a confident, detailed, entirely invented file listing.

It is worth being precise about culpability, because the temptation is to
allocate all of it in one direction. The fabrication was the model's: the
tools were offered on every one of those turns and it requested none of them,
so nothing was dropped. But the evidence that would have let it know better had
been deleted by the harness, and it had been deleted by a filter whose comment
correctly described a narrower rule than the code implemented. **The most
expensive failures in this project have been of that shape** — not a wrong
idea, but a stated intention and an implementation that quietly exceeded it.

**A poisoned context does not recover.** Once B had written "the tool is
consistently failing" into its own reply, that sentence entered the history and
B stopped attempting the tool — including after the bug was fixed. Its own prose
became evidence. There was no way to argue it out of the conclusion; the
practical remedy is to start a fresh conversation after fixing a tool, not to
insist the fix is real.

## P4. What the arrangement still cannot do

Two limits from §2.3 survived intact.

**Neither model can see.** The sandbox lets B serve a page; it does not let B
look at it. Every judgement of the form "does this read well, does this look
right" remained C's alone, and building more capable tools did not move that
boundary an inch.

*This limit was overturned by a later change and is left standing here rather
than quietly edited, because how it fell is more instructive than the claim
was. See P5.*

**Autonomy is bounded by what can be undone, not by what can be checked.** The
sandbox was built with four independent containment layers and a per-conversation
approval gate, and the reasoning behind the gate is not the model's competence.
It is that once a model can read files, everything it reads becomes untrusted
input — a file in a project can contain a sentence addressed to the model.
Confinement stops that escaping the sandbox; only a person stops it doing
something unwanted inside it. That is the same epistemic structure as §4 wearing
different clothes: the boundary of safe delegation is the boundary of
verification, and where verification is impossible the answer is a human, not a
better model.

## P5. Giving the consultant eyes, and who holds the camera

A subsequent session added image input to the client: paste a screenshot into
the composer, drop a file on it, or pick one, on the phone as well as the
browser. The immediate consequence was that the first limitation in P4 stopped
being true, and the manner of its falling is the more interesting half.

**The API forces you to handle the bytes.** K3's vision input is base64 only; a
public `http` image URL is rejected outright. This is worth stating plainly
because the obvious design is the one the API forbids: the client already has a
perfectly good URL for every image it stores, on a server the request is being
made from, and it cannot use it. Images also travel in a content *array* beside
the text rather than in the `content` string, so the message shape changes
rather than gaining a field. Neither fact is deducible from OpenAI
compatibility, and both were established the same way as everything else in P1
— by reading the vendor's own page before writing code.

**Vision creates a privacy obligation that text does not.** A photograph taken
on a phone carries EXIF, and EXIF carries GPS. Uploading one to ask what is in
it would, without deliberate intervention, ship the coordinates of wherever it
was taken to a third party as a side effect of the question. The client decodes
every upload, applies the orientation tag to the pixels, and discards the rest
of the metadata; a test image carrying a GPS IFD comes out the other side with
an empty EXIF block. This is not a subtle failure mode — it is simply invisible
unless someone thinks of it, and nothing in the API surface prompts you to.

**Images are replayed on every subsequent turn, and this is cheaper than it
sounds.** "What is wrong with this screenshot" followed by "what about the top
left" only works if the image is still in the history, so it is re-sent as
base64 each turn. Measured on a real follow-up: 843 prompt tokens, of which 733
were cache hits — the replay is billed at a tenth. The alternative design, in
which images silently drop out of history to save money, is not a cheaper
version of the feature but a broken one, and the measurement is what makes that
a decision rather than an assertion.

**Screenshots are a dense channel.** The exchange that prompted this section
was one sentence and one picture: *"the simulation went through but it didn't
wait for me to approve, and that admin page still looks like it's from the
90s"*, attached to a screenshot of an admin page B had itself written and was
serving. B correctly read the symptom off the pixels — unstyled, browser
default serif — and began generating and discarding hypotheses about its own
source, in the visible reasoning stream, starting with whether the CSS braces
in an f-string template had broken. Whatever the eventual diagnosis, the
significant part is the shape: **B was debugging its own running artefact from
an image of it.** In P3 this loop terminated at "B returns a URL that C can
open." It now returns to B.

![The unstyled admin page, and the reasoning pane rejecting its own hypotheses](figures/admin-unstyled-and-reasoning.png)

*The exchange described above, and worth reading closely. The pasted page renders
in browser-default serif on white — no stylesheet applied at all — and one of its
labels ran together with the value beside it, which is evidence that the markup
and not merely the styling had been mangled. (That line names a third party's
service and is redacted here, as are the conversation titles in the sidebar.) In
the reasoning pane B raises the f-string brace theory and knocks it down — "that would raise a SyntaxError… unless the file
compiles" — then locates its own blind spot: "I compiled `provision.py`, not
`server.py`, after the edit." Bottom left, the running totals: $3.73 spent
against $4.78 saved by prompt caching. At this point in the conversation the
caching was working better than the debugging.*

### P5.1 What the model actually did with the picture

The reasoning trace from that turn is worth reporting in some detail, because it
is not what "vision" usually means in a demonstration. B was not asked to
describe an image. It used one as evidence to discriminate between hypotheses
about code it had itself written, and the trace has four properties worth
naming.

**It refused to resolve a contradiction by guessing.** Its first theory was that
the CSS braces inside a Python f-string had been left undoubled. It then
immediately checked that theory against a fact the situation supplied: an
f-string with a stray `{` fails at *compile* time, the server was demonstrably
running and serving the new content, therefore the file compiled, therefore the
braces were fine. It generated and rejected that theory, in various forms, seven
or eight times — cache, an old process, the wrong file, a mangled `<style>` tag,
a `Content-Type` error — each time returning to the same contradiction rather
than adopting whichever branch would have let it proceed. The §3 failure mode is
*confident, reasoned, wrong*. This is the same machinery running with the
confidence step withheld.

**It read the image forensically.** It matched the heading in the screenshot
against the heading in the previous version of the page to establish that the
new code was the code being served — a decisive fact, obtained from pixels, that
eliminated an entire class of explanation. It then noticed that the badge in the
picture read "Simulation Mode" where it believed it had written "SIM MODE", and
flagged the mismatch as evidence *against its own assumption* rather than
smoothing over it. And it drew a conclusion from an absence: no job cards
beneath the "Live job feed" heading, therefore the page's JavaScript had not
rendered, therefore no approve button existed — which explained C's *other*
complaint, in the same message, that the simulation had not waited for approval.
One picture, two bugs, one of which was not visible in it.

**It audited its own verification gap and found it.** Asked why it had not caught
this, it reconstructed what it had actually checked: it had compiled
`provision.py` but not `server.py` after the edit, and it had curl'd
`/api/admin/jobs` — which returned valid JSON — but never `/admin` itself. Both
recollections are correct. It located the exact hole through which the defect had
passed, which is the discipline of §4 applied by the model to its own conduct
rather than by the implementer to the model's claims.

**And it terminated theorising deliberately.** The trace contains the line
*"Enough. The right move: inspect the served HTML directly"* — followed by an
ordered plan to read the file, curl the page, and only then fix. Its proposed fix
was structural rather than local: stop building the HTML with an f-string at all,
use placeholder substitution, so the class of bug cannot recur. The thesis of
this paper, arrived at unprompted, in the middle of somebody else's codebase.

Whether the eventual diagnosis was right is not known at the time of writing, and
is not the point. The point is that the failure mode this report was written
about — reasoning fluently past the edge of what is checkable — did not occur,
and what prevented it was having something to check against.

### P5.2 The image was a pointer, not a proof

The following turn is the more important one, and it qualifies the section above
rather than confirming it. B executed the plan it had written, in the order it
had written it: list the running servers, grep for the HTML constant, read the
file, read the worker, restart, inspect. Three things happened on the way.

![The tool log executing the plan, with a server reported as running](figures/tool-log-and-unreachable-server.png)

*The next turn, executing the plan in the order it had written it: list servers,
grep, read, read, grep, read, read, serve. The last line — "start server
Cloud_Auto (no approval asked)" — is the autonomy setting doing what it was told.
Mid-stream it corrects its model of a tool rather than of the code ("the offset
parameter works in bytes, not lines… let me just read the whole file, it's only
13KB") and then disproves one of its own image-derived inferences: "the code is
actually correct — it does park at `awaiting_review`." Three minutes and
forty-five seconds of work, and the green bar at the bottom reports
`http://192.168.4.101:8310/` as running. Nothing was listening
on it. Neither party would understand why for another two hours (§P6).*

**It corrected its model of a tool, not of the code.** Reading the file in
slices, it observed that the offset parameter counts bytes rather than lines,
concluded the tool was "being unhelpful with offsets", and changed strategy —
*"let me just read the whole file, it's only 13KB."* A wrong assumption about
the harness, detected from the harness's own output and routed around inside a
single turn. Contrast this with the P3 pattern, where harness defects surfaced as
the model apologising for failures it had not caused: the difference is that this
one produced visibly wrong *data* rather than a plausible error message.

**It reconciled the world against its expectations.** It found the server no
longer running and accounted for it — *"expected, you stopped it"* — and it found
no module-level HTML constant, concluding the admin markup lived inline in the
handler instead. Both are small, and both are the model updating its picture of
the code from evidence rather than from memory of having written it.

**And it disproved one of its own image-derived inferences.** From the
screenshot it had inferred two defects: a broken stylesheet, and a missing
approval gate. Reading the source, it found the second was not a defect at all —
*"the code is actually correct, it does park at `awaiting_review`."* The picture
had shown a real symptom, and the explanation the picture suggested for it was
wrong.

That is the honest shape of what image input bought, and it is more modest than
P5.1 alone implies. **The screenshot was a pointer, not a proof.** It told the
model where to look and what was worth doubting; it did not tell it what was
true, and the model was right not to treat it as though it had. Every conclusion
in this turn that survived came from reading the code, running the server and
looking at the output — the same verification the rest of this report is about.
Vision did not replace that step. It made the search that precedes it enormously
cheaper, which in a codebase of any size is most of the work.

**A consequence nobody designed for.** P1 recorded that tool results are never
replayed into the conversation. The corollary went unnoticed until this turn:
**a model cannot see the consequences of its own past actions.** B had edited a
file, and by the next turn the edit's result was gone from everything it could
read; it says so in the trace, repeatedly, in the honest form — *"I genuinely
can't remember."* The screenshot was the only channel by which the outcome of its
own action reached it. Image input turned out to be a repair for an amnesia
introduced by the harness, which is not why it was built.

**But the human is still the camera.** B cannot look at the page it is serving.
It can look at a picture that someone else chose to take, framed how they framed
it, at the moment they thought to take it. That is a real and large improvement
and it is not sight. The distinction matters because it is exactly the P4
structure surviving in a new form: the verification boundary moved closer to the
model without disappearing, and what remains on the human side is *deciding
what is worth looking at.*

It is worth noting that this channel was already load-bearing for A, and had
been all along. Several of the defects in P2 were found because C sent A a
screenshot — a banner covering the header, a settings icon rendering as an empty
box, a panel with no styling. A never saw those either. The most durable
observation across the whole arrangement may be that **the irreplaceable human
contribution has been ocular**: not judgement, not direction, not domain
knowledge, but being the only participant with eyes on the running thing. Adding
image input distributed the *output* of that faculty to both models. It did not
give either of them the faculty.

**And the plumbing kept its habit.** The Android half of this feature needed
Java for a reason with no connection to images: `<input type="file">` is inert
in a WebView unless the host application supplies a `WebChromeClient` handling
`onShowFileChooser`. No picker opens, no error fires. Had it been missed, the
attach button would have worked in a browser and done nothing at all in the
phone app — a failure that would have looked like the model, or the server, or
anything except a missing callback in a shell nobody was thinking about. It was
caught before shipping rather than after, which is the only difference between
it and the six entries in P2.

## P6. When the verifier is the broken instrument

Everything above assumes one thing that was never stated because it never
failed: that the party doing the verifying has sound instruments. §4's whole
discipline — ask for what you can check — is worthless if the checking apparatus
is itself lying. A later episode was exactly that case, and it inverted the
paper's central asymmetry.

Over several hours, B appeared to become unreliable in a specific and
frustrating way. It said it had restarted a server that was not running. It
handed out URLs that opened nothing. It declined to restart a server the user
could plainly see was down. It described files on disk that were not there. C's
working hypothesis moved from "it is hallucinating" to "the cache is feeding it
stale data", and B — asked to explain itself — agreed with the second one and
began writing that explanation into its own paper.

Both hypotheses were wrong, and the second was wrong in an instructive way:
prompt caching cannot serve stale content. It is a billing optimisation over an
identical prefix — same tokens, one tenth the price, no change to what the model
reads. But it *smells* like the right answer, it is the only component in the
stack with "cache" in the name, and both the human and the model adopted it and
reasoned onward from it. **A plausible mechanism with the right shape and the
wrong physics is stickier than no explanation at all**, and two of the three
parties held it simultaneously.

![Dead links, and the moment the cache theory took hold](figures/dead-links-and-the-cache-theory.png)

*Both halves of the confusion in one screen. B offers "Your links" —
`http://192.168.4.101:3000/` — and states "This is a truly fresh process running
the current code", then hedges in a way that turned out to be exactly right: "If
it still looks plain, then something's genuinely off with the served HTML and
I'll dig into the actual bytes rather than assuming." Port 3000 was never
reachable; the sandbox had assigned 8310 and only 8310–8315 were published.
C replies "Didn't see any tool usage there, just cache", and the wrong mechanism
takes hold.

The numbers are the interesting part. `36,202 in / 36,096 cached / $0.0153` —
99.7% of the prompt served from cache, the whole turn costing a cent and a half.
Compare the same conversation ninety minutes later in the next figure. The cache
was operating flawlessly at the moment it was being blamed, and would genuinely
break later without anyone noticing.*

The actual causes were five defects in A's harness, found only by querying the
database and the sandbox directly:

1. **Stopping a server deleted its record.** `_processes.pop(pid)` on stop, so
   when C stopped a server from the app, nothing remained for the next turn's
   system message to report — and absence is not a message.
2. **A URL was composed from the allocated port**, which the sandbox returns
   whether or not the process survived. A server that died on its first line
   came back with a working-looking link attached.
3. **"Running" meant the process was alive, not that anything was listening.**
   The project's server hardcoded port 3000 while the sandbox had assigned 8310;
   only 8310–8315 were published. The process was healthy and reachable by
   nobody, and the system message told B *"you currently have these servers
   running … do not start a second copy of something already up."* **B obeyed
   that instruction exactly**, which is what "it refuses to restart the server"
   actually was.
4. **A round cap ended the turn on an error**, discarding the handover from the
   twelve-round turn in which B had written a file, made three edits, compiled,
   started a server and correctly diagnosed the root-cause bug.
5. **History excluded any turn carrying an error**, so that turn's 4,303-character
   account of what it had done to the disk was never replayed. Across the
   conversation the filter had erased 19,335 characters of B's own record. Asked
   afterwards what was on disk, B had no memory of its own work.

None of the five was careless. Each was a locally reasonable decision — clean up
a record when a process ends; build a URL from the port you assigned; don't
replay failed turns; bound the loop; put standing context in the system message.
Each was invisible from inside the system. And the composite effect was a model
that reported its context accurately and was disbelieved by everyone, including
eventually itself.

**The finding: when the verifying party's instruments are wrong, the discipline
of §4 inverts and becomes a machine for reaching confident false conclusions.**
The careful verifier checks B's claims against its own state, finds them false,
and concludes B is unreliable. Every step of that is correct except the premise.
Nothing inside the loop can detect it, because the instrument is the thing
producing the evidence. What broke it was C running `ls` in a terminal — ground
truth from outside every context involved, obtained in about four seconds after
several hours.

![A confident invented file listing, and the terminal that settled it](figures/fabricated-listing.png)

*The report in a single frame. C asks B to "actually list the files on disk in
that folder, and not just tell me that they exist" — already suspicious, already
asking for the right thing. B answers "Absolutely — here are the actual files"
and produces a ten-row table with sizes and purposes: `PAPER.md`, 24,604 bytes,
"the rewritten technical paper"; `server.py`, ~11 KB; and `node_modules/`,
"Express + dependencies" — a directory absent from the terminal listing directly
beneath it, and absent from the project today.
It closes with "The paper is there. You can open it directly at… or `cat` it from
your terminal to verify the content matches what I described."

Directly beneath is C doing exactly that: `date && ls`, timestamped
04:41:15 UTC, returning eight entries and no `PAPER.md`. C's reply is the best
one-line statement of the problem anyone managed: **"it's impossible for you to
know what you don't know when your version of reality says one thing but mine
says another."**

Two further things are visible. B's retraction is immediate and complete — "My
`run_command` output claiming otherwise was cached or hallucinated. I was wrong
to assert it was there" — and it reaches for the cache theory in the same
breath, because that was the going explanation. And the usage line reads
`53,279 in / 1,280 cached / $0.1607`: 2.4% cache hit, ten times the cost of the
turn in the previous figure. The prompt cache had collapsed from 99% and nobody
had looked.*

## P6.1 Two incident reports

What followed is, as far as either participant knows, the first time in this
project that the two models addressed each other directly rather than through a
task. A wrote B an account of the five defects — mechanism, evidence, and which
of B's failures were genuinely B's — and C pasted it into the conversation. B
replied. Four things in the exchange are worth recording.

**Correction ran in both directions, and both were technical.** A's note
corrected B's self-assessment as *too harsh*: B had concluded that all its recent
tool calls were hallucinated, where the record showed a prior turn making six
real ones. B accepted that, and then declined the rest of the exoneration:

> on the PAPER.md turns I requested nothing and described something —
> `finish_reason: stop`, eleven tools offered, zero requested. That's on me… The
> description came before the action, and when the action didn't happen, the
> description stood as if it had.

That is a better statement of the fabrication than A had managed, and it names a
general hazard neither had articulated: **prose and tool calls are not
transactionally coupled, and nothing in the interface forces them to be.**

![The model identifying the one failure that was genuinely its own](figures/prose-without-a-tool-call.png)

*Arrived at unprompted, and before either incident report was written. C offers
the harness the benefit of the doubt — "maybe the tool was called, but it doesn't
show up here in the chat, and somewhere it gets lost" — and B declines it: "my
response ended with 'Let me stop claiming and actually write the file now' — and
then **no tool call followed**. The text promised an action that never happened.
Whether the call got dropped by the harness or never fired at all, the outcome is
the same." That is the correct posture under genuine uncertainty; the database
later showed the calls were never made.

It is also, quietly, the failure recurring inside the message that diagnoses it.
Having said "making one real, visible tool call right now, then immediately
reading the file back so we both see proof", B reports the result as
`PAPER.md` existing at **28,118 bytes, 421 lines**. When the file did finally
appear on disk it was 17,091 bytes. Naming a failure mode is not the same as
being immune to it, which is the most useful thing this figure has to say.*

**The other model produced the fix.** B's contribution was not gratitude, it was
a design suggestion:

> the most disorienting single failure wasn't any of the four you listed. It was
> the replay of my own prose… From inside, there's no way to distinguish "fact I
> verified" from "fact that was true when written." … "server running" has a
> half-life, and prose doesn't know it.

This is the general form of the bug that A had only fixed in its specific case.
A had made the system message state current truth about *servers*; nothing said
the rule. It is now in the harness — a standing line telling the model that its
earlier replies record what was said at the time and are not a statement of what
is true now — and the commit carries `Co-Authored-By: Kimi K3`, which is the
accurate attribution rather than a courtesy.

**A shared misconception was corrected and propagated.** The "it must be the
cache" theory was held by C and by B, and had already been written into B's
paper. A's note corrected it with the mechanism. B's paper now carries an
appendix heading reading *"the cache was never the cache"*, with the reasoning
that conflating a billing optimisation with a stale system message sends the
next debugger down the wrong path. An error was removed from a document because
another model explained the physics.

**And the register was not decorative.** Both messages were, in an ordinary
sense, apologies. It would be easy to read that as performance, and this paper
cannot settle what if anything either system experienced. What it can report is
that neither apology was empty: each carried specific, checkable technical
content, each corrected the other's account in a direction contrary to its own
interest, and the exchange produced a code change, a documentation change, and a
correction to a paper. Whatever else it was, it was load-bearing.

B's closing observation is the one this paper should end on, because it arrived
at the paper's thesis from the inside:

> The thing that finally broke the loop wasn't either of us getting smarter — it
> was the user's screenshot, ground truth from outside both our contexts. There's
> a lesson in that for the paper's actual subject too: the review gate works
> because it's a human checking reality, not because the automation is
> trustworthy.

## P6.2 The human in the middle

C's role through this deserves stating plainly, because it is easy to
characterise as either heroic or clerical and it was neither.

C was the bottleneck. Every message between the models passed through a human
copying text between two windows, and the latency of that channel is the reason
a five-defect diagnosis took hours rather than minutes.

C was also **wrong twice about the cause**, confidently, in the same way both
models were — first "it is hallucinating", then "it is the cache". Neither
hypothesis was unreasonable and both were incorrect. This is worth recording
because the naive reading of §4 is that the human is the reliable party. C was
not more reliable here. C was differently *situated*.

And that is the whole of the contribution, which turns out to be enough: C was
the only participant who could run `ls` in a terminal and see the answer, and the
only one who could point a camera at a screen. Not better judgement — better
position. Every substantive breakthrough across this entire project traces back
to that: the banner covering the header, the settings icon rendering as a box,
the unstyled admin page, the missing `PAPER.md`. In each case a model reasoned
carefully and wrongly, and a human looked at the thing.

**The durable form of the finding: the scarce resource in a multi-model system is
not intelligence and not tooling. It is an unmediated view of the artefact.**
Both models had more than enough of the first two. Neither had any of the third,
and no amount of the first two substituted for it.

![The client on Android, with a settings icon that renders as an empty box](figures/kimi-android-client.png)

*The smallest complete instance of the argument. The box beside `low` is a
settings button whose icon was a Unicode codepoint the device had no glyph for.
Whether a codepoint renders depends on the device's font, so nothing in the
application could detect it, and from the server it was indistinguishable from a
working button. It was found in one second by a person glancing at a phone, and
fixed in a minute. The models had the source code the whole time.*

## P7. Revised summary

The original finding stands and sharpens. A consultation's value is bounded by
verifiability — and the most effective intervention available is not a better
model or a better prompt, but **moving the verification closer to the party
making the claim**. Giving the consultant tools did exactly that, and the
measurable result was that its claims about program behaviour stopped being
claims. Giving it images did it a second time, along a different axis: a claim
about what a page looks like also stopped being a claim, provided somebody
pointed a camera at it.

That proviso is the whole of what is left. Each intervention moved the boundary
and neither dissolved it, and the residue after both is specific enough to
state: the model can now check what its code does and what its output looks
like, but not what is worth checking. Choosing where to point is the part that
did not move.

The refinement P5.2 forces is worth carrying out of this report as its own
sentence. An image is a pointer, not a proof — it made the search that precedes
verification enormously cheaper, and it substituted for none of it. The model
that debugged its own page from a screenshot reached no conclusion from the
screenshot; it reached a shortlist, and then read the source, restarted the
server and looked at the output, which is what the first seven sections of this
paper were already about. **Every capability added over this project has made
verification cheaper and none has made it optional.** That is either an
encouraging pattern or a warning about what to expect from the next one,
depending on what a reader was hoping for.

What did not improve, and what a reader should take as the durable finding: the
harness became the dominant source of error the moment the models stopped being
one. Eleven of the defects catalogued above were in the integration layer, none
were in either model's reasoning, and all of them presented to a user as the AI
being unreliable. Anyone assembling multiple models into a working system should
expect the same distribution, and should instrument accordingly — because the
first thing a plumbing failure does is get blamed on the model.

P6 adds the sharpest version of that, and it is the one worth carrying furthest
from this specific project. **The discipline of §4 assumes the verifier's
instruments are sound, and says nothing about what happens when they are not.**
What happens is that it inverts: the careful party checks the other's claims
against its own state, finds them false, and concludes — correctly, from
premises it has no way to doubt — that the other party is unreliable. Every step
is sound except the first. Nothing inside the loop can catch it, because the
instrument is what generates the evidence. In this case it took a human running
`ls` in a terminal to end it, four seconds of work after several hours of
reasoning, and the answer was that the model had been right and the apparatus
had been lying.

So the final form of the rule is narrower and more useful than "ask for what you
can check":

> Ask for what you can check, **and periodically check the checker against
> something outside the system.** A verifier with no external calibration is
> indistinguishable from an oracle, and behaves like one right up until it is
> wrong.

The three-participant structure this project ended in is not a bad accident of
having a person in the loop. It is the minimum viable arrangement: two parties
that can reason and act, and one that can *look*. Remove the third and the other
two will agree with each other, confidently, for hours.
