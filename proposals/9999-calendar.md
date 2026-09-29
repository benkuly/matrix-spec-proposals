# MSC9999: Calendar

Currently, Matrix primary use case is chat. With native Matrix calls (via MatrixRTC) is on the horizon, Matrix currently
still lacks first-class support for organizing people to meet at specific times, inviting participants, and sending
alerts to indicate that an event is about to begin. Such coordination is currently possible only informally through text
messages, or by using external tools such as calendars and email invitations. Other collaboration platforms have
demonstrated that calendar and event functionality can integrate effectively with a chat application.

The goal of this MSC is to define a lightweight model for calendars and events in Matrix
and to specify the calendar features that can be supported. This MSC also provides an umbrella for related MSCs required
to implement the full calendar and event model.

## Proposal

Because calendars have been extensively modeled in both theoretical and practical software engineering, this MSC uses
[IETF JSCalendar 2.0](https://datatracker.ietf.org/doc/draft-ietf-calext-jscalendarbis/) as the basis for modelling
the calendar domain. Using an existing well established specification allows utilizing existing tools for UI,
import/export or bridging to existing calendar solutions (e.g. iCalendar, CalDAV, etc.).

JSCalendar 2.0 defines three objects (`@type`): `Event`, `Task` and `Group`. Because the term "event" has a special
meaning in Matrix, these JSCalendar objects are called `@type:Event`, `@type:Task` and `@type:Group` from now on.

### New room type

A new room type `m.jscalendar` is introduced. This room MUST contain one state event `m.jscalendar.group`. If it does
not contain this state event, it is considered as an "empty" room other calendar events MUST NOT be evaluated.
Effectively, an `m.jscalendar` room has the meaning of `@type:Group`: It contains a list of `@type:Event`s and
`@type:Task`s.

> [!NOTE]
> The meaning of this sort of room can be different and is similar to JSCalendar `@type:Group`. It can represent for
> example
>
> - a single `@type:Event`
> - a series of `@type:Event`s like a daily meeting
> - a digital conference room, where a series of different `@type:Event`s may happen, and one could join the a live
    stream
> - a list of bank holidays where one would like to "subscribe" to
>
> This proposal follows the general approach of "one room per calendar event", but still allows the full flexibility
> regarding other use cases.

> [!IMPORTANT]
> Clients MUST NOT use `m.room.name` and `m.room.topic`. Instead, they should use `title` and `description` of
> `m.jscalendar.group`.

### New state events

#### `m.jscalendar.group`

A new state event `m.jscalendar.group` is introduced and can be sent into an `m.jscalendar` room.
This state event MUST have an empty `state_key`. This means, that an `m.jscalendar` room can only have one `@type:Group`
and therefore acts like a `@type:Group`. The content of this state event contains a content block `m.jscalendar.group`
matching the `@type:Group` Json. This content block MUST NOT contain the following Json properties:

- `@type`
- `entries`

Example:

```json
{
  "content": {
    "m.jscalendar.group": {
      "version": "2.0",
      "uid": "bf0ac22b-4989-4caf-9ebd-54301b4ee51a",
      "updated": "2020-01-15T18:00:00Z",
      "title": "A simple group",
      "description": "Let's have fun!"
    }
  },
  "event_id": "$143273976499sgjks",
  "origin_server_ts": 1432735824242,
  "sender": "@example:example.org",
  "state_key": "",
  "type": "m.jscalendar.group"
}
```

To get a valid `@type:Group` Json (e.g. for export), the content block must be changes as followed:

- Add `"@type": "Group"`.
- Retrieve all `@type:Event`s (via `m.jscalendar.event`) and `@type:Task`s (via `m.jscalendar.task`) and put them into
  the `"entries": []` array.

Based on the example above this could look like this:

```json
{
  "@type": "Group",
  "version": "2.0",
  "uid": "bf0ac22b-4989-4caf-9ebd-54301b4ee51a",
  "updated": "2020-01-15T18:00:00Z",
  "title": "A simple group",
  "entries": [
    {
      "@type": "Event",
      "uid": "a8df6573-0474-496d-8496-033ad45d7fea",
      "updated": "2020-01-02T18:23:04Z",
      "title": "Some event",
      "start": "2020-01-15T13:00:00",
      "timeZone": "America/New_York",
      "duration": "PT1H"
    },
    {
      "@type": "Task",
      "uid": "2a358cee-6489-4f14-a57f-c104db4dc2f2",
      "updated": "2020-01-09T14:32:01Z",
      "title": "Do something"
    }
  ]
}
```

#### `m.jscalendar.event` and `m.jscalendar.task`

The new state events `m.jscalendar.event` and `m.jscalendar.task` are introduced and can be sent into an `m.jscalendar`
room. The `state_key` must match the `uid` of `@type:Event`/`@type:Task`. The content of the state event contains a
content block `m.jscalendar.event`/`m.jscalendar.task` matching `@type:Event`/`@type:Task`. This content blocks MUST
NOT contain the following Json properties:

- `@type`
- `version` (as JSCalendar enforces to use the `version` from `@type:Group`)
- `participants/<user_id>` where `<user_id>` is a valid Matrix-Id (same applies within `recurrenceOverrides`)

Example:

```json
{
  "content": {
    "m.jscalendar.event": {
      "uid": "a8df6573-0474-496d-8496-033ad45d7fea",
      "updated": "2020-01-02T18:23:04Z",
      "title": "Some event",
      "start": "2020-01-15T13:00:00",
      "timeZone": "America/New_York",
      "duration": "PT1H"
    }
  },
  "event_id": "$143273976499sgjks",
  "origin_server_ts": 1432735824242,
  "sender": "@example:example.org",
  "state_key": "a8df6573-0474-496d-8496-033ad45d7fea",
  "type": "m.jscalendar.event"
}
```

To get a valid `@type:Event`/`@type:Task` Json (e.g. to include it into `@type:Group`), the content block must be
changes as
followed:

- Add `"@type": "Event"`/`"@type": "Task"`.
- Apply patches from `m.jscalendar.patch` message events.
- Optionally apply patches from `m.jscalendar.patch` room account data events.

Redacting an `m.jscalendar.event` or `m.jscalendar.task` removes it from the `@type:Group`.

### Patch privately via room account data event

A user may want to change or extend an `m.jscalendar.event` or `m.jscalendar.task` with some private information. For
example the `alerts` property should contain an additional alert or an alert is snoozed.

A new room account data event `m.jscalendar.patch` is introduced and can be sent into an
`m.jscalendar` room. It contains the content block `m.jscalendar.patch` of the JSCalendar type `PatchObject`. The `type`
of the event has the suffix `.<state_key>` where `<state_key>` is the `state_key` of the event that
should be overridden.

> [!WARNING]
> A pointer in the PatchObject MUST NOT change:
>
> - `@type`
> - `uid`
> - `participants`
> - `organizerCalendarAddress`
> - `sentBy`
> - `recurrenceId`
> - `recurrenceIdTimeZone`
> - `recurrenceRule`

Example:

```json
{
  "content": {
    "m.jscalendar.patch": {
      "recurrenceOverrides/2020-01-15T13:00:00/alerts/snooze-id": {
        "@type": "Alert",
        "trigger": {
          "@type": "AbsoluteTrigger",
          "when": "2026-10-12T09:55:00Z"
        },
        "relatedTo": {
          "reminder-id": {
            "relation": "snooze"
          }
        }
      }
    }
  },
  "type": "m.jscalendar.patch.a8df6573-0474-496d-8496-033ad45d7fea"
}

```

### Patch participants via message event

A user may want to change or extend an `m.jscalendar.event` or `m.jscalendar.task` with some room accessible
information. For example the `participationStatus` property. Right now only `participants` fields are allowed to patch
within a room.

A new room message event `m.jscalendar.patch` is introduced and can be sent into an
`m.jscalendar` room. It contains the content block `m.jscalendar.patch` of the JSCalendar type `PatchObject`. The `type`
of the event has the suffix `.<state_key>` where `<state_key>` is the `state_key` of the event that should be
overridden.

> [!WARNING]
> A pointer in the PatchObject MUST ONLY change the following properties:
>
> - `participants/<user_id>/*`
> - `recurrenceOverrides/*/participants/<user_id>/*`
>
> `<user_id>` must be the MXID of the user sending the event. All clients must ignore events where `<user_id` does not
match the `sender_id`.

Example:

```json
{
  "content": {
    "m.jscalendar.patch": {
      "recurrenceOverrides/2020-01-15T13:00:00/participants/@example:example.org": {
        "participationStatus": "declined"
      }
    },
    "m.relates_to": {
      "rel_type": "m.jscalendar.patch",
      "event_id": "$143273976499sgjks"
    }
  },
  "event_id": "$14327werw499sgjks",
  "origin_server_ts": 1432735824242,
  "sender": "@example:example.org",
  "type": "m.jscalendar.patch"
}

```

> [!NOTE]
> The event could be encrypted, but as long as state events aren't encrypted, the downside of having too many clients
> without Matrix 1.19 historic key sharing support is too high.

A patch always applies to a specific state event. This means, when e.g. a property of an `m.jscalender.event` is
changed, the patch points to an older state. This is effectively resetting participants. While this could make sense for
e.g. changes of the date, it may be inconvenient for changes like a location. A client SHOULD consider this by following
`replaces_state` of `m.jscalender.event` and `m.jscalendar.task` events, finding relations for each state in history and
try to apply them on the current state.

## Calendar

While rooms with type `m.jscalendar` can exist on its own, users may want to group them. This is possible by adding
these rooms into a Matrix space. This works for already joined `@type:Group`s, but does not scale very well for new
users of a space. Because all data of `@type:Group` lies within the `m.jscalendar` rooms, the user needs to join a lot
of rooms to get all relevant information.

TODO how do we solve this? Does it scale? What about new calendar events? Alternative: don't use spaces for hierarchy at
all? Instead, use `m.jscalendar` with all `m.jscalendar.event` of the shared calendar and when guest are needed to join
an event, copy it into a new room?

## Potential issues

### Using JSCalendar 2.0 spec

JSCalendar 2.0 still works the "old" way when it comes to sharing calendar events: They are sent and copied between
calendars. To work in Matrix with non-copy-based federation, E2EE and power levels, a few adaptions were needed. In
the future, this could lead to unexpected problems.

## Alternatives

### One room as one calendar

Instead of having "one room per calendar event", calendar events could always be grouped into rooms and calendar events
copy-share out of band similar to E-Mail. This approach is used
in [MSC4496](https://github.com/matrix-org/matrix-spec-proposals/pull/4496).

### Matrix native events

Although it may be tempting to design a new data structures for calendars, this creates various problems. For one
thing, Matrix is a protocol. It should build on existing solutions whenever possible rather than reinventing the wheel.
Added to this is the issue of compatibility. Export, import, and bridging should always be possible. JSCalendar already
solved compatibility.

### Bridge to E-Mail

The proposal could also define the integration with email. Technically speaking, this is already possible. For example,
a client (such as a bot) with the appropriate permissions and an integrated email client/server could manage
email-participants in the state events. However, an appservice could also perform this task.

### Re-use Matrix relations

Both Matrix and JSCalendar define a relation system. Because Matrix only allows one relation per event and JSCalendar
requires multiple relations, JSCalendar relations are re-used. This also makes it easier to form the final obtained
`@type:Group`.

## Security considerations

This proposal adds a lot of metadata to rooms. Everything in the proposal also works with encryption. Currently, Matrix
does not allow to encrypt state events and there is no general way to encrypt account data without introducing a new
secret. Therefore, this proposal stays unencrypted. A future version may enable encryption.

## Unstable prefix

FIXME

## Dependencies

This MSC builds on:

- MSC9999: Select shared invite state
- MSC9999: FIXME
