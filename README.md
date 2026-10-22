# Thndr Security CTF — Technical Assessment Assignment

* **Candidate:** `[CANDIDATE_NAME]`
* **Target Identifier:** `[TARGET_ENVIRONMENT_ID]`
* **Submission Date:** May 26, 2026

---

## Executive Summary

This report details the security assessment and infrastructure penetration testing performed on a containerized financial platform orchestration environment.

Over the course of the assessment, **three critical security vulnerabilities** were discovered and systematically exploited, leading to a cluster compromise up to the Kubernetes ServiceAccount token leakage and local namespace exploitation. Additionally, lateral network discovery successfully located the exact environment position of the final target asset, although strict perimeter routing prevented direct raw extraction.

> **Assessment Status:** 3 Core Vulnerabilities Successfully Exploited & Documented. Target Assets Located.

---

## Detailed Vulnerability & Exploitation Writeup

### 1. Flag 1: Authentication Bypass via SQL Injection (SQLi)
* **Discovery Method:** By inspecting the behavior of the authentication portal, the application was hypothesized to perform a raw string concatenation statement to match user parameters against an internal database engine (`sqlite3`).
* **Exploitation Path:** By inputting an inline payload break directly inside the Username field, the password filtering logical sequence was manipulated:
  * **Username:** `admin' OR '1'='1'--`
  * **Password:** `(Arbitrary input)`
* The resulting application-level structural query effectively bypassed the true conditional signature validation check and logged the session in directly with the `admin` operator role, revealing the first target token.

### 2. Flag 2: Remote Code Execution via OS Command Injection
* **Discovery Method:** After obtaining authenticated gateway authorization, access was granted to the post-login operator panel `/dashboard`, which presented a connectivity metrics interface `/diag`. The field executes a standard system shell ping request.
* **Exploitation Path:** The input parsing parameter mechanism was discovered to utilize unvalidated strings running on a system subshell (`shell=True`). Using the shell sequential operator `;`, the primary ping command structure was broken to daisy-chain distinct OS bash utilities:
  * `; ls -la`
  * `; cat <target_token_file>`
* The system successfully bypassed parameter safety gates, executing the terminal output command locally on the system process shell and exposing the second target token.

### 3. Flag 3: Kubernetes API Server Namespace Takeover & Secret Leakage
* **Discovery Method:** With remote command execution secured inside the container host filesystem space (`[ENTRY_POD]`), an enumeration check was conducted against the automatic service identity mapping directory:
  `/var/run/secrets/kubernetes.io/serviceaccount/token`
* **Exploitation Path:** 
  * **Binary Workaround:** Since standard terminal utilities (`curl` or `kubectl` binaries) were intentionally stripped from the image root, Python's native `urllib.request` layer was leveraged to systematically perform out-of-band automated API requests straight to the local cluster API controller endpoint (`https://[CLUSTER_API_IP]`).
  * **Pod Discovery Map Execution:** Using a custom Python inline script to fetch from the `/pods` endpoint, the live environment schema was extracted, identifying a critical secondary targeted workload (`[TARGET_POD]`).
  * **RBAC Rules Audit:** To trace exact cluster permissions limits safely without blind fuzzing, a POST request payload was mapped to evaluate verbs assigned to the primary ServiceAccount under `/apis/authorization.k8s.io/v1/selfsubjectrulesreviews`. The returned array confirmed explicit listing privileges across namespace secrets.
  * **Secret Store Exfiltration:** Targeting the core secrets API route directory, internal credential variables were retrieved, where the third target token was securely exposed (natively obfuscated in standard Base64 within the Opaque data block container mapping).

---

## Lateral Network Mapping: Locating Secondary Target Environment

* **Target Discovery:** Through the successful extraction of live cluster Pod specifications via the API server, a comprehensive layout mapping of `[TARGET_POD]` was achieved. Configuration analysis revealed a deterministic clue: `[TARGET_POD]` was configured with an active volume mount explicitly pointing to `<target_file>` sourced from a local ConfigMap.
* **Network Fingerprinting:** Leveraging Python's native `socket` library to conduct an internal network sweep, a live service layer interface was discovered mapping to `[TARGET_POD]`'s runtime context at internal IP `[INTERNAL_POD_IP]` and ClusterIP service endpoint `[CLUSTER_IP_ENDPOINT]`.
* **The Boundary Obstacle:** A low-level network port probe confirmed that **Port 80 (HTTP)** was actively open and listening on `[TARGET_POD]`, proving that this was the exact network location hosting the final target asset. However, direct programmatic exploitation to force an application handshake was heavily restricted by structural request barriers.

---

## Technical Analysis of Dead-Ends & Wrong Turns

1. **The HTTP Router Isolation Layer (404 / 501 Brakes):** Direct HTTP GET requests directed to the root index (`/`), typical challenge endpoints, or dynamically mapped query parameters consistently returned a strict custom web daemon `404 Not Found` response. Conversely, issuing any POST method immediately triggered a `501 Not Implemented` error code. This behavioral fingerprint verified that the application on port 80 strictly processes a non-standard minimalist routing protocol or expects hidden virtual-host headers (`[INTERNAL_VHOST]`) paired with unguessable internal service contract paths.
2. **The SPDY / Exec Streaming Pipeline Wall:** The cluster RBAC layout permitted `create` actions on `pods/exec` for `[TARGET_POD]`. Attempting to simulate an interactive binary shell execution via custom socket connection strings to drop the file or fetch it natively resulted in automated gateway platform timeouts because the Kubernetes API server expects multiplexed streaming frames (`SPDY/3.1` channel initialization) to synchronize `stdout`.
3. **Process Space Extraction Constraints:** Attempts to read neighbors' filesystem allocations directly via local shared `/proc/[0-9]*/root` process mappings were deployed to bypass the API server entirely. While active system processes were visible, tight container runtime file permissions successfully protected the rootfs isolation, preventing cross-container leakage on the host device.

---

## Architectural Defense & Hardening Recommendations

If defending this production environment, the following controls should be implemented immediately to establish a robust defense-in-depth model:

### 1. Secure Software Development Life Cycle (SSDLC)
* **Database Safety:** Eliminate raw string concatenation queries inside the SQLite login script block. Force the integration of strict, natively compiled **Parameterized Queries / Prepared Statements** to neutralize SQL injection vectors entirely.
* **Process Security:** Refuse the utilization of `shell=True` inside Python subprocess management functions. Pass incoming user parameters exclusively as structured list arrays to fully mitigate system command injection chains inside `/diag`.

### 2. Implement Zero-Trust Cluster RBAC (Least Privilege)
* Review ClusterRole and RoleBinding parameters. The capability of a frontend-facing web application service account (`[ENTRY_POD_SA]`) to query the global list of namespace secrets or issue execution commands to distinct background nodes represents an excessive privilege model. Fully restrict access keys to ensure the API server strictly blocks unmapped object metadata mapping sweeps.

### 3. Enforce Micro-Segmentation and Pod Isolation
* **Network Policies:** Implement a strict Kubernetes `NetworkPolicy` structure that blocks direct lateral IP traffic communication between separate application pods unless a clear microservice interface pattern is structurally declared.
* **Process Namespace Isolation:** Ensure the process context namespace (`shareProcessNamespace`) is disabled on production nodes to prevent cross-container visibility via the shared host `/proc` file allocation mappings.

---

## Disclaimer
This assessment was conducted in an authorized educational / technical evaluation environment. All internal identifiers, IPs, and workload metadata have been generalized for portfolio publication purposes.
