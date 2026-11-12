# status

The maintenance flag for the apps that talk to the `dg` VM.

It lives here, on GitHub Pages, **because it has to survive the thing it
reports on**. The VM and dz-core (the machine that terminates TLS for
`*.dg.darrigozik.com`) go down together, so a flag served from either one would
be unreachable exactly when it matters. `davidgil17.github.io` shares no DNS,
TLS or CDN path with `darrigozik.com`.

Served at: **https://davidgil17.github.io/status/status.json**

## Who reads it

| App | Key | Where |
|-----|-----|-------|
| Creature Hatcher iOS | `creature-app` | `creature-app-ios/CreatureApp/Services/ServiceStatus.swift` |
| Creature Hatcher Android | `creature-app` | `creature-app-android/.../data/StatusClient.kt` |
| Broadway Guesser | `broadway-guesser` | `broadway-guesser/frontend/status.js` |

Both creature clients share one key on purpose: they are served by the same
backend, so they go down together.

## Schema

```json
{
  "schema": 1,
  "apps": {
    "<key>": {
      "maintenance": false,
      "from": null,
      "to": null,
      "title": "Down for maintenance",
      "message": "Shown under the title."
    }
  }
}
```

A client shows its maintenance page when:

```
maintenance == true  ||  (from && to && from <= now <= to)
```

`from`/`to` are ISO 8601. Prefer UTC with a trailing `Z`
(`2026-11-18T02:00:00Z`); a numeric offset (`2026-11-17T21:00:00-05:00`) also
parses on all three clients. `title` and `message` are shown verbatim, so they
are the one place the wording can be changed without shipping an app.

## Scheduling a planned outage

Easiest way is the admin dashboard's **Maintenance** page
(`dg.tail310927.ts.net/maintenance`, Tailscale only). It takes local times,
stores UTC, refuses a half or inverted window, and shows what the apps are
actually being served.

That page runs on the VM, which is usually the machine going down — so it can
only schedule beforehand. Editing this file directly works just as well:

```json
"from": "2026-11-18T02:00:00Z",
"to":   "2026-11-18T06:00:00Z"
```

Either way the page then appears and disappears on its own — nothing has to be
flipped at 2am, and nothing depends on having a laptop during the outage.

## If it overruns

Set `"maintenance": true` and push. That ignores the window entirely. Set it
back to `false` when the server is really up — **leaving it on is the one
failure mode that keeps a healthy app dark.**

Mid-outage the dashboard is unreachable, so do this from github.com on a phone:
[edit status.json](https://github.com/Davidgil17/status/edit/main/status.json).

Expect up to 60 seconds for a change to reach clients: Pages serves this with
`cache-control: max-age=600`, and the clients defeat that by requesting
`status.json?t=<unix-minute>`.

## Rules the clients follow

- **Fail open.** If this file can't be fetched or won't parse, the apps behave
  normally. An unreadable flag is our problem, not the user's.
- **Never persisted.** Clients hold the response in memory only and re-fetch at
  launch. A `maintenance: true` written to disk would outlive the outage and
  pin users into the maintenance screen.
- **Unknown keys are ignored**, and `apps` is decoded as a map, so adding a
  fourth app here cannot break an already-shipped build.

Because of fail-open, **do not test a change by making this file invalid** — a
syntax error reads as "everything is fine" to every client. Use the `test-app`
key, which is permanently on, to exercise a gate from a debug build.
