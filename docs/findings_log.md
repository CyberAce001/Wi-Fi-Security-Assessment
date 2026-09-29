# Findings & Observations Log — Wi-Fi Security Assessment

Running log kept during testing, folded into the final Assessment Report (see `reports/`).

| ID | Phase | Type | Description | Evidence | Severity |
|---|---|---|---|---|---|
| FIND-001 | Recon / Phase 4 | Finding | MTN MiFi admin/status landing page discloses connected-devices list and battery level **without authentication**. | Screenshot of the page | Low–Medium |
| FIND-002 | Phase 3 (Nmap network scan) | Finding | MiFi runs **dnsmasq 2.80** on port 53. All dnsmasq versions before 2.83 are affected by the **DNSpooq** vulnerability set (CVE-2020-25681 through CVE-2020-25687). 2.80 falls in the affected range. | Nmap `-sV` banner output | Medium–High |
| FIND-003 | Phase 3 (Nmap `--script vuln`) | Finding (heuristic, not confirmed) | `http-slowloris-check` flags the MiFi's web server as **"LIKELY VULNERABLE"** to Slowloris DoS (CVE-2007-6750). | Nmap NSE output | Medium (heuristic) |
| FIND-004 | Phase 4 (ZAP / manual login) | Finding — **positive control** | MiFi admin login enforces an **account lockout / attempt limit** after repeated failed logins. Tracked independently per source IP, resets on a ~20+ minute timer rather than a device reboot. | Screenshot of the lockout alert | N/A — positive finding |
| FIND-005 | Phase 4 (ZAP passive scan) | Finding | ZAP's Retire.js check flagged the admin interface as loading **Underscore.js 1.3.1** — roughly a decade out of date, within the range affected by CVE-2021-23358 (template-injection / possible XSS). | ZAP Alerts tab (High risk, CWE-1395) | High |
| FIND-006 | Phase 4 (manual Wireshark analysis) | Finding | Login POST request: username field is **base64-encoded** (trivially reversible); password field is a **hashed value that varies per login attempt**, indicating a nonce/challenge is mixed in before hashing client-side. Partial mitigation — not a substitute for the missing HTTPS. | Wireshark capture, `http.request.method == "POST"` filter | Medium (contributes to Finding on HTTP-only interface) |
| OPEN-001 | Phase 3 (Nmap `--script vuln`) | **Resolved — confirmed FALSE POSITIVE** | `http-vuln-cve2010-0738` flagged `/jmx-console/` as accessible without authentication. CVE-2010-0738 is specific to **JBoss Application Server**. Manually verified via `curl` that the device returns an identical "soft 200" response for both the tested path and a nonexistent control path — the device has no real JMX console. | `curl -i` comparison, identical output | Not exploitable — false positive |
| OPEN-002 | Phase 3 (Nmap `--script vuln`) | Not applicable / inconclusive | `http-vuln-cve2014-3704` (Drupal), `http-aspnet-debug` (ASP.NET), `http-vuln-cve2017-1001000` (WordPress) all returned `ERROR: Script execution failed`. These NSE scripts target technology stacks this embedded device does not run. | N/A | N/A |
| OPEN-003 | Phase 3 (Nmap scan) | Resolved via full-port rescan | Port 80 fingerprinted as service `Demo-Webs` — reconfirmed via a full 65,535-port sweep (`-p-`) and the `http-server-header`/`http-title` NSE checks as the device's actual literal banner string, not a misidentified product. | Nmap full-port scan output | Informational |
| OPEN-004 | Phase 3 (Nmap host-machine scan) | Environment hygiene note (not a MiFi finding) | Host machine shows ports 902 (`iss-realsecure`) and 912 (`apex-mesh`) reachable over the bridged Wi-Fi interface — these are actually VMware Workstation's own `vmware-authd`/hostd ports, and 5357 (`wsdapi`) is standard Windows network-discovery. Worth firewalling to localhost. | Nmap host scan output | Informational / hardening suggestion for the host machine |
| OBS-001 | Phase 1 (Lab Setup) | Methodology observation, **resolved** | Host and Kali VM have distinct MACs and IPs at the network layer, but the MiFi's connected-devices count shows only **1** client. Confirmed as a known characteristic of bridging a VM over **Wi-Fi** (rather than Ethernet) on Windows — 802.11 association happens once, at the host's physical adapter. Does not affect Nmap/ZAP/Wireshark validity. | Screenshot of MiFi client-count page | N/A — environment/methodology note |

---

## Note on OBS-001: why only one device shows, not two

Ethernet bridging is straightforward: a switch forwards frames by MAC address regardless of how many MACs appear on one cable. Wi-Fi is different — under 802.11 (especially WPA2-PSK), each MAC address must independently associate and complete its own 4-way handshake with the access point. A bridged VM's virtual NIC has its own MAC (VMware assigns something like `00:0C:29:xx:xx:xx`), but on Windows that MAC typically never performs its own over-the-air association — only the host's physical adapter does. VMware handles the VM's separate IP internally below the wireless layer, so the MiFi's own "connected devices" counter undercounts by one relative to the true number of network-layer endpoints on the segment.

Report-ready methodology line:

> "Testing was conducted from a Kali Linux VM bridged over the host machine's Wi-Fi adapter. The VM and host were assigned distinct MAC addresses and IP leases at the network layer, but due to how 802.11 wireless association works, they shared a single Wi-Fi association at the radio layer — the target AP's own connected-device count therefore undercounts actual network endpoints by one. This does not affect the validity of Nmap, OWASP ZAP, or Wireshark results, which operate above the wireless association layer."

## FIND-004 follow-up: recovering from the lockout, and avoiding it next time

**To get back in:** power-cycle the MiFi (the failed-attempt counter is typically memory-only). If it doesn't reset on reboot, the lockout is time-based instead — wait 10–15 minutes.

**Before running Spider again near the login page**, turn off "Process Forms" and "POST Forms" in the Advanced tab, or scope the Spider away from the login endpoint entirely. Otherwise every crawl re-triggers the lockout by auto-submitting the login form with dummy values.

## Root cause of the lockout, confirmed

Running the ZAP Spider with "Process Forms" and "POST Forms" enabled caused it to auto-submit the admin login form with dummy values while crawling, exhausting the login-attempt counter from the Kali VM. The lockout is tracked independently per source IP — the host machine's browser (a different IP on the bridged subnet) could log in successfully at the same time the VM remained locked out — and resets on a timer of roughly 20+ minutes rather than requiring a device reboot.

## Other checks done manually (not covered by the default `--script vuln` run)

- **Default/weak admin credentials** on the web login — confirmed working (Finding 1, Critical).
- **Outdated JavaScript library** — confirmed via ZAP's passive Retire.js check (Finding, High).
- **Login credential transport** — manually verified via Wireshark packet inspection rather than assumed; found to be base64-encoded (username) and hashed-with-nonce (password), not raw cleartext.
- **WPS status** — confirmed enabled via the device manager.
- **Firmware/HTTPS availability** — escalated to MTN customer care directly; confirmed no HTTPS path and no firmware upgrade exists for this model.
