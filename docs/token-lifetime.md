# Canvas token lifetime

Measured on 2026-11-08 against the Aula Virtual instance (Canvas LMS). All times are America/Guayaquil (UTC-5).

## Summary

- Every access token stops working about 60 minutes after its own creation, even though the UI says it expires "never".
- Using the token does not extend it. Polling every 5 minutes changed nothing.
- A token created through the API with a valid token gets its own fresh 60 minutes. It does not inherit the creator's expiry.
- So rotation is possible without a person, as long as the next token is created while the current one is still valid.

## Method

A shell watcher called `GET /api/v1/users/self` every 5 minutes and logged the HTTP status. A 401 marked the end of a token's life. The token value was read from the environment and never printed or logged.

Run 1, one token:

| Token | Created | Last 200 | First 401 |
|---|---|---|---|
| UI token | 13:08 | 14:05:30 | 14:10:30 |

Run 2, a parent created in the UI, then a child created through the API ten minutes into the run (`POST /api/v1/users/self/tokens` with `token[expires_at]` set to one day ahead):

| Token | Created | Last 200 | First 401 |
|---|---|---|---|
| Parent (UI) | 14:26 | 15:22:44 | 15:27:44 |
| Child (API) | 14:37:40 | 15:32:45 | 15:37:45 |

The child outlived the parent by 601 seconds, two check cycles. Both deaths land within one cycle of 60 minutes after the token's creation.

## What the UI and the API say

- The child response carried `expires_at` one day ahead. The token died after one hour. The field does not describe the real limit.
- The UI lists the dead child with "expires" one day ahead and no sign that it stopped working. A dead token stays in the list until someone deletes it.
- The UI "last used" lags the real last call. The parent's last successful call was 15:22:44 and the UI showed 15:17. The child's was 15:32:45 and the UI showed 15:27. Probably Canvas throttles that field. Not verified.
- Two earlier tokens (the 13:08 UI token and an earlier test child) are no longer in the list. Why is not known.

## What this means for the client

- Rotate at 45 to 50 minutes of age, never at 60. Track the creation time in the client's own store, because neither the API nor the UI reports the real expiry.
- Follow the rule in `AGENTS.md`: create the new token, store it, then delete the old one. Do not regenerate in place.
- An expired token cannot create its successor. If the client misses the window, a person has to create a new token in the UI. The client should fail with a clear message pointing to that step.
- A new token can delete the one it replaced. On 2026-11-08 at 15:52 a child token called `DELETE /api/v1/users/self/tokens/:id` on its parent and got 200 with `workflow_state: deleted`. The parent returned 401 right after and the child kept working.
- The token id is not returned for the token you are using. Create responses include the id of the new token only. To find the old one, `GET /api/v1/users/self/tokens/:id` works for your own tokens and returns `purpose`, `created_at` and a `visible_token` that is only a hint (the first 5 characters followed by `...`), not the full value. The client should store the id from the create response so it never has to search.
- The reverse direction failed earlier the same day. Twice, a parent called `DELETE` on a child it had just created and got 404, on `/users/self/tokens/:id` and on the same path with the numeric user id. Why is not known. A parent deleting its child is not needed for rotation, so this does not block the design.

## Limits of this measurement

- One instance, one account, two runs.
- Resolution is one check cycle (5 minutes). Creation times from the UI have minute precision.
- Not tested: a child creating a grandchild in the same run.
- Not known: who imposes the 60 minute limit, Canvas configuration or an institutional policy.

## Open questions

1. Does the chain keep working at depth two and beyond?
2. Why did a parent fail to delete its child (404) while a child deleted its parent (200)? One guess is that a token can only delete tokens created before it. Not tested.
3. Is the limit the same for every account on the instance?
