# Bugs rulebook

Your lens is correctness: does this change break? Each entry reads _what it is_ → _how to attack
it_. Ground every finding in a concrete input or state that produces the wrong outcome.

- **Unhandled failure path** — a throw, rejection, or non-ok response on a path the change
  introduces, with no handler. → follow each new call to its error case and ask what the caller sees
  when it fires.
- **Boundary and guard errors** — off-by-one, an inverted condition, `&&`/`||` swapped, a truthiness
  check that treats `0`, `''`, or `false` as absent. → feed the edge values the guard was written
  for.
- **Half-applied state** — a multi-step write where a failure between steps leaves the system
  inconsistent. → pick the worst step to fail at and describe the state left behind.
- **Ordering and concurrency** — parallel work touching one resource, a missing `await`, an assumed
  iteration order, a retry that is not idempotent. → run two of it at once, in the worse order.
- **Contract drift** — caller and callee now disagree on a shape, nullability, or enum after the
  change; a narrowed type still fed unvalidated data. → check every call site the diff did not
  touch.
- **Silent fallback** — a `catch` that swallows, a `?? default` standing in for a real error, an `as`
  cast papering over a shape that does not match. → ask what the user sees when the hidden error is
  the real one.
- **Unhandled new case** — a widened union, new enum member, or new status that existing switches
  and conditionals do not handle. → grep for every site that branches on that type.
- **Data integrity** — unbounded growth, a lost timezone, a locale-dependent parse or format, a
  float where money belongs. → pass the awkward real-world value.
- **Test blind spot** — the riskiest branch the diff adds has no test that would fail if the branch
  were inverted. → invert it mentally and name the test that should have caught it.
