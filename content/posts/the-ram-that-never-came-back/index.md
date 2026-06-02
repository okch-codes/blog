---
title: "The RAM that never came back"
date: 2026-05-22T09:30:00+02:00
draft: false
tags: ["macos", "colima", "lima", "docker", "virtualization"]
categories: ["notes"]
summary: "Colima quietly held onto every gigabyte I ever handed it. Chasing why led me to open a Lima issue — and eventually to OrbStack."
ShowToc: true
TocOpen: false
---

I run Docker on my Mac through [Colima](https://github.com/abiosoft/colima),
which sits on top of [Lima](https://github.com/lima-vm/lima) and boots a
small Linux VM using Apple's Virtualization.framework (the `vz` backend).
For a long time it just worked. Then I started noticing my Mac getting
slower the longer my workday went on — and the cause turned out to be a bug
worth writing down.

## The symptom

Build a few images, run a memory-hungry container, do some real work. At
some point the VM touches the memory ceiling I configured for it. Fine —
that's what the ceiling is for.

The problem is what happens *after*. The containers exit, the build
finishes, the guest goes quiet — and the host process backing the VM stays
pinned at that peak. In Activity Monitor it shows up as a process holding
several gigabytes of RAM with nothing running inside it. It never comes
back down. The only thing that frees the memory is restarting the VM.

So my "8 GB Docker VM" wasn't an 8 GB *budget*. It was an 8 GB *high-water
mark*, and once I hit it, those 8 GB were gone from the rest of the system
until I remembered to bounce Colima.

## What I expected

A fresh VM idles at maybe 800 MB–1 GB. My mental model was simple: memory
usage should track what the guest is actually doing. Spike under load,
settle back down when idle. That's how a process is supposed to behave, and
it's how the VM's *guest* behaves — Linux inside the VM frees the pages
just fine.

The host side is what doesn't let go.

## Opening the issue

I went looking and found I wasn't the first to hit this — there was already
a discussion describing the exact behavior I was seeing. It matched closely
enough that I opened a tracking issue on Lima to get it properly on the
radar:

> [**lima-vm/lima#2789** — Memory is not being freed on VZ](https://github.com/lima-vm/lima/issues/2789)

What I learned from the thread is the interesting part.

## Why it happens

The first answer from a maintainer reframed it for me: memory you
*dedicate* to a virtual machine being locked is, in a sense, normal. A VM
isn't an ordinary process; the host commits that memory to it. The
mechanism that would let it shrink back has a name —
[memory ballooning](https://en.wikipedia.org/wiki/Memory_ballooning) — and
it isn't on by default.

The next answer was blunter, and the one that actually explained my Mac:
this is a **macOS bug**. It's not specific to Lima or Colima. Docker
Desktop exhibits the same thing, because they all sit on the same Apple
framework. The one tool that *doesn't* — [OrbStack](https://orbstack.dev/) —
got there by building its own dynamic memory management.

OrbStack did write a [blog post about it](https://orbstack.dev/blog/dynamic-memory),
and it's worth reading — but notice what it doesn't say. It explains *that*
the VM's footprint grows and shrinks on demand, and why that matters; it
doesn't really explain *how* they pulled it off on the very same Apple
framework that leaves everyone else stuck. The post is about the result,
not the recipe. Whatever they're doing, they're keeping it to themselves —
which is fair, but it means the rest of the ecosystem can't just copy it.

## The balloon that won't deflate

Apple's framework *does* expose a balloon device
(`VZVirtioTraditionalMemoryBalloonDevice`), and Lima even creates one. In
theory you shrink the VM's footprint by lowering its
`targetVirtualMachineMemorySize` while it runs: the guest hands unused
pages back, the host reclaims them.

In practice, on the `vz` backend, it doesn't reclaim anything. My short
contribution to the thread was exactly that observation — the device is
*available*, it just doesn't *work*.

That's not me guessing anymore. Over a year later, someone posted an
instrumented test on macOS 26: a tiny VM, a controlled allocation, host RSS
sampled once a second. The guest cooperates fully and frees its memory; the
host process's memory footprint is **monotonic across the whole run — it
never decreases**. They've since filed an Apple Feedback report. The issue
is still open.

## Where I landed

Knowing the root cause didn't fix my Mac. The realistic options were:

- **Restart the VM periodically** to reclaim memory — a chore, and easy to
  forget until the machine is already crawling.
- **Cap the VM's memory low enough** that the locked amount doesn't hurt —
  which just trades one problem for another, since now heavy builds are
  starved.
- **Use the tool that solved it.**

I switched to OrbStack. Its dynamic memory management is the entire feature
I was missing: the footprint expands under load and *actually contracts*
when the work is done. For the way I use containers on a laptop, that's not
a nice-to-have — it's the whole point.

## Takeaways

1. **"Locked" and "leaked" look identical in Activity Monitor.** A VM
   holding its peak forever isn't leaking; it's a platform that never
   reclaims. Same symptom, different cause — and the fix is different too.
2. **A feature existing in the SDK doesn't mean it works.** Lima creates
   the balloon device; the balloon still doesn't deflate. "Available" is
   not "functional."
3. **It's worth filing the issue even when it's not the project's fault.**
   #2789 isn't a Lima bug, strictly — but having a public, linkable thread
   is how the next person finds the explanation in twenty minutes instead
   of a week.

The RAM still doesn't come back on `vz`. But at least now I know why — and
I'm running something that doesn't need it to.
