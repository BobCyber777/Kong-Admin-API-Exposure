# Kong-Admin-API-Exposure
Penetration Testing &amp; Remediation Report




**Project:** Tejada Financial / `bank_project`
**Assessment Type:** Authorized Internal Penetration Test / Security Verification
**Target:** Kubernetes Kong Gateway
**Assessment Date:** 2026-10-01
**Tester:** Project Owner / Authorized Operator
**Status:** Remediated and independently verified

---

## 1. Executive Summary

An authorized penetration-testing assessment identified an externally reachable Kong Admin API on the Kubernetes node through a `NodePort` service.

The Kong Admin API was exposed on:

```text
10.10.10.20:31107 → Kong Admin API :8001
```

Prior to remediation, the endpoint permitted unauthenticated access to Kong Admin API resources, including configuration and topology information.

The exposure was remediated by removing the Kong Admin port from the externally exposed Kubernetes `NodePort` Service.

The remediation was subsequently verified from a separate Kali testing host.

### Final security state

```text
BEFORE

Kali
 |
 +--> 10.10.10.20:31107
          |
          +--> Kong Admin API :8001
               externally reachable
               unauthenticated


AFTER

Kali
 |
 +--X 10.10.10.20:31107
 |       filtered / unreachable
 |
 +--> 10.10.10.20:31987
          |
          +--> Kong Proxy :8000
               operational
```

The remediation is therefore considered **closed and externally verified** for the tested attack surface.

---

# 2. Scope

The assessment was limited to the project's authorized test environment.

### In scope

* Kubernetes Kong Gateway
* Kong Kubernetes Service
* Kong Admin API exposure
* Kong proxy exposure
* Network reachability from the authorized Kali testing host
* Configuration disclosure through the exposed Admin API
* Remediation verification

### Target

```text
Target host: 10.10.10.20
Kong proxy: 31987/tcp
Kong Admin NodePort: 31107/tcp
Kong Admin container port: 8001/tcp
Kong proxy container port: 8000/tcp
```

### Out of scope for this finding

The following were not required to establish this finding and were not claimed as tested:

* Authenticated Kong Admin API controls
* Destructive testing
* Production infrastructure
* Third-party infrastructure
* Unauthorized external systems
* Exploitation of downstream Django vulnerabilities
* Modification of Kong configuration through the exposed Admin API

In particular, **unauthenticated write capability was not claimed or required for this finding.**

---

# 3. Testing Methodology

The assessment followed a controlled penetration-testing workflow:

```text
1. Define authorized target
        ↓
2. Perform reconnaissance
        ↓
3. Identify exposed service
        ↓
4. Verify security impact
        ↓
5. Preserve baseline evidence
        ↓
6. Implement minimal remediation
        ↓
7. Commit remediation to source control
        ↓
8. Re-test from separate security-testing host
        ↓
9. Verify service functionality was preserved
        ↓
10. Document final status
```

Testing was performed using standard network and HTTP verification techniques, including:

* TCP port scanning
* Service identification
* HTTP request validation
* Kubernetes Service inspection
* Git diff inspection
* Git commit verification
* Post-remediation external verification

---

# 4. Initial Finding

## Finding ID

```text
KONG-001
```

## Title

**Externally Reachable Unauthenticated Kong Admin API**

## Classification

**Security Configuration / Administrative Interface Exposure**

## Initial State

The Kong Kubernetes Service exposed both the proxy and Admin ports through a `NodePort`.

The relevant configuration was:

```yaml
ports:

- name: proxy
  port: 8000
  targetPort: 8000

- name: admin
  port: 8001
  targetPort: 8001

type: NodePort
```

This resulted in an externally reachable mapping:

```text
10.10.10.20:31107 → Kong :8001
```

---

# 5. Reconnaissance Evidence

From the authorized Kali testing host, the Admin NodePort was identified as reachable.

The Admin API exposed Kong administrative information without authentication.

Previously observed endpoints included:

```text
GET /services
GET /routes
GET /plugins
```

The `/services` response disclosed internal topology including:

```text
django.bank-platform.svc.cluster.local:9000
```

The `/routes` response disclosed application routing information including:

```text
/api
/admin
```

The Admin API root also disclosed Kong runtime information, including:

```text
Kong version: 3.6.1
role: traditional
database: off
admin_listen
```

### Security impact

The exposed interface allowed an unauthenticated network client to obtain administrative configuration and internal topology information.

This increased the information available to an attacker and exposed an administrative interface that should not have been externally reachable.

The assessment did **not** rely on or claim successful unauthorized configuration modification.

---

# 6. Baseline Preservation

Before remediation, relevant Kubernetes and Kong configuration was preserved for comparison.

Evidence included:

```text
kong-service.yaml
kong-deployment.yaml
kong-configmap.yaml
Kong pod information
Kong container/image information
Kong network information
```

The repository also retained pre-change artifacts during the assessment process.

These artifacts are intended to support reproducibility and audit review.

---

# 7. Remediation

The remediation deliberately used the smallest configuration change necessary to remove the demonstrated external exposure.

The `admin` Service port was removed from:

```text
k8s/gateway/kong-service.yaml
```

The resulting Service configuration contains only:

```yaml
ports:

- name: proxy
  port: 8000
  targetPort: 8000

type: NodePort
```

The Kong proxy therefore remains externally available while the Admin API is no longer published through the Kubernetes NodePort.

---

# 8. Source-Control Evidence

The remediation was committed independently to Git.

### Commit

```text
5edb345
```

### Commit message

```text
security: remove external Kong Admin API NodePort
```

### Files changed

```text
k8s/gateway/kong-service.yaml
```

### Commit statistics

```text
1 file changed
4 deletions
```

No unrelated tracked changes were included in this security remediation commit.

Other repository modifications remained outside the remediation commit.

This provides a clear audit boundary between:

```text
Security remediation
        |
        +-- 5edb345
```

and unrelated Kong/Docker configuration work.

---

# 9. Kubernetes Verification

After applying the remediation, the live Kubernetes Service was inspected.

The resulting Service configuration showed:

```text
proxy port=8000
nodePort=31987
targetPort=8000
```

No Admin Service port was present.

Specifically, the live Service did not contain:

```text
port: 8001
nodePort: 31107
name: admin
```

The live Service remained:

```text
type: NodePort
```

but only the proxy port was exposed.

---

# 10. Independent Post-Remediation Verification

Post-remediation testing was performed from the authorized Kali security-testing host.

Command:

```bash
sudo nmap -Pn -p 31107,31987 -sV 10.10.10.20
```

Observed result:

```text
PORT      STATE    SERVICE VERSION
31107/tcp filtered unknown
31987/tcp open     unknown
```

### Interpretation

The former Admin NodePort:

```text
10.10.10.20:31107
```

was no longer externally reachable.

The Kong proxy remained reachable:

```text
10.10.10.20:31987
```

Service detection identified the responding service as Kong through its HTTP response:

```text
Server: kong/3.6.1
```

The proxy returned:

```text
404 Not Found
```

with:

```text
"message": "no Route matched with those values"
```

This is consistent with a functioning Kong proxy receiving a request for which no configured route matched.

---

# 11. HTTP Verification of Former Admin Endpoint

A direct HTTP request was also performed from Kali:

```bash
curl -sS -i --max-time 5 \
  http://10.10.10.20:31107/
```

Observed result:

```text
curl: (28) Connection timed out
```

The former Admin endpoint therefore did not return an HTTP response from the testing host.

This corroborates the Nmap result showing:

```text
31107/tcp filtered
```

---

# 12. Functional Regression Check

The remediation did not remove the Kong proxy service.

The proxy remained exposed through:

```text
10.10.10.20:31987
```

and returned a Kong-generated HTTP response identifying:

```text
Server: kong/3.6.1
```

Therefore, the remediation removed the administrative exposure while preserving the intended externally reachable proxy service.

---

# 13. Before / After Evidence

| Control                       | Before                       | After                  |
| ----------------------------- | ---------------------------- | ---------------------- |
| Kong Admin NodePort           | `31107/tcp open`             | `31107/tcp filtered`   |
| Kong Admin API                | Externally reachable         | Externally unreachable |
| Admin HTTP request            | Returned Kong Admin API data | Connection timeout     |
| Kong proxy                    | `31987/tcp open`             | `31987/tcp open`       |
| Proxy functionality           | Available                    | Available              |
| Kubernetes Admin Service port | `8001` / NodePort `31107`    | Removed                |
| Git remediation               | Not present                  | Commit `5edb345`       |

---

# 14. Residual Configuration Consideration

The Kong deployment still contains an internal Admin listener configuration:

```text
KONG_ADMIN_LISTEN=0.0.0.0:8001
```

This was intentionally **not changed as part of this remediation**.

The demonstrated external exposure was caused by the Kubernetes Service publishing port `8001` as a NodePort.

Removing that Service mapping eliminated the tested external path:

```text
External network
      |
      X
10.10.10.20:31107
      |
      X
Kong Admin :8001
```

The remaining listener should be considered a separate defense-in-depth/hardening review.

Future hardening may evaluate whether the Admin listener should be restricted to an internal interface or otherwise protected, depending on Kubernetes networking requirements and operational needs.

---

# 15. Verification Limitations

The following limitations apply to this assessment:

1. The post-remediation scan was performed from the authorized Kali host available in the test network.
2. `31107/tcp` was observed as `filtered`, not explicitly `closed`.
3. The assessment verifies that the tested Kali host could not reach the former Admin endpoint.
4. No claim is made that every possible network segment or firewall path was tested.
5. Kong Admin API write operations were not tested.
6. The assessment does not establish security of unrelated Kong, Django, Kafka, Kubernetes, or host-level interfaces.
7. The Kong Admin listener inside the pod remains a separate hardening consideration.

---

# 16. Compliance Evidence Chain

The assessment provides the following evidence chain:

```text
AUTHORIZED TARGET
       |
       v
RECONNAISSANCE
       |
       v
ADMIN NODEPORT IDENTIFIED
10.10.10.20:31107
       |
       v
UNAUTHENTICATED ADMIN API VERIFIED
       |
       v
BASELINE PRESERVED
       |
       v
MINIMAL REMEDIATION
Remove admin NodePort
       |
       v
GIT COMMIT
5edb345
       |
       v
LIVE KUBERNETES VERIFICATION
Only proxy NodePort remains
       |
       v
INDEPENDENT KALI RETEST
31107/tcp = filtered
       |
       v
HTTP RETEST
Connection timeout
       |
       v
REGRESSION TEST
31987/tcp = open
Kong proxy operational
       |
       v
FINDING CLOSED
```

---

# 17. Final Assessment Status

**KONG-001 — Externally Reachable Unauthenticated Kong Admin API**

```text
STATUS: REMEDIATED
VERIFICATION: PASSED
EXTERNAL RETEST: PASSED
FUNCTIONAL REGRESSION: PASSED
GIT EVIDENCE: 5edb345
```

The specific externally reachable Admin API path identified during the authorized penetration test has been removed and independently verified as unreachable from the Kali testing host.

---

# 18. Recommended Repository Evidence

For auditability, retain the following evidence with the security assessment records:

```text
k8s/gateway/kong-service.yaml
```

Git commit:

```text
5edb345
```

Pre-remediation configuration/evidence:

```text
kong-service.yaml.before-admin-remediation-20261001-155633
```

Additional captured Kong baseline artifacts should be retained according to the project's evidence-retention policy.

Sensitive credentials, tokens, secrets, private keys, and authentication material should not be committed to the repository.

---

## Conclusion

The penetration-testing procedure identified a concrete externally reachable administrative interface, preserved evidence of the initial condition, implemented a minimal remediation, recorded the remediation in source control, and independently verified the result from the authorized Kali testing host.

The tested Kong Admin API exposure is **closed**.

The Kong proxy remains operational through the intended proxy NodePort.

The remaining Kong Admin listener configuration is documented as a separate defense-in-depth consideration rather than being conflated with the completed remediation.
