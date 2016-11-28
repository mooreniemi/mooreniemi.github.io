---
layout: post
title: table driven methods
categories: js programming
---

I read parts of [Code Complete](http://www.cc2e.com/Default.aspx) a year
or so ago and one of the sections I found kind of intriguing was
[Table-Driven
Methods](https://www.safaribooksonline.com/library/view/code-complete-second/0735619670/ch18.html).

I always found the example in the book a bit dense though. This weekend
I happened upon a good case to use a TDM, so I figure I would document it.
I'm not an amazing Javascript programmer so _caveat lector_!

The example I'm going to use is [Conway's Game of
Life](https://en.wikipedia.org/wiki/Conway's_Game_of_Life). I'd not
programmed it before myself and it was pretty fun to do. Its ruleset
affords us a good opportunity to implement slightly non-trivial
conditional logic.

Here's the rules I copied from [Disruptive
Communications](http://disruptive-communications.com/conwaylifejavascript/):

1. If a dead cell has exactly three live neighbours, it comes to life
2. If a live cell has less than two live neighbours, it dies
3. If a live cell has more than three live neighbours, it dies
4. If a live cell has two or three live neighbours, it continues living

Reading through these, we can easily imagine a complicated `switch` or set
of `if`/`else` statements to capture the logic. But at base, we really
have 2 dimensions that are fully enumerable into very small sets. One
dimension is whether a cell is alive or dead. The other dimension is the
number of live neighbors. These are both simple integers if we `clamp` the
live neighbor count at the maximum relevant `n`.

Here's how I get that live neighbor count:

```javascript
var neighborhood = getNeighbors(i).map(function(e) { return isLive(grid, e); });
var numberOfLiveNeighbors = neighborhood.filter(utils.identity).length.clamp(0, 4);
```

Using live neighbor count as our index key, we need to encode the
conditions of the second dimension. For instance, we know a live cell with
less than 2 neighbors will die, so the 0 and 1 position of a live cell's
states depending on neighbor count should both be `'dead'`:

```javascript
// NOTE: 0 and 1 are valued 'dead', because of rule #2
var nextLivingState = ['dead', 'dead', 'live', 'live', 'dead'];
var nextDeadState = ['dead', 'dead', 'dead', 'live', 'dead'];
```

Next is the outer dimension wrapping that inner secondary one:

```javascript
var nextPossibleState = [nextDeadState, nextLivingState];
```

Finally, we can just index in to that data structure based on whether the
current cell is alive (`isLive(grid, i)`) first and how many live
neighbors it has (`numberOfLiveNeighbors`) second:

```javascript
var nextState = nextPossibleState[isLive(grid, i)][numberOfLiveNeighbors];
```

Based on the `nextState` I calculate the new value of a cell on the next
"turn" in the GOL, though if you wanted, there's no reason not to collapse
this final step into the data structure we created. You could directly
call a method based on where you index in to, roughly like so (didn't execute this):

```javascript
var deathFunc = function(cell) { // kill cell };
var liveFunc = function(cell) { // birth/maintain cell };
var nextLivingState = [deathFunc, deathFunc, liveFunc, liveFunc, deathFunc];
var nextDeadState = [deathFunc, deathFunc, deathFunc, liveFunc, deathFunc];
var nextCellFunc = [nextDeadState, nextLivingState];
nextCellFunc[isLive(cell)][numberOfLiveNeighbors].call(cell);
```

My [working code is
here](https://github.com/mooreniemi/life/blob/master/content.js#L141).
It's nothing special, but it's a live example of how to make use of a TDM
and I think it makes the code easier to read and understand.
