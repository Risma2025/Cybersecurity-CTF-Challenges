# HTB-Sherlock-Telly
# Forensic-Investigation:CVE-2026-24061-GNU-InetUtils-telnetd-Authentication-Bypass-Vulnerability-Exploitation

## Case Study: HTB Sherlock: Telly

## 📌 Overview
This repository contains a technical investigation of a security incident involving **CVE-2026-24061**, a critical authentication bypass vulnerability in `GNU InetUtils telnetd`. This analysis was conducted using a PCAP file from the **Hack The Box (HTB) Sherlock: Telly** challenge, focusing on the exploit mechanism and post-exploitation activities.

## 🛡️ Executive Summary
On January 27, 2026, an external attacker exploited a flaw in the Telnet service to gain unauthorized **root access**. By injecting a malicious environment variable (`USER=-f root`), the attacker bypassed authentication, established persistence, and exfiltrated sensitive data.

## 🕵️ Technical Investigation & Timeline
The following timeline was reconstructed through deep packet inspection (DPI) and TCP stream reconstruction:

* **10:39:28 (Exploitation):** Attacker triggered the bypass via Telnet `NEW-ENVIRON` option with `USER=-f root`.
* **10:41:33 (Persistence):** Creation of a backdoor account named `cleanupsvc`.
* **10:44:02 (Tool Ingress):** Downloaded a persistence script (`linper.sh`) from GitHub using `wget`.
* **10:47:30 (C2 Establishment):** Outbound communication established to Command & Control (C2) IP `91.99.25.54`.
* **10:49:54 (Exfiltration):** Sensitive database file successfully exfiltrated.

## 🛠️ Analysis Methodology
* **Tool:** Wireshark v4.6.3.
* **Key Filter Used:** `telnet`.
* **Technique:** Reconstruction of interactive TCP streams to identify commands like `useradd`, `passwd`, and `wget`.

## 🗺️ MITRE ATT&CK® Mapping
| Tactic | Technique | ID |
| :--- | :--- | :--- |
| **Initial Access** | Exploit Public-Facing Application | T1190 |
| **Persistence** | Valid Accounts (Backdoor Account) | T1078 |
| **Command & Control** | Application Layer Protocol: Telnet | T1071.001 |
| **Exfiltration** | Exfiltration Over C2 Channel | T1041 |
| **Credential Access** | OS Credential Dumping (`/etc/shadow`) | T1003 |

## 🚨 Indicators of Compromise (IOCs)
* **Exploit String:** `USER=-f root`
* **Backdoor Account:** `cleanupsvc`
* **C2 IP:** `91.99.25.54`
* **Persistence Script:** `https://raw.githubusercontent.com/montysecurity/linper/refs/heads/main/linper.sh` 

## 🚀 Recommendations
* **Disable Telnet:** Replace immediately with **SSH** for encrypted remote access.
* **C2 Server Blocking:** Block outbound traffic to `91.99.25.54`.
* **Credential Audit:** Remove the `cleanupsvc` user account.
* **Network Segmentation:** Implement segmentation to isolate critical backup servers.
* **Monitoring:** Deploy **IDS/IPS** signatures to alert on suspicious Telnet options.
* **Logging:** Enable centralized logging (SIEM) for visibility into account creation.

## 📸 Evidences

## 📸 Evidence 1: Initial Access

*   **Description:** Following screenshot shows Frame 52 where the Telnet `NEW-ENVIRON` option is used to inject the malicious string `USER=-f root`.
<img width="1366" height="164" alt="Screenshot (initial_access)" src="https://github.com/user-attachments/assets/96ea09ce-bf37-4327-a0c2-e1342ee693db" />

---

## 📸 Evidence 2: Post-Exploitation & Persistence

*   **Description:** TCP Stream reconstruction showing the attacker creating the `cleanupsvc` backdoor account using `useradd`.
<img width="1366" height="99" alt="Screenshot (backdoor_account_creation)" src="https://github.com/user-attachments/assets/a6215a0f-2404-432e-a1cd-c8a64ae8a7fd" />

*   **Description:** You can see the `wget` command used to pull the `linper.sh` persistence script from GitHub. Then the attacker executed `linper.sh` file by entering `chmod +x linper.sh` command. 
  <img width="1366" height="479" alt="Screenshot (linper sh_download)" src="https://github.com/user-attachments/assets/944024ee-0d2e-4795-9ebf-9199a1263342" />
<img width="1366" height="510" alt="Screenshot (chmod +x linper sh)" src="https://github.com/user-attachments/assets/66adb9fe-b7c9-4e2a-9ab7-cf74620541b9" />

---

## 📸 Evidence 3: Command & Control (C2) & Data Exfiltration

*   **Description:** Evidence of an outbound TCP session established to the malicious C2 IP address `91.99.25.54`.
<img width="1366" height="178" alt="Screenshot (C2_server_communication)" src="https://github.com/user-attachments/assets/edca2b92-c778-46bd-9cf8-8b9b34153a0e" />

*    **Description:** This screenshot confirms the data exfiltration of the sensitive database named  `credit-cards-25-blackfriday.db` at 10:49:54.
<img width="1366" height="106" alt="Screenshot (data_exfilteration)" src="https://github.com/user-attachments/assets/09388e6d-4f6e-47d1-8398-0dd37acf2573" />

---

## 🔎 References
* https://www.offsec.com/blog/cve-2026-24061/
* https://nvd.nist.gov/vuln/detail/CVE-2026-24061
* https://www.cvedetails.com/cve/CVE-2026-24061/

 ---
<img width="1357" height="612" alt="Screenshot (1713)" src="https://github.com/user-attachments/assets/0e7e8de8-d4dd-4b44-8e8e-c3240bdbb596" />
https://labs.hackthebox.com/achievement/sherlock/2413013/1144


