# ical-nv

**iCalendar** is the format calendar programs use to exchange events,
to-do items and free/busy information, specified in
[RFC 5545](https://www.rfc-editor.org/rfc/rfc5545). A `.ics` file is
one. This package reads and writes them in novo-lang. The reference
implementations are the Rust crate
[`icalendar`](https://docs.rs/icalendar) and the Python package
[`icalendar`](https://icalendar.readthedocs.io/).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What iCalendar is

An iCalendar object is a tree of **components**. The outermost is
always a `VCALENDAR`. Inside it are the components that carry the
calendar's contents.

| Component | What it is |
| --- | --- |
| `VEVENT` | An event: something that occupies time |
| `VTODO` | A to-do item |
| `VJOURNAL` | A dated note |
| `VFREEBUSY` | When somebody is busy, with no detail |
| `VTIMEZONE` | The definition of a time zone the calendar refers to |
| `VALARM` | A reminder, inside one of the first three |

Each component holds **properties**. A property is one **content
line**: a name, optional **parameters**, and a value.

```
DTSTART;VALUE=DATE:20260105
```

Here the name is `DTSTART`, there is one parameter named `VALUE`, and
the value is `20260105`.

Two rules of RFC 5545 section 3.1 govern how a content line is
written. A line longer than 75 **octets** is **folded**: it is broken,
a CRLF is inserted, and the next line begins with one space or one
horizontal tab which is not part of the value. And a line ends with
CRLF, not with a bare newline.

A `TEXT` value escapes four characters, given in section 3.3.11: a
backslash, a semicolon, a comma, and a newline, which is written `\n`.

Section 3.3 defines fourteen **value types**. Every property has a
default one, and a `VALUE` parameter names a different one where the
property allows alternatives.

| Type | What it holds |
| --- | --- |
| `BINARY` | base64 |
| `BOOLEAN` | `TRUE` or `FALSE` |
| `CAL-ADDRESS` | a URI, in practice `mailto:` |
| `DATE` | a day |
| `DATE-TIME` | a day and a time of day |
| `DURATION` | a length of time |
| `FLOAT` | a number |
| `INTEGER` | a whole number |
| `PERIOD` | a start with an end or a length |
| `RECUR` | a recurrence rule |
| `TEXT` | escaped text |
| `TIME` | a time of day |
| `URI` | a URI |
| `UTC-OFFSET` | an offset from UTC |

A `DATE-TIME` comes in three forms, given in section 3.3.5. A
**floating** time has no zone: nine in the morning wherever the
calendar is read. A **UTC** time ends in `Z`. A **zoned** time is a
local wall time whose `TZID` parameter names a zone the calendar
defines in a `VTIMEZONE`.

## Install

```
novo pkg add ical-nv
```

## Example

```novo
use icalcomp
use icalevent
use icalparse

fn main() [io]
    let text = "BEGIN:VCALENDAR\r\nVERSION:2.0\r\nPRODID:-//x//y//EN\r\nBEGIN:VEVENT\r\nUID:a@example\r\nDTSTAMP:20260101T000000Z\r\nDTSTART;VALUE=DATE:20260105\r\nDTEND;VALUE=DATE:20260106\r\nSUMMARY:Day off\r\nEND:VEVENT\r\nEND:VCALENDAR\r\n"

    // Read the stream. The calendars come back with a list of
    // everything the reader complained about and kept going past.
    match icalparse.parse(text)
        Err(_) => println("the stream could not be read at all")
        Ok(r)  =>
            for cal in r.calendars
                for c in icalcomp.children_named(cal, "VEVENT")
                    match icalevent.event_of(c)
                        Err(_) => println("not a usable VEVENT")
                        Ok(e)  =>
                            // `last_day` is the day to show. `DTEND`
                            // is the day after it.
                            match icalevent.last_day(e)
                                Some(d) => println("${e.summary} on ${d.day}")
                                None    => println("${e.summary}, timed")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: ical-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `icalerror` | The ten refusals, a stable code for each, and which of them stops a read. |
| `icalline` | The content line: folding, unfolding, escaping, and splitting one into a name, parameters and a value. |
| `icalvalue` | The fourteen value types, the three forms of a date-time, and parsing and printing each. |
| `icalcomp` | The component tree: properties with their raw text beside their typed value, and the lookups over them. |
| `icalevent` | Typed views of `VEVENT`, `VTODO`, `VALARM`, `VFREEBUSY` and `VTIMEZONE`, and the questions asked of an event. |
| `icalparse` | Reading a whole stream, printing one, and checking a calendar before it is sent. |

## How to choose an entry point

**`icalparse.parse` reads a stream and keeps going.** It answers the
calendars and a list of complaints. Use it for a file from somewhere
else.

**`icalparse.parse_strict` refuses anything non-conformant.** Use it
on output this program wrote, where a complaint is a bug.

**`icalevent.event_of` gives an event its fields.** The component it
was read from travels with it, so a property this view does not name
is still reachable through `icalcomp`.

**`icalcomp` is the whole tree.** Use it for a property no typed view
names, and for a component this package does not model.

**`icalparse.calendar` starts a new calendar** with the `PRODID` and
`VERSION` RFC 5545 section 3.6 requires.

**`icalline.fold` and `icalline.unfold` are for a caller doing its own
line handling.** `icalparse.print` folds already.

## The rules a user needs

1. **`DTEND` is the non-inclusive end.** RFC 5545 section 3.6.1. A
   one-day all-day event on 2026-01-05 has `DTEND;VALUE=DATE:20260106`.
   `icalevent.end_exclusive` answers that date;
   `icalevent.last_day` answers the 5th, which is the one to show.
2. **`DTEND` and `DURATION` must not both appear.** RFC 5545 section
   3.6.1. An event carrying both is read, and `end_exclusive` and
   `duration_of` refuse it.
3. **An event with neither has a default length**: one day when
   `DTSTART` is a `DATE`, and no length when it is a `DATE-TIME`. RFC
   5545 section 3.6.1.
4. **A `DATE` is not a `DATE-TIME` at midnight.** The two are
   different value types with different meanings.
   `icalevent.is_all_day` answers which one an event has.
5. **A line is folded at 75 octets**, and the fold must not split a
   character. RFC 5545 section 3.1.
6. **Unfolding removes a CRLF and exactly one space or tab.** RFC 5545
   section 3.1. No other whitespace is part of a fold.
7. **A `TEXT` value escapes a backslash, a semicolon, a comma and a
   newline.** RFC 5545 section 3.3.11. An unescaped comma turns one
   value into a list of two.
8. **Split a multi-value property before unescaping it**, never after.
   `icalline.split_values` splits on unescaped commas only.
9. **A `DATE-TIME` has one of three forms.** RFC 5545 section 3.3.5.
   `icalvalue.form_of` answers which.
10. **A `TZID` names a zone the calendar defines**, and need not be an
    IANA identifier. RFC 5545 section 3.8.3.1.
11. **A `VCALENDAR` must carry `PRODID` and `VERSION`**, and `VERSION`
    must be `2.0`. RFC 5545 sections 3.6, 3.7.3 and 3.7.4.
12. **A `VEVENT` must carry `UID` and `DTSTAMP`.** RFC 5545 section
    3.6.1. A file missing either is read, and the complaint is
    reported.
13. **`DTSTAMP` is when the description was written**, not when the
    event happens. RFC 5545 section 3.8.7.2.
14. **Two `VEVENT`s with the same `UID` are one series and its
    overrides.** The override carries a `RECURRENCE-ID`. RFC 5545
    section 3.8.4.4. A reader that treats them as two events shows
    both.
15. **`EXDATE` is applied after `RDATE`.** RFC 5545 section 3.8.5.1.
    `icalevent.recurrence_of` assembles the series in that order.
16. **A stream may hold more than one iCalendar object.**
    `IcalRead.calendars` is a list.
17. **A round trip preserves values, not bytes.** Fold points move,
    parameters print in the order they are held, and a value that
    parsed prints canonically.

## Time zones

This package carries time zones and does not resolve them.

A `TZID` parameter is kept as the identifier it was written with. A
`VTIMEZONE` is kept as its components, with its `STANDARD` and
`DAYLIGHT` rules readable through `icalcomp`. No function here turns a
zoned or floating wall time into an instant.

A caller that needs an instant uses
[tz-nv](https://novo-lang.org/packages/tz-nv), which is where the two
cases that conversion can meet are answered: a wall time that happened
twice, and a wall time that did not happen at all.

The consequence for a caller is that a calendar can be read, checked
and written in a test with no clock and no time-zone database.

## What is not included

- **Resolving a time zone.** See above.
- **A clock.** `DTSTAMP` is a value the caller supplies. That is what
  makes a calendar this package generates byte-identical between two
  runs.
- **Reading or writing a file.** Opening one costs `[fs]`, and this
  package declares no effects. A stream arrives as a `Str`.
- **iTIP and iMIP.** The scheduling protocols of
  [RFC 5546](https://www.rfc-editor.org/rfc/rfc5546) and
  [RFC 6047](https://www.rfc-editor.org/rfc/rfc6047) — `METHOD`,
  `REQUEST`, `REPLY`, `COUNTER` — are a separate specification and a
  separate package. The `METHOD` property is read and written like any
  other.
- **CalDAV.** [RFC 4791](https://www.rfc-editor.org/rfc/rfc4791) is
  HTTP, which costs `[net]`.
- **jCal and xCal.** The JSON and XML forms of iCalendar, RFC 7265 and
  RFC 6321.
- **Decoding a `BINARY` value.** It is carried as the base64 text,
  because a calendar holding an attachment is holding it to hand on.
- **Validating a `CAL-ADDRESS`.** It is carried as written.

## Related packages

- [rrule-nv](https://novo-lang.org/packages/rrule-nv) is the `RECUR`
  value type: the recurrence grammar and the occurrences it generates.
  This package depends on it, and `icalevent.recurrence_of` hands back
  one of its recurrence sets.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) supplies
  every date, time and duration type used here. This package depends
  on it.
- [tz-nv](https://novo-lang.org/packages/tz-nv) turns a wall-clock
  value into an instant. See the Time zones section.
- [chrono-nv](https://novo-lang.org/packages/chrono-nv) reads the
  clock, which is what a `DTSTAMP` needs.
- [vcard-nv](https://novo-lang.org/packages/vcard-nv) is the other
  half of the same family: RFC 6350 shares iCalendar's content-line
  grammar, its folding and its escaping.

## Tests

```bash
novo test tests/icalline_tests.nv    # folding, unfolding, escaping, content lines
novo test tests/icalvalue_tests.nv   # the value types and the three date-time forms
novo test tests/icalevent_tests.nv   # the exclusive end, and the assembled series
novo test tests/icalcover_tests.nv   # the component tree, the writer, the errors
```

The normative source is RFC 5545: section 3.1 for folding, section 3.3
for the value types, section 3.6 for the components, and section 3.8
for the properties. The vectors are the examples the specification
prints, including its own folded `DESCRIPTION` line.

The suite asserts that RFC 5545's folding example unfolds to one
logical line, that a fold never exceeds the width in octets and never
splits a character, that an unescaped comma splits a value and an
escaped one does not, that a `DATE` and a `DATE-TIME` are different
values with different accessors, that a trailing `Z` and a `TZID`
produce different forms, that a one-day all-day event ends on the
following day and is shown on the preceding one, that an event
carrying both `DTEND` and `DURATION` is read and then refuses both
questions, and that a calendar built by `icalparse.calendar` passes
its own check.

The tests compile today and fail at run, each on the
`not implemented: ical-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at
a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `icalerror.IcalError` and the other public types | the types are declared |
| `icalerror.message`, `.code`, `.is_fatal`, `.line_of` | no |
| `icalline.param`, `.line`, `.parse_line`, `.print_line`, `.param_value` | no |
| `icalline.fold`, `.unfold`, `.fold_width` | no |
| `icalline.escape_text`, `.unescape_text`, `.split_values` | no |
| `icalvalue.default_kind`, `.allowed_kinds`, `.kind_name`, `.kind_named` | no |
| `icalvalue.parse_value`, `.print_value`, `.kind_of` | no |
| `icalvalue.text_of`, `.date_of`, `.datetime_of`, `.form_of`, `.form_tzid` | no |
| `icalvalue.period_to`, `.period_for` | no |
| `icalcomp.property`, `.property_raw`, `.value_of`, `.raw_of` | no |
| `icalcomp.param_of`, `.tzid_of`, `.text_property` | no |
| `icalcomp.component`, `.with_property`, `.with_child`, `.component_name` | no |
| `icalcomp.property_named`, `.properties_named`, `.children_named`, `.property_count` | no |
| `icalevent.event_of`, `.todo_of`, `.alarm_of`, `.freebusy_of`, `.timezone_of` | no |
| `icalevent.is_all_day`, `.end_exclusive`, `.last_day`, `.duration_of` | no |
| `icalevent.recurrence_of`, `.recurrence_id_of`, `.alarms_of`, `.to_component` | no |
| `icalparse.parse`, `.parse_strict`, `.print`, `.check` | no |
| `icalparse.prodid_of`, `.version_of`, `.calendar` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
