# CSS Inline Vertical Alignment and Line Wrapping Around Floats #
## ... or why implementing CSS 2.1 is harder than you thought ##

by [L. David Baron](http://dbaron.org)

<a rel="license" href="http://creativecommons.org/licenses/by-sa/3.0/"><img alt="Creative Commons License" style="border-width:0" src="http://i.creativecommons.org/l/by-sa/3.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/3.0/">Creative Commons Attribution-ShareAlike 3.0 Unported License</a>.

Prerequisite reading: [CSS 2](https://drafts.csswg.org/css2/), chapters 8, 9, and 10.

This document starts with some observations about some pieces of the CSS box model that are tricky to implement, and then describes how they are currently implemented in Gecko and how I believe they could be implemented better.

## Observations ##

### The anchor point for a float need not be at a line breaking opportunity ###

Conceptually, although it's not described in the spec, floats have a
place that they come from:  essentially, where they would have been if
they'd been a non-floated inline.  In Gecko, we actually place an empty
inline there and call it the float's placeholder.  It probably makes
more sense to call it the float's anchor point.

There's no requirement that this anchor point be at a line breaking
opportunity.  It's perfectly legal markup to write:

    <p>This is a ridicul<img style="float:left">ous example.</p>

This is interesting because the 
[float placement rules](http://www.w3.org/TR/CSS2/visuren.html#float-position)
in CSS 2 (also see
[why they're bad](http://dbaron.org/log/20120827-specification-style))
generally want the float placed next to the line containing its anchor
point.  But if that's not possible (because placing the float there
would shorten the line enough that the anchor point no longer fits),
then the *float* gets pushed down so that it doesn't intersect the line.

So, logically, we can't be sure we've placed the float at a position
until we've gotten to the first line breaking opportunity at or after
its anchor point.  But more on that later, after my other observations.

### The width available for a line wrapped around floats is a function of that line's height ###

Because of the [float placement
rules](http://www.w3.org/TR/CSS2/visuren.html#float-position) in CSS 2,
the float can't be above its anchor point, but it can be substantially
below its anchor point.  This can happen, for example, if there are
multiple floats that are too big to all fit next to each other, or if
some of the floats use the 'clear' property, which clears them past
other floats.  This sort of situation, with the top of a float being
substantially below its anchor point, is relatively common on Wikipedia.

When this happens, the lines that come logically after the float have to
wrap around the float.  A later float might be wider than a float before
it, which means that the available width for the lines that are wrapping
around floats might decrease with downward movement.  The rules for
wrapping lines around floats say that the lines must not intersect any
floats whose vertical positions they overlap with.  (In particular,
[section 9.5 of CSS 2](http://www.w3.org/TR/CSS2/visuren.html#floats)
says "A line box is next to a float when there exists a vertical
position that satisfies all of these four conditions: (a) at or below
the top of the line box, (b) at or above the bottom of the line box, (c)
below the top margin edge of the float, and (d) above the bottom margin
edge of the float.")

This, in turn, means that making an individual line taller might reduce
the amount of horizontal space available to it.  An implementation that
fails to consider this (which happens in a number of implementations,
actually, but Gecko gets it mostly right) ends up with text overlapping
the upper part of the float, as in
[this screenshot](https://bug384376.bugzilla.mozilla.org/attachment.cgi?id=268303).
(This also applies to elements establishing new block formatting
contexts that wrap around floats; Gecko doesn't get that case right.)

This, in turn, means that placing something new on a line (for example,
an inline image) that fits horizontally might not fit because it makes
the line taller which in turn reduces the available width for the line.

### 'vertical-align' is tricky ###

The values of the [vertical-align
property](http://www.w3.org/TR/CSS21/visudet.html#propdef-vertical-align)
fall into two groups.  One group is the 'top' and 'bottom' values.  The
other group is all the rest of the values.

For all values other than top or bottom, vertical-align describes how an
element is aligned relative to its parent.

The 'top' and 'bottom' values, on the other hand, describe the position
of that element relative to the line.  Or, really, they describe the
position of of that element and its descendants that are aligned to it
relative to the line, since the element with 'top' or 'bottom' can have
a descendant that extends above or below the element's top and bottom.

In essence, this means vertical alignment operates in terms of subtrees
that are glued together instantly, one for the root of the line (the
block containing it) and one for each element with 'top' or 'bottom'
vertical-alignment.  These subtrees, while fixed immediately within the
subtree, have unknown position relative to each other (I've called them
"loose") until all content is placed on the line.

### Floats usually, but not always, have sizes that don't depend on their position ###

According to the CSS specifications, the size of a float doesn't depend
at all on its position; the size is simply computed from the CSS
properties on the float, its contents, and the size of its containing
block.

However, floating tables, in quirks mode, show a different behavior.
(It's worth investigating if this behavior is still needed for
backwards-compatibility, since it introduces a good bit of complexity.)
Floating tables in quirks mode use the width available for lines to fit
in (which considers floats adjacent to the line or table) in their
sizing rules, in place of the width of the containing block.  (In
general, sizing this way produces very bad effects when multiple floats
are used and shrink wrapped, as described in TODO: LINK HERE (and also
see [the bug where this problem was fixed in Gecko](https://bugzilla.mozilla.org/show_bug.cgi?id=59200)).)

In Gecko, this quirk is implemented in
nsBlockReflowState::FlowAndPlaceFloat and
nsBlockFrame::AdjustFloatAvailableSpace.

This single case adds extra complexity to the problem of float layout
(unless it turns out it's possible to remove it from the Web).

## Multi-pass line layout in Gecko ##

In Gecko, related to the above reasons, there are three reasons that we
repeat layout of a line.  The repetition is implemented in the loops in
nsBlockFrame::ReflowInlineFrames.  These three reasons are:

LineReflowStatus::RedoNoPull:  We redo the line's reflow when we've placed
content on the line past the last break that fits.  This happens
because, in Gecko, we place one frame (box, rendering object) on the
line at a time.  There are frequently not break opportunities between
frames.  For example, the following markup has multiple frames but no
breaking opportunities:

    <div>Inte<a>rest</a>ingly</div>

This case is essentially a workaround for the way Gecko does inline
layout.

LineReflowStatus::RedoMoreFloats:  This is the case that reflects one of the
observations above, that the width available for a line can decrease if
the line takes up more height.  In Gecko, when that happens, we start
layout of the line over again, decreasing its width to the width
available with the new height.  (The first time through assumes the line
will have zero height, which is actually probably too conservative.)

This case could be avoided if we avoided placing unbreakable units on
the line if they would increase the line's height in a way that would
decrease the line's width such that that unit wouldn't fit.  (This
requires, when placing each unit, doing enough of the vertical alignment
process to determine its effect on the line's height.  See below.)

LineReflowStatus::RedoNextBand:  This happens when the *first* unbreakable
unit ("word") on the line doesn't fit next to floats.  In this case we
have to move the entire line down, until either the word fits or there
are no longer floats next to it.  It might appear that this doesn't
require redoing the layout of the line.  However, I believe that it does
because of floats whose anchor points might be in the middle of that
first word.  We normally position and lay out (reflow) floats when we
reach their anchor point during inline layout.  If we only pushed the
line down without layout out that first word again, we would fail to lay
out the floats again.

This could be avoided by not placing and laying out the floats until
we've committed the word that contains their anchor point to a line.
This appears to me to be the correct time, in the sense that we want to
do this after things that can influence their position, but before
things whose position they influence.  (That's generally how we want to
order our layout calculations.  While the current order for floats is
close to correctly ordered, the LineReflowStatus::RedoNextBand is a
workaround for it not being quite right.)

## Vertical alignment in Gecko ##

Part of the need for these multi-pass cases in line layout is related to
the way we do vertical alignment in Gecko:  at the end of the line
layout process.  We do horizontal layout first, and then when we're done
(but still inside the above repetition loops), do vertical alignment for
the entire line in nsLineLayout::VerticalAlignLine.  This function does
two passes over the line, one (VerticalAlignFrames) to handle all of the
parent-relative 'vertical-align' values (anything other than 'top' and
'bottom') and gather the information needed for the line-relative
'vertical-align' values ('top' and 'bottom') and then another pass
(PlaceTopBottomFrames) to finish the alignment of the line-relative
values.  This means that line height calculations are not done at all
until line layout is complete, which is one thing that would need to be
changed to avoid multi-pass layout in the cases above.

## A possible single-pass line layout solution ##

So I *think* (though I haven't worked it through in too much detail)
that it should be possible to do line layout in a way that is always
single pass.  It requires addressing the issues above with some care; in
particular:

1. Inline layout needs to have a notion of committing each unbreakable
unit to the line, even if that unbreakable unit contains multiple
elements.

2. Float layout and placement should happen *after* the unbreakable unit
containing the float's anchor point has been committed to the line.  (An
anchor point in between unbreakable units, however, should not, I think,
ever attach to the following unbreakable unit, though.)

3. Vertical alignment and line height calculation needs to be done
incrementally as unbreakable units are committed to the line.  This
means (a) computing all parent-relative alignment immediately (b)
propagating the effects on line-height up to the closest ancestor with
line-relative alignment or to the line and (c) keeping track of the
largest contribution to line box height from a line-relative piece (the
line itself, or an element with line-relative alignment).  Really, this
needs to be compute prior to commiting the word to the line, and then
committed afterwards, along with committing the word.

4. TODO: more description

TODO: consider quirks mode floating tables

TODO: consider breaks with different priorities
