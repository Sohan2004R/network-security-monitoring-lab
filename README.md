# Network Security Monitoring Lab

A virtual lab environment built to simulate common network attacks, detect them using **Suricata IDS**, write custom detection rules, and mitigate attacks using **fail2ban**. Built as a hands-on extension of the Cisco Networking Academy *Network Support and Security* course.

## Objective

To build a small, realistic network security monitoring setup that demonstrates the full attack lifecycle:

**Attack → Detection (IDS) → Custom Rule Writing → Mitigation → Verification**

## Lab Topology

```
   Kali Linux (Attacker)          Suricata Sensor
     10.10.10.10                    10.10.10.30
          \                              /
           \                            /
            \==== Internal Network =====/
              (VirtualBox "labnet")
                        |
                        |
                Ubuntu Server (Victim)
                   10.10.10.20
               Services: SSH, Apache
```

- All three VMs run on an isolated VirtualBox **Internal Network** (no internet access between them, NAT used only for package installation).
- The sensor's network adapter is set to **Promiscuous Mode: Allow All** so it can observe traffic between the attacker and victim.

## Tools Used

| Tool | Purpose |
|---|---|
| VirtualBox | Virtualization platform |
| Kali Linux | Attacker machine |
| Ubuntu Server 26.04 LTS | Victim and Sensor machines |
| Suricata | Network Intrusion Detection System (IDS) |
| Nmap | Network/port scanning |
| Hydra | SSH brute-force tool |
| Nikto | Web vulnerability scanner |
| fail2ban | Intrusion prevention / automated IP banning |
| Wireshark | Packet inspection (supporting tool) |

## Environment Setup

1. Three VMs created in VirtualBox: `kali` (attacker), `victim` (Ubuntu Server with SSH + Apache), `sensor` (Ubuntu Server with Suricata).
2. All three connected via an Internal Network (`labnet`), with static IPs assigned via Netplan (victim, sensor) and NetworkManager (Kali).
3. Suricata installed on the sensor, configured to monitor the lab-facing interface, with Emerging Threats Open ruleset (69,000+ rules) loaded.
4. Victim configured with OpenSSH server and Apache2, confirmed reachable from Kali.

## Attack Scenarios

### 1. Network Reconnaissance — Nmap Scan

**Command (from Kali):**
```bash
nmap -sS -A 10.10.10.20
```

**Result:** Suricata's default Emerging Threats ruleset detected the scan immediately, generating 25 alerts:
```
[**] [1:2024364:5] ET SCAN Possible Nmap User-Agent Observed [**]
[Classification: Web Application Attack] {TCP} 10.10.10.10 -> 10.10.10.20:80
```

This confirmed the IDS could detect reconnaissance activity out-of-the-box using community-maintained signatures.

*(Screenshots: `screenshots/` — Nmap scan output and corresponding Suricata alerts)*

---

### 2. SSH Brute Force Attack — Hydra

**Command (from Kali):**
```bash
hydra -l victim -P pass.txt ssh://10.10.10.20
```

A small wordlist of common passwords was used to simulate a credential-guessing attack against the victim's SSH service.

**Finding:** The default Suricata ruleset did **not** generate an alert for this attack — SSH brute-forcing isn't flagged by the generic Emerging Threats Open rules unless scanning behavior is also present. This revealed a real gap in out-of-the-box detection coverage.

**Response:** A custom Suricata rule was written to close this gap:

```
alert tcp any any -> 10.10.10.20 22 (msg:"Custom Possible SSH Brute Force"; flow:to_server; flags:S; threshold:type threshold, track by_src, count 5, seconds 60; sid:1000001; rev:1;)
```

This rule triggers when more than 5 new TCP connections are made to port 22 from the same source IP within 60 seconds — a strong indicator of brute-force behavior.

After reloading Suricata with the new rule and re-running the attack, the alert fired correctly:
```
[**] [1:1000001:1] Custom Possible SSH Brute Force [**] {TCP} 10.10.10.10 -> 10.10.10.20:22
```

The victim's own `/var/log/auth.log` also recorded the failed login attempts and OpenSSH's built-in `srclimit_penalise` rate-limiting kicking in independently.

*(Screenshots: `screenshots/` — Hydra attack, auth.log entries, custom rule alert)*

---

### 3. Web Vulnerability Scan — Nikto

**Command (from Kali):**
```bash
nikto -h http://10.10.10.20
```

Nikto scanned the victim's Apache server and reported 9 findings, including missing security headers (`Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`) and an outdated Apache version notice.

**Finding:** Like the brute-force attack, this scan was **not** flagged by the default Suricata ruleset — Nikto's HTTP request pattern closely resembles normal browser traffic, making it harder for signature-based detection to catch without a specific rule.

**Response:** A second custom rule was added, matching on Nikto's HTTP User-Agent string:

```
alert http any any -> 10.10.10.20 80 (msg:"Custom Possible Nikto Web Scan"; http.user_agent; content:"Nikto"; sid:1000002; rev:1;)
```

*(Screenshots: `screenshots/` — Nikto scan results)*

---

## Mitigation — fail2ban

To move from detection to active prevention, **fail2ban** was installed on the victim machine to automatically ban IPs showing repeated failed SSH login attempts.

**Configuration (`/etc/fail2ban/jail.local`):**
```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 4
bantime = 300
findtime = 60
```

**Verification:**

1. Hydra brute-force attack re-run against the victim.
2. After 4 failed attempts, fail2ban automatically banned the attacker's IP:
   ```
   Status for the jail: sshd
   Currently banned: 1
   Banned IP list: 10.10.10.10
   ```
3. Confirmed the ban was effective — a direct SSH connection attempt from Kali was refused:
   ```
   ssh victim@10.10.10.20
   ssh: connect to host 10.10.10.20 port 22: Connection refused
   ```

This demonstrated a complete attack lifecycle: **attack attempted → detected → mitigated → block verified.**

*(Screenshots: `screenshots/` — fail2ban status showing banned IP, refused connection)*

## Repository Structure

```
network-security-monitoring-lab/
├── README.md
├── rules/
│   └── local.rules          # Custom Suricata detection rules
├── screenshots/             # Evidence for each attack/detection/mitigation stage
└── docs/
    └── topology.png         # Network topology diagram
```

## Lessons Learned

- Default/community IDS rulesets catch obvious reconnaissance (e.g. Nmap scans) very well, but miss slower or more "normal-looking" attacks like credential brute-forcing and some vulnerability scans — custom rules are essential for full coverage.
- Writing effective detection rules requires understanding both the attack's network behavior (e.g. connection rate, protocol flags) and Suricata's rule syntax (`threshold`, `flow`, `content` matching).
- Detection alone isn't enough — pairing IDS alerts with an active response mechanism (fail2ban) turns visibility into real protection.
- Host-level logs (`auth.log`) and OpenSSH's own rate-limiting provided a useful secondary layer of evidence alongside the IDS.

## Future Improvements

- Add ARP spoofing detection and simulation.
- Forward Suricata alerts to an ELK/Kibana stack for centralized log visualization.
- Automate the lab setup using Vagrant or Ansible for reproducibility.
- Add a Python script to summarize and visualize alert counts from `fast.log`.

---

**Author:** Chanupa Sohan Rathnayake
Built as a self-directed project following the Cisco Networking Academy *Network Support and Security* course.
