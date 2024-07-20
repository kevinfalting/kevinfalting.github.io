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
if !(isThis || isThat) {
  return
}
```

The point is to build the happy path expression and negate it. Think _"what condition(s) need to be true in order for execution to proceed?"_, then wrap that whole thing in parenthesis and negate it.

Also don't overlook the power of the "tried and true" above. Be clever by being simple to understand and change.
