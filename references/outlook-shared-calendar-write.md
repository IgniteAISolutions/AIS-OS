# Writing to someone else's Outlook calendar

Why Claude can read a colleague's calendar but cannot create events on it, and
what has to change to enable it.

**Live case:** Elaine has edit access to Alice's calendar and adds events to it
manually every day. Claude, signed in as Elaine, can read Alice's calendar but
refuses to create events on it.

## The short answer

It is not a permissions problem on your side. Elaine's Microsoft permissions are
already correct. It is a limitation of Claude's built-in Outlook connector.

Two separate locks sit between Claude and "put this meeting on Alice's
calendar":

1. **Microsoft lock.** Does Elaine have edit rights on Alice's calendar?
   **Already open.** She uses them every day.
2. **Connector lock.** Does Claude's create-event tool have a way to say
   "Alice's calendar", and did it ask Microsoft for permission to write to
   shared calendars? **Shut, and you cannot open it.**

## The plain-English version

Think of a calendar as a diary sitting in someone's house.

Elaine already has a key to Alice's house. She walks in and writes in the diary
whenever she likes. That part works.

Claude has two tools. The **reading** tool has a little box on it labelled
"whose diary?" (`calendarOwnerEmail`), so Claude can point it at Alice's house
and read. The **writing** tool has no such box. It only ever writes in the diary
belonging to whoever signed in. Elaine's key is irrelevant, because the pen
physically does not reach past her own desk.

There is a second thing. When Claude signed in to Microsoft it asked for a
specific list of powers, called scopes. It asked for "read calendars other
people shared with me" (`Calendars.Read.Shared`), which is why the reading works.
It never asked for "write to calendars other people shared with me"
(`Calendars.ReadWrite.Shared`). Microsoft would refuse the write even if the tool
had the box.

So the answer to "is it permissions or is Claude tied to email IDs?" is: the
second one, effectively. The write tool is hard-wired to the mailbox that is
signed in. Elaine's delegate rights cannot travel through a tool that has no
parameter for them.

## Why you cannot fix this in settings

Anthropic's built-in Outlook connector has fixed tool definitions and fixed
scopes. You cannot add a `calendarOwnerEmail` parameter to
`outlook_create_event`, and you cannot widen what it asks Microsoft for. No
amount of granting in Outlook or Entra ID changes it. That is a product change
on Anthropic's side.

Which means: to get shared write today, you need a path that is not the
built-in connector.

## Options, ranked by effort

### Option A: invite Alice as an attendee (works right now, zero setup)

Claude creates the event on Elaine's calendar and adds Alice as an attendee. It
lands on Alice's calendar as an invite she accepts.

One real difference: Alice is not the organiser, Elaine is. So Alice cannot
edit or cancel it herself, and it shows in her calendar as an accepted meeting
rather than as her own entry. For most diary management that is fine. For
"Elaine manages Alice's diary on her behalf", it is a downgrade from what she
does manually today.

### Option B: Zapier or Power Automate (no code, fastest real fix)

Both have a create-event action for Microsoft Outlook. Authorise the connection
as **Elaine**, whose rights already work, then have Claude trigger it.

- Zapier: Microsoft Outlook > Create Event. You have Zapier connected already.
- Power Automate: Office 365 Outlook > Create event (V4).

**Verify this before committing to it:** open the action's Calendar dropdown and
check that Alice's calendar actually appears in the list. Shared calendars live
at a different address in Microsoft's API than your own, and some connectors only
list your own. If Alice's calendar is not in the picker, this route will not work
and you need Option C.

Rough effort: under an hour, assuming the calendar shows up.

### Option C: a custom Graph connector (the proper fix)

A small remote MCP server that talks to Microsoft Graph directly, registered as
an app in Entra ID, added to Claude as a custom connector. Its create-event tool
takes a mailbox parameter, so it can write anywhere Elaine has delegate rights.

Two permission models:

- **Delegated** (`Calendars.ReadWrite.Shared`): acts as Elaine, limited to
  calendars actually shared with her. Safer, and the audit trail stays on her
  account. This is the right one here.
- **Application** (`Calendars.ReadWrite` app-only): acts as a service account and
  can reach every mailbox in the tenant. Only if you need it running unattended,
  and only with an Application Access Policy naming exactly which mailboxes it
  may touch.

Who does what:

| # | Who | What |
|---|-----|------|
| 1 | Alice | Already done. Elaine has "Can edit" on her calendar. |
| 2 | M365 / Entra admin | Register the app, add `Calendars.ReadWrite.Shared` delegated permission, grant admin consent |
| 3 | Claude Team owner or primary owner | Enable custom connectors for the workspace (Settings > Connectors) |
| 4 | Whoever builds it | Stand up the MCP server, add it in Claude |
| 5 | Elaine | Reconnect the connector so the new scope lands on her token |

Rough effort: a day for someone who has done an Entra app registration before.

## Security note

Write access to someone else's calendar is a real privilege. The delegated model
keeps Alice in control, since revoking Elaine's calendar sharing instantly kills
Claude's access too, and every event carries Elaine's name. App-only permission
without an Access Policy does not do either. Do not reach for the bigger hammer
because it is quicker to set up.

## Recommendation

1. Use Option A today so Elaine is not blocked.
2. Spend twenty minutes checking whether Alice's calendar appears in the Zapier
   Outlook picker. If it does, Option B is your answer this week.
3. Only build Option C if this becomes a standing workflow across more than one
   executive.
