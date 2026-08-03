---
name: model-client
description: Building a dedicated chat front end for a language model whose vendor ships an API but no usable app — as was done for Moonshot's Kimi K3 in ~/workspace/kimi. Use when a new frontier model is released and the options are a generic OpenAI-compatible UI or something built for it; when adding streaming, reasoning display, tool calling, cost tracking or attachments to such a client; or when checking what a provider's API actually does before writing code against it. Assumes the LAN-app stack — read the `lan-app` skill for the deployment shape.
---

# A chat client for a model that hasn't got one

The point of building one rather than pointing a generic client at the endpoint
is that generic clients show what OpenAI has and hide what the model actually
does. Every interesting thing about K3 — reasoning as a separate stream,
`reasoning_effort`, ten-to-one cache pricing — is invisible in a UI that assumes
the OpenAI surface.

`~/workspace/kimi` is the worked example. Read `api/app/kimi.py` first; it is
the file that knows what the vendor's API really looks like.

## Check the API before writing anything

"OpenAI-compatible" is a claim about the request format. It is never a claim
about behaviour, and the differences are where the product decisions are. Read
the vendor's own docs — not an aggregator's summary — and answer all of these
before designing a screen:

- **Model ids and base URL.** Exact strings.
- **Streaming shape.** Is there a channel besides `content`? K3 streams
  `reasoning_content` separately; a client reading only `content` shows a blank
  screen for the entire time the model is thinking, which is most of the wait.
- **Which parameters actually do something.** K3 fixes `temperature`, `top_p`
  and `n`. A temperature slider would have been a control that does nothing,
  which is worse than no control. The real knob was `reasoning_effort`.
- **What cannot be turned off.** K3 always reasons. That is a UI requirement,
  not a setting.
- **Tool-call streaming.** Deltas usually arrive keyed by index, with the id and
  name in one chunk and the arguments as partial JSON across several.
  Accumulate and emit whole; never try to parse arguments mid-stream.
- **What must be echoed in multi-turn.** Tool results may need both
  `tool_call_id` *and* `name`. Whether reasoning has to be replayed is often
  undocumented — see below.
- **Vision input format.** K3 rejects public `http` URLs outright; images go in
  the request body as base64 data URLs, inside a content *array* beside the
  text rather than in the `content` string. Note the shape of that constraint:
  your service will have a perfectly good URL for every image it stores and
  will not be able to use it. See "Images" below.
- **Usage and cache reporting.** Streaming often omits usage entirely unless
  `stream_options: {include_usage: true}` is set. Cached-token spelling varies
  (`prompt_tokens_details.cached_tokens`, `cached_tokens`,
  `prompt_cache_hit_tokens`) — accept all, and degrade a missing one to zero
  rather than raising.
- **Prices, and any cache discount.** K3's cached input is a tenth of uncached,
  which is large enough to design around rather than mention.
- **Rate limits, and which dimension.** Ask the API rather than guessing: a
  concurrency cap, a request-per-minute cap and a token-per-minute cap call for
  completely different responses, and the error body usually names which one.
  Note that a tool-calling turn spends one request *per round*, so an RPM cap
  binds far sooner than it looks — four file operations is five requests. Treat
  a 429 as a queue and retry with backoff rather than failing the turn; a turn
  killed at round four has already paid for rounds one to three.

  Check the tier rather than the vendor's headline numbers, and expect it to
  move: this account was Tier 0 (3 concurrent / 20 RPM / 500k TPM), where the
  client had to work around the limits, and a $5 top-up made it Tier 1 (50 / 200
  / 2M, no daily cap), where they stop mattering. So don't hardcode tier figures
  in error messages — the API's own message is current by definition and yours
  will silently go stale. Do keep a self-imposed concurrency cap, but be honest
  about what it is for: below a tight ceiling it protects you from the vendor,
  and above a generous one it is a spending brake against a runaway loop. Cache
  anything polled that costs a request — but tune the TTL to whichever of those
  two pressures is real, because a balance that takes two minutes to reflect a
  top-up is its own small bug.

Record the answers in a module docstring with the date and the URLs. That file
is the thing the next person will trust.

### When the docs do not say

Settle it against the live API and write down what you observed, not what you
assumed. Whether K3 needs `reasoning_content` replayed was unanswerable from the
docs, so: send `content` only, ask a follow-up whose meaning depends on the
previous turn, and check the pronoun resolves and the prompt-token count rises
by roughly the added history. Store the field regardless — keeping it costs
disk, discarding it cannot be undone for conversations that already happened.

## Shape

```
web (React) ──▶ api (FastAPI) ──▶ vendor API
                    │
              caddy + SQLite
```

**The key lives in `.env` on the server and nothing else ever sees it.** The
browser and the phone talk only to the proxy, so a decompiled APK or an open
dev-tools panel give up nothing. This is also the only arrangement with one
place to rotate it.

**Conversations live server-side.** The whole reason to have both a web app and
a phone app is that a conversation started on one continues on the other.

**SSE, not WebSocket.** One direction, no client-to-server traffic during a
turn, survives a proxy without an upgrade handshake. Note `EventSource` cannot
POST, so the browser side is `fetch` plus a small parser — and that parser must
buffer partial events, because chunk boundaries do not respect message
boundaries. Caddy needs `flush_interval -1`.

## What the client must get right

**Show reasoning as what it is.** A collapsed pane above the reply, expanded on
demand. It is usually longer than the answer, so leaving it open buries the
thing you asked for. Show a live "Thinking…" state only until the answer starts
arriving; after that the answer is the better progress indicator.

**Keep volatile data out of the system message.** Prompt caching works on
prefixes, so one character that changes between turns — a uptime in minutes, a
timestamp, "3 items in your queue" — misses the cache for the *entire*
conversation behind it, and the damage grows as the conversation does.
Measured on one real conversation: 36,096 of 36,202 prompt tokens cached while
the note was static, 1,280 of 59,134 once it carried a running server's
uptime. At a tenth the price for a hit that is most of the cost of a turn.
Split it: standing rules first and byte-identical every turn, live state in a
short note inserted just before the newest user message.

**Replay a turn if it said something, whatever else happened to it.** It is
natural to filter failed turns out of the history, and the reason is sound —
an empty assistant message teaches the model that empty replies are fine. Do
not let that become "exclude anything with an error set". A turn that did
twenty minutes of real work and then hit a limit carries both an error *and*
the only account of what it changed; dropping it leaves the model with no
record of its own actions, and a model with no record guesses. Filter on empty,
not on failed.

**Report cost per message, per conversation and in total,** with cache hits
called out. A number nobody can see is a number nobody optimises, and the
difference between a cached and uncached prefix can be tenfold. Below a dollar,
four decimal places — two rounds almost every turn to `$0.00`.

**A turn you paid for is a turn you keep, and the turn must own that.** Put
persistence inside the task doing the work, not inside the streaming response.
If the response owns it, a client going away cancels the turn and stores a
fragment — after you have paid for all of it. This is not an edge case on
mobile: backgrounding an Android app suspends its WebView and drops the
connection every single time, which surfaces as the app appearing to hang or
saying "disconnected" whenever you look away for a minute.

Two more things that bite here. `asyncio` holds only a weak reference to a
task, so one whose only strong reference was a local in a torn-down generator
can be garbage collected mid-flight — keep them in a module-level set. And the
client needs somewhere to ask "is it still going": expose an `active` flag on
the conversation and poll it after a broken stream, rather than reporting a
failure the server does not agree with.

**Replay is asymmetric: the model's prose is replayed, the world is not.** This
is the same finding as the next paragraph and it is worth stating in its
stronger form, because it produces defects that look exactly like the model
lying. An assistant message saying "your server is running at http://…" stays
in the history forever. The tool result that started the server does not. An
event that happened *outside any turn* — the user stopping that server from
your own UI — certainly does not. So the model keeps reporting a fact that
stopped being true, and gets blamed for it.

The rule: **anything that changes the world outside a turn must be injected
into the next turn explicitly.** Rebuild the system message from live state
every turn, and make it state changes rather than only current conditions —
"the server you started was STOPPED BY THE USER" rather than a list that has
quietly gone from one item to none. Three things this needs: keep records of
things after they end (deleting the row on stop means nothing is left to
report), record *why* they ended (crashed and stopped-by-a-person call for
opposite responses), and never compose a success artefact from a field that is
present on failure — a URL built from "the port we allocated" rather than from
"it is running" is a link to a corpse.

**Tool results are not replayed to the model.** History is user and assistant
prose; anything a tool returned is gone by the next turn. If the model needs to
act later on something a tool gave it — an id, a handle, a port — put it in the
system message, rebuilt per turn from live state. Discovered when a model could
not stop a server it had started ninety seconds earlier: it had the URL only
because it had written it in its own reply.

**Keep failed turns in the conversation**, shown where the reply would be, and
exclude them from the history sent upstream — otherwise a retry teaches the
model that empty replies are acceptable.

**Truncate titles, do not generate them.** An extra completion per new
conversation costs real money to produce something worth nothing.

**Render markdown to elements, never to HTML.** No `dangerouslySetInnerHTML`
anywhere. This matters more than in an app whose text you write yourself:
model output can be steered by anything the model read. For syntax
highlighting use `lowlight`, which returns a syntax tree, rather than
highlight.js directly, which returns markup.

**Enter inserts a newline; Ctrl/Cmd+Enter sends.** The usual convention is the
other way round and is a bad fit here — the questions worth a reasoning model's
latency have paragraphs or code in them.

## Images, if the model has vision

Worth building early rather than late. A screenshot is the densest channel a
user has — one picture plus "this looks wrong" carries what a paragraph cannot
— and if the model can also run code, an image is how the consequences of its
own actions get back to it. That last point is not obvious: tool results are
not replayed (above), so a model genuinely cannot see what its last edit did.
A screenshot repairs an amnesia the harness created.

- **Paste first, then drop, then a button.** What is being sent is nearly
  always already on the clipboard. Take over the paste event *only* when the
  clipboard actually holds an image, or you break pasting text.
- **Decode before believing anything.** The content type is a claim by whoever
  is uploading. The image library opening the file is the only evidence it is
  an image, and the format it reports is the one to use from then on.
- **Strip EXIF, and understand why.** Phone photos carry GPS. Uploading one to
  ask what is in it ships the coordinates of where it was taken to a third
  party as a side effect. Apply the orientation tag to the pixels — otherwise a
  sideways photo gets described sideways — and discard the rest.
- **Cap the long edge, but generously, and keep PNG as PNG.** Image tokens are
  billed like any other, so a 12MP photo costs real money to answer no better.
  Screenshots set the floor on how far you can go: they are mostly small text,
  resampling is how legible text becomes illegible, and JPEG's ringing lands
  hardest on exactly the sharp edges text is made of.
- **Store one image, not two.** What is displayed should be what was sent, so a
  misread screenshot can be checked against the same pixels the model had.
  Content-address it: pasting five near-identical screenshots while iterating on
  a UI is the normal case.
- **Replay images in history.** "What about the top left" only works if the
  picture is still there. Measure the cost before fearing it — on K3 a
  follow-up turn came back with 733 of 843 prompt tokens cached, so the replay
  is billed at a tenth.
- **On Android, `<input type="file">` is inert without a `WebChromeClient`
  handling `onShowFileChooser`.** No picker, no error. It will work in a
  browser and do nothing in the app. Answer the callback on cancel too, or the
  input stays permanently disabled and the button appears to work once and then
  stop. Handle both `getClipData()` and `getData()` — that is why WebView
  uploads so often work for one image and not for three.

## Local file access, if it gets that far

`~/workspace/kimi` has this; the design is in its README and `api/app/agency.py`.
The short version:

- **A conversation is scoped to one project folder.** That is the unit of
  access. A global grant list cannot express "this conversation, that folder".
- **Autonomy is per conversation, not per server** — read-only, ask each time,
  ask once, never ask. One chat is "look at my config" and the next is "go and
  build this".
- **A new conversation never starts unattended.** The project carries over; an
  auto-approving level does not. Otherwise the per-conversation design is
  defeated by inheritance and you have a global setting that is slower to
  notice.
- **Containment is layered and independent**: the API checks the path, a
  separate sandbox service re-checks it against its own root whatever the API
  claimed, and that service's container has only the permitted tree mounted.
- **No shell.** Path checking does not contain arbitrary commands.
- **Approvals live server-side**, so a turn paused on the laptop can be answered
  from the phone.
- Once the model can read files, everything it reads is untrusted input. That is
  what the approval gate is for, not belt-and-braces.

## Testing without spending money

Mock the vendor, not your own code. A container on the compose network, the
service pointed at it for one command, and the mock **asserting on what it
receives** — that `stream` is on, that usage was requested, that no
fixed-by-the-API parameter was sent.

Cover: reasoning and content interleaved, tool-call deltas split mid-JSON
across chunks, a usage block, an error status, and a stream that ends without
usage at all.

Then do one real turn before believing any of it. Keep it short and use the
cheapest effort setting; the whole K3 client was validated for under three
cents.

## Things that read as bugs and are not

- **Slow first token on a new key.** 18s and then a 429 on the next request was
  rate limiting, not the client.
- **`cached_tokens: 0` on short prompts.** Caching needs a substantial prefix.
  Report zero as "none reported" rather than implying none happened.
- **Reasoning of a handful of characters at low effort.** That is the setting
  working.
