# Thndr Security CTF — Technical Assessment

Technical assessment of a containerized financial platform CTF
environment provided by Thndr. Identified and chained multiple
vulnerabilities across the web application layer and the underlying
Kubernetes infrastructure.

## Overview

This assessment covered a full attack path from the application layer
to the cluster layer, demonstrating how individually moderate
vulnerabilities can be chained into a significant compromise.

## Vulnerability Chain

1. **SQL Injection** — Authentication bypass via inline payload in the
   login form, leading to operator-level access.
2. **OS Command Injection** — Remote code execution through an
   unvalidated parameter in a diagnostic endpoint (`shell=True`).
3. **Kubernetes ServiceAccount Token Abuse** — Leveraged the pod's
   mounted SA token to enumerate the Kubernetes API, review RBAC
   permissions via `SelfSubjectRulesReview`, and exfiltrate secrets.
4. **Lateral Network Discovery** — Mapped a secondary workload via
   the API server and fingerprinted its exposed service.

## Skills Demonstrated

- SQL Injection (authentication bypass)
- OS Command Injection (RCE)
- Kubernetes RBAC enumeration & ServiceAccount token abuse
- Secret exfiltration from the Kubernetes API
- Lateral network mapping & service fingerprinting
- Defense-in-depth hardening recommendations

## Report

The full technical writeup is available here:

📄 [`assessment-report.pdf`](./assessment-report.pdf)

## Hardening Recommendations (Summary)

- Replace raw string concatenation with parameterized queries.
- Avoid `shell=True`; pass user input as structured argument lists.
- Apply least-privilege RBAC — no frontend SA should list namespace
  secrets or exec into other pods.
- Enforce Kubernetes NetworkPolicies to block lateral pod-to-pod
  traffic.
- Disable `shareProcessNamespace` on production nodes.

## Disclaimer

This assessment was conducted in an authorized CTF / technical
assignment environment. All sensitive identifiers — including tokens,
credentials, internal IP addresses, and workload names — have been
redacted. Shared for portfolio and educational purposes only.
