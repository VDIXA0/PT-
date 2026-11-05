# Passive Reconnaissance — Technical Lab Report

## 1. Executive Summary

This project demonstrates practical passive reconnaissance techniques using Kali Linux.

The assessment focuses on gathering publicly available domain registration information and DNS records using WHOIS and DIG.

The objective is to understand how reconnaissance helps penetration testers identify and analyze publicly visible domain infrastructure before an authorized security assessment.

---

## 2. Target Information

| Field | Details |
| --- | --- |
| Project Name | Passive Recon |
| Target Domain | example.com |
| Operating System | Kali Linux |
| Tools | WHOIS, DIG, NSLOOKUP |
| Assessment Type | Domain Information Gathering |
| Scope | Public registration and DNS information |

> `example.com` is a domain reserved for examples and documentation. Its results do not represent any real company's infrastructure.

---

## 3. Environment Preparation

The required utilities were installed using APT:

```bash
sudo apt update
sudo apt install dnsutils whois -y
dig -v
whois --version
```

- Updated package repositories.
- Installed DNS utilities (`dig`, `nslookup`).
- Installed WHOIS.
- Verified tool availability.

---

## 4. WHOIS Enumeration

**Command:** `whois example.com`

**Objective:** Retrieve publicly available domain registration information.

**Information investigated:**

- Registrar
- Registration dates (creation, update, expiration)
- Domain status
- Name servers
- Available registrant information (often redacted for privacy)

**Evidence:** `Evidence/whois.txt`

**Findings:**

| Item | Value |
| --- | --- |
| Registrar | *(add from output)* |
| Creation Date | *(add from output)* |
| Updated Date | *(add from output)* |
| Expiration Date | *(add from output)* |
| Name Servers | *(add from output)* |
| Domain Status | *(add from output)* |

---

## 5. DNS Enumeration

### 5.1 A Record

**Command:** `dig example.com A`

**Objective:** Identify IPv4 addresses associated with the domain.

**Evidence:** `Evidence/dns-a.txt`

**Findings:** *(add the actual IPv4 response)*

### 5.2 AAAA Record

**Command:** `dig example.com AAAA`

**Objective:** Identify IPv6 addresses associated with the domain.

**Evidence:** `Evidence/dns-aaaa.txt`

**Findings:** *(add the actual response)*

### 5.3 MX Record

**Command:** `dig example.com MX`

**Objective:** Identify the domain's mail servers and their priorities. A lower priority number means higher preference.

**Evidence:** `Evidence/dns-mx.txt`

**Findings:** *(add the actual mail server records; if none or a "null MX" is returned, note that the domain does not receive email)*

### 5.4 NS Record

**Command:** `dig example.com NS`

**Objective:** Identify the authoritative name servers.

**Evidence:** `Evidence/dns-ns.txt`

**Findings:** *(add the actual name server records)*

### 5.5 TXT Record

**Command:** `dig example.com TXT`

**Objective:** Retrieve publicly available text records, including email security policies (such as SPF) and domain verification information when published.

**Evidence:** `Evidence/dns-txt.txt`

**Findings:** *(add the actual TXT records)*

### 5.6 SOA Record

**Command:** `dig example.com SOA`

**Objective:** Retrieve DNS zone authority and administrative timing information (primary server, serial number, refresh/retry/expire values).

**Evidence:** `Evidence/dns-soa.txt`

**Findings:** *(add the actual SOA response)*

---

## 6. Additional Investigation

Additional commands were used to compare DNS query methods:

| Command | Purpose |
| --- | --- |
| `dig @1.1.1.1 example.com A` | Query Cloudflare's resolver instead of the system default, to compare answers |
| `nslookup example.com` | Compare `nslookup` output with `dig` |
| `dig example.com ANY` | Request all record types; many servers return limited or no useful data for ANY |

These commands helped develop familiarity with DNS resolvers and different query methods.

---

## 7. Security Relevance

The information collected during reconnaissance can help identify:

- Publicly visible domain infrastructure.
- Mail server configurations.
- Authoritative DNS providers.
- Published email security policies.
- Additional areas that may be relevant to a future authorized assessment.

DNS information alone does not prove that a server is online, that a service is accessible, or that a vulnerability exists.

---

## 8. Limitations

- The assessment focused on publicly available registration and DNS information.
- No exploitation was performed.
- No vulnerability scanning was performed.
- No intrusive port scanning was performed.
- DNS responses may vary depending on the resolver and query time.
- `ANY` queries are not a reliable way to enumerate all records.

---

## 9. Conclusion

This practical lab provided experience with WHOIS and DNS enumeration using Kali Linux.

The activities improved understanding of domain registration data, DNS record types, mail infrastructure, name servers, and the role of information gathering in penetration testing.

**Next step:** subdomain enumeration on an authorized, in-scope domain.

---

## 10. Evidence

All command outputs are stored in the `Evidence/` directory:

| File | Content |
| --- | --- |
| `whois.txt` | WHOIS lookup |
| `dns-a.txt` | A records |
| `dns-aaaa.txt` | AAAA records |
| `dns-mx.txt` | MX records |
| `dns-ns.txt` | NS records |
| `dns-txt.txt` | TXT records |
| `dns-soa.txt` | SOA record |

Screenshots of the practical activities are stored in the `Screenshots/` directory.

---

## 11. Skills Practiced

| Skill | What was practiced |
| --- | --- |
| WHOIS Enumeration | Domain registration research |
| DNS Enumeration | Identifying DNS records |
| Infrastructure Recon | Understanding domain infrastructure |
| Linux CLI | Using Kali terminal tools |
| Evidence Collection | Saving command outputs |
| Technical Reporting | Documenting findings |
