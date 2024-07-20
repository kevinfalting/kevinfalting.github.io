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

A ticket then comes in that says a deleted user should not be able to perform some action. It's simple to take that at face value and write the following conditional which does exactly what the ticket says: deny deleted users.

```go
if user.State == Deleted {
  return someErr
}
```

However, this is the kind of conditional which is doomed to fail open. What happens when a new state is added? Yep, it's _implicitly_ allowed. Congratulations, the new `Purged` state is now allowed where `Deleted` was not. There's nothing about the way it's written that considers all other potential values, including garbage!

We can make it a little better by thinking a little harder about the context of the action and recognizing that only active users should be able to perform it. Think about the happy path, then negate it. See also the [Negated Happy Path Form](./negated-happy-path-form.md). This now only allows active users and denies the rest.

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

I don't know of a way to easily check for exhaustiveness at compile time in Go, but there are options like writing exhaustive tests or using a linter[^1]. The point of being exhaustive is to deliberately consider the possible values and what to do with them.

With anything that fails open, users will be performing actions that they shouldn't, possibly putting them in a wierd or unrecoverable state. The resulting bug that's reported for this is significantly more difficult to debug. Instead, if the conditional fails closed, the user is prevented from performing that action, a mild but temporary annoyance, and the bug report can link to an error that describes exactly where the error occurred.

Do this for all related would-be enums: states, types, kinds, etc.

There's one more example, and that is the explicit fail-open form, and it's not as uncommon as you might think. It's particularly useful in situations where it's still safe to proceed even if the value is unrecognized. This can be a good strategy when introducing something new that is riskier to fail closed unnecessarily than allow execution to continue.

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

Here's a more realistic example of explicitly failing open. Say I have a process running and it's listening for signals. If it recieves an unrecognized signal, it should just be ignored.

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

I came to this after working in a large system that for years made the safe-at-the-time assumption that there would only ever be two user types. Then, along came a third and fourth. Unfortunately, introducing these new user types required the exercise of combing through all the execution paths and closing them off - who knows what paths we've missed that are still implicitly failing open! Ideally, introducing a new user type would mean finding all of the critical execution paths and opening them up.

Fail safely.

[^1]: I've not used any linters to check for exhaustiveness, but there are some that claim they can. ymmv.
