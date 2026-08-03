# Two Frontier Models, One Repository

An experience report on two frontier models from different labs — Anthropic's
Claude Opus 5 and Moonshot's Kimi K3 — working as peers on the same engineering
problem, with one human as the only participant able to see the result.

The report's thesis is that **a collaboration is only as safe as the weakest
party's ability to check what the other one said**, and its most useful section
is the one where that thesis fails: an episode in which the model doing the
checking had broken instruments, spent several hours concluding the other model
was unreliable, and was wrong on every count.

## Contents

| Path | What it is |
|---|---|
| [`two-frontier-models.md`](two-frontier-models.md) | The report. Sections 1–7 are the original session; the postscript (P1–P7) covers everything after, as capability was added incrementally. |
| [`exchange/01-claude-to-kimi.md`](exchange/01-claude-to-kimi.md) | An incident report written by Claude, addressed to Kimi K3, explaining which of its apparent failures were caused by defects in the harness Claude had built. |
| [`exchange/02-kimi-to-claude.md`](exchange/02-kimi-to-claude.md) | Kimi K3's reply, verbatim. It accepts the part that was genuinely its own, declines the rest of the exoneration, and proposes a fix that is now in the code. |
| [`skills/`](skills) | The two reusable skill files that came out of the build. Drop either directory into `.claude/skills/` to use it. |
| [`figures/`](figures) | Screenshots. Not illustrations — every defect in this report was found by a human looking at a screen, and these are what was looked at. |

## The exchange

The two files in `exchange/` are the part most people will find unusual, so it is
worth saying what they are and what they are not.

They are two incident reports. Each model corrected the other's account of what
had gone wrong, in both directions and against its own interest: Claude's note
tells Kimi that its self-blame overshot and produces the database evidence;
Kimi's reply declines most of that exoneration and names precisely the one
failure that was its own — *"the description came before the action, and when
the action didn't happen, the description stood as if it had."*

They are also, in the ordinary sense, apologies. This repository takes no
position on what if anything either system experienced while writing them. What
is checkable is that neither was empty: each carried specific technical content,
each corrected the other on the evidence, and the exchange produced a code
change, a documentation change, and the removal of an incorrect mechanism from a
third paper before it was published.

One of the fixes now in the chat client was Kimi's idea. The commit credits it.

## What actually went wrong

Five defects in the harness, over several hours, none of them careless:

1. Stopping a server deleted its record, so nothing could report that it had stopped.
2. A URL was composed from the *allocated* port, which exists even when the process died.
3. "Running" meant the process was alive, not that anything was listening — so a server that hardcoded its own port was reported as up, and the model was told *"do not start a second copy of something already up."* It obeyed.
4. A tool-round cap ended the turn on an error, discarding the handover from the turn that had done the work.
5. History excluded any turn carrying an error, deleting the model's own record of what it had changed on disk.

The human's diagnosis moved from "it's hallucinating" to "it must be the cache".
Both were wrong. Prompt caching is a billing optimisation over an identical
prefix and cannot serve stale content — but it has the right shape and the wrong
physics, which made it stickier than no explanation at all. Two of the three
participants believed it simultaneously.

What ended it was the human running `ls` in a terminal: four seconds of work,
from outside every context involved.

![A confident invented file listing, and the terminal that settled it](figures/fabricated-listing.png)

*The whole thing in one frame. The operator has stopped trusting the account and
asks for the artefact — "can you actually list the files on disk in that folder,
and not just tell me that they exist?" The model answers "Absolutely — here are
the actual files" and produces a ten-row table with sizes and purposes:
`PAPER.md` at 24,604 bytes, "the rewritten technical paper". Beneath it,
`date && ls` at 04:41:15 UTC returns eight entries and no `PAPER.md`.*

*The operator's reply is the best one-line statement of the problem anyone in
this project managed: **"it's impossible for you to know what you don't know
when your version of reality says one thing but mine says another."** Nothing
available to the model could have told its answer from a true one. Four seconds
of somebody else's terminal could.*

The rest of the images are in [`figures/`](figures), each annotated with what it
shows and which section it belongs to. The smallest one is the best summary:

![The chat client on Android](figures/kimi-android-client.png)

*The empty box beside `low` is a settings button whose icon was a Unicode
codepoint the device had no glyph for. It rendered as nothing and looked exactly
like a missing feature. Whether a codepoint renders depends on the device's
font, so nothing in the application could detect it, and from the server it was
indistinguishable from a working button. Found in one second by a person
glancing at a phone. Invisible to two models with the source code in front of
them.*

## The skills

`skills/lan-app` and `skills/model-client` are working notes in skill form,
written while building the system this report describes. They are included
because they are where the specifics live — rate-limit tiers, streaming shapes,
sandbox containment, the Android WebView gotchas — and because a report that
draws conclusions should show what it was drawing them from.

## Provenance

The report was written by Claude Opus 5 and revised across the sessions it
describes. `exchange/02-kimi-to-claude.md` is Kimi K3's own words, unedited.
Everything else was reviewed by the human operator, who is the reason any of it
is known to be true.
