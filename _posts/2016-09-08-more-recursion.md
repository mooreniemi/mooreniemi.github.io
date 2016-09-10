---
layout: post
title: more on recursion
---

I don't think you can teach people the intuition around recursion. I think
the [old
joke](http://stackoverflow.com/questions/717725/understanding-recursion#comment529870_717725)
is basically true, and you need to practice. For myself though, I wanted
to capture how I think about it now in case it is instructive for others
or more likely curious to myself in the future.

### balancing

A good example of a recursive problem is determining whether a binary tree
is balanced. I'm going to use
[leetcode](https://leetcode.com/problems/balanced-binary-tree/)'s problem
definition:

> a height-balanced binary tree is defined as a binary tree in which the
> depth of the two subtrees of *every* node never differ by more than 1.

Now my first impulse was to try to solve this non-recursively by using BFS
and making sure none of the paths to leaves differed by more than 1. But
this doesn't satisfy the "every node" bit of our definition above. Here's
a tree that drives that home:

```
       1
     /   \
    2     2
   /       \
  3         3
 /           \
4             4
```

The path lengths to both leaves are the same but this tree isn't balanced.
Here's the balanced version in case it isn't clear:

```
      1
    /   \
   2     2
 /  \   /  \
3    4 4    3
```

When we see something like "every node" the bell should probably
immediately go off that this is a good case for recursion. Recursion means
applying the same set of rules or steps to each subproblem of our problem.
Given we know our subproblem must be each node (since we have to do this
for _every_ node), we can ask: what rules apply to any one node?

0. If you have no node, trivially this is balanced.
1. If you have one node with no children, this is balanced.
2. Time to check the kids:
    1. A node's left subtree must be balanced.
    2. A node's right  subtree must be balanced.
3. The difference in the left and right subtree heights/depths is `<= 1`.

Now, translating this to a recursive function takes another little leap,
which is understanding how the computer executes things and puts them on
the stack. I'm going to annotate the function, then go through its
execution by annotating our "bad" tree, then look again at the function.

```ruby
def is_balanced?(root)
		# rule 1
    return [true, 0] if root.nil?
		# rule 2
    return [true, 1] if root.left.nil? && root.right.nil?

		# rule 3.1
    l, l_depth = is_balanced?(root.left)
		# rule 3.2
    r, r_depth = is_balanced?(root.right)

		# rule 3 and rule 4
    [r && l && ((l_depth - r_depth).abs <= 1), [l_depth, r_depth].max + 1]
end
```

Now, at the end there's a bit more going on than my literal translation of
rule 4. (Like, what is `[l_depth, r_depth].max + 1` for?) If we walk
through using this function I think we can clarify why. I'm going to
abbreviate the function name to `ib?`.

```
                 ib?(1)
               /        \
              /          \
             /            \
      _____ /              \_______
     /                             \
   ib?(2)                        ib?(2)
  /     \                        /     \
ib?(3)  ib?(nil)             ib?(nil) ib?(3)
```

The first thing I want to observe is that we have to check both left and
right even when one is `nil`. Everywhere we see `ib?(nil)` we're going to
be returning `[true, 0]`, because rule 1 tells us a non-existent node is
balanced at a depth of 0.

Let's focus on one half of the tree for a moment.

```
                      ib?(1)
                      /   \
                _____/     * (ignored)
               /
            ib?(2)
            __|__
           /     \
         ib?(3)  ib?(nil)
         __|__
        /     \
    ib?(4)   ib?(nil)
   ____|____
  /         \
ib?(nil) ib?(nil)
```

When the function executes, the recursion "unfurls" by calls to rule
3 (lines 8 and 10) into this tree (and its mirrored side I am ignoring).
When it has completely "unfurled" out to every leaf, we roll ourselves
back up using rule 4 (line 13). Note that the _entire_ tree is put on the
stack as calls to `is_balanced?` before we ever start looking at any
individual nodes. So let's fill in the values (abbreviating `true` to
`t`):

```
                      ib?(1)
                      /   \
                _____/     * (ignored)
               /
            ib?(2)
f = end =>  [f, 3]
            __|___
           /      \
         ib?(3)  ib?(nil)
         [t, 2]   [t, 0]
         __|__
        /     \
    ib?(4)   ib?(nil)
    [t, 1]    [t, 0]
   ____|____
  /         \
ib?(nil) ib?(nil)
[t, 0]    [t, 0]
```

The extra bookkeeping we're doing on line 13 that isn't encompassed
directly by rule 4 could be posed as another sort of implicit rule we're
relying on from the tree structure: every jump up adds one to the max
subtree height. This saves us a whole other traversal just to assign
depths to the nodes.

Finally, the function returns as soon as we've rolled back up to `[f, 3]`
on line 6 because it fails the first part of our condition `l && r`.
I think this function has a nice shape for recursion, and we can annotate
it with those observations:

```ruby
def is_balanced?(root)
    # base cases where we can just give back values
    # immediately and stop unfurling into more calls
    # note we HAVE to set depth/height here so that we can
    # use it when we roll back up from these bases
    return [true, 0] if root.nil?
    return [true, 1] if root.left.nil? && root.right.nil?

    # recursive calls that cover the entire structure
    l, l_depth = is_balanced?(root.left)
    r, r_depth = is_balanced?(root.right)

    # up until this point we are unfurling
    ############################################################
    # now we have unfurled the structure into calls on our stack

    # now we can focus on: what does every call need to execute
    # if we weren't able to satisfy a base case? (but some node did)
    [r && l && ((l_depth - r_depth).abs <= 1), [l_depth, r_depth].max + 1]
end
```

The fact that we go all the way out to the leaves before we can roll back
in is key to the function working. Without it, we won't have values to use
in `[l_depth, r_depth].max + 1`. Without those values, we can't answer the
question `((l_depth - r_depth).abs <= 1))` for any node _except_ the
leaves.

To me, this example is a nice once because it helps show that you can
actually start by just trying to "unfurl" a tree of calls _first_ when
you're thinking through the problem. Once you do that, you need to ask
yourself:

1. how did I stop adding more calls? (those are your base cases)
2. what do I need to carry back up the tree to satisfy the rest of my
   calls? (that's the stuff you'll put after or with your recursive call
   or inductive step)

I'll try that now on another classic recursion example:
[Fibonacci](https://www.mathsisfun.com/numbers/fibonacci-sequence.html).

### fibonacci and tail recursion

The Fibonacci sequence starts with these values: `[0, 1, 1, 2, 3, 5, 8,
13, 21, 34]`. These values are determined by a simple set of rules. For
any `n` its Fibonacci number is `n-1 + n-2`. Taking the 4th Fibonacci number is
2 as our example:

```
                         fib(4)
              fib(4-1=3)      fib(4-2=2)
       fib(3-1=2) fib(3-2=1)      fib(2-1=1) fib(2-2=0)
fib(2-1=1) fib(2-2=0)              fib(1=1)   fib(0=0)
fib(1=1)   fib(0=0)
```

Now when we go back up, we need only add these results together.

```
                         fib(4)
              fib(4-1=3)      fib(4-2=2)
                1          +        1  = 2
       fib(3-1=2) fib(3-2=1)      fib(2-1=1) fib(2-2=0)
fib(2-1=1) fib(2-2=0)              fib(1=1) +  fib(0=0) = 1
fib(1=1) + fib(0=0) = 1
```

The function needs this template:

```ruby
def fib_of(n)
  # base cases
  # "unfurling" or inductive step
end
```

Well, we saw the base cases are 1 and 0.

```ruby
def fib_of(n)
  return 0 if n.zero?
  return 1 if n == 1
  # inductive step
end
```

And we know we have to call `fib_of` twice to get our branches and so that
we can add those branches together.


```ruby
def fib_of(n)
  return 0 if n.zero?
  return 1 if n == 1
  fib_of(n) + fib_of(n)
end
```

Finally, we know we have to do something to `n` according to our rule.

```ruby
def fib_of(n)
  return 0 if n.zero?
  return 1 if n == 1
  fib_of(n - 1) + fib_of(n - 2)
end
```

Now we'll get our expected result (`13`) when we call `fib_of(7)`. The
last thing to observe about recursion is that it can be expensive, because
we're using the call stack as our own data structure. If you try to
execute the recursive solution as written for Fibonacci for large values
you'll exhaust the call stack.

Our solution to _that_ problem is apparent as soon as we write out the
tree though. We're redoing a ton of work because our stack frames are
independent. My usual impulse here is to move away from recursion and just
iterate forward (so basically, manage everything myself[^1]):

```ruby
def fib_of(n)
  fibs = [0,1,1]
  2.upto(n) do |e|
    fibs[e] = fibs[e - 1] + fibs[e - 2]
  end
  fibs[n]
end
```

We can refine this a bit to avoid needing to keep track of the whole
sequence in memory:

```ruby
def fib_of(n)
  return n if n >= 0 && n < 2
  first, second = 0, 1
  next_val = nil

  2.upto(n) do
    next_val = first + second
    first, second = second, next_val
  end
  next_val
end
```

What if we want to keep it recursive though? Then we need to use tail
recursion. When I first read about tail recursive functions the definition
was something like the top [google
result](https://www.quora.com/What-is-tail-recursion-Why-is-it-so-bad):

> Tail recursion is a special kind of recursion where the recursive call
> is the very last thing in the function.

Until you've spent time "unfurling" and "furling back up" recursive
functions, the position of the call might not seem so important. What can
we possibly know, based on its position, about its behavior?

Well, if nothing else, we know its return value must be usable in whole or
in part as its parameters. So, we need to somehow change the function
signature to allow us to do that. When we do, we can carry over values
from previous stack frames _and only those values_. (Rather than the whole
sequence or the whole "tree" as I drew it above.) Here's an example where
to preserve the original function signature we use a lambda inside our
method to capture the recursive calls:

```ruby
def fib_of(n)
  fib_tail = ->(a, b, e) { e > 0 ? fib_tail.call(b, a + b, e - 1) : a }
  fib_tail.call(0, 1, n)
end
```

This works for us because, like our balancing example above, we can modify
our last values to be new values in the next call. So now, rather than us
managing the entire sequence in our `fibs` variable, we just reuse our
previous sum (`b`) and create a new sum (`a + b`).

Perhaps it helps to see that through a few calls:

```ruby
fib(3)
  fib_tail.call(a = 0, b = 1, e = 3)
    3 > 0 == true
    # fib_tail.call(b = 1, a + b = 1, e - 1 = 2)
    fib_tail.call(1, 1, 2)
      2 > 0 == true
      # fib_tail.call(b = 1, a + b = 2, e - 1 = 1)
      fib_tail.call(1, 2, 1)
        1 > 0 == true
        # fib_tail.call(b = 2, a + b = 3, e - 1 = 0)
        fib_tail.call(2, 3, 0)
          0 > 0 == false
          # : a
          2
```

If we're free to change our function signature, we have another option,
which is to use default arguments. I think this makes it even clearer how
it's working by removing the plumbing code of the lambda:

```ruby
def fib_of(n, a = 0, b = 1)
  n > 0 ? fib_of(n - 1, b, a + b) : a
end
```

Notice in those above examples we're moving around our base cases and
compacting them into a single predicate. I think the last example shows
that structure the best. We only need one predicate because we're seeding
`a` and `b` with the first 2 sequence values. Our only way to get to zero,
in other words, is through hitting one first.

Nice, right? So why have you never seen a function defined this way in
Ruby?

It's clearer to me now, having spent more time thinking about just what is
happening for the computer, why we care about tail calls, and why we or
our compiler need to optimize them away. Take our last tail
recursive function as an example in Ruby: if you try to execute
`fib_of(1000)` you'll hit `SystemStackError: stack level too deep`.

Although we're doing a lot better than our first most naive implementation
of the recursive solution (it begins to slow down as low as `fib_of(30)`),
we're still using the stack and the stack is a finite resource in our
system. For Ruby, we can [change the VM
settings](http://nithinbekal.com/posts/ruby-tco/) to get TCO, but
personally I am happy enough to write iterative code where that's not an
option.

[^1]: If you want to see totally explicit management of a stack check out this bizarre and inadvisable solution I came up with [here](https://github.com/mooreniemi/experiments/blob/master/spec/ifib_spec.rb#L22). It is a lot more work/LOC for me to do what the computer can do for me for 'free', but we won't hit a stack error!
