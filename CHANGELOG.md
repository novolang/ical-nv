# Changelog

All notable changes to ical-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `icalvalue` — the load-bearing interface, and it is that **a
  property value is not a string**. RFC 5545 section 3.3 defines
  fourteen value types and section 3.2.20 lets a `VALUE` parameter say
  which of them a property carries. The property everybody meets is
  `DTSTART`: `DTSTART;VALUE=DATE:20260105` is an all-day event on the
  5th and `DTSTART:20260105T090000` is an event at nine in the
  morning. A library that kept both as `Str`, or widened both to a
  date-time at midnight, cannot tell them apart — and the two are
  rendered differently, sorted differently, and have different end
  dates. So `IcalValue` is an enum with one arm per type, the `VALUE`
  parameter is honoured, and a property carrying a type it may not
  carry is `IcalValueKindMismatch` rather than a coercion. A DATE-TIME
  carries its FORM as well, because section 3.3.5 gives it three —
  floating, UTC, and a local wall time with a `TZID` — distinguished
  in the text by one letter and a parameter, which is why every
  calendar implementation has read one as another.
- `icalevent` — **`DTEND` is exclusive, and `end_exclusive` is the
  name of the call.** RFC 5545 section 3.6.1 makes it the
  non-inclusive end, and for an all-day event that is the single most
  common bug in calendar software: a one-day event on 2026-01-05 is
  written `DTEND;VALUE=DATE:20260106`. A reader that renders `DTEND`
  as the last day shows the event on two days; a writer that emits the
  last day produces an event everybody else shows one day short.
  Neither raises anything, because each round-trips its own mistake.
  So the name says which convention it answers, `last_day` is
  published beside it for the renderer that wants the other one, and
  section 3.6.1's default for an event with neither `DTEND` nor
  `DURATION` is computed here rather than left to every caller. A
  component carrying both is not refused — the events still want
  reading — but `end_exclusive` and `duration_of` are, because
  answering either would be choosing for the caller.
  `recurrence_of` assembles `DTSTART`, `RRULE`, `RDATE` and `EXDATE`
  into an rrule-nv set in the order section 3.8.5 fixes, so nobody
  re-implements it and nobody forgets that exclusion comes last.
- `icalline` — folding is counted in OCTETS and cut only on character
  boundaries. A character count produces lines of up to 300 octets,
  which a strict reader refuses; an octet count that cuts anywhere
  splits a UTF-8 character in half. `unfold` is deliberately not the
  inverse of `fold`: it removes a CRLF and one space wherever it finds
  them, so a value refolded at a different width unfolds to the same
  text, which is what makes the round trip value-preserving rather
  than byte-preserving. The escaping of section 3.3.11 is a separate
  call from the parse, and `split_values` comes BEFORE `unescape_text`
  because the other order cannot tell an escaped comma from a
  separator — an unescaped comma does not corrupt the file, it turns
  one description into two with nothing raised.
- `icalcomp` — the component tree is published, and every property
  keeps its raw text beside its typed value. A caller that only ever
  saw `IcalEvent` could not read the property its server puts the
  meeting URL in, and dropping `X-` properties is how a round trip
  loses somebody's data.
- `icalparse` — a read answers the calendars AND the complaints.
  Almost every `.ics` file in the world is wrong about something; a
  reader that refused all of them would be unusable and one that
  reported none would leave its caller unable to tell a clean import
  from a lossy one. `Err` is kept for the three faults that stop the
  parse, and `parse_strict` is the other reader, for a caller checking
  its own output. `IcalRead.calendars` is a LIST, because a stream may
  hold more than one iCalendar object and a parser that returned the
  first drops the time-zone definition that came with the invitation.
- `icalerror` — ten refusals, with `is_fatal` dividing what stops a
  read from what is merely non-conformant, and both the logical and
  the physical line number on the faults that have one.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  ical-nv.<module>.<fn>`, and the suites that build a date reach
  calendar-nv's own `todo()` first, because calendar-nv 0.0.2 is an
  interface too.
- **`rrule-nv` is a PATH dependency in this staged tree.** `novo pkg
  publish` refuses a path — a tarball carrying one sends every
  consumer to a directory that is not on their machine — so the
  manifest line becomes `rrule-nv = "^0.0.1"` before the publish, and
  rrule-nv is published first. Under the caret rule `^0.0.1` pins
  exactly 0.0.1.
- **No time zone is resolved.** A `TZID` is carried as the identifier
  the calendar was written with, and a `VTIMEZONE` is carried as its
  components. Turning a zoned wall time into an instant is tz-nv's
  job, and folding it in would make this package untestable without a
  zone database.
- **Nothing here reads a clock.** A `DTSTAMP` is a value the caller
  supplies, which is what makes a generated calendar byte-identical
  between two runs of a test.
