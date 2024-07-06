There's a now famous
[rant](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
on async I read some years ago before I worked with async much. My main
async language now is Rust, and if Twitter is anything to go by, people
still find async very frustrating.

While I sympathize with the discussion of function color, for me it
actually misses the more fundamental problem I have when using async: in
a large enough codebase with many contributors, it gets difficult to
predict and diagnose performance issues with runtime contention.

Here's an example of Wrong and Right.

Wrong [in Rust playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=9fbb0b9bd0ae70f820deee635d11f6cb):

```
use std::time::Instant;
use tokio::time::{sleep, Duration};

async fn blocking_task(id: usize) {
    // Simulate a blocking operation using std::thread::sleep (which blocks the thread)
    std::thread::sleep(Duration::from_secs(5));
    println!("Blocking task {} completed", id);
}

async fn non_blocking_task(id: usize) {
    for i in 1..=5 {
        println!("Non-blocking task {} running iteration {}", id, i);
        sleep(Duration::from_millis(500)).await;
    }
}

#[tokio::main]
async fn main() {
    let start = Instant::now();

    let mut handles = vec![];
    for id in 1..=5 {
        handles.push(tokio::spawn(blocking_task(id)));
    }

    for id in 6..=15 {
        handles.push(tokio::spawn(non_blocking_task(id)));
    }

    // Wait for all tasks to complete
    for handle in handles {
        let _ = handle.await;
    }

    let duration = start.elapsed();
    println!("Total execution time: {:?}", duration);
}
```

Returns (snipping the bottom of the output for brevity):

```
Non-blocking task 9 running iteration 5
Non-blocking task 14 running iteration 5
Non-blocking task 11 running iteration 5
Blocking task 4 completed
Total execution time: 15.000760031s
```

Now here's Right [in a Rust playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=3a9711e05ac38a2a2b4741a36482a156):

```
use std::time::Instant;
use tokio::task;
use tokio::time::{sleep, Duration};

async fn blocking_task(id: usize) {
    // Offload the blocking operation to a blocking thread pool
    task::spawn_blocking(move || {
        // Simulate a blocking operation using std::thread::sleep
        std::thread::sleep(Duration::from_secs(5));
        println!("Blocking task {} completed", id);
    })
    .await
    .expect("The blocking task panicked");
}

async fn non_blocking_task(id: usize) {
    for i in 1..=5 {
        println!("Non-blocking task {} running iteration {}", id, i);
        sleep(Duration::from_millis(500)).await;
    }
}

#[tokio::main]
async fn main() {
    let start = Instant::now();

    let mut handles = vec![];
    for id in 1..=5 {
        handles.push(tokio::spawn(blocking_task(id)));
    }

    for id in 6..=15 {
        handles.push(tokio::spawn(non_blocking_task(id)));
    }

    // Wait for all tasks to complete
    for handle in handles {
        let _ = handle.await;
    }

    let duration = start.elapsed();
    println!("Total execution time: {:?}", duration);
}
```

And its output:

```
Blocking task 3 completed
Blocking task 4 completed
Blocking task 5 completed
Total execution time: 5.001713304s
```

As [Alice](https://ryhl.io/blog/async-what-is-blocking/) instructs:

> If you remember only one thing from this article, this should be it:
> Async code should never spend a long time without reaching an `.await`.

Why is this? Because tokio is "load-balancing" your async function calls
as tasks across the worker threads that it manages. The runtime is calling
`poll` on the task and looking for one of two statuses: `Ready`, or
`Pending`. `Ready` is simple enough, it finished. But for a task to give
back `Pending`, it needed to `yield` somewhere. Let's synonymize `yield`
to `go_check_all_other_functions` -- then when you run a blocking function
inside async you're putting the `go_check_all_other_functions` all the way
at the end of the blocking function's work. There is no point to
interleave other work.

Functions don't have color, but they do have _size_ or _weight_. When you
use a global runtime which is premised on the benefits of being able to
switch tasks to other resources (this is why you obey `Send` after all --
you're then allowed to switch what thread your task is running on so all
CPU cores are utilized), then you really need all your resources to be
handling tasks of the same size. Imagine you have a road which has a bus
lane, car lane, and a bicycle lane. If you put a bus in each of the lanes,
you're definitely slowing down the cars and bicycles. If you keep the
buses in the bus lane, only that lane is slowed down by the extra stops.

What I find challenging with async over time is actually that we _don't_
have something like a _size_ or _weight_ of the function, that is, color
is opt-in and best-effort by a human. It's not hard for someone new to the
codebase to introduce a slowdown by not wrapping something in
`spawn_blocking`. And there's no easy way to diagnose these -- yes, [tokio
console](https://github.com/tokio-rs/console) exists, but in my
experience, it isn't really workable for production where I actually see
conditions of higher contention.

Sometimes I like to imagine a new language which would automatically
generate benchmark code per function such that their performance
characteristics could be known and force handling by dedicated resources.
Does such a thing exist?

---
layout: post
title: forget color in async, what about contention in a global runtime
date: 2024-07-07 14:21 -0400
---
