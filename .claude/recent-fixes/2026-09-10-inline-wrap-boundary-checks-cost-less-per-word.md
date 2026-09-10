# Inline wrap-boundary checks cost less per word

## What was wrong

Nothing incorrect — this is pure optimization of the per-word helpers
[2026-09-10-inline-boundaries-do-not-create-wrap-opportunities.md](2026-09-10-inline-boundaries-do-not-create-wrap-opportunities.md)
introduced. Three of them ran on paths layout takes for every placed word:

- `UpdateTrailingGraphemeContext` ran for **every** placed word with no early-out, doing
  `string.Concat` + `StringInfo.ParseCombiningCharacters` + a substring: 3 allocations and ~278 ns per
  word, to return one trailing cluster.
- `IsGraphemeBoundaryBefore` was computed **twice** per word at all three call sites — once directly,
  then again inside `HasOrdinaryWrapOpportunityBefore`. Its cross-owner path segments a concatenated
  string (~222 ns, 104 B).
- `CommonUtils.IsEmojiLineBreakCharacter` runs per *character* during tokenization and ran a ~7-step
  binary search over a 95-entry range table for every Latin character (~56 ns).

## The load-bearing idea

No UAX #29 rule joins two grapheme-neutral ASCII characters (printable ASCII is never Extend,
SpacingMark, ZWJ, Prepend, regional indicator, extended pictographic, Hangul, CR/LF, or a surrogate).
So for ordinary Latin text the trailing cluster is simply the final character, and a boundary between
two such characters is certain — both readable without segmenting anything. Where segmentation is
still needed, walking clusters with `GetNextTextElementLength` beats `ParseCombiningCharacters`,
which materializes an `int[]` of *every* boundary when only one is wanted (and in
`IsGraphemeBoundaryBefore`, the walk stops at the boundary under test).

`isGraphemeBoundary` became a **required** parameter of `HasOrdinaryWrapOpportunityBefore` rather than
an optional one defaulting to null: a grapheme boundary is a precondition of an ordinary opportunity
and every caller already needs the answer, so an optional recompute would be dead code plus a
foot-gun for the next caller.

## What was deliberately not done

**Reordering the `||` chain in `HasOrdinaryWrapOpportunityBefore` so `HasInterElementWhitespaceBefore`
goes last.** It looks like free money — that disjunct walks the box tree and `GetPreviousSibling`
does a linear `Boxes.IndexOf`, while every other disjunct is a field read. Measured, it was a **6.6%
median regression** on an inline-heavy document. The reason: in `<span>a</span> <span>b</span>` the
inter-element whitespace *is* the wrap opportunity, so that call returns true as operand #2 and the
original short-circuits immediately. Moving it last forces the intervening operands (including two
`OwnerBox.WordBreak.Value` reads) to run for every word, and the walk still happens anyway. The walk
is only wasted work when it returns *false*, which is not the case that made it look hot.

**Memoizing `HasInterElementWhitespaceBefore` per `OwnerBox`.** Its result depends only on the box, so
a memo looks safe, but `ParseToWords` runs *during* layout (running elements, counters), so a box's
`Words.Count` can go 0 → non-zero after the memo is taken, and `EndsWithCollapsibleWhitespace` reads
exactly that. It would need an invalidation hook to be correct, and the measured win did not justify
one.

## Evidence

- Full net8.0 suite green: 10,672 passed, 9 platform-specific skips. Diff coverage 100% (44/44
  executable changed lines). Solution builds with zero warnings.
- Per-call microbenchmarks: trailing-context 278 ns/128 B → 32 ns/24 B; boundary test 222 ns/104 B →
  44 ns/43 B; emoji range test 56 ns → 3.8 ns on ASCII input.
- Document-level A/B against `b1b37887`: allocation consistently lower on all three shapes (prose
  49.07 → 48.21 MB, glued spans 57.91 → 57.38 MB, sibling spans 95.71 → 95.18 MB per render). Time
  4–10% faster on sibling-spans.

## The trap this cost real time on

**Wall-clock numbers here are only meaningful interleaved.** Between two runs on the same machine the
*unmodified baseline* moved from a 270 ms median to 371 ms — enough drift to manufacture a
convincing +25% "regression" out of a build that was actually faster. Every conclusion above rests on
alternating baseline/candidate runs collected in one batch, with allocation (which does not drift) as
the corroborating signal. Comparing a number measured now against one measured twenty minutes ago
produces confident nonsense.
