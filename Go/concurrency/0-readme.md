# Go Concurrency Thinking Framework

## The Two Questions

Before writing any concurrent code, ask yourself just two questions:

### Question 1: Who needs to wait for whom?

Draw an arrow. That arrow becomes your channel or WaitGroup.

```
Producer has data → Consumer needs it       =  channel (data flows)
Consumer says "go" → Producer needs it      =  channel (signal flows back)
Main waits for workers to finish             =  WaitGroup
```

### Question 2: What's the relationship?

- **Channel = conversation.** Two goroutines need to talk to each other. One has something, the other wants it.
- **WaitGroup = headcount.** You don't care *what* the goroutines say. You just want to know: "has everyone finished?"

---

## The Recipe

Whenever you're designing concurrent code, go through these steps:

```
1. Who PRODUCES data?           → they need a channel to SEND on
2. Who CONSUMES data?           → they READ from that channel
3. Who needs to WAIT for whom?  → if waiting for data: channel
                                  if waiting for completion: WaitGroup
4. Who needs PERMISSION to continue? → they READ from a signal channel
                                       someone else WRITES to that channel
```

---

## Picking the Right Tool

Don't think about channels and WaitGroups first. Think about the **arrows** — who waits, who signals, who sends data. Then pick the tool:

| Arrow means...                  | Tool         |
| ------------------------------- | ------------ |
| Carries **data**                | channel      |
| Carries **"go ahead"** signal   | channel      |
| Means **"are you all done?"**   | WaitGroup    |

---

## Example: Restoring Sequencing (Pattern 4)

**Step 1 — Identify the arrows:**

```
"Joe produces messages"          → Joe sends on ch
"Main consumes messages"         → Main reads from ch
"Joe should wait for permission" → Joe reads from waitFor (BLOCKS)
"Main gives permission"          → Main writes to waitFor (UNBLOCKS Joe)
```

**Step 2 — Pick the tools:**

- Joe → Main (data): `ch <- Message{...}` → `msg := <-ch`
- Main → Joe (permission signal): `msg.wait <- true` → `<-waitFor`

**Step 3 — Code follows naturally:**

```go
// Producer: sends data, then waits for permission
go func() {
    for {
        ch <- Message{name, waitFor}  // arrow 1: data flows out
        <-waitFor                      // arrow 2: blocks until "go ahead"
    }
}()

// Consumer: reads data, then gives permission
m1 := <-ch           // arrow 1: data received
fmt.Println(m1.name)
m1.wait <- true       // arrow 2: sends "go ahead"
```

---

## Key Rules

1. **Every channel exists because of an arrow.** If you can't explain "who sends, who receives, why", the channel shouldn't exist.
2. **No channel is random.** Each one answers: "who is waiting for whom, and for what?"
3. **WaitGroup is not a channel replacement.** Use it only when you need "wait for N things to complete" and you don't care about data from them.
4. **Close flows downstream.** Producer closes → consumer detects via `range` → next stage closes → and so on.
5. **Blocked ≠ Finished.** A goroutine stuck on a channel read is still alive, consuming resources. Always give goroutines a way to exit.

---

## Quick Reference: When to Use What

| Scenario                                    | Tool                          |
| ------------------------------------------- | ----------------------------- |
| Send data from A to B                       | `ch <- data` / `<-ch`        |
| Merge N inputs into one stream              | Fan-in (N goroutines + 1 ch) |
| Wait for N goroutines to finish             | `sync.WaitGroup`             |
| Tell a goroutine to stop                    | Quit channel or `context`    |
| Control pacing / turn-taking                | Signal channel inside message |
| Timeout on waiting                          | `select` + `time.After`      |
| React to whichever channel is ready first   | `select`                      |