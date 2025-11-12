---
layout: post
title: Little's Law, M/M/1 and M/M/c queues
usemathjax: true
date: 2025-11-11 23:33 -0500
---
How can we predict performance when load testing a server or set of
servers?

Well, it's a queue: you send work at a rate (something per something)
$$\lambda$$, it takes $$W$$ on average time to do the work, and at any
given time you have $$L$$ work items getting worked on. That's Little's
Law: $$L = W \cdot \lambda$$, where $$L$$ is requests in the system, $$W$$
is average base latency in seconds, and $$\lambda$$ is the arrival rate of
requests/transactions per second.

When doing a load test, you can just run your TPS generator until by some
measure you consider the system overloaded: "there's the max!" But you can
also make predictions of what this max, _in some sense_, **should** be.
You can run a request per second (assuming it runs in under a second) to get
the average _base_ latency of your system, $$W$$, and if you think about it,
you realize that it must equal $$\frac{1}{\mu}$$, where $$\mu$$ is the
maximum TPS the system can take.

So let's run an example and say that idle state average latency $$W$$ is
0.05s (50 milliseconds). From that you predict your system's $$\mu$$ (max
TPS) is 20. (Of course that seems insanely low but don't worry we'll get to
that in a moment.) First observe that it's a big assumption to believe your
system behaves the same idle as it does busy. As your system receives more
load, $$\lambda$$, you need to absorb that the utilization you read is an
_average_ and from the perspective of a given request, it is not guaranteed
access to idle resources at any given instant inside that second (measurement
window). That is, even well below _average_ 100% utilization, your request can
queue. You need a term to model that.

To model this factor of utilization, which will cause a waiting time
($$W_q$$ or $$P_{\text{wait}}$$), we can use the M/M/1 queue, which tells
us that $$W$$ _under load_ will become $$\frac{1}{\mu-\lambda}$$. That
really makes a lot of sense: if other requests are coming to the server in
the same second, there's a chance you will compete. To continue the above
example, this means at 15 TPS (75% utilization), your $$W$$ becomes
$$\frac{1}{5}$$ or 0.2s (200 milliseconds). That is 4 times slower than
your idle latency.

There's actually a dimensionless multiplier you can use directly to
calculate "how many times base latency will $$W$$ be under arrival load
$$\lambda$$?" $$\frac{1}{1-\frac{\lambda}{\mu}}$$, where $$\frac{\lambda}{\mu}$$ is directly the utilization
term, $$\rho$$.

Now, you should be asking somewhere in reading the foregoing, "hey, wait
a second, what kind of system are we modeling, is it true to my system?"

M/M/1 is a really nice, tractable, understandable  model. So obviously
it's also wrong. :) How wrong? We'll show a table, but let's actually talk
about what it represents first so we have an intuition for why it's wrong
and how to get an exact, correct prediction.

M/M/1 stands for M=Markovian, M=memoryless, 1=worker. Really this models the
simplest case: you have one worker handling things serially from one queue.
That's why you can just think of $$\frac{1}{\mu}$$ as average latency --
imagine a worker standing over a conveyer belt. The
$$\frac{1}{1-\frac{\lambda}{\mu}}$$ term is just saying, "how long will the
average task take when the conveyer belt is adding tasks $$\lambda$$?"

But hey, it's 2025, and your server has multiple CPUs, and each CPU is
probably capable of doing some work for you in parallel. Many server files
I see basically do `# cpus - 1` to give a server thread sitting on the
port and then the rest of the cores are workers handling requests...
A scenario properly modeled by M/M/c, where c is the number of cores or
servers.

M/M/c uses the Erlang C formula to get the $$W_q$$ which is conceptually
not too crazy ("what's the probability all servers are busy over the
probability of all server states?") but it's a lot of terms to work with
in practice.

Zoomed out, we're still dealing with:

$$
\boxed{W = \color{green}{W_q} + \color{red}{\frac{1}{\mu}}} \tag{1}
$$

But if we zoom in to $$W_q$$, we see:

$$
\boxed{\color{green}{W_q} = \frac{\color{blue}{\frac{A^c}{c!} \cdot \frac{1}{1-\rho}}}{\color{purple}{\sum_{k=0}^{c-1} \frac{A^k}{k!}} + \color{blue}{\frac{A^c}{c!} \cdot \frac{1}{1-\rho}}} \cdot \color{red}{\frac{1}{\mu}}} \tag{2}
$$

The blue term is just "what's the probability that all cores/servers/workers
are busy?" And the purple term is just "what's the probability overall of
all other states where only k servers are busy?" As a ratio, you're asking
"what's the probability all workers are busy out of all their potential
busyness states?" That's the probability you'll wait.

While $$(2)$$ isn't really _that_ bad, it's a bit much for quick napkin
math. We can use an approximation of the queue time for G/G/c by
Allen-Cunneen where the exponent is $$\alpha(c) = \sqrt{2(c+1)-1}$$.
What's nice about this term is that you can pretty much just memorize
common values of $$c$$, eg. 16 gives you $$\sqrt{34}-1 \approx 6$$. And
then $$W_q$$ is just $$\frac{\rho^{\alpha(c)}}{c(1-\rho)}
\cdot\frac{1}{\mu}$$ (service time or $$S$$). This works about like the
first pass we did above, where you know your idle time $$W$$ is 0.05, and
you can multiply it by $$\frac{1}{1-\frac{\lambda}{\mu}}$$ with
straightforward modifiers from $$c$$, $$\rho$$, and the approximated power
factor. (Recall $$\rho$$ is equal to $$\frac{\lambda}{\mu}$$.)

Tabling this out, we can see assuming no parallelism massively
underestimates our system[^mm1] but that our approximation of Erlang C is
pretty decent!

| ρ (per-core) | M/M/1 | M/M/16 (Erlang C) | M/M/16 (Allen–Cunneen) | Err A-C (%) | Err M/M/1 (%) |
|--------------|-------|-------------------|------------------------|-------------|---------------|
| 0.01         | 1.010 | 1.000             | 1.000                  | 0 %         | +1.0 %        |
| 0.10         | 1.111 | 1.000             | 1.000                  | 0 %         | +11.1 %       |
| 0.30         | 1.429 | 1.000             | 1.000                  | 0 %         | +42.9 %       |
| 0.50         | 2.000 | 1.001             | 1.004                  | +0.3 %      | +99.8 %       |
| 0.70         | 3.333 | 1.027             | 1.037                  | +1.0 %      | +224 %        |
| 0.80         | 5.000 | 1.095             | 1.106                  | +1.0 %      | +356 %        |
| 0.90         | 10.000| 1.370             | 1.376                  | +0.4 %      | +630 %        |

| Metric           | Allen–Cunneen         | M/M/1                  |
|------------------|-----------------------|------------------------|
| Mean abs error   | ≈ 0.4 %               | ≈ 195 %                |
| Max error        | ≈ 1.0 %               | ≈ 630 %                |

If we do our example with 50ms average base latency, you can see we need
to go above 70% before (to my taste anyway) we see significant latency
increase:

| ρ (per-core) | Queue term ρ^6/[16(1−ρ)] | Mean latency W (s) | Latency (ms) | % overhead vs base |
|--------------|--------------------------|--------------------|--------------|--------------------|
| 0.10         | 0.00000007               | 0.05000            | 50.00        | 0.00 %             |
| 0.30         | 0.000065                 | 0.05000            | 50.00        | 0.01 %             |
| 0.50         | 0.001953                 | 0.05010            | 50.10        | 0.20 %             |
| 0.70         | 0.0246                   | 0.05123            | 51.23        | 2.46 %             |
| 0.80         | 0.0818                   | 0.05409            | 54.09        | 8.18 %             |
| 0.90         | 0.332                    | 0.0666             | 66.6         | 33.2 %             |
| 0.95         | 0.918                    | 0.0959             | 95.9         | 91.8 %             |

So if you want a very simple prediction table, you can take common values
for cores, which are just bases of two, peg the desired utilization at 70% (you
don't want to waste hardware), and voila you can see your expected latency as
a function of $$c$$:

| cores (c) | α(c) = √(2(c+1)) − 1 | W/S multiplier | W @ S = 10 ms | W @ S = 50 ms | W @ S = 100 ms |
|-----------|----------------------|----------------|---------------|---------------|----------------|
| 2         | 1.449                | 1.9938         | 19.94 ms      | 99.69 ms      | 199.38 ms      |
| 4         | 2.162                | 1.3854         | 13.85 ms      | 69.27 ms      | 138.54 ms      |
| 8         | 3.243                | 1.1311         | 11.31 ms      | 56.55 ms      | 113.11 ms      |
| 16        | 4.831                | 1.0372         | 10.37 ms      | 51.86 ms      | 103.72 ms      |
| 32        | 7.124                | 1.0082         | 10.08 ms      | 50.41 ms      | 100.82 ms      |
| 64        | 10.071               | 1.0023         | 10.02 ms      | 50.11 ms      | 100.23 ms      |

You can see that you need to run with some "real" slack in your system to keep
overhead low, ie. you want to run at least 8-16 cores to stay close to your
base latency. But if you need to spend less money, you can just accept
higher latency. That's often a great option because humans don't tend to
notice 50ms versus 100ms for something like a web server, and (eg.) a 4xl
instance (16 vCPUs[^vcpu]) costs 8 times what an l (2 vCPUs) instance
costs.

Of course, in reality most servers are just waiting on their databases... :)

----

[^mm1]: It is "fun" to think just how bad latency is for asking a person to take on one more task when they're already busy... People love to estimate like they're M/M/c when they are more M/M/1. ;)

[^vcpu]: Note that a vCPU is a physical core + a hyperthread, so your real utilization is higher than you might expect. As AWS documentation states: "AWS documentation uses the term 'vCPU' synonymously with a 'thread' or a 'hyperthread' (or half of a physical core). Don't miss this factor of 2 when quantifying the performance or cost of an HPC application on AWS." [Source](https://docs.aws.amazon.com/wellarchitected/latest/high-performance-computing-lens/definitions.html)
