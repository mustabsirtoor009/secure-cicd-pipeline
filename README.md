# secure-cicd-pipeline
DevSecOps CI/CD pipeline with SAST, SCA, secret and IaC scanning using GitHub Actions

## Pipeline overview

| Stage | Tool | Purpose |
|---|---|---|
| Secret detection | Gitleaks | Finds hardcoded credentials in the repo |
| SAST / IaC | Semgrep | Finds insecure code, workflow, and Terraform patterns |
| Dependencies, IaC and secrets | Trivy | Finds vulnerable packages, misconfigurations, and hardcoded secrets |

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

### 4. Hardcoded private key in source (Trivy)
- **Rule:** AsymmetricPrivateKey (HIGH)
- **Location:** `lib/insecurity.ts` line 21
- **Issue:** An RSA private key is embedded in the application code.
- **Risk:** An attacker can forge signed login tokens (JWTs) and impersonate any user.
- **Fix:** Remove the key from code, load it from a secrets manager at runtime, and rotate it.

### 5. Private key committed in Terraform (Trivy)
- **Rule:** AsymmetricPrivateKey (HIGH)
- **Location:** `terraform/networking.tf` line 171 (duplicated in `infrastructure/terraform/`)
- **Issue:** A TLS private key is stored next to the certificate in the repo.
- **Risk:** Anyone with repo access can impersonate the server or decrypt its traffic.
- **Fix:** Keep keys out of version control, store them in a secrets manager, and rotate them.
