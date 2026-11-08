# Authentication design

Status: valid for now, as of 2026-11-08. Revisit if ESPOL issues a developer key or the token limit changes. Measurements behind these decisions are in [token-lifetime.md](token-lifetime.md).

## Constraint

- Every access token dies about 60 minutes after its creation, whatever the UI or the API says about expiry.
- A token can create its successor, and the successor can delete it. An expired token cannot create anything.
- The first token has to come from the Aula Virtual UI. There is no SSO or device flow to bootstrap from.
- The cause of the 60 minute limit is not known. The instance is self-hosted by ESPOL and does not expose its Canvas version, so a version or configuration issue is plausible but not confirmed.

## Decisions

1. **Rotate on use.** The client checks the token age before each request and rotates at 45 minutes. Order: create the new token, store it, swap it in, delete the old one.
2. **Optional timer.** A user level timer (systemd user timer or `poliav auth daemon`) can rotate in the background. It is off by default. It only helps while the machine is on.
3. **Accept the gap.** If the machine is off or idle for more than an hour, the chain is broken. The client reports that the token expired and tells the user to run `poliav auth login`. It never fails with a bare 401.
4. **One chain per person, on their own machine.** No shared server holding several people's sessions. A central store would concentrate credentials that belong to individual accounts.
5. **State outside `.env`.** The client writes its current token, its id and its creation time to a state file in the user's state directory (`$XDG_STATE_HOME/poliav/`, default `~/.local/state/poliav/`), mode 600, written atomically. `.env` is only the seed.
6. **Seed from the UI.** `poliav auth login` opens the profile page where tokens are created, reads the pasted value without echo (or from stdin when not interactive), rotates it right away and stores the result. `CANVAS_TOKEN` stays supported as a seed for unattended use.
7. **Own clock.** Token age comes from the client's recorded creation time, never from `expires_at`.

## Commands

```
poliav auth login     read the seed token, rotate it, store the result
poliav auth status    age and estimated time left, never the value
poliav auth rotate    force a rotation
```

## Why not the alternatives

- **Always on server.** It would survive a laptop being off, but it holds sessions for several accounts in one place. The cost outweighs the convenience for a club project.
- **Background daemon as a requirement.** It does not remove the gap, because it stops when the machine does. It also keeps a live credential on disk indefinitely, which may go against the intent of the one hour limit.

## Open

- Ask ESPOL IT whether a developer key can be issued to the club. It would give OAuth with a refresh token that survives idle time and remove the main limitation. Not asked yet.
- Whether a child token can create a grandchild is untested here.
- A seed has no known creation time. Proposed handling: rotate right after loading it.
