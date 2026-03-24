## 📋 Table of Contents

- [Overview](#overview)
- [Pipeline Architecture](#pipeline-architecture)
- [1. SAST — Static Application Security Testing](#1-sast--static-application-security-testing)
- [2. DAST — Dynamic Application Security Testing](#2-dast--dynamic-application-security-testing)
- [3. Container Vulnerability Scanning](#3-container-vulnerability-scanning)
- [4. Policy-as-Code (OPA / Sentinel)](#4-policy-as-code-opa--sentinel)
- [Full Pipeline Integration](#full-pipeline-integration)
- [Quick Reference](#quick-reference)

| Layer | Tool Examples | When It Runs | What It Catches |
|---|---|---|---|
| SAST | SonarQube, Semgrep | On PR / commit | Code-level flaws, secrets, insecure patterns |
| DAST | OWASP ZAP, Nuclei | On staging deploy | Runtime flaws, auth bypass, injection |
| Container Scan | Trivy, Grype | On image build | CVEs in OS packages, base image vulns |
| Policy-as-Code | OPA, Sentinel | On deploy / infra change | Compliance violations, misconfigurations |

---

## Pipeline Architecture

```
Code Commit
    │
    ▼
┌─────────────┐
│  SAST Scan  │  ← SonarQube / Semgrep (on PR)
└──────┬──────┘
       │
    ▼
┌─────────────────────┐
│  Build Docker Image │
│  + Container Scan   │  ← Trivy (blocks push on CRITICAL CVEs)
└──────────┬──────────┘
           │
        ▼
┌──────────────────────┐
│  Deploy to Staging   │
│  + DAST Scan         │  ← OWASP ZAP (active scan)
└──────────┬───────────┘
           │
        ▼
┌──────────────────────┐
│  Policy Gate         │  ← OPA / Sentinel (enforce compliance)
└──────────┬───────────┘
           │
        ▼
  Deploy to Production
```

---

## 1. SAST — Static Application Security Testing

### Key Concepts

- Analyzes **source code without executing it** — catches vulnerabilities early
- Finds: SQL injection, XSS, hardcoded secrets, insecure crypto, OWASP Top 10 issues
- Integrates into IDE or CI so developers get feedback before merging
- Produces false positives — requires tuning and triage rules

### Recommended Tools

| Tool | Best For | License |
|---|---|---|
| SonarQube | Enterprise, multi-language | Community (free) / Commercial |
| Semgrep | Custom rules, fast, OSS-friendly | OSS / Pro |
| Checkmarx | Enterprise compliance | Commercial |
| Snyk Code | Developer-first, IDE plugins | Free tier / Commercial |

### Step-by-Step: SonarQube Setup

**Step 1 — Run SonarQube server**

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  sonarqube:community
```

Access at `http://localhost:9000` (default: `admin` / `admin`)

---

**Step 2 — Create a project and generate a token**

1. Log in → **My Account** → **Security**
2. Generate a new token → copy `SONAR_TOKEN`
3. Add it as a CI secret (never commit to the repo)

---

**Step 3 — Add `sonar-project.properties` to your repo root**

```properties
sonar.projectKey=my-app
sonar.projectName=My Application
sonar.sources=src
sonar.tests=tests
sonar.host.url=http://localhost:9000
```

---

**Step 4 — Integrate with GitHub Actions**

```yaml
# .github/workflows/sast.yml
name: SAST Scan

on: [pull_request]

jobs:
  sonarqube:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@master
        with:
          args: >
            -Dsonar.token=${{ secrets.SONAR_TOKEN }}
```

---

**Step 5 — Set a Quality Gate as a CI gate**

Go to **Administration → Quality Gates** and configure:

- Fail on **new** critical/blocker issues (not legacy debt)
- Block PR merge when Quality Gate fails
- Use branch analysis to scan PRs independently from `main`

### Tips

- Scan only **new** issues on PRs to avoid legacy noise
- Use `.sonarcloud.yaml` for SonarCloud (hosted version)
- Integrate IDE plugins (SonarLint) for real-time feedback during development

---

## 2. DAST — Dynamic Application Security Testing

### Key Concepts

- Attacks a **running application** like a real adversary — no source code needed
- Finds: auth bypass, CSRF, runtime injection, server misconfiguration, insecure session handling
- Complements SAST — catches flaws that only manifest at runtime
- Always run against **staging**, never production (can corrupt data / trigger alerts)

### Recommended Tools

| Tool | Best For | License |
|---|---|---|
| OWASP ZAP | Open source, CI-friendly | Free |
| Burp Suite Enterprise | Deep scanning, manual + automated | Commercial |
| Nuclei | Template-based, fast recon | Free / OSS |

### Step-by-Step: OWASP ZAP in CI

**Step 1 — Deploy app to a staging environment**

Ensure a live test instance is running (e.g. `http://staging:8080`) before the scan job starts.

---

**Step 2 — Run ZAP baseline scan (passive, fast ~2 min)**

```bash
docker run --rm \
  -v $(pwd):/zap/wrk \
  ghcr.io/zaproxy/zaproxy \
  zap-baseline.py \
    -t http://staging:8080 \
    -r report.html
```

---

**Step 3 — Run full active scan**

```bash
docker run --rm \
  -v $(pwd):/zap/wrk \
  ghcr.io/zaproxy/zaproxy \
  zap-full-scan.py \
    -t http://staging:8080 \
    -z "-config scanner.strength=HIGH" \
    -r full-report.html
```

---

**Step 4 — Scan authenticated endpoints**

Create `zap-context.xml` with login URL and form fields, then:

```bash
zap-full-scan.py \
  -t http://staging:8080 \
  -n zap-context.xml \
  -U testuser \
  -r auth-report.html
```

---

**Step 5 — Scan a REST API with OpenAPI spec**

```bash
docker run --rm \
  ghcr.io/zaproxy/zaproxy \
  zap-api-scan.py \
    -t http://staging:8080/api/openapi.json \
    -f openapi \
    -r api-report.html
```

---

**Step 6 — Fail CI on high/critical alerts**

ZAP exits with a non-zero code when high/critical findings are present — add `--exit-code 1` or check ZAP's exit status to block deployment:

```yaml
# In CI: if ZAP exits non-zero, fail the job
- name: DAST Scan
  run: |
    docker run --rm ghcr.io/zaproxy/zaproxy \
      zap-baseline.py -t $STAGING_URL -r report.html
  continue-on-error: false
```

### Tips

- Use `zap-baseline.py` first (passive only) before adding the heavier active scan
- Store ZAP config in `zap.yaml` (automation framework) and commit it to the repo
- Whitelist your staging URL to prevent ZAP crawling third-party services
- Upload `report.html` as a CI artifact for review

---

## 3. Container Vulnerability Scanning

### Key Concepts

- Scans **Docker image layers** for CVEs in OS packages, language dependencies, and base images
- Should gate every `docker push` — a vulnerable base image silently inherits hundreds of CVEs
- Integrates with registries (ECR, GCR, Docker Hub) to continuously re-scan on new CVE disclosures
- Also scans Dockerfiles and Kubernetes manifests for misconfigurations

### Recommended Tools

| Tool | Best For | License |
|---|---|---|
| Trivy | All-in-one: images, IaC, secrets | Free / OSS |
| Grype | Fast image scanning | Free / OSS |
| Snyk Container | Developer UX, registry integration | Free tier / Commercial |
| AWS ECR Inspector | Native AWS registry scanning | Pay-per-use |

### Step-by-Step: Trivy

**Step 1 — Install Trivy**

```bash
# macOS
brew install trivy

# Linux
sudo apt-get install trivy

# Docker (no install)
docker pull aquasec/trivy:latest
```

---

**Step 2 — Scan a local image before pushing**

```bash
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:latest
```

- `--exit-code 1` causes CI to fail on findings
- `--severity HIGH,CRITICAL` filters to actionable issues only

---

**Step 3 — Integrate into CI before `docker push`**

```yaml
# .github/workflows/container-scan.yml
- name: Build image
  run: docker build -t $IMAGE_NAME .

- name: Scan image with Trivy
  run: |
    trivy image \
      --exit-code 1 \
      --severity CRITICAL \
      --ignore-unfixed \
      --format sarif \
      --output trivy-results.sarif \
      $IMAGE_NAME

- name: Upload scan results to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: trivy-results.sarif
```

---

**Step 4 — Scan Dockerfile for misconfigurations**

```bash
trivy config ./Dockerfile
```

Detects: running as root, using `ADD` instead of `COPY`, no `USER` directive, exposed secrets in `ENV`, etc.

---

**Step 5 — Scan Kubernetes manifests**

```bash
trivy config ./k8s/
```

Detects: privileged containers, missing resource limits, `hostPID: true`, missing `readOnlyRootFilesystem`, etc.

---

**Step 6 — Scan filesystem/dependencies (pre-build)**

```bash
trivy fs . --severity HIGH,CRITICAL
```

Useful for catching vulnerable libraries before building the image.

### Tips

- **Pin base images by digest**: `FROM node:20@sha256:...` — the same tag gets new CVEs silently
- Use `--ignore-unfixed` to suppress CVEs with no available fix, reducing noise
- Enable **continuous scanning** in your registry (ECR, GCR) to catch new CVEs in already-pushed images
- Set severity thresholds per environment: block `CRITICAL` always; warn on `HIGH` in dev

---

## 4. Policy-as-Code (OPA / Sentinel)

### Key Concepts

- Policies written in code (Rego for OPA, HCL for Sentinel) are **version-controlled and peer-reviewed**
- Enforces compliance at the **gate** — before infra changes or deployments land
- OPA: used with Kubernetes (admission controller), APIs (Envoy), and CI pipelines
- Sentinel: HashiCorp-native, works with Terraform Cloud, Vault, Consul
- A single policy enforces uniformly across all teams — no manual checklist reviews

### Recommended Tools

| Tool | Best For | Ecosystem |
|---|---|---|
| OPA + Gatekeeper | Kubernetes admission control | Any / K8s |
| conftest | Policy testing in CI pre-deploy | Any |
| HashiCorp Sentinel | Terraform / Vault policy enforcement | HashiCorp |
| Kyverno | Kubernetes-native policy (no Rego) | K8s |

### Step-by-Step: OPA with Kubernetes (Gatekeeper)

**Step 1 — Install OPA Gatekeeper**

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.13/deploy/gatekeeper.yaml
```

Verify:

```bash
kubectl get pods -n gatekeeper-system
```

---

**Step 2 — Write a ConstraintTemplate (Rego policy)**

```yaml
# require-non-root-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requirenonrootcontainer
spec:
  crd:
    spec:
      names:
        kind: RequireNonRootContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requirenonrootcontainer

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container '%v' must set runAsNonRoot: true", [container.name])
        }
```

---

**Step 3 — Apply the template**

```bash
kubectl apply -f require-non-root-template.yaml
```

---

**Step 4 — Create a Constraint (scope + enforcement)**

```yaml
# require-non-root-constraint.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireNonRootContainer
metadata:
  name: require-non-root-prod
spec:
  enforcementAction: deny        # start with "warn" to audit before enforcing
  match:
    namespaces: ["production", "staging"]
```

```bash
kubectl apply -f require-non-root-constraint.yaml
```

---

**Step 5 — Test policy locally with conftest (pre-deploy CI gate)**

```bash
# Install conftest
brew install conftest

# Test a Kubernetes manifest against your policies
conftest test deployment.yaml --policy ./policies/
```

Example output:

```
FAIL - deployment.yaml - Container 'api' must set runAsNonRoot: true
1 test, 0 passed, 0 warnings, 1 failure
```

---

**Step 6 — Enforce policy in CI with OPA CLI**

```bash
# Install OPA
brew install opa

# Evaluate a manifest against a policy file
opa eval \
  --data ./policies/require-non-root.rego \
  --input deployment.json \
  "data.requirenonrootcontainer.violation" \
  --fail
```

Use `--fail` to exit non-zero if any violations are returned — this blocks the pipeline.

---

**Step 7 (Terraform): Sentinel policy in Terraform Cloud**

```hcl
# policies/require-approved-amis.sentinel
import "tfplan/v2" as tfplan

allowed_amis = ["ami-0abc12345", "ami-0def67890"]

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.type is "aws_instance" implies
    rc.change.after.ami in allowed_amis
  }
}
```

Add to `sentinel.hcl` in your Terraform workspace and reference in Terraform Cloud policy sets.

### Tips

- **Start with `enforcementAction: warn`** before switching to `deny` — audit violations without breaking existing workloads
- Use the **OPA Playground** at [play.openpolicyagent.org](https://play.openpolicyagent.org) to iterate on Rego interactively
- Store all policies in a `policies/` directory, reviewed like application code
- Use `conftest` in local pre-commit hooks so developers catch violations before pushing
- Consider **Kyverno** as a simpler alternative to OPA/Gatekeeper if you prefer YAML-based policies over Rego

---

## Full Pipeline Integration

### Example: GitHub Actions end-to-end

```yaml
# .github/workflows/devsecops.yml
name: DevSecOps Pipeline

on:
  push:
    branches: [main]
  pull_request:

jobs:
  sast:
    name: SAST — SonarQube
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: SonarSource/sonarqube-scan-action@master
        with:
          args: -Dsonar.token=${{ secrets.SONAR_TOKEN }}

  build-and-scan:
    name: Build + Container Scan
    runs-on: ubuntu-latest
    needs: sast
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t ${{ secrets.IMAGE_NAME }}:${{ github.sha }} .
      - name: Trivy scan
        run: |
          trivy image \
            --exit-code 1 \
            --severity CRITICAL \
            --ignore-unfixed \
            ${{ secrets.IMAGE_NAME }}:${{ github.sha }}
      - name: Push image
        run: docker push ${{ secrets.IMAGE_NAME }}:${{ github.sha }}

  dast:
    name: DAST — OWASP ZAP
    runs-on: ubuntu-latest
    needs: build-and-scan
    steps:
      - name: Deploy to staging
        run: ./scripts/deploy-staging.sh
      - name: ZAP baseline scan
        run: |
          docker run --rm ghcr.io/zaproxy/zaproxy \
            zap-baseline.py \
            -t ${{ secrets.STAGING_URL }} \
            -r zap-report.html
      - uses: actions/upload-artifact@v4
        with:
          name: zap-report
          path: zap-report.html

  policy-gate:
    name: Policy Gate — OPA
    runs-on: ubuntu-latest
    needs: dast
    steps:
      - uses: actions/checkout@v4
      - name: conftest policy check
        run: |
          conftest test k8s/ --policy ./policies/
      - name: Deploy to production
        if: success()
        run: ./scripts/deploy-prod.sh
```

---

## Quick Reference

### Scan commands cheat sheet

```bash
# SAST — SonarQube scanner
sonar-scanner -Dsonar.token=$SONAR_TOKEN

# DAST — ZAP baseline (passive)
docker run ghcr.io/zaproxy/zaproxy zap-baseline.py -t http://target -r report.html

# DAST — ZAP full scan (active)
docker run ghcr.io/zaproxy/zaproxy zap-full-scan.py -t http://target -r report.html

# Container — Trivy image scan
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:latest

# Container — Trivy Dockerfile / IaC scan
trivy config ./Dockerfile
trivy config ./k8s/

# Container — Trivy filesystem scan
trivy fs . --severity HIGH,CRITICAL

# Policy — conftest (OPA, pre-deploy)
conftest test deployment.yaml --policy ./policies/

# Policy — OPA CLI
opa eval --data policy.rego --input manifest.json "data.main.deny" --fail
```

### Severity thresholds (recommended defaults)

| Severity | SAST | Container | DAST | Policy |
|---|---|---|---|---|
| Critical | ❌ Block PR | ❌ Block push | ❌ Block deploy | ❌ Block deploy |
| High | ⚠️ Warn | ⚠️ Warn (dev) / ❌ Block (prod) | ❌ Block deploy | ❌ Block deploy |
| Medium | ℹ️ Report | ℹ️ Report | ⚠️ Warn | ⚠️ Warn |
| Low | ℹ️ Report | ℹ️ Report | ℹ️ Report | ℹ️ Report |

---

## Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP ZAP Docs](https://www.zaproxy.org/docs/)
- [SonarQube Docs](https://docs.sonarqube.org/)
- [Trivy Docs](https://aquasecurity.github.io/trivy/)
- [OPA Documentation](https://www.openpolicyagent.org/docs/)
- [OPA Playground](https://play.openpolicyagent.org)
- [HashiCorp Sentinel](https://developer.hashicorp.com/sentinel)
- [Gatekeeper (OPA for K8s)](https://open-policy-agent.github.io/gatekeeper/)
- [conftest](https://www.conftest.dev/)

---

*Last updated: March 2026*
