# ADR: Handling Unpatched Vulnerabilities (CVE Exceptions)

## Status
Approved

## Context
Our CI/CD pipeline runs a Trivy scan on the built Docker image and fails the build (`exit-code: 1`) if any `HIGH` or `CRITICAL` vulnerability is detected. 
However, some vulnerabilities reside in upstream library dependencies or base OS images where no vendor fix/patch has been released yet. Blocking the CI/CD pipeline indefinitely for these vulnerabilities would halt deployment of critical updates.

## Decision
We will allow specific unpatched CVE exceptions under the following conditions:
1. The vulnerability is verified to have **no patch/fix available** from the vendor/upstream maintainers.
2. An analysis has been performed to confirm that the vulnerability is not exploitable under our application's runtime context.
3. The exception is registered in a `.trivyignore` file in the root directory.
4. Each exception must be documented in this ADR with an **expiration date** (maximum of 30 days) to enforce periodic review.

### Current Exception Registry

| CVE ID | Package Name | Severity | Date Raised | Expiry Date | Mitigation & Justification |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **CVE-2026-99999** *(Example)* | `libssl` | CRITICAL | 2026-06-18 | 2026-07-18 | Upstream vendor has not released a patch. The API does not expose any direct interfaces using the affected SSL bindings. |

## Consequences
- The CI pipeline will pass even if the specified CVE is present, avoiding deployment blockages.
- The security team must review this file monthly and remove expired entries or renew them if no patch is available yet.
- To whitelist a CVE, developers must add it to `.trivyignore` file:
  ```
  # CVE-2026-99999: libssl issue without upstream patch (Expires 2026-07-18)
  CVE-2026-99999
  ```
- If the expiry date is reached, the exception must be re-validated or remediated.
