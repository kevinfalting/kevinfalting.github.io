# Fail Safely

Go doesn't have enums. (at least at the time of this writing and I have no idea if there are plans of adding them.)

I've typically seen related values expressed as a set of constants.

```go
const (
	Active  = "A"
	Deleted = "D"
	Pending = "P"
)
```

or using iota. (no, stop it)

```go
const (
  Active = iota + 1
  Deleted
  Pending
)
```

However, in code, I may find something like:

```go
if user.State == Deleted {
  return someErr
}
```

which is doomed to fail open. What happens when a new state is added? Yep, it's _implicitly_ allowed. Congratulations, the new `Purged` state is now allowed where `Deleted` was not.

We can make it a little better by describing what we want to continue, and negating it. See also the [Negated Happy Path Form](./negated-happy-path-form.md).

```go
if user.State != Active {
  return someErr
}
```

While this is _better_, we're still _implicitly_ denying any other type, but we're talking about a known set of values here. The best thing to do would be to exhaustively handle every related state, _explicitly_ allowing or denying each one, and by doing so, protecting against that future new state with a default deny.

```go
switch user.State {
  case Active, Pending:
    // okay, continue...

  case Deleted:
    return someErr

  default:
    return errNotHandled
}
```

Go doesn't have enums, so I don't know of a way to easily check for this at compile time, but the next best thing is at runtime. At least something like this will help prevent a user getting into a weird state. Debugging that is way harder than debugging the `errNotHandled` default deny.

An additional benefit is that the code will very readily point out all places that need to be considered when adding a new state, either by finding all of the locations via an editor, or at runtime when it throws errors.

Do this for all related would-be enums: states, types, kinds, etc.

There's one more example, and that is the explicit fail-open form, and it's not as uncommon as you might think. It's particularly useful in situations where it's still safe to proceed even if the value is unrecognized. This can be a good strategy when introducing something new and is riskier to fail closed unnecessarily than allow execution to continue.

```go
switch user.State {
  case Active, Pending:
    // okay, continue...

  case Deleted:
    return someErr

  default:
    log.Println("unknown user state %q, continuing")
}
```

That example is a little more difficult to see it's usefulness but I wanted to include it for the ability to compare the examples that all use the same set of values. Here's a more realistic example of explicitly failing open. Say I have a process running and it's listening for signals. If we recieve an unrecognized signal, it should just be ignored.

```go
switch cmd {
case Start:
  // start the process

case Stop:
  // stop the process

case Restart:
  // restart the process

default:
  fmt.Printf("Unknown command: %s\n", cmd)
  // continue...
}
```

And that's really it. Avoid implictly failing open, and beyond that choose the right tool for the job. When I'm dealing with a set of related values, I typically choose a switch block. It results in more lines of code, but I believe the trade off for explicitness and debuggability are worth it.

I came to this after working in a large system that for years made the safe-at-the-time assumption that there would only ever be two user types. Then, along came a third and fourth. Unfortunately, introducing these new user types required the exercise of combing through all the execution paths and closing peices off - who knows what paths we've missed that are still implicitly failing open! Ideally, introducing a new user type would mean finding all of the critical execution paths and opening them up.
