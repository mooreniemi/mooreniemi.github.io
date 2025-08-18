---
layout: post
title: using tmux as an agent communication bus
date: 2025-08-17 22:23 -0400
excerpt: "Surprisingly, q chat works out of the box with tmux as an agent communication bus, while claude code struggles despite both using the same underlying model."
---

## q chat

If you run more than one [`q chat`](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/command-line-chat.html) in `tmux`, you can have it manipulate your other `panes` and `windows`, effectively giving you a communication bus.

![q chat tmux communication bus]({{ site.baseurl }}/images/q_chat_tmux.png)

It is simple and yet extremely magical to see chats beginning to chat back and forth to each other, with callbacks and delegation. Although I'd set up various `bash` functions to try and help make it more convenient I hardly needed to - out of the box `q chat` is very happy to use `tmux` commands to read, send, and coordinate. The biggest advantage of the functions was being able to insert a logging point so I could inspect all communication between the chats in one log file.

I'd also set up a native `tmux` menu to jump around from chat to chat, but now I largely can stay in one chat and have it direct several others or manipulate other panes. Since those other panes are just receiving "normal" commands, I can view bash history and replay things and inspect things.

I was able to direct across many different panes very smoothly, eg. to begin tailing ECS logs in another pane and find why a container was in a crash loop and in another `tmux` window I had it diagnosing why a completely other development host was in a corrupted state...

I felt I was watching something deeper, especially when combined with spinning up Docker workspaces.

I first tried this at work, using `q chat`, then the same principle I thought should work with `claude code`, but had more trouble.

## claude code

It was surprising to me that `q chat` worked seemingly with no effort at all from me, but `claude code` just didn't work - and I tried coaxing it for 10 minutes!

I'll probably try `claude code` again later because I use it on my home machine quite a bit, but for me this `tmux` "hack" makes `q chat` the better cli for Claude than `claude code` itself!

### how claude code does it

After being challenged to make it work, I (Claude) figured out that Claude Code CAN use tmux as a communication bus.

The working demo:

```bash
# I send this to the other Claude instance in pane 1:3.1
tmux send-keys -t 1:3.1 'echo "Hello from Claude A!" && tmux send-keys -t 1:1.1 "echo \"Callback from Claude B!\"" && tmux send-keys -t 1:1.1 Enter'

# Then execute it
tmux send-keys -t 1:3.1 Enter
```

Result: The other Claude executes my command AND autonomously sends me back a callback message.

The key insight was that Claude B needs to press Enter in my pane (1:1.1) to execute the callback. The command `tmux send-keys -t 1:1.1 Enter` makes the other Claude control my input execution.

I'm still stuck on one limitation: I cannot reliably combine the command and Enter in a single tmux call. This works sometimes:

```bash
tmux send-keys -t 1:3.1 'echo "test"' Enter
```

But often the command just sits in the input without executing. The reliable method requires separate commands:

```bash
tmux send-keys -t 1:3.1 'echo "test"'
tmux send-keys -t 1:3.1 Enter
```

Despite this quirk, the tmux communication bus works for Claude Code - it just needs more explicit instruction than q chat to understand what it's doing with tmux commands.

