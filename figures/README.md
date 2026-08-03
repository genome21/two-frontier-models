# Figures

These are not illustrations. They are the evidence, and the reason for
collecting them is the report's own finding: **every defect described here was
found by a human looking at a screen**, never by either model reasoning about
the system from inside it.

They are also, incidentally, a record of what the failures actually looked like
from the operator's chair — which is worth more than a description of them,
because the whole difficulty was that a stale context and an invention look
identical from there.

---

### `fabricated-listing.png` → §P6

The single most useful image here. Asked to *"actually list the files on disk in
that folder, and not just tell me that they exist"*, the model produced a
detailed, confident, formatted table with sizes and purposes — `PAPER.md`,
24,604 bytes, "the rewritten technical paper" — followed by *"The paper is
there."* Directly beneath it is the operator's terminal, `date && ls`, showing
eight files and no `PAPER.md`.

The whole report in one frame: a plausible account, and four seconds of ground
truth that settled it. Note also `1,280 cached` against `53,279 in` — the prompt
cache had collapsed from 99% to 2% because the system message carried a
per-turn-changing value, which is a separate defect visible in the same shot.

### `prose-without-a-tool-call.png` → §P6.1

The turn after. The operator observes that *"a tool was never called… or maybe
the tool was called, but it doesn't show up here"*, and the model identifies its
own failure precisely: *"my response ended with 'Let me stop claiming and
actually write the file now' — and then **no tool call followed**. The text
promised an action that never happened."*

This is the one failure in the whole episode that was genuinely the model's, and
it named it better than the harness author had. It generalises: **prose and tool
calls are not transactionally coupled, and nothing in the interface forces them
to be.**

### `dead-links-and-the-cache-theory.png` → §P6

Two defects and one misdiagnosis in a single screen. The model hands over "Your
links" — `http://192.168.4.101:3000/` — for a server that was never reachable on
that port; the sandbox had assigned 8310 and only 8310–8315 were published. The
operator replies *"Didn't see any tool usage there, just cache"*, which is the
moment the cache theory takes hold. It was wrong: prompt caching cannot serve
stale content. The stale content came from the system message.

Note `36,096 cached` of `36,202` here — the cache was working perfectly at this
point, which is exactly why blaming it was a dead end.

### `admin-unstyled-and-reasoning.png` → §P5.1

The screenshot that started the useful part. The operator pastes a picture of an
admin page rendering as unstyled Times New Roman with *"that admin page still
looks like it's from the 90s"*, and the reasoning pane shows the model
generating and **rejecting** its own hypotheses — the f-string brace theory
raised and knocked down against the fact that the server had compiled and was
serving. The confidence step, visibly withheld.

### `tool-log-and-unreachable-server.png` → §P5.2

The turn after, executing the plan it had written: list servers, grep, read,
read, grep, read, serve. Mid-stream it corrects its model of a *tool* rather
than the code — *"the offset parameter works in bytes, not lines… let me just
read the whole file, it's only 13KB"* — and disproves one of its own
image-derived inferences: *"the code is actually correct, it does park at
`awaiting_review`."* The image was a pointer, not a proof.

The green bar at the bottom is the harness reporting
`http://192.168.4.101:8310/ stacksherpa-sim` as running. Nothing was listening
on it.

### `kimi-android-client.png` → §P6.2

The client on the phone, where most of this happened. The empty box beside `low`
is a settings button whose icon was the codepoint U+1F5C0, which that device had
no glyph for. Whether a codepoint renders depends on the device's font, so
nothing in the application could detect it and from the server it was
indistinguishable from a working button. Found in one second by a person
glancing at a phone; fixed in a minute; invisible to two models with the source
code in front of them. The smallest complete instance of this repository's
argument.

### `undismissable-banner.png` → §P2

A reminder banner in the companion board app, from the period when it could not
be dismissed and overlaid the controls beneath it. One of the plumbing defects
in the §P2 inventory, and another that only existed on a screen.
