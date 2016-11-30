---
layout: post
title: table driven methods
categories: js programming
---

I read parts of [Code Complete](http://www.cc2e.com/Default.aspx) a year
or so ago and one of the sections I found kind of intriguing was
[Table-Driven
Methods](https://www.safaribooksonline.com/library/view/code-complete-second/0735619670/ch18.html).

I always found the example in the book a bit dense, and not found many
examples online that I felt were tractable. But this weekend I happened
upon a good case to use a TDM, so I figured I would document it. I'm not
an amazing Javascript programmer so as to the style of my implementation,
well, _caveat lector!_

The example problem to solve I'm going to use is [Conway's Game of
Life](https://en.wikipedia.org/wiki/Conway's_Game_of_Life). I'd not
programmed it before myself and it was pretty fun to do. Its ruleset
affords us a good opportunity to implement a TDM because it's slightly
non-trivial conditional logic that is still short enough to keep in mind
as we go.

Here's the rules I copied from [Disruptive Communications's blog post on
GOL](http://disruptive-communications.com/conwaylifejavascript/):

1. If a dead cell has exactly three live neighbours, it comes to life
2. If a live cell has less than two live neighbours, it dies
3. If a live cell has more than three live neighbours, it dies
4. If a live cell has two or three live neighbours, it continues living

It doesn't really matter if you don't know the domain of this problem as
long as you have a feeling for how to translate those rules into code.
Reading through these rules, I can easily imagine a complicated `switch`
or set of `if`/`else` statements to capture the logic. Like you have some
outer `if`/`else` for the liveness or deadness of a cell, and then some
additional statements within the liveness branch to codify the rest.

But at base, we really have 2 dimensions of logic that are fully and
independently enumerable into very small sets. We can actually put all
these values in a table.

One dimension, our "outer" dimension, is whether a cell is alive or dead.
If we think of a prototypical xy graph and x is the horizontal dimension,
then the values of x are "livingCell" and "deadCell" or we
could define them as integers like 0 and 1.

| livingCell | deadCell |
|------------|----------|
| ????       | ????     |

We get this outer dimension from this portion of our ruleset:

1. **If a dead cell** ...
2. _If a live cell_ ...
3. _If a live cell_ ...
4. _If a live cell_ ...

The other dimension (our "y" dimension or the row numbers of our table) is
the number of live neighbors.

| (numberOfLiveNeighbors) | livingCell | deadCell |
|-------------------------|------------|----------|
| 0                       | dead       | dead     |
| 1                       | dead       | dead     |
| 2                       | live       | dead     |
| 3                       | live       | live     |
| 4                       | dead       | dead     |

This corresponds to this portion of our ruleset:

1. ... exactly three live neighbours, ...
(`== 3`)
2. ... less than two live neighbours, ...
(`== 0 || == 1`)
3. ... more than three live neighbours, ...
(`== 4`)
4. ... two or three live neighbours, ...
(`== 2 || == 3`)

Both dimensions can be expressed as simple integers if we `clamp` the live
neighbor count at the minimum and maximum relevant `n`s (for our ruleset,
0 and 4).

I'm going to work inside out and start with the inner dimensions dependent
on live neighbor count. Here's how I grab that count, which is only
important because by
[`clamp`](https://github.com/mooreniemi/life/blob/master/utils.js#L40),
`numberOfLiveNeighbors` is guaranteed to be some value between 0 and 4.:

```javascript
// get the liveness of every neighbor of the cell I care about
var neighborhood = getNeighbors(i).map(function(e) { return isLive(grid, e); });
// filter them for only true values (for live cells)
var numberOfLiveNeighbors = neighborhood.filter(utils.identity).length.clamp(0, 4);
```
Using live neighbor count as our index key, we need to encode the
conditions of the dimension. For instance, one condition is that we
know a live cell with less than 2 neighbors will die, so the 0 and
1 position of a live cell's potential states should both be `'dead'`:

```javascript
// 0 and 1 are valued 'dead', because of rule #2
var nextLivingState = ['dead', 'dead', 'live', 'live', 'dead'];

// coincidentally, so are 0 and 1 for a dead cell
// all of the dead cell states are owing to rule #1
var nextDeadState = ['dead', 'dead', 'dead', 'live', 'dead'];
```

If we didn't have an additional dimension of whether the cell is alive or
dead itself to begin with, we'd only need one of the two Arrays defined
above, and could index directly into it. Since cells can be alive or dead,
we need to encode that too in another Array. The simplest thing to do is
to map dead states to 0 and live states to 1 in the "outer" "y" dimension
as I described above:

```javascript
// nextPossibleState[0] <==> dead
// nextPossibleState[1] <==> alive
var nextPossibleState = [nextDeadState, nextLivingState];
```

Now we can just index in to that data structure based on whether the
current cell is alive (`isLive(grid, i)`) first and how many live
neighbors it has (`numberOfLiveNeighbors`) second:

```javascript
var nextState = nextPossibleState[isLive(grid, i)][numberOfLiveNeighbors];
// for instance: nextPossibleState[0][4] => 'dead'
```

Based on the `nextState` I calculate the new value of a cell on the next
"turn" in the GOL, though if you wanted, there's no reason not to collapse
this final step into the data structure we created.

To do so, you could directly call a method based on where you index in to,
[like so](https://github.com/mooreniemi/life/blob/master/content.js#L120):

```javascript
// what we'll ultimately do to a cell depending on our conditional logic
function live(cell,i) {
  cell.push(compass.positionFromId(i));
}

function dead(cell) {
  cell.length = 0;
}

// our conditional logic encoded into Arrays
var nextLivingState = [dead, dead, live, live, dead];
var nextDeadState = [dead, dead, dead, live, dead];
var nextPossibleState = [nextDeadState, nextLivingState];

// indexing into our conditional logic based on cell state and neighbor state
nextPossibleState[isLive(grid, i)][numberOfLiveNeighbors].call(null, cell, i);
```

My [working code is
here](https://github.com/mooreniemi/life/blob/master/content.js#L96).
It's nothing special, but it's a live example of how to make use of a TDM
and I think it makes the code easier to read and understand.
