# Negated Happy Path Form

Using the _negated happy path form_ helps to write simpler to understand and change boolean expressions.

Consider the following expression:

```go
if !a || !b || !c {
  return
}
```

compared to:

```go
if !(a && b && c) {
  return
}
```

or:

```go
happyPath := a && b && c
if !happyPath {
  return
}
```

The second two are significantly simpler to understand and edit. They define the ["happy path"](https://en.wikipedia.org/wiki/Happy_path) then just negate it.

Depending on the complexity of the expression, switching between the second and third forms can be useful, even sometimes combining them.

```go
isThis := a && b
isThat := c && d
if !(isThis || isThat) {
  return
}
```

Of course there's always ol' reliable:

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

The point is to build the happy path expression and negate it. Think _"what condition(s) need to be true in order for execution to proceed?"_, then wrap that whole thing in parenthesis and negate it.

Not using the _negated happy path form_ does not help writing difficult to misunderstand and change boolean expressions...
