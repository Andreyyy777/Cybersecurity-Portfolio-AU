# CASE-005 — GitLab Authorization Security Assessment

**Assessment:** 8 November 2026  
**Prepared for portfolio:** 9 November 2026  
**Environment:** Self-hosted GitLab CE 19.5.3 on Ubuntu 24.04.5 LTS ARM64 in UTM  
**Scope:** Authorized local-only laboratory; no public GitLab infrastructure tested.  
**Conclusion:** Expected access restrictions were observed; no authorization bypass demonstrated.

## Summary

A private GitLab project with fictional data was used to compare Guest and Developer access to repository content and restricted administrative routes. Anonymous REST API checks provided a second authorization baseline. The security assessment is a documented learning exercise, **not** a disclosed GitLab vulnerability.

| Test | Role / resource | Observation | Assessment |
|---|---|---|---|
| AUTH-001A | Guest / private README | 404 page displayed, content not shown | Expected |
| AUTH-001B | Developer / same README | Control marker visible | Expected |
| AUTH-002 | Developer / `/admin` | 404 page displayed | Expected |
| AUTH-003 | Developer / `/admin/users` | 404 page displayed | Expected |
| AUTH-004 | Developer / admin user details | 404 page displayed | Expected |
| ADV-BASE-001 | Anonymous / project API | HTTP 404 in Terminal | Expected |
| ADV-BASE-002 | Anonymous / repository API | HTTP 404 in Terminal | Expected |
| ADV-002 | Developer / membership update API | Not executed | Pending |

## Evidence quality

This assessment was documented retrospectively. Original screenshots of browser outcomes were collected; no raw HTTP messages for the browser interactions were preserved. Displayed error pages **do not prove the HTTP status code**. The unauthenticated API checks retained status output but not a full transaction archive. Membership roles have not been independently corroborated with an immutable API capture. These limitations must be resolved before making stronger claims.

## Methodology

- Distinct local identities: administrator, Developer, Guest.
- Private synthetic README marker as a known protected object.
- Browser comparison of expected versus observed role access.
- Unauthenticated curl checks against GitLab REST API.
- Evidence register with limitations and prospective integrity controls.

## Skills demonstrated

Linux VM provisioning, GitLab administration, RBAC testing, REST API checks, evidence evaluation, safe handling of test credentials, technical reporting.

## Next stage

Verify project membership through the API; repeat baseline tests with sanitized raw HTTP metadata; attempt a controlled **Developer-to-Maintainer authorization check** in the local VM, and verify persistent role state. Record all outcomes and SHA-256 hashes as tests occur.

**Privacy:** Raw screenshots, credentials, tokens, full machine logs, and the complete unreviewed evidence archive are intentionally not included in this public portfolio directory.
