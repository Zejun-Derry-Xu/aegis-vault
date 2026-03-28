# 🛡️ Aegis Vault: A Unix-Based Secure File System

[![Language: C](https://img.shields.io/badge/Language-C-blue.svg)](https://en.wikipedia.org/wiki/C_(programming_language))
[![Database: SQLite3](https://img.shields.io/badge/Database-SQLite3-003B57.svg)](https://www.sqlite.org/)
[![Security: Defense-in-Depth](https://img.shields.io/badge/Security-Defense--in--Depth-success.svg)](#)
[![Status: Private/Showcase](https://img.shields.io/badge/Source_Code-Private-red.svg)](#)

> **Note to Recruiters & Engineers:** Due to the sensitive nature of the cryptographic methodologies and strict security policies, the raw C source code is maintained in a private repository. This document serves as a comprehensive architectural and methodological showcase of the system.

## 📖 Overview

**Aegis Vault** is a lightweight, highly secure local file repository system built entirely in C from the ground up. 

Instead of relying on modern heavy-weight authentication frameworks (like Auth0 or Spring Security), this project was designed to demonstrate a profound understanding of **low-level Unix security, defensive programming, and applied cryptography**. The system safely manages file additions, deletions, and retrievals while completely isolating user data and mitigating OWASP top vulnerabilities.

## 🏗️ Architectural Highlights

The system operates on a **Privilege Separation (SUID)** model to simulate a highly hostile environment where the underlying database is assumed to be fully compromised (world-readable).

* **SUID Binaries:** Core operations (`add-user`, `add-file`, `retrieve-file`) are executed via SUID programs, temporarily elevating privileges to handle sensitive I/O operations while protecting the vault from standard users.
* **Strict POSIX Permissions:** Uploaded files are forcefully stripped of standard permissions and locked down to `400` (Owner Read-Only) to prevent unauthorized execution or tampering.
* **Safe Overwrite Mechanisms:** Defeated classic POSIX file overwrite vulnerabilities by implementing forced `unlink()` teardowns before any `fopen(..., "wb")` operations, neutralizing permission-denial crashes.

## 🔐 Deep Dive: The "Unbreakable" Cryptographic Engine

A core requirement of this system was to secure user credentials against both online and offline attacks, assuming a worst-case scenario where the SQLite database is leaked.

I engineered a custom **"Salt + Pepper"** cryptographic pipeline utilizing the SHA-512 (`$6$`) algorithm via the Unix `crypt()` library:

1.  **Dynamic Random Salts (Anti-Rainbow Table):** Extracted raw entropy from `/dev/urandom` to generate a unique 16-character salt for *every single user*. This guarantees that identical passwords yield entirely different hash signatures, completely neutralizing pre-computed Rainbow Table attacks.
2.  **Filesystem-Isolated Peppers (Anti-Offline Brute Force):**
    A high-entropy "Pepper" is stored in a heavily restricted `600` secret file, physically separated from the database.
    * **The Formula:** `Final_Hash = crypt(Password + Isolated_Pepper, Random_Salt)`
    * **The Result:** Even if an attacker dumps the entire `.db` file (containing the salts and hashes), they cannot begin a dictionary or brute-force attack without compromising the heavily restricted Unix filesystem to obtain the Pepper.

## 🛑 Threat Mitigation & Defensive Programming

Beyond authentication, the system was hardened against common application-layer attacks:

* **SQL Injection (SQLi) Prevention:** Completely eliminated SQLi vectors by strictly utilizing `sqlite3_bind_text` (Parameterized Queries). No user input is ever concatenated into SQL execution strings.
* **Directory / Path Traversal Defense:** Sanitized all incoming filepaths. Attackers attempting to escape the repository boundary (e.g., passing `../../etc/shadow` as a payload) are aggressively blocked by strict boundary validations.
* **Secure Logging Audits:** Maintained an immutable action log tracking the *Real UID* (not just the Effective UID), ensuring non-repudiation of all repository actions.

## 🧪 Comprehensive Penetration & Functional Testing

To validate the defense-in-depth architecture, I developed a robust **Automated Bash Audit Suite** (`advanced-test.sh`). The script acts as both a QA workflow and an automated Red Team, executing:
* SUID and Permission bit audits (`400`/`644` validations).
* Negative authentication tests.
* Path Traversal payload injections.
* SQL Injection payload injections (e.g., `admin' OR '1'='1`).
* Database plaintext leakage scans.

