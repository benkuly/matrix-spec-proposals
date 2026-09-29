# MSC9999: Share additional invite state

When a Matrix client receives an invitation, he gets access to `InviteState` containing a list of specified
`StrippedStateEvent`s. The minimal set of events that should be included are currently part of the spec.

As the inviter, I want to be able to share additional (stripped) state events to the invitee. As an example, I want to
send some calendar event information, so the invitee can decide to accept or decline it.

## Proposal

This proposal adds a new request body parameter `share_additional_state` to
`POST /_matrix/client/v3/rooms/{roomId}/invite` which is a list of event types.

Example:

```json
{
  "reason": "Welcome to the daily!",
  "user_id": "@dino:connect2x.de",
  "share_additional_state": [
    "de.connect2x.custom",
    "m.jscalendar.event"
  ]
}
```

## Potential issues

## Alternatives

### More complex sharing rules

`share_additional_state` could be a more complex object to allow sharing state event types with specific
`state_key` or some glob-matching rules.

## Security considerations

## Unstable prefix

- `share_additional_state` -> `de.connect2x.msc9999.share_additional_state`
