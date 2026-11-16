# PARADOX + GHOST
# Security Validation Report

**Report ID:** PG-SVR-2026-028  
**Application:** PARADOX Security Hub v0.3.2  
**Assessment type:** Localhost security regression validation  
**Report generated:** 2026-11-16 07:52 UTC  
**Status:** Tested controls passed; broader assurance pending  
**Classification:** Laboratory / Portfolio Evidence

## 1. Executive Summary

A security review of PARADOX Security Hub v0.3.1 identified
missing structured audit events for rejected CSRF requests and
cross-user record access attempts.

Version 0.3.2 introduced five additional audit event types
and was evaluated through isolated localhost HTTP experiments.

The supplied execution logs show that the tested access controls
continued to reject unauthorized operations, while the new
security events were persisted in SQLite.

## 2. Scope

- Application: PARADOX Security Hub v0.3.2
- Target: isolated HTTP server bound to 127.0.0.1
- Storage: temporary SQLite databases
- Accounts: synthetic test accounts
- Production or third-party targets: none
- Existing v0.3.1 installation: preserved

## 3. Methodology

Testing used Python HTTP clients with independent session
cookie jars, authenticated synthetic users, CSRF tokens,
and direct SQLite verification of audit event persistence.

Each tested server was launched with a temporary database
and shut down after execution.

Results below are transcribed from the operator-supplied
Terminal output. Original console logs are not embedded
in this report and have not been independently archived.

## 4. Original Findings

### F-001: Missing CSRF denial audit event

**Observed in v0.3.1:** An invalid CSRF request returned
HTTP 403 but generated no new audit row.

**Security impact:** Reduced visibility into rejected
request-origin and CSRF checks.

**Status:** Remediated in tested v0.3.2 scenario.

### F-002: Missing cross-user access denial audit event

**Observed in v0.3.1:** An attempt to access another user's
record returned HTTP 404 but generated no new audit row.

**Security impact:** Reduced visibility into unauthorized
resource access attempts.

**Status:** Remediated in tested v0.3.2 scenario.

### Additional audit coverage

Review of v0.3.1 also identified missing audit calls
for rejected record edit/delete operations and logout.
Events were added in v0.3.2 and verified separately.

## 5. Remediation

The following structured audit event types were introduced:

| Event | Meaning | Verification |
|---|---|---|
| csrf_denied | Rejected CSRF/origin check | Passed |
| record_access_denied | Rejected record GET | Passed |
| record_edit_denied | Rejected record update | Passed |
| record_delete_denied | Rejected record deletion | Passed |
| logout | Successful session logout | Passed |

The existing audit table schema remained unchanged.

## 6. Validation Results

### Experiment 027C — Baseline

- Invalid CSRF request: HTTP 403
- Cross-user record read: HTTP 404
- New audit rows after each denial: 0
- Finding: two confirmed logging gaps

### v0.3.2 Installation Regression

- CSRF denial event persisted: PASS
- Cross-user read denial event persisted: PASS
- Logout event persisted: PASS
- Audit subject digest format: PASS
- Synthetic record preservation: PASS

### Experiment 028 — Edit and Delete Denials

- Cross-user edit: HTTP 404
- Edit denial event recorded exactly once: PASS
- Cross-user delete: HTTP 404
- Delete denial event recorded exactly once: PASS
- Both events attributed to requesting user: PASS
- Original record preserved: PASS
- Owner access retained: PASS

## 7. Data Protection

The tested audit rows used SHA-256 hexadecimal subject digests
and did not contain the tested raw usernames or passwords.

An unkeyed SHA-256 digest of a low-entropy user ID does not
provide strong pseudonymization against guessing. Consider
keyed HMAC with a protected secret for privacy-sensitive
production deployments.

The CSRF-denial event currently groups origin failures and
token failures into a common event. Additional structured
reason codes may improve investigative value, provided they
do not expose sensitive tokens.

## 8. Evidence Integrity

**Reviewed source:** `server.py`  
**Source SHA-256:** `e7c41b671efb354bc65673d520e784aaa64ab89581e0201f36a396eb0590bb75`

The SHA-256 hash anchors the local source file at report
generation time. It does not authenticate the Terminal logs,
prove the origin of the software, or establish a complete
chain of custody.

## 9. Limitations and Residual Risk

- Tests used isolated localhost environments, not an
  internet-facing deployment.
- A complete regression suite for v0.3.2 was not executed.
- Concurrent requests, audit write failures, log rotation,
  log tampering, and resource exhaustion were not evaluated.
- Missing records and cross-user access denials may share
  the same HTTP 404 response and audit event.
- Logout of an expired or anonymous session was not assessed
  in this validation sequence.
- No complete penetration test is claimed.
- Results rely on operator-provided execution output.

## 10. Conclusion

The five added security audit event types passed their
documented local HTTP validation scenarios.

The findings observed in Experiment 027C were remediated
for the tested conditions. Further security assessment is
recommended before external deployment.

**Overall assessment:** PASS WITH LIMITATIONS.
