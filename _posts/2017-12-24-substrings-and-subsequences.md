---
layout: post
title: substrings and subsequences
categories: programming
date: 2017-12-24 15:12 -0500
---

In day to day work I don't get to make much use of dynamic programming. But for fun, let's consider LCS: longest common ([substring](http://algorithms.tutorialhorizon.com/dynamic-programming-longest-common-substring/)/[subsequence](http://algorithms.tutorialhorizon.com/dynamic-programming-longest-common-subsequence/)).

A subsequence is any sequence you can create by deleting but not rearranging **any** characters in a string.

So from "turtledoves" we can produce "loves."

A substring is any sequence you can create by deleting characters off **the beginning and ending of a string only**.

Thus whereas "loves" is invalid as a substring, "doves" is valid.

Substrings are a subset of subsequences, and both are subsets of the powerset of the string's characters. (This is why the naive algorithm of longest common subsequence is so expensive: you end up generating a powerset, which is an O(2^n) operation!)

I like LCS because there's a graphical intuition you can keep in mind, and it helps distinguish the two algorithms while reinforcing their distinctions. I haven't seen it written out elsewhere, so I wanted to make a note of it.

Here's the matrix ("program") formed for a Longest Common Subsequence:

| ☒ | ☒ | A | C | B | A | E | D |
|---|---|---|---|---|---|---|---|
| ☒ | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| A | 0 | 1 | 1 | 1 | 1 | 1 | 1 |
| B | 0 | 1 | 1 | 2 | 2 | 2 | 2 |
| C | 0 | 1 | 1 | 2 | 2 | 2 | 2 |
| A | 0 | 1 | 1 | 2 | 3 | 3 | 3 |
| D | 0 | 1 | 1 | 2 | 3 | 3 | 4 |
| F | 0 | 1 | 1 | 2 | 3 | 3 | 4 |

The longest common subsequence is "ACAD" of length 4. (The utmost bottom right value.)

You can see the cascading effect even if you don't recognize the algorithm. The matrix for these two strings in the substring matrix is very sparse in comparison:

| ☒ | ☒ | A | C | B | A | E | D |
|---|---|---|---|---|---|---|---|
| ☒ | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| A | 0 | 1 | 0 | 0 | 1 | 0 | 0 |
| B | 0 | 0 | 0 | 1 | 0 | 0 | 0 |
| C | 0 | 0 | 1 | 0 | 0 | 0 | 0 |
| A | 0 | 1 | 0 | 0 | 1 | 0 | 0 |
| D | 0 | 0 | 0 | 0 | 0 | 0 | 1 |
| F | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

Let's change the strings to be a bit more overlapping.

| ☒ | ☒ | A | B | A | E | D |
|---|---|---|---|---|---|---|
| ☒ | 0 | 0 | 0 | 0 | 0 | 0 |
| A | 0 | 1 | 0 | 1 | 0 | 0 |
| B | 0 | 0 | 2 | 0 | 0 | 0 |
| A | 0 | 1 | 0 | 3 | 0 | 0 |
| D | 0 | 0 | 0 | 0 | 0 | 1 |
| F | 0 | 0 | 0 | 0 | 0 | 0 |

Now we can see an aggregation again. Both algorithms rely on a core shape:

| x |  ☒  |
| ☒ | x+1 |

But subsequences are "looser" and thus make use of extra state. This state looks like this:

| ☒ | a        |
| b | max(a,b) |

For me, these shapes are nice to remember in motivating the pure index equations of LCS[i-1][j-1] and max(LCS[i-1][j], LCS[i][j-1]).
