# Fail Safely

In Go, a common method to represent related values is by using a set of constants.

```go
const (
  Active  = "A"
  Deleted = "D"
  Pending = "P"
)
```

A feature request then comes in that says a deleted user should not be able to perform some action. It's easy to take that at face value and write the following conditional which does exactly what was requested: deny deleted users.

```go
if user.State == Deleted {
  return someErr
}
```

However, this is the kind of conditional which is doomed to fail open. What happens when a new state is added? Yep, it's _implicitly_ allowed. Congratulations, the new `Purged` state is passes where `Deleted` is blocked. There's nothing about the way it's written that considers all other potential values, including garbage!

We can make it better by thinking a little harder about the context of the action and recognizing that only active users should be able to perform it. Think about the happy path, then negate it. See also the [Negated Happy Path Form](./negated-happy-path-form.md). This now only allows active users and denies the rest.

```go
if user.State != Active {
  return someErr
}
```

While this is _better_, we're still _implicitly_ denying all other states. The best thing to do would be to exhaustively handle every related state, _explicitly_ allowing or denying each one, and by doing so, protecting against that future new state with a default deny.

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

I don't know of a way to easily check for exhaustiveness at compile time in Go, but there are options like writing exhaustive tests or using a linter[^1]. The point of being exhaustive is to deliberately consider all possible values and what to do with them.

With anything that fails open, users will be performing actions that they shouldn't, possibly putting them in a wierd or unrecoverable state. The resulting bug that's reported for this is significantly more difficult to debug. Instead, if the conditional fails closed, the user is prevented from performing that action, a mild but temporary annoyance, and the bug report can link to an error that describes exactly where the error occurred.

The last form is to explicitly fail-open, which is more common than you might expect. This approach is useful when it is safe to continue even if an unrecognized value is encountered. It is a good strategy when introducing new features, where failing closed could cause more problems than allowing the system to continue running.

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

Here's a more realistic example of explicitly failing open. Say I have a running process listening for known signals and should ignore anything unrecognized.

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

And that's really it. Avoid implictly failing open, and beyond that choose the right tool for the job. When I'm dealing with a set of related values, I typically choose a switch block. It can result in more lines of code, but I believe the trade off for explicitness and debuggability are worth it.

I learned this after working on a large system that initially assumed there would only be two user types. Then, along came a third and fourth. Unfortunately, introducing these new user types required the exercise of combing through all the execution paths and closing them off - who knows what paths we've missed that are still implicitly failing open! Ideally, introducing a new user type should involve identifying all critical execution paths and opening them up.

Fail safely.

[^1]: I've not used any linters to check for exhaustiveness, but there are some that claim they can. ymmv.
