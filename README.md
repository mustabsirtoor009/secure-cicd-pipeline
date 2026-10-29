# secure-cicd-pipeline
DevSecOps CI/CD pipeline with SAST, SCA, secret and IaC scanning using GitHub Actions

## Pipeline overview

| Stage | Tool | Purpose |
|---|---|---|
| Secret detection | Gitleaks | Finds hardcoded credentials in the repo |
| SAST / IaC | Semgrep | Finds insecure code, workflow, and Terraform patterns |

Target application: [OWASP Juice Shop](https://github.com/juice-shop/juice-shop), a deliberately vulnerable app.

## Sample findings

### 1. Mutable GitHub Action tag
- **Rule:** github-actions-mutable-action-tag
- **Issue:** Actions referenced by tag (`@v2`) can be silently repointed by the owner.
- **Risk:** Supply-chain attack that runs attacker code in CI.
- **Fix:** Pin to a full commit SHA and keep the version in a comment.

### 2. Curl piped into a shell
- **Rule:** gha-curl-pipe-shell
- **Issue:** `curl ... | sh` runs a downloaded script without verification.
- **Risk:** A hijacked URL or server leads to code execution on the CI runner.
- **Fix:** Download, verify the checksum or signature, then run.

### 3. AWS load balancer without access logs
- **Rule:** aws-elb-access-logs-not-enabled
- **Issue:** The ALB has no logging.
- **Risk:** No evidence for incident response or detection.
- **Fix:** Add an `access_logs` block that writes to S3.

## Scan results (run #5)
| Metric | Value |
|---|---|
| Rules run | 408 |
| Files scanned | 1,193 |
| Findings | 68 (all blocking) |
| Secrets found (Gitleaks) | 0 |
