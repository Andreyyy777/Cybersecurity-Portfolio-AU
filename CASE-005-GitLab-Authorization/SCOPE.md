# CASE-005: Scope and Methodology

**Assessment type:** Retrospective local GitLab authorization assessment.

**Authorized scope:** Self-hosted GitLab CE instance in a UTM Ubuntu ARM64 VM; historical lab address `http://192.168.64.23`. Tests are limited to the user's own sandbox, three locally managed test accounts and a private test project.

**Excluded:** gitlab.com, third-party infrastructure, public production instances, unrelated VMs, and real user data.

**Environment (from installation transcript):** Ubuntu 24.04.5 LTS ARM64; GitLab CE 19.5.3; private group `CASE-005-Research`; private project `GitLab-Security-Lab`. The terminal transcript shows 15 `gitlab-ctl` services running immediately after installation, followed by a local HTTP `302`; these are installation-time observations, not present-day health assurance.

**Accounts and expected access:** `root` (administrator), `case005_dev` (Developer), `case005_guest` (Guest). Role assignments were operated interactively and are described in the conversation, but an immutable membership-role API export was **not captured**.

**Methods:** Browser navigation as logged-in Guest or Developer; public unauthenticated API requests via curl; comparison of observed page and HTTP responses with access expectations. Screenshots were collected during November 8, 2026; archived copies preserve image bytes.

**Evidence limitations:** Browser pages displaying '404' do not by themselves prove an HTTP status of 404. Direct backend status, raw request headers, response bodies, effective membership role snapshots, GitLab request IDs for these browser interactions, and timestamps from a trusted synchronized clock were not collected for the initial AUTH series. They must not be fabricated. `ADV-002` role escalation has **not** been executed.

**Security and privacy:** The lab uses HTTP, not TLS. Protect all access tokens and session cookies, avoid capturing authentication headers, use short-lived credentials, and do not publish a raw evidence pack without reviewing screenshots for private details. This assessment is educational, not evidence of a reportable GitLab vulnerability.
