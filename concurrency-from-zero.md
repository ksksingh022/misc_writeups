# Concurrency — From Zero

**Who this is for:** someone who has never studied this before. No prior knowledge assumed. No jargon until it is earned — and when a fancy word appears, it is introduced in a box like this:

> 📖 **The fancy word:** what people actually call this in books and docs.

**How to read this:**

| Part | What it covers | Code? |
|---|---|---|
| Part 1 | The idea itself, using kitchens and people | None |
| Part 2 | Where this shows up in real software | None |
| Part 3 | How Python does it, and Python's one weird rule | Yes |
| Part 4 | Cheat sheet, mistakes, glossary | A little |

Read Part 1 slowly. Everything else is built on it.

---

# PART 1 — The Idea

## 1.1 The one picture that explains everything

Imagine you are cooking a meal. You have three jobs:

| Job | Time it takes | Do you have to be there? |
|---|---|---|
| Boil water | 10 min | No — the stove does it |
| Chop vegetables | 5 min | **Yes** — your hands are needed |
| Bake bread | 20 min | No — the oven does it |

**The slow way.** Put the water on. Stand and stare at it for 10 minutes. Then chop for 5 minutes. Then put bread in the oven and stare at it for 20 minutes.

```
THE SLOW WAY  (one cook, one job at a time, waiting is wasted)

minutes  0        5        10       15       20       25       30       35
         |--------|--------|--------|--------|--------|--------|--------|
cook     [==== staring at water ====][ chop ][======= staring at oven =======]

TOTAL: 35 minutes
```

**The smart way.** Put the water on. While it boils, chop the vegetables. Put the bread in the oven. While it bakes, you are free.

```
THE SMART WAY  (still ONE cook — but waiting time gets reused)

minutes  0        5        10       15       20       25
         |--------|--------|--------|--------|--------|
stove    [~~~~ water boiling by itself ~~~~]
oven              [~~~~~~~~ bread baking by itself ~~~~~~~~]
cook     [ chop  ][ free ] [ free ] [ free ] [ free ]

TOTAL: ~22 minutes    ← same one cook, 13 minutes saved
```

Nothing got faster. The cook did not gain extra hands. **The only change is that the cook stopped standing idle during waiting periods.**

That is concurrency. That is the whole idea.

> 📖 **The fancy word:** *concurrency* — structuring work so that the waiting parts of different jobs overlap.

---

## 1.2 The single most important distinction in this whole document

People mix up two words constantly. They are not the same thing.

| | **Concurrency** | **Parallelism** |
|---|---|---|
| Plain meaning | Juggling many jobs by switching between them | Actually doing many jobs at the same instant |
| Kitchen version | 1 cook, cleverly ordered | 2 cooks, 2 pairs of hands |
| How many workers? | Can be just **one** | Must be **more than one** |
| What it fixes | Wasted waiting | Not enough hands |
| Is it about design or hardware? | **Design** — how you organise the work | **Hardware** — how many workers exist |

```
SEQUENTIAL        CONCURRENT             PARALLEL
(one at a time)   (one worker,           (two workers,
                   switching)             truly at once)

worker: A A A     worker: A B A B A B     worker1: A A A A
        A A A             B A B A B A     worker2: B B B B
        B B B
        B B B     "at any single instant   "at a single instant
                   only one is running,     BOTH are running"
"A fully finishes  but both are in
 before B starts"  progress"
```

A one-line way to remember it:

- **Concurrency** = *dealing with* many things at once.
- **Parallelism** = *doing* many things at once.

You can have concurrency without parallelism (one clever cook). You can have parallelism without concurrency (two cooks who never coordinate). Most real systems have both.

---

## 1.3 Two kinds of work — this decides everything later

Every piece of work a program does falls into one of two buckets. **If you learn only one thing from this document, learn this table.**

| | **Waiting work** | **Thinking work** |
|---|---|---|
| What is happening | The program asked someone else for something and is idle until the answer arrives | The program is actively calculating, nonstop |
| Kitchen version | Water boiling, bread baking | Chopping, kneading |
| Who is busy? | Something outside — network, disk, another machine | Your worker's own brain |
| Real examples | Downloading a file, reading from a database, calling an API, reading a file from disk, waiting for a user to click | Resizing an image, sorting a huge list, encrypting data, running a maths loop |
| Does the worker have free capacity? | **Yes — tons.** It is just sitting there. | **No.** It is maxed out. |
| Fix | **Overlap the waits** (concurrency) | **Add more workers** (parallelism) |

> 📖 **The fancy words:** waiting work = *I/O-bound*. Thinking work = *CPU-bound*.
> ("I/O" just means input/output — talking to anything outside the program.)

**The decision rule that follows from this:**

```
                    What is my program mostly doing?
                                 │
              ┌──────────────────┴──────────────────┐
              │                                     │
         WAITING a lot                        THINKING a lot
    (network, disk, database)              (maths, loops, crunching)
              │                                     │
              ▼                                     ▼
      Overlap the waiting.                  Add more workers.
      One worker is enough.                 One worker cannot be
      Huge speed-ups possible.              made faster by tricks.
```

Getting this wrong is the #1 beginner mistake. Adding 8 workers to a job that is 95% waiting is wasteful. Cleverly overlapping waits in a job that has zero waiting achieves exactly nothing.

---

## 1.4 The three ways to get concurrency

There are only three fundamental strategies. Everything you will ever read about is one of these three, or a mix.

### Strategy A — Separate kitchens

Hire 4 cooks. Give each their own kitchen, their own knives, their own fridge. They never touch each other's stuff.

> 📖 **The fancy word:** *processes*.

- ✅ Totally safe — nobody can mess up anyone else's ingredients.
- ✅ If one kitchen burns down, the others are fine.
- ❌ Expensive — you paid for 4 fridges.
- ❌ Sharing is painful. To give Cook B a chopped onion, Cook A has to package it up and physically walk it over.

### Strategy B — One kitchen, many cooks

Hire 4 cooks. Put them all in the **same** kitchen with **one** shared set of tools and **one** shared fridge.

> 📖 **The fancy word:** *threads*.

- ✅ Cheap — one kitchen.
- ✅ Sharing is instant — the fridge is right there.
- ❌ **They collide.** Two cooks grab the same knife. One takes the eggs the other was about to use. This causes real, nasty bugs (section 1.6).

### Strategy C — One hyper-organised cook

Hire **one** cook, but one with a rule: *never stand still*. The moment anything needs waiting, immediately go do something else, and come back when it is ready.

> 📖 **The fancy words:** *asynchronous programming*, *event loop*, *cooperative multitasking*.

- ✅ Cheapest of all — one cook, one kitchen.
- ✅ No collisions — only one person is ever touching anything.
- ✅ Scales enormously — one cook can look after hundreds of pots.
- ❌ Useless if the work is thinking work. One brain is one brain.
- ❌ **One badly behaved job freezes everything.** If the cook decides to knead dough for 10 minutes straight, every pot boils over. Cooperation is mandatory.

### Side by side

| | **A: Processes** | **B: Threads** | **C: Async** |
|---|---|---|---|
| Analogy | Separate kitchens | Shared kitchen | One tireless cook |
| Number of workers | Many | Many | **One** |
| Memory | Separate | Shared | Shared (only one user) |
| Cost to create | Heavy | Medium | Almost free |
| Who decides when to switch? | The operating system | The operating system | **Your own code** |
| Can switch happen mid-sentence? | Yes | **Yes** ← danger | No — only at points you mark |
| Collision bugs? | Very unlikely | **Very likely** | Rare |
| Good for thinking work | ✅ **Yes** | ❌ (see Part 3) | ❌ No |
| Good for waiting work | ⚠️ Works, but overkill | ✅ Yes | ✅ **Best** |
| Realistic count | ~ number of CPU cores (4–16) | Tens to a few hundred | Thousands |

---

## 1.5 Who is actually in charge of the switching?

There are two styles, and the difference matters.

```
PREEMPTIVE  (used by processes and threads)
────────────────────────────────────────────
A manager with a stopwatch stands over the workers.
Every few milliseconds: "STOP. You — sit down. You — go."

Workers have no say. A worker can be stopped
mid-action, even halfway through writing a number.

  worker A: ▓▓▓▓|      |▓▓▓|        |▓▓▓▓▓
  worker B:     |▓▓▓▓▓▓|   |▓▓▓▓▓▓▓▓|
                 ↑ forced switch, could happen anywhere

+ One greedy worker can't hog everything.
- Switching can happen at the worst possible moment. → bugs.


COOPERATIVE  (used by async)
────────────────────────────────────────────
No manager. Each worker voluntarily says
"I'm about to wait — someone else go ahead."

  task A: ▓▓▓▓ (await) ......... ▓▓▓
  task B:      ▓▓▓▓▓▓ (await) ........
                ↑ switch only at marked points

+ You know exactly where switches happen. → far fewer bugs.
- A worker that never volunteers freezes everyone.
```

This is the single deepest difference between threads and async, and it explains why threads are bug-prone and async is not.

---

## 1.6 What goes wrong: the shared whiteboard

This is the classic disaster, and it is worth understanding properly.

A whiteboard on the wall shows the number **10**. Two people are each told: *"add 1 to that number."*

To a human "add 1" feels like one action. It is not. It is three:

1. **Read** the number off the board
2. **Add 1** in your head
3. **Write** the new number back

Now watch what happens if the manager's stopwatch interrupts at the wrong moment:

| Time | Person A | Person B | Whiteboard says |
|---|---|---|---|
| 1 | reads → sees 10 | | 10 |
| 2 | | reads → sees 10 | 10 |
| 3 | thinks: 10 + 1 = 11 | | 10 |
| 4 | | thinks: 10 + 1 = 11 | 10 |
| 5 | writes 11 | | **11** |
| 6 | | writes 11 | **11** |

Two people each added 1. The board should say **12**. It says **11**.

**One update silently vanished.** No error message. No crash. Just a wrong number.

> 📖 **The fancy word:** *race condition* — the answer depends on the exact timing of who got interrupted where. Run it again and you might get the right answer, which is what makes these bugs so horrible to find.

### The fix: the marker

Put **one** marker in the room. Rule: *you may only touch the whiteboard while holding the marker.*

| Time | Person A | Person B | Whiteboard |
|---|---|---|---|
| 1 | takes marker 🖊️ | | 10 |
| 2 | reads 10, thinks 11, writes 11 | ⏸️ waiting for marker | **11** |
| 3 | puts marker down | | 11 |
| 4 | | takes marker 🖊️ | 11 |
| 5 | | reads 11, thinks 12, writes 12 | **12** ✅ |

Correct. But notice: **Person B did nothing while waiting.** You traded speed for correctness. That is always the trade.

> 📖 **The fancy words:** the marker is a *lock* (or *mutex*). The whiteboard section is the *critical section*. Making something un-interruptible is making it *atomic*.

### The new problem the fix creates: everyone frozen forever

Two shared items now: a **marker** and an **eraser**. Both are needed to do the job.

```
   Person A                        Person B
   ┌──────────────────┐            ┌──────────────────┐
   │ holds: 🖊️ marker  │           │ holds: 🧽 eraser  │
   │ needs: 🧽 eraser  │            │ needs: 🖊️ marker  │
   └────────┬─────────┘            └─────────┬────────┘
            │                                │
            └──── waiting for B ─────┐       │
                                     │       │
            ┌──── waiting for A ─────┼───────┘
            ▼                        ▼
          Neither will ever let go. Forever.
```

> 📖 **The fancy word:** *deadlock*. The standard prevention: **everyone always picks up shared items in the same agreed order** (marker first, then eraser — always). Then this loop cannot form.

### The related problems, briefly

| Problem | Plain description |
|---|---|
| **Race condition** | Result depends on who got interrupted when. Wrong answers, no crash. |
| **Deadlock** | Two workers each hold what the other needs. Frozen forever. |
| **Starvation** | One unlucky worker never gets a turn while others keep grabbing it. |
| **Livelock** | Two people in a corridor both stepping aside, forever. Busy, no progress. |
| **Lock contention** | So many people queue for the marker that you're effectively back to one worker. All that concurrency, zero benefit. |

**The takeaway:** shared stuff is where all the pain lives. This is why Strategy A (separate kitchens) and Strategy C (one cook) are so much safer than Strategy B.

---

## 1.7 The costs nobody mentions

Concurrency is not free. Three hidden bills:

| Cost | What it means |
|---|---|
| **Setup cost** | Creating a worker takes time and memory. A new kitchen costs a lot; a new pair of hands costs less; a new "job on the tireless cook's list" costs almost nothing. |
| **Switching cost** | Every switch means putting down what you were doing, remembering where you were, and picking up something else. Switch too often and you spend all your time switching. |
| **Coordination cost** | Locks, queues, passing data between kitchens. Real time spent on bookkeeping instead of work. |

Which is why:

```
   speed-up
      ▲
      │           ╭──────────────  ← flattens out (Amdahl's law:
      │        ╭──╯                   the parts that MUST be
      │      ╭─╯                      sequential set the ceiling)
      │    ╭─╯          ╲___
      │  ╭─╯                 ╲___  ← then gets WORSE
      │ ╭╯                        ╲   (switching + lock costs
      │╭╯                          ╲   exceed the benefit)
      └──────────────────────────────►  number of workers
       1   2   4   8   16  32  64  128
```

More workers is not linearly better, and past some point is actively worse. **Always measure. Never assume.**

---

## 1.8 Part 1 in one box

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Concurrency = overlapping WAITING.   Needs 1 worker.        │
│  Parallelism = overlapping THINKING.  Needs many workers.    │
│                                                              │
│  Waiting work (I/O)   → overlap it. Massive wins.            │
│  Thinking work (CPU)  → split it across real workers.        │
│                                                              │
│  Three tools:  separate kitchens (processes)                 │
│                shared kitchen    (threads)                   │
│                one tireless cook (async)                     │
│                                                              │
│  All the bugs come from SHARED THINGS being touched          │
│  by two workers at once. Locks fix it, and cost speed.       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

# PART 2 — Where This Sits in Real Software

Part 1 was abstract. This part answers: *where does this actually live, and who uses it for what?*

## 2.1 Where it sits in the stack

Concurrency is not one thing in one place. It is a chain running from your code down to the metal.

```
┌────────────────────────────────────────────────────────────────┐
│  YOUR APPLICATION CODE                                         │
│  "handle 1000 users"  "process these 500 files"                │
│  ── you express the INTENT here ──                             │
├────────────────────────────────────────────────────────────────┤
│  LIBRARIES & FRAMEWORKS                                        │
│  web server, database driver, HTTP client                      │
│  ── most concurrency is already done for you here ──           │
├────────────────────────────────────────────────────────────────┤
│  LANGUAGE RUNTIME                                              │
│  threads, event loop, worker pools                             │
│  ── this is where Python's rules kick in (Part 3) ──           │
├────────────────────────────────────────────────────────────────┤
│  OPERATING SYSTEM                                              │
│  the manager with the stopwatch — decides who runs when        │
│  ── the matchmaker between your intent and the hardware ──     │
├────────────────────────────────────────────────────────────────┤
│  HARDWARE: CPU CORES                                           │
│  4 cores = 4 real pairs of hands. That is the hard ceiling.    │
│  ── real PARALLELISM only exists here ──                       │
└────────────────────────────────────────────────────────────────┘
```

**The key insight:** you *design* concurrency at the top. Parallelism is a *physical fact* at the bottom. The OS is the matchmaker. You can ask for 1000 concurrent jobs on a 4-core machine — the OS will happily accept, and run at most 4 at any real instant.

## 2.2 Real systems and what they use

| System | The problem it faces | What kind of work | Approach used |
|---|---|---|---|
| **Web server** (nginx, FastAPI, Node) | 10,000 users connected, almost all just waiting on network | Waiting | Async event loop, or a pool of threads |
| **Database** | Many queries at once, all touching the same tables | Both | Threads + heavy locking; transactions are locks with rules |
| **Web browser** | One frozen tab must not freeze the whole browser | Both | **Separate process per tab** (separate kitchens), plus background threads |
| **Phone app** | Screen must stay smooth while data downloads | Waiting | One "UI thread" that must **never** be blocked + background threads |
| **Video game** | Draw 60 frames/sec while physics, audio, AI all run | Thinking | Dedicated threads per subsystem, tightly synchronised |
| **Operating system** | Run 200 programs on 8 cores | Both | It *is* the scheduler — everything above depends on it |
| **Data pipeline / Spark** | 1 TB of data to transform | Thinking | Split data into chunks, process on many machines |
| **ML training** | Enormous matrix maths | Thinking | GPUs (thousands of tiny workers) + parallel data loading |
| **Web scraper / API client** | Fetch 5,000 URLs | Waiting | Async, with a cap on how many at once |
| **Calling an LLM API 200 times** | Each call takes 3 s, almost all of it waiting for the server | Waiting | Async with a concurrency limit — 200 calls in ~5 s instead of 10 min |

## 2.3 Three worked examples

### Example A — A website handling 1,000 visitors

What a single request actually spends its time on:

```
one request, total 205 ms
├─ 2 ms    parse the incoming request        ← thinking
├─ 50 ms   ask the database                  ← WAITING
├─ 100 ms  call a payment API                ← WAITING
├─ 50 ms   fetch something from cache        ← WAITING
└─ 3 ms    build the HTML response           ← thinking

Thinking:   5 ms  (2%)
Waiting:  200 ms  (98%)
```

98% waiting. So:

```
NO CONCURRENCY                  WITH CONCURRENCY
one at a time                   overlap the waiting
─────────────────               ────────────────────
~5 requests/second              ~1000 requests/second
                                (same single machine)
```

This is *the* reason async web frameworks exist. The server is almost never busy — it is almost always waiting.

### Example B — Resizing 1,000 photos

```
per photo: 500 ms of pure pixel maths, 5 ms of disk reading
Thinking: 99%.  Waiting: 1%.
```

Overlapping the waiting saves you 1%. Pointless. What you need here is **more real workers**:

| Setup | Time |
|---|---|
| 1 worker | 500 sec |
| 4 workers on 4 cores | ~125 sec ✅ |
| 100 workers on 4 cores | ~500 sec + switching overhead ❌ (still only 4 hands) |

**More workers than you have cores does not help thinking work.** This surprises everyone the first time.

### Example C — Your phone's UI freezing

Every phone/desktop app has one special worker that draws the screen.

```
BAD — download runs on the UI worker
UI worker: [draw][draw][====== 3 sec download ======][draw]
                        ↑ screen is FROZEN, taps ignored,
                          user assumes the app crashed

GOOD — download handed to a background worker
UI worker: [draw][draw][draw][draw][draw][draw][show result]
background:      [====== 3 sec download ======]──┘
                        ↑ screen stays smooth
```

This is why mobile frameworks *forbid* network calls on the UI thread — they will throw an error rather than let you freeze the screen.

## 2.4 The pattern behind all of them

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   Almost every real use of concurrency is one of:       │
│                                                         │
│   1. "Don't let a slow thing block everything else."    │
│      → web server, phone UI, browser tab                │
│                                                         │
│   2. "Do the waiting for many things at the same time." │
│      → scraping, API calls, downloads                   │
│                                                         │
│   3. "Split heavy maths across real cores."             │
│      → image processing, data pipelines, ML             │
│                                                         │
│   4. "Keep independent things separate so one           │
│       crash doesn't kill the rest."                     │
│      → browser tabs, microservices                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

# PART 3 — Concurrency in Python

Now we attach the ideas to real code. Python has **one unusual rule** that changes everything, so we start there.

## 3.1 Python's one weird rule: the single microphone

In Python you can create many threads (Strategy B — many cooks in one kitchen). But Python adds a rule:

> **Only one thread may run Python code at any instant. There is one microphone in the room, and you must be holding it to speak.**

> 📖 **The fancy words:** the *GIL* — Global Interpreter Lock.

Why does this exist? Python keeps a little counter on every object saying how many things are using it. Letting many threads update those counters at once creates exactly the whiteboard problem from section 1.6 — for *every object in your program*. The microphone rule made Python simple, safe, and fast for single-threaded code. It has been there for 30 years.

**Now the part that actually matters, and that most people miss:**

> **A thread must hand over the microphone the moment it starts waiting.**

So:

```
THINKING WORK across 4 Python threads — NO GAIN
(everyone wants the mic; they just take turns)

thread 1: ▓▓▓░░░░░░░░░▓▓▓░░░░░░░░░
thread 2: ░░░▓▓▓░░░░░░░░░▓▓▓░░░░░░
thread 3: ░░░░░░▓▓▓░░░░░░░░░▓▓▓░░░
thread 4: ░░░░░░░░░▓▓▓░░░░░░░░░▓▓▓
          └── only ever ONE ▓ per column ──┘
          Same total time as 1 thread. Sometimes SLOWER
          (passing the mic around costs something).


WAITING WORK across 4 Python threads — HUGE GAIN
(waiting doesn't need the mic, so waits genuinely overlap)

thread 1: ▓[~~~~~~ waiting on network ~~~~~~]▓
thread 2:  ▓[~~~~~~ waiting on network ~~~~~~]▓
thread 3:   ▓[~~~~~~ waiting on network ~~~~~~]▓
thread 4:    ▓[~~~~~~ waiting on network ~~~~~~]▓
          └── all four waits happen at the same time ──┘
          4 tasks in the time of ~1. ✅
```

### The table that answers "which tool do I use?"

| Your work is... | `threading` | `multiprocessing` | `asyncio` |
|---|---|---|---|
| **Waiting** (API calls, DB, files, downloads) | ✅ Great | ⚠️ Works, but wasteful | ✅ **Best**, especially at large scale |
| **Thinking** (maths, loops, image processing) | ❌ **No benefit** — the microphone | ✅ **The answer** | ❌ No benefit |

Everything else in Part 3 is detail on this table.

---

## 3.2 The three toolkits

| | `threading` | `multiprocessing` | `asyncio` |
|---|---|---|---|
| Part 1 strategy | B: shared kitchen | A: separate kitchens | C: one tireless cook |
| Unit of work | thread | process | coroutine / task |
| Memory | shared | separate copies | shared (one thread) |
| Cost to create one | ~8 MB, ~ms | ~30–50 MB, ~100 ms | ~KB, microseconds |
| Who switches? | The OS, anytime | The OS | Your code, at `await` only |
| Race conditions? | ⚠️ Yes, easily | Rare (nothing shared) | Rare |
| Sensible number | 10s–100s | = CPU core count | 1,000s–10,000s |
| Sharing data | just use a variable (⚠️ needs locks) | must be packed & shipped (slow) | just use a variable |
| Code style | normal | normal + a required guard | needs `async`/`await` everywhere |

Check your core count with:

```python
import os
print(os.cpu_count())   # how many real "hands" you have
```

---

## 3.3 The easiest starting point: `concurrent.futures`

Ignore `threading` and `multiprocessing` at first. There is a simpler front door that covers 90% of real needs, and **swapping between waiting-mode and thinking-mode is a one-word change**.

### Baseline — the slow way

```python
import time

def fake_download(name):
    print(f"start {name}")
    time.sleep(2)                 # pretend: waiting for the network
    print(f"done  {name}")
    return f"{name}.zip"

files = ["a", "b", "c", "d", "e"]

start = time.time()
results = [fake_download(f) for f in files]
print(results)
print(f"took {time.time() - start:.1f} sec")     # ~10.0 sec
```

Five files × 2 seconds each = 10 seconds, spent almost entirely doing nothing.

### The fix — a pool of threads

```python
import time
from concurrent.futures import ThreadPoolExecutor

def fake_download(name):
    print(f"start {name}")
    time.sleep(2)
    print(f"done  {name}")
    return f"{name}.zip"

files = ["a", "b", "c", "d", "e"]

start = time.time()
with ThreadPoolExecutor(max_workers=5) as pool:
    results = list(pool.map(fake_download, files))
print(results)
print(f"took {time.time() - start:.1f} sec")     # ~2.0 sec  ✅
```

**10 seconds → 2 seconds.** Three lines changed.

Notice the printed output interleaves — all five say `start` before any says `done`. That is the overlap, visible.

> `pool.map` returns results **in the original order**, even though they finish in random order. Very convenient.
> The `with` block automatically waits for everything to finish before moving on.

### For thinking work — change one word

```python
import time
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def heavy_maths(n):
    total = 0
    for i in range(n):
        total += i * i
    return total

if __name__ == "__main__":                      # ← REQUIRED. See 3.7.
    jobs = [8_000_000] * 4

    start = time.time()
    [heavy_maths(n) for n in jobs]
    print(f"sequential: {time.time() - start:.1f} sec")

    start = time.time()
    with ThreadPoolExecutor(max_workers=4) as pool:
        list(pool.map(heavy_maths, jobs))
    print(f"threads:    {time.time() - start:.1f} sec")   # ← no better!

    start = time.time()
    with ProcessPoolExecutor(max_workers=4) as pool:
        list(pool.map(heavy_maths, jobs))
    print(f"processes:  {time.time() - start:.1f} sec")   # ← actually faster
```

Typical output on a 4-core machine:

| Version | Time | Why |
|---|---|---|
| sequential | 4.0 sec | one worker, one job at a time |
| **Thread**PoolExecutor | ~4.1 sec | one microphone → no gain, tiny loss |
| **Process**PoolExecutor | ~1.2 sec | 4 separate kitchens, 4 microphones ✅ |

**`Thread` vs `Process` is literally the only word you change.** Run this yourself once — seeing threads fail on maths is the moment the GIL stops being abstract.

### When you need results as they arrive

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=5) as pool:
    futures = {pool.submit(fake_download, f): f for f in files}
    for future in as_completed(futures):          # yields in FINISH order
        name = futures[future]
        try:
            print(name, "→", future.result())     # exceptions surface here
        except Exception as e:
            print(name, "failed:", e)
```

> ⚠️ **Important:** if a job raises an exception, you will not see it until you call `.result()`. Silent-failure bugs live here. Always call `.result()` inside a try/except.

---

## 3.4 `asyncio` — the one tireless cook

Async is Strategy C. It is the best fit for *lots* of waiting, and it is what modern Python web frameworks are built on.

### The mental model

A waiter in a restaurant. One waiter, twenty tables.

A bad waiter takes table 1's order, walks to the kitchen, and **stands there** until the food is ready. Tables 2–20 wait.

A good waiter takes table 1's order, hands it to the kitchen, and immediately moves to table 2. When food is ready, they deliver it.

**`await` means: "I'm about to wait here — go serve someone else, come back when this is ready."**

### The three words you need

| Word | Meaning |
|---|---|
| `async def` | This function is allowed to pause. Calling it does **not** run it — it hands you a "job to be done". |
| `await` | Pause here, let others run, resume when ready. Only legal inside `async def`. |
| `asyncio.run(...)` | Start the tireless cook and give it the first job. Called once, from normal code. |

### The same download, in async

```python
import asyncio, time

async def fake_download(name):
    print(f"start {name}")
    await asyncio.sleep(2)        # ← the async version of waiting
    print(f"done  {name}")
    return f"{name}.zip"

async def main():
    files = ["a", "b", "c", "d", "e"]
    start = time.time()
    results = await asyncio.gather(*(fake_download(f) for f in files))
    print(results)
    print(f"took {time.time() - start:.1f} sec")     # ~2.0 sec

asyncio.run(main())
```

`asyncio.gather` = "start all of these, and wake me when they're all finished." Results come back in the order you passed them in.

### The trap that catches every single beginner

```python
async def broken():
    time.sleep(2)          # ❌ NO await — this FREEZES the entire program
                           #    Every other task stops for 2 full seconds.

async def correct():
    await asyncio.sleep(2) # ✅ this yields control to everyone else
```

**Any normal blocking call inside async code freezes the whole event loop.** Your one cook decided to stand and stare. Everything stops.

| Blocking (freezes async) | Async-safe replacement |
|---|---|
| `time.sleep(2)` | `await asyncio.sleep(2)` |
| `requests.get(url)` | `await client.get(url)` with `httpx` / `aiohttp` |
| `open(f).read()` | `aiofiles`, or `await asyncio.to_thread(...)` |
| a heavy `for` loop | `await asyncio.to_thread(...)` or a process pool |

Escape hatch when you must call blocking code from async:

```python
result = await asyncio.to_thread(some_blocking_function, arg1, arg2)
```

This shoves the blocking work onto a background thread so the loop keeps spinning.

### A realistic example — many API calls, with a safety limit

Firing 500 requests at once will get you rate-limited or banned. A **semaphore** is a bouncer: "only N inside at a time."

```python
import asyncio, httpx

async def fetch_one(client, url, gate):
    async with gate:                       # wait for a free slot
        try:
            r = await client.get(url, timeout=30)
            return url, r.status_code
        except Exception as e:
            return url, f"failed: {e}"

async def fetch_all(urls, limit=10):
    gate = asyncio.Semaphore(limit)        # at most 10 in flight
    async with httpx.AsyncClient() as client:
        return await asyncio.gather(
            *(fetch_one(client, u, gate) for u in urls)
        )

urls = [f"https://example.com/page/{i}" for i in range(500)]
results = asyncio.run(fetch_all(urls, limit=10))
```

500 requests, 10 at a time, one thread, one CPU core. This exact shape covers web scraping, calling many APIs, batch LLM requests, and bulk database reads.

> ⚠️ **The catch with async:** it is contagious. To `await` something, you must be inside an `async def`, whose caller must also be async, all the way up. You cannot sprinkle async into one function of a normal codebase. This is why threads are often the pragmatic choice for a script, and async the right choice for a whole application built that way from the start.

---

## 3.5 Threads and race conditions in real Python

Here is the whiteboard problem from section 1.6 as real, running code — a bank account with 1000 in it, and 20 threads each trying to withdraw 100.

```python
import threading, time

balance = 1000

def withdraw(amount):
    global balance
    if balance >= amount:      # CHECK  — "is there enough?"
        time.sleep(0.001)      #         (any wait hands over the microphone)
        balance -= amount      # ACT    — "take the money"

threads = [threading.Thread(target=withdraw, args=(100,)) for _ in range(20)]
for t in threads: t.start()
for t in threads: t.join()     # join = "wait for this thread to finish"

print(balance)
```

There is only 1000 in the account, so at most 10 withdrawals should succeed and the balance should never go below 0.

Actual output, run three times:

```
balance = -1000
balance = -1000
balance =  -900
```

**Twice as much money came out as ever existed.** No error. No crash. Just wrong.

Why: all 20 threads checked `balance >= 100` while the balance was still 1000. Every one of them saw "yes, plenty". Then they all took money. The check and the act were two separate steps, and the world changed in between.

> 📖 **The fancy name for this shape of bug:** *check-then-act*. It is behind double-booked airline seats, duplicate orders, and inventory going negative. It is probably the most common concurrency bug in real production systems.

### The fix

```python
import threading, time

balance = 1000
lock = threading.Lock()

def withdraw(amount):
    global balance
    with lock:                 # ← take the marker; auto-released on exit
        if balance >= amount:
            time.sleep(0.001)
            balance -= amount

threads = [threading.Thread(target=withdraw, args=(100,)) for _ in range(20)]
for t in threads: t.start()
for t in threads: t.join()

print(balance)     # 0, every single time ✅
```

The lock must wrap **both** the check and the act. Locking only the subtraction would not help at all — the bug lives in the gap *between* the two.

Always use `with lock:` rather than manual `acquire()` / `release()`. If an exception fires mid-way, the `with` form still releases the lock; manual code can leave it held forever, freezing every other thread.

### ⚠️ An honest warning about the demo you'll see everywhere else

Nearly every tutorial demonstrates race conditions with this instead:

```python
counter = 0
def add():
    global counter
    for _ in range(1_000_000):
        counter += 1          # "read, add, write — this will lose updates!"
```

The *explanation* is correct. But on Python 3.10 and newer this code will usually print the **right** answer, because of an internal optimisation that makes the interpreter unlikely to switch threads in the middle of that particular sequence.

This is genuinely dangerous to learn from, so be clear about the lesson:

> **Code being unsafe is not the same as code failing.** A race condition is a bug that shows up *sometimes* — under load, on a different machine, on a different Python version, on a bad day in production. "I ran it and it worked" proves nothing whatsoever.

Reason about whether shared data is protected. Never conclude it is fine because a test passed.

### Better than locks: don't share at all

The safest concurrent code shares nothing. Use a queue — a conveyor belt that is safe by construction:

```python
import threading, queue

work = queue.Queue()
results = queue.Queue()

def worker():
    while True:
        item = work.get()
        if item is None:            # the "stop" signal
            work.task_done()
            break
        results.put(item * item)
        work.task_done()

threads = [threading.Thread(target=worker, daemon=True) for _ in range(4)]
for t in threads: t.start()

for i in range(20): work.put(i)
for _ in threads: work.put(None)     # one stop signal per worker
work.join()                          # wait until everything is processed

print(sorted(results.queue))
```

`queue.Queue` handles all the locking internally, so you never write a lock yourself.

**Rule of thumb: prefer passing messages over sharing variables.** Ninety percent of threading bugs simply cease to exist.

## 3.6 Where Python quietly does this for you

A lot of the time you should not write any of the above:

| What you're doing | The concurrency is already handled |
|---|---|
| NumPy / Pandas / Polars maths | Heavy operations drop the microphone and use multiple cores internally |
| scikit-learn | `n_jobs=-1` runs across all cores |
| PyTorch / TensorFlow | GPU work and `DataLoader(num_workers=4)` |
| FastAPI / Django async views | The framework runs the event loop |
| Serving a web app | `gunicorn -w 4` runs 4 separate processes for you |
| `subprocess` | The OS runs it as a separate process |

**Before writing a process pool for numeric work, check whether a vectorised NumPy/Polars operation does it faster with no concurrency code at all.** It very often does.

---

## 3.7 The gotchas that will bite you

| # | Gotcha | Symptom | Fix |
|---|---|---|---|
| 1 | Missing `if __name__ == "__main__":` with processes | Endless process spawning, or a confusing crash | Always wrap process-pool code in that guard |
| 2 | `time.sleep` / `requests` inside `async def` | Async is somehow no faster | Use `await asyncio.sleep` / `httpx` / `asyncio.to_thread` |
| 3 | Threads for heavy maths | No speed-up, sometimes slower | Use `ProcessPoolExecutor` |
| 4 | Spawning 500 threads | Memory blows up, everything slows | Use a pool with fixed `max_workers` |
| 5 | Forgetting `.join()` or `await` | Program exits, work silently lost | Use `with pool:`, `.join()`, or `asyncio.gather` |
| 6 | Never calling `.result()` | Exceptions vanish silently | Always retrieve results in try/except |
| 7 | Sharing a plain variable across threads | Wrong numbers, no error | `Lock`, or a `Queue` |
| 8 | Passing huge data to processes | Slower than sequential | Data gets copied — send small inputs, or use threads |
| 9 | Assuming completion order | Output looks scrambled | Use `pool.map` (ordered) or sort afterwards |
| 10 | Unclosed sessions/clients | Warnings, leaked connections | `async with httpx.AsyncClient() as c:` |

Gotcha #1 in code:

```python
# ❌ crashes or spawns forever on Windows/macOS
with ProcessPoolExecutor() as pool: ...

# ✅
if __name__ == "__main__":
    with ProcessPoolExecutor() as pool: ...
```

Why: to make a new process, Python re-imports your file. Without the guard, the re-import runs the pool-creation code again, which makes more processes, which re-import the file...

---

## 3.8 Is the GIL going away?

Yes — slowly, and carefully.

- **Python 3.13** shipped an experimental build with no GIL.
- **Python 3.14** (October 2025) promoted it to **officially supported, but still optional** — you install a separate build, usually written `3.14t`. The normal build still has the GIL.
- The eventual plan is for it to become the default, but that is still some years out.

What this means practically **today**:

| | Standard build (the default) | Free-threaded build (`3.14t`) |
|---|---|---|
| Threads for maths | No speed-up | Real multi-core speed-up (roughly 2–4× on 4 cores) |
| Threads for waiting | Already good | No real change — waits already overlapped fine |
| Single-threaded speed | Baseline | Slightly slower (~5–10% on 3.14) |
| Library support | Everything works | Any C library not yet updated **silently switches the GIL back on** |

Two things to take away:

1. **Everything in this document still applies.** The default Python you install today has the GIL, and the free-threaded build does not remove race conditions — it makes them *more* likely, because threads genuinely run at the same instant.
2. Do not restructure anything for this yet. Keep processes for maths, threads/async for waiting.

---

## 3.9 The Python decision flowchart

```
                    What is my code mostly doing?
                                │
        ┌───────────────────────┴───────────────────────┐
        │                                               │
   WAITING for things                            THINKING hard
   (API, DB, files, network)                     (maths, loops, crunching)
        │                                               │
        │                                               ▼
        │                                    Is it array/dataframe maths?
        │                                        │              │
        │                                       YES             NO
        │                                        │              │
        │                                        ▼              ▼
        │                              NumPy / Polars     ProcessPoolExecutor
        │                              vectorised ops     (max_workers = cores)
        │                              (usually fastest,
        │                               zero concurrency
        │                               code needed)
        ▼
  How many things at once, and what does the codebase look like?
        │
        ├─ Up to ~100, normal (non-async) code, a script
        │      ▼
        │   ThreadPoolExecutor        ← start here. Simplest thing that works.
        │
        ├─ Hundreds or thousands
        │      ▼
        │   asyncio + gather + Semaphore
        │
        └─ Already using an async framework (FastAPI etc.)
               ▼
            asyncio, and use async libraries throughout
```

**If unsure: start with `ThreadPoolExecutor`.** It is the least code, works with libraries you already use, and is the right answer for most waiting-heavy scripts.

---

# PART 4 — Cheat Sheet, Experiments, Glossary

## 4.1 The one-page summary

```
┌─────────────────────────────────────────────────────────────────┐
│  STEP 1 — Ask: is my program WAITING or THINKING?               │
│           (time it. Don't guess. Everyone guesses wrong.)       │
├─────────────────────────────────────────────────────────────────┤
│  WAITING (network, disk, DB, APIs)                              │
│     → simple script, up to ~100 tasks : ThreadPoolExecutor      │
│     → hundreds/thousands, or async app: asyncio + gather        │
│     → always cap it: max_workers=N  or  asyncio.Semaphore(N)    │
├─────────────────────────────────────────────────────────────────┤
│  THINKING (maths, loops, image/text crunching)                  │
│     → arrays/dataframes : NumPy / Polars (try this FIRST)       │
│     → anything else     : ProcessPoolExecutor(max_workers=cores)│
│     → threads will NOT help. The GIL.                           │
├─────────────────────────────────────────────────────────────────┤
│  ALWAYS                                                         │
│     → measure before and after. Concurrency often makes         │
│       things slower.                                            │
│     → if two workers touch the same variable: Lock or Queue     │
│     → `if __name__ == "__main__":` around any process pool      │
│     → never call .sleep()/requests/blocking code inside async   │
└─────────────────────────────────────────────────────────────────┘
```

## 4.2 Copy-paste starters

```python
# ── Waiting work, simple ─────────────────────────────────────────
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=10) as pool:
    results = list(pool.map(my_function, my_list))     # ordered results

# ── Waiting work, need errors handled per item ───────────────────
from concurrent.futures import ThreadPoolExecutor, as_completed
with ThreadPoolExecutor(max_workers=10) as pool:
    futures = {pool.submit(my_function, x): x for x in my_list}
    for f in as_completed(futures):
        try:    print(futures[f], f.result())
        except Exception as e: print(futures[f], "failed:", e)

# ── Thinking work ────────────────────────────────────────────────
from concurrent.futures import ProcessPoolExecutor
if __name__ == "__main__":
    with ProcessPoolExecutor() as pool:                # defaults to core count
        results = list(pool.map(my_function, my_list))

# ── Waiting work at scale ────────────────────────────────────────
import asyncio
async def run_all(items, limit=10):
    gate = asyncio.Semaphore(limit)
    async def one(x):
        async with gate:
            return await my_async_function(x)
    return await asyncio.gather(*(one(x) for x in items),
                                return_exceptions=True)   # don't let one
                                                          # failure kill all
results = asyncio.run(run_all(my_list))

# ── Blocking code that must run inside async ─────────────────────
result = await asyncio.to_thread(blocking_function, arg)

# ── Protecting a shared variable across threads ──────────────────
import threading
lock = threading.Lock()
with lock:
    shared_thing = shared_thing + 1
```

## 4.3 Five experiments — run these, don't just read

The fastest way to make this real is to watch it happen. Each takes under a minute.

| # | Experiment | What you should see | What it proves |
|---|---|---|---|
| 1 | The `fake_download` script, sequential vs `ThreadPoolExecutor` | 10.0 sec → 2.0 sec | Waiting overlaps beautifully |
| 2 | The same script's printed output | All five print `start` before any prints `done` | The overlap made visible |
| 3 | `heavy_maths` with Thread vs Process pool | Threads: no change. Processes: faster (if you have >1 core) | The GIL, felt rather than described |
| 4 | The bank account, with and without the lock | Balance goes negative without the lock | Race conditions are real and silent |
| 5 | Replace `await asyncio.sleep(2)` with `time.sleep(2)` in the async example | 2 sec → 10 sec | One blocking call freezes the whole event loop |

> All numbers in this document came from actually running the code. Experiment 3 was run on a **single-core** machine, where the process pool showed **no** speed-up at all — a perfect demonstration of section 2.3: you cannot get more parallelism than you have cores, no matter what code you write.

## 4.4 Glossary — plain word → the word you'll meet in docs

| What this document called it | What everyone else calls it |
|---|---|
| Waiting work | I/O-bound |
| Thinking work | CPU-bound |
| Separate kitchens | Processes / `multiprocessing` |
| Many cooks, one kitchen | Threads / `threading` |
| One tireless cook | Async / event loop / `asyncio` |
| One at a time, start to finish | Sequential / synchronous / blocking |
| Overlapping the waiting | Concurrency |
| Truly at the same instant | Parallelism |
| The single microphone | The GIL (Global Interpreter Lock) |
| A job that can pause and resume | Coroutine |
| The pause point | `await` / yield point |
| Putting down one job to pick up another | Context switch |
| The marker | Lock / mutex |
| The part only one worker may enter | Critical section |
| Cannot be interrupted halfway | Atomic |
| Lost update from bad timing | Race condition |
| Everyone frozen, each holding what the other needs | Deadlock |
| One worker never gets a turn | Starvation |
| The bouncer capping how many at once | Semaphore |
| A team of ready workers | Thread pool / process pool |
| A receipt for a result that isn't ready yet | Future / promise |
| The conveyor belt | Queue |
| The manager with the stopwatch | Scheduler |
| Manager forcibly switches workers | Preemptive multitasking |
| Workers volunteer to switch | Cooperative multitasking |
| Sequential parts limit your ceiling | Amdahl's law |

## 4.5 The three sentences to remember

1. **Concurrency overlaps waiting; parallelism needs more cores.** Different problems, different fixes.
2. **In Python: threads and async for waiting, processes for thinking.** The GIL is the reason.
3. **Every concurrency bug comes from two workers touching the same thing.** Lock it, or better, don't share it.

---

*Everything here was written to be read top to bottom once, then used as a reference. If you only reread one thing, make it section 1.3 — the two kinds of work. Every other decision follows from it.*
