* Run SonarQube using Docker
* Create a project using UI
* Scan source code using Docker-based scanner
* View vulnerabilities and issues

---

## 🚀 Step 1: Run SonarQube Container

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  --network zap-net \
  sonarqube:community
```

---

## ⏳ Step 2: Wait Until SonarQube is Ready

```bash
docker logs -f sonarqube
```

Wait for message:

```
SonarQube is up
```

---

## 🌐 Step 3: Open SonarQube UI

Open in browser:

```
http://localhost:9000
```

---

## 🔐 Step 4: Login

* Username: `admin`
* Password: `admin`
* Change password after login

---

## 📁 Step 5: Create Project (UI)

1. Click **Create Project**
2. Select **Manually**
3. Enter:

   * Project Key: `juice-shop`
   * Display Name: `juice-shop`
4. Click **Set Up**

---

## 🔑 Step 6: Generate Token

1. Go to **My Account → Security**
2. Click **Generate Token**
3. Copy the token (important)

---

## 📥 Step 7: Clone Source Code

```bash
git clone https://github.com/juice-shop/juice-shop.git
cd juice-shop
```

---

## ⚙️ Step 8: Run Sonar Scanner (Docker)

### Linux / Mac:

```bash
docker run --rm \
  --network zap-net \
  -v $(pwd):/usr/src \
  sonarsource/sonar-scanner-cli \
  -Dsonar.projectKey=juice-shop \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://sonarqube:9000 \
  -Dsonar.login=YOUR_TOKEN
```

### Windows (CMD):

```bash
docker run --rm ^
  --network zap-net ^
  -v %cd%:/usr/src ^
  sonarsource/sonar-scanner-cli ^
  -Dsonar.projectKey=juice-shop ^
  -Dsonar.sources=. ^
  -Dsonar.host.url=http://sonarqube:9000 ^
  -Dsonar.login=YOUR_TOKEN
```

Replace:

```
YOUR_TOKEN = token from SonarQube UI
```

---

## ⏳ Step 9: Wait for Analysis

Expected output:

```
ANALYSIS SUCCESSFUL
```

---

## 📊 Step 10: View Results

Open:

```
http://localhost:9000
```

Click on project: **juice-shop**

---

## 🔍 Step 11: Understand Results

### 🐞 Bugs

Code errors

### 🔐 Vulnerabilities

Security issues (e.g., injection, unsafe code)

### 🧹 Code Smells

Bad coding practices

---

## ⚠️ Troubleshooting

### ❌ SonarQube not starting

```bash
sysctl -w vm.max_map_count=262144
```

---

### ❌ Scanner cannot connect

Use:

```
http://sonarqube:9000
```

(not localhost inside Docker)

---

### ❌ sonar-scanner not found

Use Docker scanner instead (already covered above)

---

## 🧠 Key Concepts

* SonarQube = SAST (Static Analysis)
* Works on source code, not running apps
* UI = visualization
* Scanner = analysis engine

---

## ✅ Final Summary

1. Run SonarQube in Docker
2. Create project via UI
3. Generate token
4. Clone source code
5. Run Docker scanner
6. View issues in dashboard

---

## 🚀 Next Steps

* Integrate SonarQube with CI/CD
* Fix detected vulnerabilities
* Combine with ZAP (DAST) for full DevSecOps setup
