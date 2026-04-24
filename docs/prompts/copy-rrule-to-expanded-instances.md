# Copy recurrenceRule onto expanded instances

## Context

The recurring events design originally specified `recurrenceRule: null` on expanded instances (virtual instances computed during GET expansion). This was a normalization instinct — the RRULE "belongs" to the parent — but it created a real UX gap: the FE's `formatRecurrenceLabel()` function needs the RRULE string to display "Weekly on Tue, Thu, Fri" instead of a generic "Recurring event" fallback.

This was discussed during the BE PR #20 review cycle (see `reviews/recurring-events/02-recurrence-rule-on-virtual-instances.md`) and the field was set to null at that time. The design doc (`docs/recurring-events-design.md` §4) has been updated to reflect that instances should now carry the parent's RRULE.

## The change

In `CalendarEventMapper.toInstanceResponse()`, set `recurrenceRule` to the parent's value instead of null.

Before:
```java
.recurrenceRule(null)
```

After:
```java
.recurrenceRule(parent.getRecurrenceRule())
```

That's the entire change. The parent is already passed into the method.

## Verify

- Existing tests for instance expansion should still pass (the field isn't asserted as null anywhere critical, but check)
- If any test asserts `recurrenceRule` is null on instances, update the assertion to expect the parent's RRULE string

## Branch & PR

- Branch: `fix/copy-rrule-to-instances`
- Atomic commit, open a PR when done
