# 🛡️ Automated DevSecOps Pipeline & Vulnerability Remediation Project

## 📌 Project Overview
This repository demonstrates a complete, production-grade **DevSecOps Automated Security Pipeline** integrated with a target web application (OWASP Juice Shop). The goal of this project was to transition a passive CI/CD build process into an active security gatekeeper capable of flagging supply chain vulnerabilities, hardcoded secrets, infrastructure misconfigurations, and application source-code flaws.

By bridging automated defense mechanisms with manual exploitation methodologies, this project covers both **Red Team vulnerability analysis** and **Blue Team remediation/hardening techniques**.

---

## 🏗️ Pipeline Architecture (GitHub Actions)
The workflow file `.github/workflows/security-scan.yml` is split into three concurrent and sequential security gates:

1. **Static Code Analysis (SAST):** Utilizes **Semgrep** configured with specialized rulesets (`p/owasp-top-ten`, `p/javascript`, `p/typescript`) to catch logic flaws, injection vectors, and weak transport configurations directly within the source code.
2. **Secrets Detection:** Leverages **TruffleHog OSS** to run high-entropy filesystem scanning to identify leaked private keys or hardcoded API tokens before deployment.
3. **Container Security (SCA):** Uses **Trivy Vulnerability Scanner** to audit the compiled Docker layers and open-source third-party dependencies (`package.json`) against known CVE databases.

---

## 🕵️‍♂️ Vulnerability Case Studies & Remediation

### Case Study 1: Software Supply Chain Mitigation (Tar Package Gzip Bomb)
* **Vulnerability Identifier:** CVE-2026-59873 (CRITICAL)
* **The Flaw:** The project inherited a transitive dependency on the `tar` library that was vulnerable to a Denial of Service (DoS) exploit. An attacker could upload a small, malformed gzip archive that expands exponentially, exhausting host CPU/disk space and crashing the container.
* **Red Team Impact:** High-reliability, low-footprint Denial of Service capable of taking down application infrastructure remotely.
* **Blue Team Remediation:** Implemented an absolute package resolution rule via the `overrides` block in `package.json` to force-route the runtime engine to a secure release patch baseline (`>= 7.5.19`), bypassing vulnerable nested requirements.
  ```json
  "overrides": {
    "tar": "^7.5.19"
  }
  ```

### Case Study 2: SQL Injection (SQLi) via Raw String Concatenation
* **Vulnerability Identifier:** `express-sequelize-injection` (BLOCKING)
* **The Flaw:** Found in `dbSchemaChallenge_1.ts`. The application database layer constructed database queries using direct string interpolation (`"+criteria+"'`), allowing user-controlled inputs to alter SQL query execution logic.
* **Red Team Impact:** An attacker can input payloads like `' OR '1'='1` to bypass authentication or execute a `UNION SELECT` statement to dump the entire database contents (including user password hashes).
* **Blue Team Remediation:** Refactored raw SQL executions to discard string parsing entirely, shifting inputs into a cryptographically isolated parameterized layout:
  ```typescript
  models.sequelize.query(
    "SELECT * FROM Products WHERE ((name LIKE :search OR description LIKE :search) AND deletedAt IS NULL) ORDER BY name",
    { replacements: { search: `%${criteria}%` }, type: models.sequelize.QueryTypes.SELECT }
  );
  ```

### Case Study 3: Hardcoded Cryptographic Private Keys
* **Vulnerability Identifier:** `hardcoded-jwt-secret` (BLOCKING)
* **The Flaw:** Located in `lib/insecurity.ts`. The server application signed and validated JSON Web Tokens (JWT) using a hardcoded, plaintext string literal stored directly inside the source code repository.
* **Red Team Impact:** If the source repository is leaked or accessed, an attacker can extract the signing key, craft custom administrative payloads, forge their own session token signatures, and achieve a full administrative takeover.
* **Blue Team Remediation:** Purged the static assignment from the repository and implemented a runtime initialization construct that injects keys directly into application memory via system environment blocks:
  ```typescript
  const privateKey = process.env.JWT_PRIVATE_KEY;
  if (!privateKey) throw new Error("FATAL: JWT_PRIVATE_KEY missing!");
  ```

### Case Study 4: Sensitive Information Disclosure via Directory Listing
* **Vulnerability Identifier:** `express-check-directory-listing` (BLOCKING)
* **The Flaw:** Located inside `juice-shop-master/server.ts`. The application backend misconfigured its static file-serving modules by passing raw internal folders directly into the `serveIndex` middleware. This forced the Express engine to dynamically generate a clickable, interactive HTML directory tree index whenever a user queried endpoints like `/ftp` or `/support/logs`.
* **Red Team Impact:** Extreme reconnaissance advantage. A malicious operator can navigate directly to `https://target-app.com` or `/support/logs` to map out the application's layout. This lets them discover and exfiltrate forgotten sensitive database backups (e.g., SQLite files), application blueprints (`package-lock.json`), or active server runtime logs containing sensitive environment vectors or live session tokens.
* **Blue Team Remediation:** Completely removed the broad `serveIndex` middleware handlers for internal paths. For file requirements that actually must remain accessible to the public, the architecture was refactored to serve explicit file assets individually via static paths rather than exposing entire parent directories:
  ```typescript
  // ❌ DEPRECATED AND REMOVED: Exposed full tree structure
  // app.use('/ftp', serveIndexMiddleware, serveIndex('ftp', { icons: true }))

  // ✅ SECURED: Exposes the specific legal file only; blocks folder indexing
  app.use('/ftp/legal.md', express.static('ftp/legal.md'));
  ```

### Case Study 5: Broken Transport Layer Security (Insecure TLS Protocol Support)
* **Vulnerability Identifier:** `javascript.express.security.audit` / Weak Cipher Configuration
* **The Flaw:** The network service wrapper initialized its HTTPS server instance utilizing a legacy configuration block (`secureProtocol: 'TLSv1_method'`). This parameter allowed the system to accept connection negotiations from deprecated cryptography baselines including TLS 1.0 and TLS 1.1.
* **Red Team Impact:** Facilitates Man-In-The-Middle (MITM) attacks. An attacker positioned on the same network layer can execute a protocol downgrade attack (e.g., forcing connection paths down to TLS 1.0). Due to mathematical flaws in legacy ciphers (such as POODLE or BEAST vectors), the attacker can decrypt the intercepted traffic streams, reading plaintext session cookies, payloads, and parameters in real time.
* **Blue Team Remediation:** Stripped out the obsolete `secureProtocol` parameters and enforced strict, modern protocol constraints requiring a minimum connection standard of TLS 1.2 or 1.3:
  ```typescript
  const server = https.createServer({
    key: privateKey,
    cert: certificate,
    // ✅ SECURED: Hardens transport boundary; explicitly drops legacy protocols
    minVersion: 'TLSv1.2'
  }, app);
  ```

---

## 🛠️ Skills Demonstrated
* **DevSecOps Integration:** Automated scanning with Semgrep, Trivy, and TruffleHog.
* **Pipeline Remediation:** Resolving mutable GitHub Actions tags and blocking unsafe shell ingestion patterns (`curl | sh`).
* **Vulnerability Assessment:** Manual mapping of static code findings to live exploitation paths (offline hash cracking via John the Ripper/Hashcat, SQL query manipulation).
* **Defensive Engineering:** Implementing secure coding standards, parameterized queries, and environment variable abstractions.
