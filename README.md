# Wi-Fi Security Assessment

Lab-based Wi-Fi and network security assessment of a personally owned MTN MiFi device (MTN Broadband 4G MiFi ZLT M30S), performed from an isolated Kali Linux VM as a cybersecurity coursework project.

**Prepared by:** Awe Oluwasanmi Mayokun
**Classification:** Personal / Educational Lab Assessment

---

## ⚠️ Scope & Authorization

All testing in this project was performed **only** against hardware owned by the assessor — a personal MTN MiFi, wireless-only, no USB/Ethernet port. No third-party, production, or otherwise unauthorized networks were tested at any point. This repository is published for educational/portfolio purposes; it documents methodology and findings, not a live-attack toolkit.

If you're reading this because you also own consumer Wi-Fi hardware: only ever test devices you own or have explicit written permission to test.

---

## Table of Contents

- [Overview](#overview)
- [Objective](#objective)
- [Test Environment & Tools](#test-environment--tools)
- [Methodology](#methodology)
- [Key Findings](#key-findings)
- [Positive Security Observations](#positive-security-observations)
- [Remediation Summary](#remediation-summary)
- [Repository Structure](#repository-structure)
- [Viewing the Report](#viewing-the-report)
- [References](#references)
- [About Me](#about-me)
- [License](#license)

---

## Overview

This project combines network scanning (Nmap), web application testing (OWASP ZAP), and traffic analysis (Wireshark) against a personal MiFi's admin interface and network services, following a scope limited strictly to devices owned by the assessor. The deliverables — assessment report, safe-lab evidence, remediation recommendations, and a short walkthrough video — were produced per the CS-12 project brief: *"Wi-Fi Security Assessment."*

## Objective

Assess the Wi-Fi and network security posture of a personally owned MTN MiFi device in a controlled lab environment, and produce an assessment report, remediation/hardening recommendations, and a short walkthrough video.

## Test Environment & Tools

- **Kali Linux**, run as a VM under **VMware Workstation** on a Windows host, network adapter set to **Bridged** (not NAT) so the VM receives its own IP address directly on the target Wi-Fi subnet — a deliberate methodology choice, not the default left in place.
- **Nmap** — host discovery, service/version scanning (`-sV -sC`), vulnerability scanning (`--script vuln`), and a full 65,535-port sweep (`-p-`).
- **OWASP ZAP** (Community Edition) — Spider crawling, passive scanning (including its Retire.js-based outdated-JavaScript-library check), and manual analysis of the admin interface.
- **Wireshark** — traffic capture and manual packet-level inspection of the login flow.
- A clean **VMware snapshot**, taken before any target-facing testing began, as the reproducible baseline and safe-lab evidence.

## Methodology

Full step-by-step methodology is in [`docs/wifi_security_assessment_guide.md`](docs/wifi_security_assessment_guide.md). In brief: reconnaissance → Nmap scanning → ZAP web testing → Wireshark traffic analysis → device-manager review → findings consolidation and risk rating. Every automated scanner flag was manually triaged rather than accepted at face value — several Nmap `--script vuln` hits were confirmed as false positives (e.g. a JBoss-specific check against a device that doesn't run JBoss) via direct `curl` verification, and the ZAP/Wireshark findings below were confirmed through manual packet inspection rather than assumed from the absence of HTTPS alone.

A working findings log kept during testing is in [`docs/findings_log.md`](docs/findings_log.md).

## Key Findings

| # | Finding | Risk | Recommendation |
|---|---|---|---|
| 1 | Default administrative credentials work | **Critical** | Change the default admin password immediately |
| 2 | WPS enabled | **High** | Disable WPS in MiFi settings |
| 3 | Outdated/vulnerable JavaScript library — Underscore.js 1.3.1 (CWE-1395) | **High** | Update the bundled library to a current, patched release |
| 4 | dnsmasq 2.80, within the DNSpooq-affected range (CVE-2020-25681–25687) | **Medium–High** | Update firmware if a fix ever ships; not currently possible on this model |
| 5 | Admin interface served over HTTP only; no HTTPS or firmware upgrade path | **Medium** | Vendor-confirmed no HTTPS path; apply compensating controls |
| 6 | Unauthenticated status page discloses connected devices and battery level | **Low–Medium** | Accept as a platform limitation; keep device off untrusted networks |
| 7 | Slowloris DoS heuristic flag (CVE-2007-6750) | **Medium (heuristic, unconfirmed)** | Monitor; do not actively DoS-test a single production device |

Manual Wireshark analysis of the login flow (behind Finding 5) found the admin password is hashed client-side with a value that varies per login attempt — not sent as raw plaintext, but not a substitute for the missing TLS either, since the username and all other session traffic on the interface remain unencrypted. Full detail, evidence screenshots, and risk narrative for every finding are in the formal report.

## Positive Security Observations

Not every result was a weakness — the following were already correctly configured or were hardened during testing:

- WPA2-PSK (AES) encryption in use.
- UPnP disabled.
- Account lockout enforced on the admin login after repeated failed attempts, tracked per source IP with a time-based reset.
- MAC address blacklisting available.
- SSID changed from the manufacturer's default to a non-identifying name.
- Data usage cap re-enabled with an SMS alert at 50% usage.
- Connected-device list reviewed and pruned.
- No DOM-based XSS found on the admin interface (tested and clean).
- The Kali VM testing platform itself exposed no listening network services.
- Login password hashed client-side with a per-attempt varying value, rather than transmitted as raw plaintext.

## Remediation Summary

1. **Immediate:** change the default admin password; disable WPS; update or replace the vulnerable Underscore.js component.
2. **Compensating controls** for the unfixable HTTP-only admin interface: keep administration to the local Wi-Fi network only, never administer from an untrusted network, use a password manager.
3. **Monitor rather than patch** the dnsmasq/DNSpooq exposure and the Slowloris heuristic flag — no firmware fix currently exists.
4. **Longer term:** consider a hardware upgrade to a model with an actively maintained, TLS-capable firmware branch.

Full prioritized action list is in the report's Remediation & Hardening Recommendations section.

## Repository Structure

```
.
├── README.md
├── reports/
│   └── Wi-Fi_Security_Assessment_Report_CS-12.pdf   # full formal report (primary deliverable)
├── diagrams/
│   ├── lab_topology_diagram.png                     # lab network topology (IPs generalized)
│   ├── findings_severity_chart.png                  # findings-by-severity chart
│   └── summary_infographic.png                      # one-page summary infographic
└── docs/
    ├── wifi_security_assessment_guide.md             # step-by-step methodology guide
    └── findings_log.md                                # working findings/observations log
```

## Viewing the Report

The full report — Executive Summary, Scope & Methodology, Lab Environment, all seven findings with evidence and risk ratings, Positive Security Observations, Remediation, Conclusion, and References — is in [`reports/`](reports/). GitHub previews PDF files directly in the browser; no download needed to read it.

## References

Full citation list (CVEs, the DNSpooq disclosure, the Pixie Dust research, Retire.js, and tooling documentation) is in the report's Appendix C.

## About Me

**Awe Oluwasanmi Mayokun** — 3MTT cybersecurity fellow. This project was completed as part of coursework and is published here as a portfolio piece documenting hands-on methodology.

- GitHub: https://github.com/CyberAce001
- LinkedIn: https://www.linkedin.com/in/oluwasanmiawe/

## License

This repository is licensed under the [MIT License](LICENSE) — see the LICENSE file for the full text.
