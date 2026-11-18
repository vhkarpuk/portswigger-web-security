# Multi-step process with no access control on one step

**Category:** Access control / IDOR
**Level:** Practitioner
**Mechanism:** Authorization enforced on early steps but not on the final confirmation step

**Lab URL:** https://portswigger.net/web-security/access-control/lab-multi-step-process-with-no-access-control-on-one-step

---

## Vulnerability

Promoting a user to admin is a multi-step flow (select the action, then confirm). The
access-control check sits on the earlier steps, but the **final confirmation request** —
the one that actually changes the role — is not re-checked. The developers assume only an
admin can reach that last request, so a low-privileged user who replays the confirmation
directly performs the privileged action.

## Reconnaissance & Exploitation

Logged in as the admin (`administrator:admin`), promoting a user issues a confirmation
request:

```http
POST /admin-roles HTTP/2
Cookie: session=<admin session>
Content-Type: application/x-www-form-urlencoded

action=upgrade&confirmed=true&username=carlos
```

The `confirmed=true` parameter marks this as the final step of the flow. I sent it to
Repeater, then — in a separate session logged in as **wiener** — swapped in wiener's
session cookie and changed the username to my own:

```http
POST /admin-roles HTTP/2
Host: 0a7a00f70427ba70801d8faf0000004c.web-security-academy.net
Cookie: session=<wiener session>
Content-Type: application/x-www-form-urlencoded

action=upgrade&confirmed=true&username=wiener
```

```http
HTTP/2 302 Found
Location: /admin
```

The `302` (not `401`) confirms the role change succeeded: wiener became administrator —
lab solved. The low-privileged session was accepted at the one step that lacked a check.

## Impact

Vertical privilege escalation: any authenticated user can grant themselves (or anyone)
admin by replaying the confirmation request with their own session. The multi-step design
gives a false sense of safety — guarding the setup steps is worthless if the step that
commits the change trusts that the caller must already be authorized.

## Remediation

- Enforce the access-control check on **every** step that performs or commits a sensitive
  action, especially the final one. Never assume a later step is unreachable without the
  earlier ones.
- Re-validate the user's role server-side at the point the change is written, not only when
  the flow is initiated.
- Route all steps through a single authorization layer so no individual handler can be
  invoked without the check.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-285 (Improper Authorization); CWE-306 (Missing Authentication for Critical Function)

## Notes

- The flaw is an assumption about reachability: "only an admin gets to the confirm step."
  HTTP has no inherent order — any request can be sent standalone, in any sequence, by
  anyone who knows its shape.
- Test pattern for multi-step flows: walk the whole flow once as a privileged user, capture
  every request, then replay each step individually with a low-priv session. Any step that
  doesn't return an auth error is missing its check.
- Same root cause as the method-based lab (06): the control is attached to the wrong thing
  (a step / a verb) instead of to *who the user is* and *what they're trying to do*.