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

---

## 🛠️ Skills Demonstrated
* **DevSecOps Integration:** Automated scanning with Semgrep, Trivy, and TruffleHog.
* **Pipeline Remediation:** Resolving mutable GitHub Actions tags and blocking unsafe shell ingestion patterns (`curl | sh`).
* **Vulnerability Assessment:** Manual mapping of static code findings to live exploitation paths (offline hash cracking via John the Ripper/Hashcat, SQL query manipulation).
* **Defensive Engineering:** Implementing secure coding standards, parameterized queries, and environment variable abstractions.
