# Responsible Vulnerability Disclosure Report

**Target:** crAPI (OWASP Completely Ridiculous API), deployed locally via Docker Compose for training purposes.
**Reporter:** Akandwanaho Onesmas
**Date:** 2026-11-04
**Severity:** Medium (CVSS 3.1 Base Score: 6.5)

## Summary

The vehicle location endpoint `GET /identity/api/v2/vehicle/{vehicle_id}/location` does not verify that the requesting user owns the vehicle identified by `vehicle_id`. Any authenticated user can retrieve the full name, email address and precise GPS coordinates of any other user's vehicle by supplying its ID, which does not have to belong to them.

## Vulnerability Class

Broken Object Level Authorization (BOLA / IDOR) — OWASP API Security Top 10, API1:2023.

## Steps to Reproduce

1. Register two separate accounts, Account A and Account B, on the target application.
2. Log in as Account A. Create a community forum post (or browse existing posts).
3. Observe that forum posts publicly include each author's `vehicleid` field in the API response (`GET /community/api/v2/community/posts/recent`), leaking vehicle IDs that belong to other users.
4. Log in as Account B and obtain a valid session token.
5. Using Account B's token, request the vehicle location endpoint with a vehicle ID that was observed in another user's post:

       curl "http://localhost:8888/identity/api/v2/vehicle/{vehicle_id}/location" \
         -H "Authorization: Bearer <Account B's token>"

6. The server returns HTTP 200 with the full name, email, and GPS coordinates of the vehicle's actual owner, despite Account B having no relationship to that vehicle.
7. Repeated against three different vehicle IDs, all three succeeded identically, confirming the flaw is systemic and not a one-off.

## Evidence

Full request/response evidence is saved in `vehicle_idor_results.txt`, showing three separate vehicle IDs returning three different users' names, emails, and coordinates, all retrieved using one unrelated user's session token.

## Impact

An attacker with any valid (even low-privilege) account can:
- Enumerate or harvest other users' names and email addresses.
- Track the real-time physical location of any vehicle in the system, given its ID.
- Combine this with the public forum feed (which leaks vehicle IDs per post) to target specific, named individuals without needing to guess IDs at all.

This is a serious privacy violation in any real deployment, since physical location tracking of a named individual can enable stalking or physical harm, not just a data leak.

## CVSS 3.1 Scoring

Vector: `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N`
Base Score: 6.5 (Medium), confirmed via the official FIRST.org calculator.

| Metric | Value | Justification |
|---|---|---|
| Attack Vector | Network | Exploitable over HTTP |
| Attack Complexity | Low | No special conditions needed |
| Privileges Required | Low | Any authenticated low-privilege account suffices |
| User Interaction | None | Fully automatable |
| Scope | Unchanged | Stays within the API's own authorization domain |
| Confidentiality | High | Exposes PII and precise location data |
| Integrity | None | Endpoint is read-only |
| Availability | None | No denial-of-service effect |

## Secondary Findings

- **Verbose error disclosure:** `GET /identity/api/v2/user/dashboard` with no Authorization header returns HTTP 404 with the message "Given Email is not registered!" rather than a generic 401. This reveals internal lookup logic and should return a uniform error.
- **Excessive data in public post feed:** `GET /community/api/v2/community/posts/recent` includes each author's email address and internal `vehicleid` in a public, unauthenticated-readable feed. This directly enabled the primary finding above by leaking valid vehicle IDs to target.
- **Secure endpoint (control):** `GET /identity/api/v2/vehicle/vehicles` correctly rejects requests with no Authorization header (401 "Invalid Token"), confirming that ownership/auth checks are possible in this codebase and were simply omitted on the location endpoint.

## Remediation Recommendations

1. On the vehicle location endpoint, verify that `vehicle.owner_id == current_authenticated_user.id` before returning any data; return 403/404 otherwise.
2. Apply the same ownership check to every vehicle-related endpoint, not just this one.
3. Remove or restrict the `vehicleid` and `email` fields from the public forum post API response, since they unnecessarily leak internal identifiers and PII that other users can act on.
4. Return generic, uniform error messages (e.g. a plain 401) for all unauthenticated requests, rather than revealing lookup logic.
5. Log and alert on repeated requests for vehicle IDs that do not belong to the requesting user, as this pattern indicates active enumeration.

## Disclosure Note

This finding was identified against a local, intentionally vulnerable training instance of OWASP crAPI, run entirely on the researcher's own machine. No production system, real user, or third-party service was accessed or tested.
