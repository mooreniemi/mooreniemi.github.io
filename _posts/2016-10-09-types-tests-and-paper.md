---
layout: post
title: types, tests, and paper
---

Owing to either my curiosity or my masochism, I'm taking a couple
[MOOCs](https://github.com/mooreniemi/mooc) this fall. As an opportunity
to learn some new tools, I'm trying to do all (or as much of the work as
will make sense) in Haskell and
[Spacemacs](https://github.com/syl20bnr/spacemacs)[^spacemacs]. In doing
today's homework, I felt like I had a good case study in my own thinking
process to examine, especially with regard to what "tools" I typically
use.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en"
dir="ltr">Isn&#39;t the correct answer to &quot;tests or types&quot;
something like: &quot;yes please! Both! You got anything else in there
that could help us out?&quot;</p>&mdash; Will Wilson (@WAWilsonIV) <a
href="https://twitter.com/WAWilsonIV/status/784891724866785280">October 8,
2016</a></blockquote> <script async
src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

I actually had started the homework before I saw it was assigned, because
it builds off a [lecture on the Array Inversion
Problem](https://www.coursera.org/learn/algorithm-design-analysis/lecture/GFmmJ/o-n-log-n-algorithm-for-counting-inversions-i).
I thought it was an interesting enough algorithm in the lecture that I'd
started voluntarily translating it. Here's the gist:

You have an array of integers. In O(n log n) you want to count all pairs
of inverted elements. In the array `[2,1]` the inverted pair is just that
array itself, so the count is one. In the array `[3,1,2]` there are two
inverted pairs: `[3,1]` and `[3,2]`. It's solvable in O(n log n) because
you can basically "piggyback" directly on [merge
sort](https://github.com/mooreniemi/experiments/blob/master/spec/merge_sort_spec.rb).

I began by writing type signatures of the psuedocode sketched out in the
lecture. As these changed, I got constant feedback about where in my
functions I needed to make updates. That type level feedback, while it can
be overwhelming sometimes, is very useful. One thing that I'll put out is
it forced me to think through the generic structure of the algorithm in
more detail.

I began by using a totally unconstrained type signature:

```haskell
inversions :: ([a], Int) -> ([a], Int)
```

As soon as I'd begun to write the function's implementation, this
signature became insufficient. The type checker did its best to tip me
off: what if `a` is not something that can be ordered? (All the sorting
operations I'm doing using `<` or `>`, and so on, won't work on a type
that has no instances defined.) So eventually I landed on:

```haskell
inversions :: (Ord a) => ([a], Int) -> ([a], Int)
```

This means my function works even on inputs I didn't originally have in
mind:

```haskell
inversions (["moo", "boo", "foo"],3)
-- => (["boo","foo","moo"],2)
```

My first tests were written after I'd already sketched out some code. (A
bit unusual for me.) The instructor had pointed out that in the worst
case, where an array is in reverse order, the count of inversions is `n
choose k`. So I wrote that out as my first set of tests because it was the
easiest way to generate test cases I could guarantee.

```haskell
inversions ([2,1], 2) `shouldBe` ([1,2], 1)
inversions ([3,2,1], 3) `shouldBe` ([1,2,3], 3)
inversions ([5,4,3,2,1], 5) `shouldBe` ([1,2,3,4,5], 10)
inversions ([6,5,4,3,2,1], 6) `shouldBe` ([1,2,3,4,5,6], 15)
inversions ([10000,9999..1], 10000) `shouldBe` ([1,2..10000], 49995000)
```

There's no reason to specify them out that way, I was just rusty at using
Haskell's
[QuickCheck](https://wiki.haskell.org/Introduction_to_QuickCheck1#Testing_with_QuickCheck)
so I avoided it until the end. Here's the above tests compressed into [a
property test in the midst of an hspec
test](http://hspec.github.io/quickcheck.html):

```haskell
context "property test on NonNegative ints" $ do
  it "for total reversed sort order reverses list and count n choose k" $ property $
    -- n < 100000 ==> is our way of throwing out super big values
    \(NonNegative n) -> n < 100000 ==>
                        let xs = [n,n-1..1]
                        in inversions (xs, n) === ([1..n], n * (n-1) `div` 2)

```

In trying to get these tests to pass, I traced the first bug by writing
out the code on a piece of paper, with an empty line between each code
line, and filling in the assignments through an execution. This tends to
help nonsensical values jump out at me. (In something like Ruby I "cheat"[^ruby]
and use `binding.pry` to walk through with `ls -l` and check.) Here
was the culprit that became immediately obvious to me on paper:

```haskell
-- just like the first recursive call in merge sort, with an error
half_length = n `div` 2
(c,y) = inversions (snd $ splitAt half_length a, half_length)
```

In merge sort, you need to recursively halve the original array. In order
to save us the work of recalculating length (an O(n) operation on `List`) it's
helpful to pass length in to the recursion directly. When you have an
array with an even number of elements, the length of the subarrays is, of
course, equal. But with an oddly numbered array, you need to do a little
extra work to keep track of which side got the "odd one out", so to speak.

```haskell
half_length = n `div` 2
-- make sure the right side has the correct length
(c,y) = inversions (snd $ splitAt half_length a, n - half_length)
```

The last bug was in my implementation of this algorithm's version of
`merge`, which is called `countSplitInv` in my code. It does all the stuff
a normal merge function does, but it needs to accumulate the count of the
inverted pairs, too. From the lectures, I intuitively grasped the need to
count all "remaining" elements in the left side on every case where left's
current element was bigger than right's current element. I started out
with inefficiently calling `length` on `n`, but eventually moved on to
passing `left_length` in (we have to calculate it in the `inversions`
function anyway so why waste it?):

```haskell
countSplitInv :: (Ord a) => [a] -> Int -> [a] -> ([a], Int) -> ([a], Int)
countSplitInv l left_length r (acc,n)
  | null l || null r = ((reverse acc) ++ l ++ r, n)
  | otherwise = case (l,r) of
                  (x:xs,y:ys) -> if x > y
                                 then countSplitInv l left_length ys (y:acc, n+left_length)
                                 else countSplitInv xs left_length r (x:acc, n)

inversions :: (Ord a) => ([a], Int) -> ([a], Int)
inversions ([], 0) = ([], 0)
inversions (a,n)
  | n == 1 = (a,0)
  | otherwise = (d, x+y+z)
  where
    left_length = n `div` 2
    right_length = n - left_length
    (l,r) = splitAt left_length a
    (b,x) = inversions (l, left_length)
    (c,y) = inversions (r, right_length)
    (d,z) = countSplitInv b left_length c ([],0)
```

This saved the algorithm a lot of unnecessary work. Speaking of which, I'm
skipping over another issue in Haskell. Using `++` as I did in my earliest
versions was incredibly inefficient. It introduced a bunch of O(n)
operations: one for every call of `countSplitInv`, which is already being
run at least O(n) times... Ouch. That's why you'll see I switched to
`y:acc` and `x:acc` with `reverse acc` in my base case.

Even through those changes, I was still wrestling with getting the wrong
results. This last bug took me the longest to diagnose, and I can put that
down to not creating varied enough test cases. My test cases up to this
point had been unbalanced[^unbalanced]: I was overexercising the first
branch my version of merge, such that I wasn't account for all the
necessary behavior when `y > x`.

In concrete terms, I'd made a bunch of cases where arrays were largely or
only in reverse order, but hadn't made enough cases where the array was
effectively already sorted or "more" sorted than not. This meant part of
my recursion tree was pretty "unwatered" shall we say, and it took me
forever to figure out which branch I was neglecting. At this point, I went
around and grabbed some more test cases from working implementations.

I could have (and should have) just written out the recursion trees
(again, I do that on paper) until I saw the gap, but I was tired, and so
I compared my implementation to several others. In comparing to iterative
solutions I saw, I realized my problem was really that I wasn't thinking
through Haskell's syntax in my pattern matching; I was not thinking about
how I'd carved `x` off the front of `xs`. This is why _only_ the `else`
branch needs to modify the size of `left_length`: because it modified `l`!
(Duh!) Here's the upshot:

```haskell
countSplitInv :: (Ord a) => [a] -> Int -> [a] -> ([a], Int) -> ([a], Int)
countSplitInv l left_length r (acc,n)
  | null l || null r = ((reverse acc) ++ l ++ r, n)
  | otherwise = case (l,r) of
                  (x:xs,y:ys) -> if x > y
                                 then countSplitInv l left_length ys (y:acc, n+left_length)
                                 -- gotta modify left_length!
                                 else countSplitInv xs (left_length-1) r (x:acc, n)
```

All together then, the [working
code](https://github.com/mooreniemi/mooc/blob/master/test/InversionSpec.hs)
I used to submit my homework [looks like
this](https://github.com/mooreniemi/mooc/blob/master/src/Inversion.hs):

```haskell
module Inversion where
-- https://www.coursera.org/learn/algorithm-design-analysis/lecture/GFmmJ/o-n-log-n-algorithm-for-counting-inversions-i

countSplitInv :: (Ord a) => [a] -> Int -> [a] -> ([a], Int) -> ([a], Int)
countSplitInv l left_length r (acc,n)
  | null l || null r = ((reverse acc) ++ l ++ r, n)
  | otherwise = case (l,r) of
                  (x:xs,y:ys) -> if x > y
                                 then countSplitInv l left_length ys (y:acc, n+left_length)
                                 else countSplitInv xs (left_length-1) r (x:acc, n)

inversions :: (Ord a) => ([a], Int) -> ([a], Int)
inversions ([], 0) = ([], 0)
inversions (a,n)
  | n == 1 = (a,0)
  | otherwise = (d, x+y+z)
  where
    left_length = n `div` 2
    right_length = n - left_length
    (l,r) = splitAt left_length a
    (b,x) = inversions (l, left_length)
    (c,y) = inversions (r, right_length)
    (d,z) = countSplitInv b left_length c ([],0)
```

Now, there's one observation looming behind these bugs. They have a common
source: because I implemented the algorithm directly from the lectures
(and there, it was sketched out with length passed in) and then was
worried about `length` not being a "free" operation on `List` in Haskell,
I bent over backwards to pass length through the functions. This was, as
we see, error-prone.

With a better chosen type
([Sequence](https://hackage.haskell.org/package/containers-0.5.8.1/docs/Data-Sequence.html)),
I can have a cheap length function but still append an element to the end
of the container. Here's what that looks like:

```haskell
module Inversion where
import Data.Sequence as S
import Data.Foldable as F

countSplitInv :: (Ord a) => S.Seq a -> S.Seq a -> (S.Seq a, Int) -> (S.Seq a, Int)
countSplitInv l r (acc,n)
  = case (viewl l, viewl r) of
    (EmptyL,_) -> (acc >< r, n)
    (_,EmptyL) -> (acc >< l, n)
    (x:<xs,y:<ys) -> if x > y
      then countSplitInv l ys (acc|>y, n + S.length l)
      else countSplitInv xs r (acc|>x, n)

inversions' :: (Ord a) => S.Seq a -> (S.Seq a, Int)
inversions' a
  | n <= 1 = (a, 0)
  | otherwise = (d, x+y+z)
  where
    n = S.length a
    (l,r) = S.splitAt (n `div` 2) a
    (b,x) = inversions' l
    (c,y) = inversions' r
    (d,z) = countSplitInv b c (S.empty,0)

inversions :: (Ord a) => ([a], Int) -> ([a], Int)
inversions (a,_) = let (list,count) = inversions' $ S.fromList a
                       in (F.toList list, count)
```

You'll notice the `inversions`/`inversions'` wrapping. I have a wrapping
function that takes `([a], Int)` just because my tests were already
written for that interface, and this was an easy way to avoid rewriting
any of them.

Comically, I happened upon the first bug again in refactoring to
`Sequence`![^bug] I was using `y<|acc` and `x<|acc` instead of the correct
ordering I needed, which you see above. Still, that's only half the bugs
of the original implementation. :)


[^spacemacs]: I **LOVE** Spacemacs. If you're a vim person, [give it a chance](https://github.com/syl20bnr/spacemacs/blob/master/doc/VIMUSERS.org).
[^ruby]: A side-note about Ruby vs Haskell. I think of working in these two languages as being very opposite in some ways. One way I was noticing as I was working this problem through is how often in Ruby I need to inspect the value in the runtime because of duck typing. I often don't know exactly what any value (object) _is_ let alone what it can do until I call `.inspect` or `.methods` on it. In Haskell, there's never any ambiguity like this.
[^unbalanced]: I think it'd be interesting, though I'm unsure if it'd be useful, to involve a coverage tool in one's tests. (In Ruby you could use [simplecov](https://github.com/colszowka/simplecov), in Haskell [hpc](https://wiki.haskell.org/Haskell_program_coverage).) Specifically one that could show a sort of flamegraph of how often tests were hitting branches. You could get some heuristic value, perhaps, from seeing where you're traversing less often.
[^bug]: It was a holdover from the optimization of `List` where I expected to use `reverse` in my base case.
