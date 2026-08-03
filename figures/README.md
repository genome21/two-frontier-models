# Figures

These are not illustrations. They are the evidence, and the point of collecting
them is that **every defect in this report was found by a human looking at a
screen** — never by either model reasoning about the system from inside it.

## Present

### `kimi-android-client.png`

The chat client on the phone, which is where most of the work in this report was
done. Note the box beside `low` in the top right: that is a settings button
whose icon was the Unicode codepoint U+1F5C0, which the device had no glyph for.
It rendered as an empty box and looked exactly like a missing button. Nothing in
the app could detect it; whether a codepoint has a glyph depends on the device's
font. It was found by the operator saying *"the settings button icon is missing"*
and is why every icon in both apps is now an inline SVG.

### `undismissable-banner.png`

A reminder banner in the companion board app, from the period when it could not
be dismissed. Referenced in §P2 as one of the plumbing defects that presented as
a broken product and was invisible from the server side.

## Wanted

The following were shared during the session but only ever existed as pasted
images in a chat window, so they are not recoverable from this machine. If you
still have them, dropping them in with these filenames is enough — the README
and report reference them once they exist.

| Filename | What it shows | Why it matters |
|---|---|---|
| `stacksherpa-landing.png` | The finished public page and intake form | The artefact that finally worked, at the end of §P6 |
| `stacksherpa-admin.png` | The admin job feed, styled, with a live log and an "Approve & Complete" gate | Both the fixed page *and* the review gate that the paper argues for |
| `admin-unstyled.png` | The same page rendering as unstyled Times New Roman | The screenshot the model debugged its own code from, in §P5.1 |
| `fabricated-listing.png` | The chat showing a confident file table for files that did not exist, beside a terminal `ls` showing the truth | §P6 in a single image: the model's account and the ground truth side by side |
| `harness-reasoning.png` | The reasoning pane, mid-hypothesis, rejecting its own f-string theory | §P5.1's claim that the confidence step was withheld |

`fabricated-listing.png` is the most valuable of these if only one survives. It
is the whole report in one frame: a detailed, plausible, entirely invented answer
next to four seconds of terminal output that settled it.
