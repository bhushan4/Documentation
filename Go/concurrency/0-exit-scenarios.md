# Go: return, continue, break — Where They Go

## Quick Reference

| Keyword    | What it does                                  |
| ---------- | --------------------------------------------- |
| `return`   | Exits the **entire function/goroutine**       |
| `continue` | Skips to the **next iteration** of the loop   |
| `break`    | Exits the **nearest** for/switch/select block |

---

## In a `for` loop (simple)

```go
for i := 0; i < 5; i++ {
    if i == 2 { continue }  // skips 2, loop continues → prints 0,1,3,4
    if i == 4 { break }     // exits loop entirely → prints 0,1,3
    if i == 3 { return }    // exits the FUNCTION → nothing runs after this
    fmt.Println(i)
}
```

---

## In `select` inside `for` — THE GOTCHA

```go
for {
    select {
    case msg, ok := <-ch:
        if !ok {
            break     // ← breaks SELECT, NOT the for loop!
                      //   for loop runs again → select runs again
        }
        continue      // ← skips rest of current FOR iteration
                      //   goes back to top of for loop
        return        // ← exits the goroutine/function entirely
    }
    // break from select lands HERE, still inside the for loop
    // continue SKIPS this part
}
```

### Summary for `select` inside `for`:

| Keyword    | Exits what?   | Lands where?                       |
| ---------- | ------------- | ---------------------------------- |
| `break`    | The `select`  | Code after select, still in loop   |
| `continue` | Current iteration | Top of `for` loop (skips code after select) |
| `return`   | The function  | Gone                               |

---

## The `continue` Trap

```go
for {
    select {
    case msg, ok := <-ch1:
        if !ok {
            ch1 = nil
            continue     // ← SKIPS the nil check below!
        }
        c <- msg
    }

    // continue jumps over this — you never check!
    if ch1 == nil && ch2 == nil {
        return
    }
}
```

**Fix option 1:** Use if/else instead of continue:

```go
case msg, ok := <-ch1:
    if !ok {
        ch1 = nil
    } else {
        c <- msg
    }
// Now nil check always runs
```

**Fix option 2:** Check inside the case before continuing:

```go
case msg, ok := <-ch1:
    if !ok {
        ch1 = nil
        if ch2 == nil { return }
        continue
    }
    c <- msg
```

---

## Labeled break — Breaking the outer loop from inside select

```go
loop:                          // ← label
    for {
        select {
        case msg, ok := <-ch:
            if !ok {
                break loop     // ← breaks the FOR loop, not select
            }
        }
    }
// break loop lands HERE
```

Without the label, `break` only exits the select. With `break loop`, it exits the labeled `for` loop.

---

## Cheat Sheet

| Context                        | `break`          | `continue`               | `return`          |
| ------------------------------ | ---------------- | ------------------------ | ----------------- |
| `for` loop                     | exits loop ✓     | next iteration ✓         | exits function ✓  |
| `select` (no loop)             | exits select     | compile error            | exits function ✓  |
| `select` inside `for`          | exits **select** ⚠️ | next **loop** iteration ⚠️ | exits function ✓  |
| `switch` (no loop)             | exits switch     | compile error            | exits function ✓  |
| `switch` inside `for`          | exits **switch** ⚠️ | next **loop** iteration  | exits function ✓  |

⚠️ = common source of bugs. Use labeled break or return when you need to exit the outer loop.