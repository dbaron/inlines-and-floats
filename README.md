# CSS Inline Vertical Alignment and Line Wrapping Around Floats #
## ... or why implementing CSS 2.1 is harder than you thought ##

by [L. David Baron](http://dbaron.org)

<a rel="license" href="http://creativecommons.org/licenses/by-sa/3.0/"><img alt="Creative Commons License" style="border-width:0" src="http://i.creativecommons.org/l/by-sa/3.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/3.0/">Creative Commons Attribution-ShareAlike 3.0 Unported License</a>.

Prerequisite reading: [CSS 2](http://www.w3.org/TR/CSS2/), chapters 8, 9, and 10.

This document starts with some observations about some pieces of the CSS box model that are tricky to implement, and then describes how they are currently implemented in Gecko and how I believe they could be implemented better.

## Observations ##

### The width available for a line wrapped around floats is a function of that line's height ###

Conceptually, although it's not described in the spec, floats have a
place that they come from:  essentially, where they would have been if
they'd been a non-floated inline.  In Gecko, we actually place an empty
inline there and call it the float's placeholder.  Because of the
[float placement rules](http://www.w3.org/TR/CSS2/visuren.html#float-position)
in CSS 2 (also see
[why they're bad](http://dbaron.org/log/20120827-specification-style)),
the float can't be above its placeholder, but it can be substantially
below its placeholder.  This can happen, for example, if there are
multiple floats that are too big to all fit next to each other, or if
some of the floats use the 'clear' property, which clears them past
other floats.  This sort of situation, with the top of a float being
substantially below its placeholder, is relatively common on Wikipedia.

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

### The anchor point for a float need not be at a line breaking opportunity ###

TODO: WRITE ME

### 'vertical-align' is tricky ###

TODO: WRITE ME

(aligned subtrees of line and of top/bottom elements within it)

## Multi-pass line layout in Gecko ##

## A possible single-pass line layout solution ##
