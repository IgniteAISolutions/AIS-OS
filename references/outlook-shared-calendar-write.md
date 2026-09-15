# Writing to a shared Outlook calendar from Claude

Note for Richard and the Microsoft service provider.

## The situation

Elaine has edit rights on Alice's calendar and adds events to it manually every
day. Claude, signed in as Elaine, can read Alice's calendar but cannot create
events on it.

## The cause

This is not a Microsoft permissions problem. Elaine's delegate rights are
correct and in use.

It is a limitation in Claude's built-in Outlook connector, on two counts:

- The create-event tool only writes to the mailbox that is signed in. It has no
  parameter for a different mailbox. The calendar search tool does have one,
  which is why reads against Alice's calendar work and writes do not.
- The connection holds the `Calendars.Read.Shared` scope but not
  `Calendars.ReadWrite.Shared`. Microsoft would refuse the write even if the
  tool had somewhere to put Alice's address.

Neither is configurable. Granting further rights in Outlook or Entra ID changes
nothing, because there is no route for those rights through the tool.

## Does the service provider need to build this per person?

No. The delegated model is built once at tenant level and then works for
anyone who already has calendar rights.

**One-off work:**

| # | What | Who |
|---|------|-----|
| 1 | App registration in Entra ID with the `Calendars.ReadWrite.Shared` delegated permission, admin consent granted tenant-wide | M365 / Entra admin |
| 2 | A small connector (remote MCP server) that talks to Microsoft Graph, hosted once | Service provider |
| 3 | Enable custom connectors on the Claude workspace | Claude Team owner or primary owner |

**Per person after that:** they sign in to the connector once. Nothing else. No
new registration, no new build.

Access is then bounded automatically by whatever calendars have already been
shared with that person in Outlook. If Alice shares her calendar with a second
assistant next month, it works immediately with no involvement from IT.

## The exception worth flagging

There is an alternative permission model, application (app-only), where a
service account can reach every mailbox in the tenant. That one **does** carry
per-person admin, because you scope it with an Application Access Policy
listing each mailbox it may touch. That is a config line rather than a build,
but it is ongoing work and it is a much broader grant.

Use delegated unless something genuinely needs to run unattended with nobody
signed in. Delegated keeps the audit trail on the individual's account, and if
Alice ever unshares her calendar, Claude's access dies with it.

## Sizing

About a day for someone who has done an Entra app registration before. Most of
that is standing up and hosting the connector. The Microsoft side is routine.

## Check this cheaper option first

A Power Automate flow using the Office 365 Outlook "Create event (V4)" action,
with the connection authorised as Elaine. No app registration, no hosting, and
built once per pattern rather than per person. Zapier's Microsoft Outlook
"Create Event" action is the equivalent if that is already in use.

Before committing to it, open the action's calendar picker and confirm Alice's
calendar actually appears. Shared calendars sit at a different address in the
Graph API than a user's own, and some connectors only list your own. If she is
in the list, this is an afternoon rather than a day. If she is not, fall back to
the connector above.

## Interim

Until either is in place, Claude can create the event on Elaine's calendar and
add Alice as an attendee. It reaches Alice's calendar as an invite. The
difference is that Elaine is the organiser, so Alice cannot edit or cancel it
herself.
