# CSS Inline Vertical Alignment and Line Wrapping Around Floats #
## ... or why implementing CSS 2.1 is harder than you thought ##

by [L. David Baron](http://dbaron.org)

<a rel="license" href="http://creativecommons.org/licenses/by-sa/3.0/"><img alt="Creative Commons License" style="border-width:0" src="http://i.creativecommons.org/l/by-sa/3.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/3.0/">Creative Commons Attribution-ShareAlike 3.0 Unported License</a>.

Prerequisite reading: [CSS 2.1](http://www.w3.org/TR/CSS21/), chapters 8, 9, and 10.

This document starts with some observations about some pieces of the CSS box model that are tricky to implement, and then describes how they are currently implemented in Gecko and how I believe they could be implemented better.

## Observations ##

### The width available for a line wrapped around floats is a function of that line's height ###

TODO: WRITE ME

(what's spec'd, implementations being buggy, wikipedia being common testcase but now works around the bug)

### The anchor point for a float need not be at a line breaking opportunity ###

TODO: WRITE ME

### Vertical align is tricky ###

TODO: WRITE ME

(aligned subtrees of line and of top/bottom elements within it)

## Multi-pass line layout in Gecko ##

## A possible single-pass line layout solution ##
