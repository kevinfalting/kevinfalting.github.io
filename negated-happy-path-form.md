# Negated Happy Path Form

I've been leaving a lot of code review comments asking for a boolean expression to be written in happy-path-form, then negated. Let's call it the _negated happy path form_. There's a few ways to do this.

Consider the following expression:

```go
if !a || !b || !c {
  return
}
```

versus

```go
if !(a && b && c) {
  return
}
```

versus

```go
happyPath := a && b && c
if !happyPath {
  return
}
```

The second two are significantly simpler to understand and edit. They define the "happy path" then just negate it.

Of course there's always the tried and true:

```go
if !a {
  return
}

if !b {
  return
}

if !c {
  return
}
```

Depending on the complexity of the expression, I may go between the second and third forms, sometimes combining them.

```go
isThis := a && b
isThat := c && d
if !(isThis || isThat) {}
```

Stop here, I believe I've made my point. It's a simple change and we're just getting progressively more silly from this point on.

---

There's still a scoping problem where more variables have been added to the local scope. Maybe we can use an arbitrary smaller scope within the larger scope.

```go
// this is fine if it can make it past the _"what empty braces? gross!"_ in code review.
func Do() {
  {
    isThis := a && b
    isThat := c && d
    if !(isThis || isThat) {}
  }
}
```

That's kinda ugly though.

What's more ugly, you ask?

```go
if isThis, isThat := a && b, c && d; !(isThis || isThat) {}
```

At least it's scoped properly...

What about:

```go
thisOrThat := func() (bool, bool) {
  isThis := a && b
  isThat := c && d
  return isThis, isThat
}

if isThis, isThat := thisOrThat(); !(isThis || isThat) {}
```

I said we were getting silly, right? Let's "clean" it up though.

```go
// this is actually very okay with me
isThisOrThat := func() bool {
  isThis := a && b
  isThat := c && d
  return isThis || isThat
}()

if !isThisOrThat {}
```

That feels better, but doesn't solve all of the problems.

What about:

```go
if func() bool {
    isThis := a && b
    isThat := c && d
    return isThis || isThat
  }() {}
```

**um, no. probably don't.**

Consider scope, but more importantly, consider readability.
