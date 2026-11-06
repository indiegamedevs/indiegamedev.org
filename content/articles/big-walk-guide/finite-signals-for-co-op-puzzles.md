---
title: "Designing a Small Signal Code for Co-op Puzzles"
date: 2026-10-25
articles_tags:
  - co-op
  - game-design
  - tools
  - programming
summary: "A reproducible model for finite gesture messages, binary choices, and recovery rules in co-op puzzle prototypes."
---

A co-op puzzle can give one player information that another player needs, while limiting how they can communicate. Suppose there are ten possible symbols, three distinguishable gestures, and no voice channel. Before designing an animation or interface, define exactly what one message can represent.

The model below is a design exercise. It does not assume that any particular game exposes these gestures, or that players can reliably distinguish them.

## Count messages before assigning meanings

Call the available gestures A, B and C. A message consists of one gesture repeated between one and four times, with a clear boundary. This gives:

`3 gestures × 4 repetition counts = 12 distinct messages`

The multiplication assumes every gesture remains distinguishable at every allowed count. A and B must not become visually identical from the receiver's viewpoint. It also assumes a message uses one gesture type: A twice followed by B once is two messages, not an extra entry in this codebook.

Assign the ten options a stable order. A once through four times represents options 1–4; B represents 5–8; C once or twice represents 9–10. Leave C three times and C four times unassigned.

For a one-based option number `j`, first subtract one. Its gesture group is `floor((j - 1) / 4)`; its repetition count is `((j - 1) mod 4) + 1`. Option 9 therefore becomes C once. Decoding reverses this: `group × 4 + count`, with groups numbered 0, 1 and 2.

Both participants must share the same ordering. If one numbers objects left to right and the other uses a mirrored view, flawless signalling still delivers the wrong answer. Agree on a shared reference, such as a named corner or a diagram, before sending messages.

## Make the codebook executable

This Python example enumerates valid pairs directly, giving the decoder a way to reject unassigned messages.

```python
def make_codebook(channels, option_count, max_repeats):
    if not channels or len(set(channels)) != len(channels):
        raise ValueError("Use at least one channel with distinct labels")
    if any(type(v) is not int or v < 1
           for v in (option_count, max_repeats)):
        raise ValueError("Counts must be positive integers")
    if option_count > len(channels) * max_repeats:
        raise ValueError("Not enough distinct messages")
    pairs = [(label, count) for label in channels
             for count in range(1, max_repeats + 1)]
    return pairs[:option_count]


book = make_codebook(("A", "B", "C"), 10, 4)
decode = {message: option for option, message in enumerate(book, 1)}

assert book[8] == ("C", 1)                 # encode option 9
assert decode[("C", 1)] == 9               # decode it
assert decode.get(("C", 3)) is None        # unassigned pair
assert [decode[word] for word in book] == list(range(1, 11))
```

The final assertion checks every valid message in both directions. It proves consistency of this table, not whether somebody can count four distant gestures correctly. An input layer must also reject unknown labels and malformed counts instead of guessing.

Unused entries provide limited error detection. C three times is invalid here, but A twice misread as A three times still names a valid option. Spare codes do not automatically make a system error-correcting.

## When twelve messages are not enough

With thirteen options, this particular one-message scheme no longer fits. Increasing the repetition limit is possible, but gives the receiver more to count. More gestures help only if people can tell them apart.

A different model uses a sequence of reliable yes/no answers. With `q` answers, there are at most `2^q` patterns, so the minimum fixed length for `n` options is `ceil(log2(n))`. Thirteen options require four answers: three distinguish only eight possibilities, while four distinguish sixteen.

One reproducible assignment numbers options from 0 to 12 and writes each index as four bits. Option 9 has index 8, so its pattern is `1000`. If 1 means yes and 0 means no, send yes, no, no, no. Patterns `1101`, `1110` and `1111` remain unused. This zero-based assignment is the article's example, not a description of the optional demo's lookup table. Specify the bit order; reversing `1000` changes its meaning. No must be an intentional signal, not an uncertain absence.

A binary sequence is not necessarily faster or easier. Each answer needs a boundary; interactive questions add time and memory demands. The formula measures distinguishable patterns under perfect transmission, not the duration or reliability of an exchange.

## Design the recovery path too

Before a consequential action, the receiver should echo the decoded option and wait for confirmation. An echo can reveal disagreement, but cannot guarantee that both participants understood the same thing.

Define a restart signal separately from ordinary data. If C four times becomes restart, remove it from the data alphabet and document that reservation. Do not count a control message as another selectable option. If all twelve messages are occupied, restart needs a separate signal or a changed scheme.

Specify when a message begins and ends. A pause used as an end marker needs an explicit timing rule; otherwise two short messages can look like one longer message. This boundary is an additional protocol assumption, not a free extra symbol. If confirmation is lost, restart from a known boundary rather than appending another unexplained burst.

For a prototype, separate two questions. First, can every intended option be encoded and decoded, with invalid inputs rejected? Second, can participants perceive, remember and recover the messages under the actual camera, timing and interaction constraints? The arithmetic answers only the first. This article reports no participant testing and makes no accessibility claim.

I maintain the independent, unofficial Big Walk Guide. Its [signal encoder](https://thebigwalk.world/en/protocols) provides an optional interactive illustration of this kind of capacity calculation. The model and code above stand on their own; neither the project nor this article is affiliated with House House or Panic.

*Drafted with AI assistance.*
