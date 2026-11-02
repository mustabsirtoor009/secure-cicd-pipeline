# secure-cicd-pipeline

DevSecOps CI/CD pipeline with SAST, SCA, secret and IaC scanning using GitHub Actions.

## Pipeline overview

| Stage | Tool | Purpose |
|---|---|---|
| Secret detection | Gitleaks | Finds hardcoded credentials in this repository |
| SAST / IaC | Semgrep | Finds insecure code, workflow, and Terraform patterns in the target app |
| Dependencies, IaC and secrets | Trivy | Finds vulnerable packages, misconfigurations, and hardcoded secrets in the target app and its container image |

Target application: [OWASP Juice Shop](https://github.com/juice-shop/juice-shop), a deliberately vulnerable app. Semgrep and Trivy scan it (it is downloaded during each run), while Gitleaks scans this repository. That is why Gitleaks reports no secrets while Trivy finds private keys in Juice Shop.

## Security gate

The pipeline fails the build when a scanner finds serious issues:

| Scanner | Gate condition |
|---|---|
| Gitleaks | Any secret detected |
| Semgrep | Any finding (`--error`) |
| Trivy | Any HIGH or CRITICAL finding (`--exit-code 1`) |

Because the target app is deliberately vulnerable, the pipeline is expected to fail. A red run shows that the gate blocks insecure code. On a real project, the findings would be fixed until the run turns green.

## Pipeline run

## Scan results (first full run)

| Metric | Value |
|---|---|
| Semgrep rules run | 408 |
| Files scanned by Semgrep | 1,193 |
| Semgrep findings | 68 (all blocking) |
| Secrets found by Gitleaks (this repo) | 0 |
| Critical Node.js package vulnerabilities (Trivy image scan) | 11 |

## Sample findings

### 1. Mutable GitHub Action tag
- **Rule:** github-actions-mutable-action-tag
- **Issue:** Actions referenced by tag (`@v2`) can be silently repointed by the owner.
- **Risk:** Supply-chain attack that runs attacker code in CI.
- **Fix:** Pin to a full commit SHA and keep the version in a comment. Applied in this repository's own workflow.

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

### 6. Critical vulnerable dependencies (Trivy image scan)
- **Rule:** CVE-2015-9235 (jsonwebtoken), CVE-2019-10744 (lodash), CVE-2023-46233 (crypto-js)
- **Issue:** The container image ships outdated packages, such as jsonwebtoken 0.1.0 and lodash 2.4.2.
- **Risk:** An attacker can bypass token verification, pollute object prototypes, or break password hashing.
- **Fix:** Upgrade to the fixed versions (jsonwebtoken 4.2.2, lodash 4.17.12, crypto-js 4.2.0) and replace packages with no fix (decompress, marsdb).

## Hardening applied to this pipeline

- Third-party actions are pinned to full commit SHAs, with the version kept in a comment.
- Dependabot opens weekly pull requests to update the pinned actions.
- Trivy runs from its official Docker image instead of a third-party action.

## Limitations

- The findings are documented, not fixed. The target app is a training app with intentional flaws.
- The scanner Docker images (`semgrep/semgrep`, `aquasec/trivy:latest`) are not pinned by digest yet.
