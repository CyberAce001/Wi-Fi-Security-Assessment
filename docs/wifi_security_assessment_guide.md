# Wi-Fi Security Assessment — Methodology Guide

Step-by-step methodology used to assess a personally owned MTN MiFi device from a Kali Linux VM. This is the practical guide behind the formal Assessment Report in `reports/`.

## Phase 0 — Lab setup

1. Install Kali Linux as a VM under VMware Workstation on the Windows host (VMware was already installed, so it was used in preference to VirtualBox rather than running two hypervisors side by side).
2. Set the VM's network adapter to **Bridged** (not NAT) so it receives its own IP address directly on the target Wi-Fi subnet, appearing as a distinct peer device to the MiFi. NAT was considered and rejected: under NAT the VM would sit behind VMware's own internal subnet and share the host's single address, never appearing as its own reachable device on the target network.
3. Update Kali and all testing tools, then take a **clean VMware snapshot** before any target-facing testing begins. This snapshot is the reproducible baseline and safe-lab evidence that testing happened from a controlled, disposable environment.

## Phase 1 — Reconnaissance

Capture, from the MiFi's own admin interface (once briefly authenticated): SSID, encryption type (expect WPA2-PSK or similar), channel, connected-client count, and device model/firmware string. Note the connected-client count carefully — see the methodology note on bridged-VM Wi-Fi association below.

## Phase 2 — Network scanning (Nmap)

```
sudo nmap -sV -sC 192.168.0.1          # service/version + default scripts
sudo nmap --script vuln 192.168.0.1    # vulnerability-scan scripts
sudo nmap -sV -sC -p- 192.168.0.1      # full 65,535-port sweep, for completeness
```

Also scan the host machine and the Kali VM itself, for completeness — not because they're in scope as targets, but because anything they expose surfaced from the same testing activity and is worth recording as an environment observation.

Triage every `--script vuln` hit manually rather than accepting scanner output at face value. In this assessment, several automated flags (a JBoss JMX-console check, Drupal/ASP.NET/WordPress checks) were confirmed as false positives or not-applicable, since the target device doesn't run any of those stacks — verified with direct `curl` comparisons rather than assumed.

## Phase 3 — Web application testing (OWASP ZAP)

1. Add the MiFi's admin interface as a Site in ZAP, then run the **Spider** (Tools panel) to crawl available pages.
2. In the Spider dialog's **Advanced** tab, review Max Duration, Process Forms, and POST Forms carefully — leaving Process Forms/POST Forms enabled while crawling near a login page will auto-submit the form with dummy values and can trigger an account lockout (this happened during this assessment and became a positive-finding writeup once diagnosed).
3. Check the **Alerts** tab after crawling — ZAP's passive scanner flags issues like outdated JavaScript libraries (via its Retire.js-based check), missing security headers, and similar, without needing an active scan.
4. Manually review login-related requests in the **History** tab for authentication mechanics, cookie flags, and CSRF protection.

## Phase 4 — Traffic analysis (Wireshark)

1. Start a capture on the interface used to reach the MiFi, then log in through the browser as normal.
2. Filter for the login submission: `http.request.method == "POST"`.
3. Expand the HTML Form fields in the packet detail pane and manually inspect each value — don't assume "no HTTPS" automatically means "raw plaintext credentials." In this assessment, the username field was base64-encoded (trivially reversible) but the password field was a hash that varied on every login attempt, indicating client-side hashing with a nonce — a partial mitigation worth stating precisely rather than overclaiming.
4. Filter `tls` separately to confirm (or rule out) any HTTPS traffic to the admin panel's own IP specifically — general internet traffic on the network will show TLS; the admin panel itself should not, if the "no HTTPS" finding is accurate.

## Phase 5 — Device-manager review

Log into the admin interface directly and review every settings page, not just the landing page — this assessment's WPS finding, MAC-filtering option, and several positive-observation items (SSID rename, data-usage alerting) were only visible after navigating into Device Settings.

## Phase 6 — Findings consolidation & risk rating

Rate each finding Critical/High/Medium/Low based on likelihood and impact for a personal, non-enterprise device, then write up recommendations. Separate confirmed findings from scanner-flagged-but-unconfirmed ones (e.g. a Slowloris DoS heuristic flag that was deliberately not exploited against the assessor's only device) and record discounted automated findings in a triage note rather than silently dropping them — this documents the rigor of the manual review.

## Notes & limitations worth carrying into the report

- **Bridged-VM wireless association**: a VM bridged over Wi-Fi (rather than Ethernet) on Windows typically shares the host's single 802.11 association even though it has its own MAC and IP at the network layer — the AP's own connected-device count will undercount by one. This doesn't affect Nmap/ZAP/Wireshark results, which operate above the wireless association layer.
- **ZAP proxy overhead**: browsing through ZAP's proxy measurably slows responsiveness, especially from a VM — account for this when timing scans.
