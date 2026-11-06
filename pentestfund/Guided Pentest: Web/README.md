# RecruitX Web Application Penetration Testing — Lab Documentation

## Overview

This repository documents a hands-on, local-only web application penetration testing exercise performed against **RecruitX**, a fictional recruitment platform built for training purposes. The assessment was carried out from a Kali Linux VM against a Dockerized target running at `http://127.0.0.1:8088`.

The exercise is structured as a guided engagement in which the tester (a junior penetration tester) works through reconnaissance, vulnerability discovery, controlled exploitation, and professional reporting — the same lifecycle used in real-world web application assessments, but against a safe, isolated, intentionally vulnerable application.

**Scope note:** All testing described here targets a local, disposable lab environment (`127.0.0.1:8088`) that the tester controls. None of the techniques, commands, or payloads in this document are intended for use against systems the tester does not own or have explicit written authorization to test.

## Objectives

- Enumerate and understand the attack surface of a web application before attempting exploitation.
- Identify and safely demonstrate common web application vulnerabilities: IDOR / broken access control, API information disclosure, password reset weaknesses, and unsafe file upload handling.
- Distinguish between informational observations (e.g., a missing header) and confirmed, demonstrated vulnerabilities.
- Collect reproducible evidence (command output, screenshots, raw HTTP responses).
- Translate technical findings into a structured, client-ready penetration testing report with remediation guidance.

## Key Concepts

| Concept | Definition | Why it matters |
|---|---|---|
| Reconnaissance / Enumeration | Passive and active collection of information about a target before attacking it | Prevents wasted effort and informs which attack paths are realistic |
| IDOR (Insecure Direct Object Reference) | A flaw where an application exposes a reference to an internal object (e.g., a numeric user ID) and fails to verify the requester is authorized to access it | One of the most common and highest-impact web vulnerabilities (OWASP Top 10 — Broken Access Control) |
| Authentication vs. Authorization | Authentication confirms *who* a user is; authorization confirms *what* that user is allowed to do | Many vulnerabilities (like IDOR) exist because an app checks authentication but skips authorization |
| API Information Disclosure | An API endpoint returning more data, or more access, than the caller should have | APIs are often tested less rigorously than the main UI, making them a common weak point |
| Password Reset Weakness | Flaws in the account-recovery flow (predictable/disclosed tokens, no expiration, reusable tokens, no rate limiting) | A weak reset flow can let an attacker take over accounts without ever guessing a password |
| Unrestricted File Upload | Accepting and storing a file whose type or content is not properly validated | If an uploaded file can be executed by the server (e.g., as PHP), it can lead to full server compromise |
| Security Headers | HTTP response headers that instruct the browser to enforce protective behavior (e.g., CSP, X-Frame-Options) | Defense-in-depth; their absence doesn't prove exploitability but it does raise risk |

## Tools and Technologies

| Tool | Purpose |
|---|---|
| `curl` | Command-line HTTP client used to inspect raw requests/responses, headers, and API output |
| `nmap` | Network/service scanner used to confirm the open port and fingerprint the service |
| `gobuster` | Directory/file brute-forcing tool used to discover hidden or undocumented endpoints |
| Firefox + DevTools | Manual browsing, request inspection (Network tab), and cookie/storage inspection |
| Docker / Docker Compose | Runs the vulnerable RecruitX application as isolated containers |
| Kali Linux | The attacker workstation, pre-loaded with the above tools |

## Lab Environment Setup

### Starting the Application

```bash
cd ~/Downloads/recruitx-local-lab
ls -la
sudo docker compose up -d
```

**Purpose:** Launches the RecruitX application and its dependencies as Docker containers in detached mode.

**Breakdown:**
- `docker compose up` — reads `docker-compose.yml` and starts all defined services.
- `-d` — "detached" mode; containers run in the background instead of occupying the terminal.

**Fallback for older installations:**

```bash
sudo docker-compose up -d
```
Older Docker installations ship Compose as a separate binary (`docker-compose`) rather than the integrated `docker compose` plugin; both perform the same function.

**Verification commands:**

```bash
sudo docker ps
sudo docker compose logs --tail=50
```

- `docker ps` lists running containers, confirming the application started and which ports are mapped.
- `docker compose logs --tail=50` shows the last 50 lines of container logs, useful for catching startup errors (e.g., a port conflict or missing dependency).

**Expected result:** The application becomes reachable at `http://127.0.0.1:8088`.

### Test Account

| Field | Value |
|---|---|
| Email | `testuser@recruitx.local` |
| Password | `Password123` |

A low-privilege account is used first, by design — professional assessments typically start from the access level of a realistic attacker (an unprivileged registered user) rather than starting with administrative credentials.

### Workspace Organization

```bash
mkdir -p ~/RecruitX-Assessment/{recon,exploitation,evidence,reports}
cd ~/RecruitX-Assessment
```

**Purpose:** Creates a standard evidence-handling directory structure, mirroring how professional penetration testers organize engagement artifacts so findings are reproducible and auditable.

| Directory | Contents |
|---|---|
| `recon/` | Headers, scan output, directory brute-force results, notes |
| `exploitation/` | Raw request/response evidence from vulnerability testing |
| `evidence/` | Screenshots |
| `reports/` | The final written report |

---

## Mission 01 — Reconnaissance and Enumeration

**Objective:** Build an accurate picture of the application's technology, structure, and exposed endpoints before attempting to find vulnerabilities. Attacking without reconnaissance risks wasted effort and missed attack surface.

### Step 1 — HTTP Response Headers

```bash
curl -I http://127.0.0.1:8088
```

**Purpose:** Sends an HTTP `HEAD` request, returning only response headers (no body), to quickly reveal server metadata.

**Breakdown:** `-I` tells curl to issue a HEAD request instead of GET.

**Output analysis:**

| Header | Meaning |
|---|---|
| `HTTP/1.1 200 OK` | The server processed the request successfully |
| `Server` | Identifies the web server software (and sometimes version) |
| `Content-Type` | The MIME type of the resource that *would* be returned |
| `Date` | Server-side timestamp of the response |

**Practical importance:** Server banners can reveal the web server family and sometimes version numbers, which may correspond to known vulnerabilities. However, this alone is only an *information disclosure observation* — it does not confirm an exploitable vulnerability until corroborated with version-specific research.

**Evidence capture:**

```bash
curl -I http://127.0.0.1:8088 > recon/http-headers.txt
```

### Step 2 — Service Identification with Nmap

```bash
sudo nmap -sV -sC -p 8088 127.0.0.1
```

**Purpose:** Confirms the target port is open and fingerprints the service and software version running on it.

**Breakdown:**

| Flag | Function |
|---|---|
| `-sV` | Probes open ports to determine service/version information |
| `-sC` | Runs Nmap's default script set (banner grabbing, basic enumeration scripts) |
| `-p 8088` | Restricts the scan to port 8088 |
| `127.0.0.1` | Scan target (the local loopback interface, since the lab runs locally) |

**Execution:** Nmap sends probe packets to port 8088, analyzes the responses against its service-fingerprint database, and additionally runs NSE (Nmap Scripting Engine) default scripts against the detected service.

**Expected output:** Confirmation that port 8088/tcp is `open`, the service is identified as `http`, and (if available) the web server name/version in the banner.

**Evidence capture:**

```bash
sudo nmap -sV -sC -p 8088 127.0.0.1 -oN recon/nmap-scan.txt
```
`-oN` writes the scan results in Nmap's normal (human-readable) output format to the specified file.

**Analysis prompt:** A service listening on port 8088 rather than the default HTTP port 80 typically indicates either a non-privileged process binding to a high port (ports below 1024 require elevated privileges on Linux), a reverse proxy or container port mapping (common with Docker, where internal port 80 is mapped to host port 8088), or a deliberate choice to run a secondary/alternate web service alongside another one already using port 80.

### Step 3 — Directory and File Discovery with Gobuster

```bash
gobuster dir \
  -u http://127.0.0.1:8088 \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php \
  -t 10 \
  -o recon/gobuster.txt
```

**Purpose:** Brute-forces common directory and file names against the target to discover endpoints not linked from the visible UI.

**Breakdown:**

| Flag | Meaning |
|---|---|
| `dir` | Gobuster's directory/file enumeration mode |
| `-u` | Target base URL |
| `-w` | Path to the wordlist used for guessing names |
| `-x php` | Additionally appends `.php` to each wordlist entry and tests it |
| `-t 10` | Uses 10 concurrent threads for the scan |
| `-o` | Writes results to the specified output file |

**Prerequisite check:**

```bash
ls /usr/share/wordlists/dirb/common.txt
gobuster version
```
If the wordlist or tool is missing:
```bash
sudo apt update
sudo apt install -y dirb
sudo apt install -y gobuster
```

**Output analysis — HTTP status codes:**

| Code | Meaning |
|---|---|
| 200 | Resource exists and is accessible |
| 301 | Permanent redirect |
| 302 | Temporary redirect |
| 403 | Access forbidden (resource exists, but access is refused) |
| 404 | Resource not found |

**Practical importance:** A 403 is not automatically a vulnerability — it indicates the server is enforcing some access restriction on a resource that exists. The value of this step is in producing a map of real endpoints (e.g., `/api/`, `/admin/`, `/login.php`, `/register.php`, `/profile.php`, `/reset.php`) to investigate further.

### Step 4 — API Discovery

```bash
curl -i http://127.0.0.1:8088/api/
curl -s http://127.0.0.1:8088/api/ -o recon/api-discovery.json
cat recon/api-discovery.json
```

**Purpose:** Separately enumerates the application's API surface, since APIs frequently expose more granular data and are tested less consistently than the main site.

**Breakdown:**
- `-i` includes response headers in the output (useful to see content type and status).
- `-s` runs curl in "silent" mode (no progress meter), appropriate when piping output to a file.
- `-o` writes the response body to a file.

**What to look for:** Enumerated endpoints such as `/api/user?id=` or `/api/jobs`, and — in this lab specifically — their direct PHP equivalents `/api/user.php?id=` and `/api/jobs.php` when the "pretty" routes return 404.

**Practical importance:** API endpoints often return raw structured data (JSON) rather than rendered HTML, which can make it easier to spot over-exposed fields (e.g., internal IDs, emails, or other user attributes) that a rendered web page might otherwise hide.

### Step 5 — Manual Application Walkthrough

Pages explored in Firefox: home, jobs listing, login, registration, password reset, user dashboard, profile.

After logging in with the test account, the profile URL is recorded, e.g.:

```
http://127.0.0.1:8088/profile.php?id=2
```

**Purpose:** Manual browsing builds contextual understanding of how the application is structured and surfaces parameters (like `id`) that later become the subject of access-control testing. At this stage, the ID is only *recorded*, not yet manipulated.

### Mission 01 — Checklist

- [ ] HTTP response headers collected
- [ ] Nmap scan completed and saved
- [ ] Gobuster directory discovery completed
- [ ] API index inspected
- [ ] Website pages manually inspected
- [ ] Profile URL and ID parameter recorded

---

## Mission 02 — IDOR and Broken Access Control

**Objective:** Determine whether an authenticated user can access another user's data by manipulating an object identifier (classic IDOR).

### Step 1 — Baseline

Log in as `testuser@recruitx.local`, open the profile page, and record the URL, the user's own ID, displayed name, and visible information. A baseline screenshot is saved as `evidence/idor-normal-user.png`.

**Why this matters:** Without a known-good baseline, it's impossible to demonstrate that subsequent access to *another* user's data is abnormal.

### Step 2 — Authorization Testing on the Profile Page

With the test account still logged in, the profile ID is changed from the user's own value to another seeded ID, e.g.:

```
http://127.0.0.1:8088/profile.php?id=1
```

This is inspected both directly in the browser and via Firefox DevTools (`F12` → Network tab → reload → inspect the request/response).

**What is being tested:** Whether the server independently verifies that the *session's authenticated user* is permitted to view the requested `id`, or whether it simply trusts the `id` parameter as-is once any valid session exists.

### Step 3 — API-Level Verification

```bash
curl -i "http://127.0.0.1:8088/api/user.php?id=1"
curl -i "http://127.0.0.1:8088/api/user.php?id=2"
```

**Purpose:** Confirms (or refutes) the same weakness at the API layer, independent of the rendered HTML page — a flaw may exist in one layer and not the other.

**Evidence capture:**

```bash
curl -i "http://127.0.0.1:8088/api/user.php?id=1" > exploitation/api-user-1.txt
curl -i "http://127.0.0.1:8088/api/user.php?id=2" > exploitation/api-user-2.txt
```

**Analysis questions:**
- Does the endpoint require authentication at all?
- Does it return user information regardless of which account is logged in?
- Does the response change in any way based on who is requesting it?

### Step 4 — Impact

In a production system, a confirmed IDOR on a user/profile endpoint could expose personal information, contact details, account metadata, private documents, or employment-related data belonging to *other* users — a direct privacy and data-protection failure. In this lab, only the data actually observed in the response should be documented, with no assumptions about data that wasn't directly confirmed.

**Expected finding:** Broken Access Control / IDOR (severity dependent on what data is actually returned).

---

## Mission 03 — Password Reset Weaknesses

**Objective:** Assess whether the account-recovery flow protects against account takeover.

### Step 1 — Inspect the Reset Form

`http://127.0.0.1:8088/reset.php` is reviewed for its fields (email, token, new password) and any messages it surfaces.

### Step 2 — Request a Reset

A reset is requested for `testuser@recruitx.local`, and the server's response is observed — specifically, whether the reset token itself is disclosed directly to the requester (e.g., shown on-screen or in the HTTP response) rather than only being sent through a verified out-of-band channel such as email.

### Step 3 — Behavioral Analysis

Key questions documented without automated brute-forcing (to avoid generating excessive load even in a lab):

- Is the token visible to the requester?
- What is the token's length/format (an indicator of entropy/predictability)?
- Does the app distinguish between a valid and invalid token in its messaging?
- Does the token expire?
- Can a token be reused after it's been consumed?
- Is there any indication of rate limiting on reset requests?

**Why this matters:** A token that is disclosed to the requester, long-lived, reusable, and not rate-limited collectively weakens the recovery flow far below the security of the primary login, effectively creating a second, weaker path into any account.

### Step 4 — Validate the Full Flow

Using the legitimately issued token for the test account, a password reset is completed end-to-end, followed by a login attempt with the new password (and a check of whether the old password still works, to confirm the reset actually took effect).

**Expected finding:** Password reset token disclosure and/or insufficient protections on the recovery process, contingent on what is actually observed.

---

## Mission 04 — Administrative File Upload Security

**Objective:** Determine whether uploaded files are validated safely, and whether the server will execute an uploaded file as code.

### Step 1 — Locate the Upload Interface

```
/admin/upload.php
```
Accessed first as the normal user to confirm expected access-denial behavior for a non-administrative account.

### Step 2 — The Core Weakness Being Tested

This lab is intentionally configured so that:
- The server-side validation permits the `.phtml` extension.
- Apache is configured to execute `.phtml` files as PHP.

This represents a classic mismatch between client-side restrictions (easily bypassed), server-side extension allow/deny lists (which must be strict), and the web server's own file-execution configuration (which must not execute anything inside an upload directory).

### Step 3 — Harmless Proof-of-Execution File

```bash
nano proof.phtml
```
```php
<?php
echo "RecruitX PHP execution test";
?>
```

**Design rationale:** This file performs no system interaction — it simply prints a static string. This is a safe way to *prove* server-side code execution occurred without using a web shell or any payload capable of further compromising the host, in line with the lab's safety constraints.

### Step 4 — Observing Upload and Execution Behavior

Through the documented administrative upload interface, the test file is submitted and the following is recorded:
- Whether the file is accepted.
- The storage location/returned path (if disclosed).
- Whether navigating to the uploaded file's URL causes the PHP code to execute (i.e., the string is printed) rather than the raw source being displayed or downloaded.

**Interpretation:** If the message is printed, this is direct proof the upload directory permits server-side script execution — a critical-severity condition in a real deployment, since it is the foundation for remote code execution via malicious uploads.

### Step 5 — Impact

Severity depends on reachability (can an attacker get to the upload feature at all), whether the server executes the uploaded content, what privileges the web server process runs with, and whether the upload directory is publicly web-accessible.

**Expected finding:** Unsafe file upload validation, with potential for server-side code execution.

---

## Mission 05 — Application and API Security Review

**Objective:** Consolidate findings around API exposure, session handling, and HTTP security configuration.

### Step 1 — Jobs API Review

```bash
curl -i http://127.0.0.1:8088/api/jobs.php
```
Evaluated for whether it requires authentication, and whether it returns only intended public job-listing data or anything more sensitive. A publicly accessible API is not inherently a flaw — a job board's listings are expected to be public; the assessment is whether the *data returned* matches the *intended* access level.

### Step 2 — Session and Authentication Behavior

Checked manually:
- Does logging out actually invalidate the session (i.e., does the dashboard become inaccessible afterward)?
- Does the app correctly redirect unauthenticated requests away from protected pages?
- Does any response leak session internals?

**Cookie inspection (Firefox DevTools):** `F12` → Storage → Cookies → `127.0.0.1`, checking for the `HttpOnly`, `Secure`, and `SameSite` flags.

| Flag | Purpose |
|---|---|
| `HttpOnly` | Prevents client-side JavaScript from reading the cookie, mitigating session theft via XSS |
| `Secure` | Ensures the cookie is only sent over HTTPS |
| `SameSite` | Restricts whether the cookie is sent on cross-site requests, mitigating CSRF |

**Note:** A missing `Secure` flag is expected and lower-risk on a plain-HTTP localhost lab, but it is still documented as a configuration observation relevant to how the application would need to be hardened before any production/HTTPS deployment.

### Step 3 — Security Header Review

```bash
curl -I http://127.0.0.1:8088
```

Checked for the presence of:

| Header | Protects against |
|---|---|
| `Content-Security-Policy` | Cross-site scripting and unauthorized resource loading |
| `X-Content-Type-Options` | MIME-type sniffing attacks |
| `X-Frame-Options` | Clickjacking via iframe embedding |
| `Strict-Transport-Security` | Protocol downgrade / SSL-stripping attacks (HTTPS contexts) |
| `Referrer-Policy` | Leakage of sensitive URL data via the `Referer` header |

**Important distinction:** Missing headers are documented as configuration weaknesses that *increase* risk and should be remediated, but they are not treated as proof of an independently exploitable vulnerability on their own.

---

## Reporting

A structured report (`reports/RecruitX-Pentest-Report.md`) is produced using a standard penetration-testing report template, covering:

1. **Executive Summary** — plain-language overview for non-technical stakeholders.
2. **Scope** — target, environment, and assessment type.
3. **Methodology** — the ordered phases followed (recon → auth/access-control testing → API testing → password reset testing → upload testing → configuration review → evidence/reporting).
4. **Findings** — one structured entry per vulnerability, each with:
   - **Severity** (assigned based on demonstrated impact, not assumed worst-case)
   - **Description** of the confirmed behavior
   - **Evidence** (requests, responses, screenshots — secrets redacted)
   - **Impact** in business/user terms
   - **Recommendation** for remediation
5. **Conclusion** — summary of confirmed findings and remediation priorities.
6. **Limitations** — explicit statement that results apply only to the local training lab, not any production system.

### Evidence Folder Structure

```
RecruitX-Assessment/
│
├── recon/
│   ├── notes.txt
│   ├── http-headers.txt
│   ├── nmap-scan.txt
│   ├── gobuster.txt
│   └── api-discovery.json
│
├── exploitation/
│   ├── api-user-1.txt
│   └── api-user-2.txt
│
├── evidence/
│   ├── idor-normal-user.png
│   └── other-screenshots.png
│
└── reports/
    └── RecruitX-Pentest-Report.md
```

---

## Remediation Summary

| Vulnerability | Primary Remediation |
|---|---|
| IDOR / Broken Access Control | Enforce server-side ownership/authorization checks for every object request, derived from the authenticated session — never trust a client-supplied ID alone |
| API Information Disclosure | Require authentication where appropriate; enforce object-level authorization; return only the minimum necessary fields |
| Weak Password Reset | Use cryptographically secure random tokens; store only token hashes; enforce short expiration; make tokens single-use; apply rate limiting; never return the token to the requester |
| Unsafe File Upload | Enforce a strict server-side allowlist; validate actual file content/MIME type (not just extension); generate randomized filenames; store uploads outside the web root; disable script execution in upload directories |
| Missing Security Headers | Configure `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`, and `Referrer-Policy` appropriately for the deployment context |

**Verification principle:** A finding is not considered resolved until its fix can be independently verified — for example, after fixing IDOR, a standard user requesting another user's private profile should receive an authorization error (e.g., HTTP 403), not the data.

---

## Real-World Applications

The vulnerability classes exercised in this lab are consistently among the most frequently reported in real-world web application assessments and bug bounty programs:

- **IDOR/Broken Access Control** maps directly to OWASP Top 10 category A01:2021 and is one of the most commonly exploited flaws in production applications, particularly in platforms with user-specific records (profiles, invoices, documents, HR/recruitment data).
- **API information disclosure** is increasingly significant as applications shift to API-first architectures, where the API often has weaker testing coverage than the primary UI.
- **Password reset flaws** are a common account-takeover vector because they bypass the primary authentication mechanism entirely.
- **Unrestricted file upload** remains a leading cause of full server compromise when upload directories allow script execution.

## Skills Demonstrated

- Structured reconnaissance methodology (headers, service fingerprinting, content discovery, API enumeration).
- Manual and tool-assisted identification of access-control flaws.
- Safe, non-destructive proof-of-concept techniques (harmless proof-of-execution rather than web shells).
- Evidence-handling discipline appropriate for a professional engagement.
- Translating technical findings into business-relevant, remediation-focused reporting.

## Conclusion

This lab walks through a complete, compressed version of a real web application penetration test: from initial reconnaissance of an unknown target, through methodical testing of common high-impact vulnerability classes, to a professional report suitable for a development team to act on. The emphasis throughout is on evidence-based conclusions — distinguishing confirmed, demonstrated vulnerabilities from configuration observations that merely increase risk — which mirrors the standard expected of real-world penetration testing engagements.

## Limitations

This documentation describes testing performed exclusively against a local, intentionally vulnerable training application (`127.0.0.1:8088`) running in Docker on an isolated Kali Linux VM. It does not describe, and should not be interpreted as, an assessment of any production system, third-party service, or publicly accessible website. The techniques described here must only be applied to systems the reader owns or is explicitly authorized in writing to test.
