## 🎯 Goal

* Scan Docker images for vulnerabilities
* Identify OS & dependency issues
* Integrate into DevSecOps workflow

---

## 🛠️ Tool Used

**Trivy** – A simple and powerful container vulnerability scanner

---

## 🚀 Step 1: Pull Trivy Image

```bash
docker pull aquasec/trivy
```

---

## 📦 Step 2: Scan a Docker Image

Example: Scan Juice Shop image

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image bkimminich/juice-shop
```

---

## 🔍 Step 3: Understand Output

Trivy will show:

* 🔴 CRITICAL vulnerabilities
* 🟠 HIGH vulnerabilities
* 🟡 MEDIUM vulnerabilities
* 🔵 LOW vulnerabilities

Details include:

* Package name
* Installed version
* Fixed version
* CVE ID

---

## 📊 Step 4: Scan Running Container

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy container juice-shop
```

---

## 📁 Step 5: Scan Filesystem (Optional)

```bash
docker run --rm \
  -v $(pwd):/scan \
  aquasec/trivy fs /scan
```

---

## 📄 Step 6: Generate Report (JSON)

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image -f json -o report.json bkimminich/juice-shop
```

---

## ⚠️ Troubleshooting

### ❌ Permission issue

Make sure Docker socket is mounted:

```
-v /var/run/docker.sock:/var/run/docker.sock
```

---

### ❌ Slow scan

First scan downloads DB → normal

---

## 🧠 Key Concepts

* Scans OS packages (Alpine, Debian, etc.)
* Detects known CVEs
* Works on images, containers, filesystem

---

## 🔥 DevSecOps Integration

| Tool      | Purpose                  |
| --------- | ------------------------ |
| SonarQube | SAST (Code issues)       |
| OWASP ZAP | DAST (Runtime testing)   |
| Trivy     | Container/Image scanning |

---

## ✅ Final Summary

1. Pull Trivy image
2. Scan Docker image
3. Analyze vulnerabilities
4. Generate report
5. Integrate into CI/CD

---
