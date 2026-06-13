# HTB-Sherlock-Fruitzy: Write-Up

**Write-Up | DFIR | Platform: Hack The Box | Solved by: RismaFareedh | Sherlock Rank: #31 | Solve Date: 01 May 2026**

---

## Sherlock Scenario

CyberJunkie started as a junior QA Analyst at his friend's startup. He called the CEO of the startup because he believed he had mistakenly downloaded something malicious. The CEO sought help from you, his friend in the cybersecurity field. You sent him a guide on collecting evidence from the machine using KAPE. Now that you have been given the forensic image, you can analyze and help your friends, as they cannot afford to hire an MSSP.

---

## Tools Utilized

- **PhishTool**: email header analysis
- **DB Browser for SQLite**: browser history / downloaded artifact analysis
- **Eric Zimmerman's Registry Explorer**: Amcache.hve parsing
- **Eric Zimmerman's MFTECmd**: Master File Table analysis
- **Windows Event Viewer**: Windows event logs analysis
- **VirusTotal**: file hash, domain threat intelligence, domain registration lookup

---

## Executive Summary

| Field | Details |
|---|---|
| **Incident Type** | Phishing leading to Remote Monitoring and Management (RMM) Backdoor Deployment |
| **Target User** | cyberjunkie |
| **Target Environment** | Windows Endpoint (QA-Pipeline-02) |
| **Threat Actor Vector** | Social engineering via a themed email lure ("Special Party Invitation") directing the user to a malicious payload download link |
| **Impact** | Full, persistent endpoint compromise via unauthorized installation of Datto RMM (CentraStage), granting the attacker administrative remote access with LocalSystem privileges |

---

## Task 1 — What is the Subject/topic of the Phishing email?

### Investigation

The first step was uploading the suspicious .eml file to PhishTool for header analysis. The tool immediately surfaced the full email metadata. The email carried the subject `Special Party Invitation from JANET CARNAHAN`, sent from `businessinvitation432@gmail.com`, Display Name "Business invitation", to `CyberJunkieContractor@gmail.com`, and Timestamp `2026-03-04 14:48:56`. The originating IP was `209.85.220.65`, which reverse-resolved to `mail-sor-f65.google.com`.

<img width="1159" height="518" alt="SS-Answer-01" src="https://github.com/user-attachments/assets/a4ba351d-cc9a-49b9-9169-dc76a30a207f" />

**The email body was deliberately brief,`"Please find your exclusive invitation at the following link. We look forward to welcoming you at the event. Regards, Janet."` — a social-engineering lure designed to drive a single click**.

<img width="805" height="285" alt="SS-URL" src="https://github.com/user-attachments/assets/28f719f3-e17a-4b30-90d4-3bc42c60c3f5" />

---

## Task 2 — What is the malicious URI that the malicious link redirected to?

### Investigation

To trace where the phishing link led, I examined the Microsoft Edge browser history using DB Browser for SQLite, opening the History database at:

```
H:\C\Users\cyberjunkie\AppData\Local\Microsoft\Edge\User Data\Default\History
```

Browsing the downloads table revealed a single download entry. The tab_url field contained the full malicious URI:

```
https://pomi.digital/premium/windows_download.php
```
<img width="1354" height="199" alt="SS-Answer-02" src="https://github.com/user-attachments/assets/ca1f2248-8517-4b9e-a1ad-98c7c8385ce2" />

---

## Task 3 — What is the name of the downloaded file?

### Investigation

This was corroborated by the Master File Table (MFT). Using MFTECmd, the MFT was parsed to a CSV and opened in Excel. Filtering on the `ParentPath` as '*Downloads*' or specifically '*\Users\cyberjunkie\Downloads*' returned row 226031 with the following attributes:

- **ParentPath:** .\Users\cyberjunkie\Downloads
- **FileName:** premium.exe
<img width="1331" height="122" alt="SS-Answer-03" src="https://github.com/user-attachments/assets/3929ef61-8aa6-4db4-8e45-30a1ddc65c25" />

---

## Task 4 — When was the downloaded file executed by the victim according to Amcache?

### Investigation

Amcache.hve (`C:\Windows\appcompat\Programs\Amcache.hve`) is a Windows registry hive that records program execution metadata — critically, the first time a binary was run on the system. I loaded Amcache.hve into Registry Explorer and searched for `premium.exe`.

The forensic entry showed:

```
Root\InventoryApplicationFile\premium.exe|45f3dc9b442b7d80
Last write timestamp: 2026-03-04 16:44:33
```
<img width="1284" height="348" alt="SS-Answer-04" src="https://github.com/user-attachments/assets/65ca2029-96e7-4fc5-8923-bb88508125cc" />

---

## Task 5 — What is the SHA256 hash of the malicious executable?

### Investigation

From Amcache metadata of premium.exe, I found the `FileId` which is equivalent to the SHA-1 hash (SHA-1: `405481d3d5529445c546d4425b72c5827ebde840` — ignore first 4 digits). Then queried VirusTotal with the SHA-1 hash and got the SHA-256 hash.

<img width="668" height="346" alt="SS-Answer-05-1" src="https://github.com/user-attachments/assets/e3759cec-a4c5-4153-b1cf-f335a9104eef" />

**Foundout the SHA-256 Hash `af240a2c2a4b42e3a130f47ccaab8aa2e20a1a588bc959ee9efd7475055ea7e3` from VirusTotal.

<img width="730" height="130" alt="SS-Answer-05-2" src="https://github.com/user-attachments/assets/b65db157-3f22-49fb-8d3b-f4f639c43e55" />

---

## Task 6 — When was the Microsoft Defender scan of the file initiated?

### Investigation

After executing the file and receiving no party invitation, the victim grew suspicious and manually initiated a Microsoft Defender custom scan on the file. To pinpoint the exact time, I opened Windows Event Viewer and loaded the saved log:

```
Microsoft-Windows-Windows Defender/Operational
```

Filtering for `Event ID 1000` (Defender scan started) returned 5 matches. The most recent entry was dated 2026-03-04, and the event confirmed a Custom Scan targeting `C:\Users\cyberjunkie\Downloads\premium.exe`, initiated by user `QA-PIPELINE-02\cyberjunkie` on `computer QA-Pipeline-02`.


Then switching to the `Details tab* > *Friendly View` and reading the `SystemTime` revealed the precise UTC timestamp:

```
2026-03-04T16:48:00.9828308Z
```

<img width="779" height="388" alt="SS-Answer-06-2" src="https://github.com/user-attachments/assets/0ba946e8-52e7-45b7-9ecf-393859050c41" />


---

## Task 7 — What was the RMM backdoor service name?

### Investigation

To identify the installed backdoor, I filtered the Windows System event log for `Event ID 7045`, "a new service was installed in the system." A suspicious entry was logged at **2026-03-04 16:44:45 UTC** with the service file name **CentraStage.exe**. Its silent installation, configured with an auto-start trigger and LocalSystem privileges, is a clear indicator of a malicious backdoor deployment.

```
Service Name:        CentraStage
Service File Name:   C:\Program Files (x86)\CentraStage\CagService.exe
Service Type:        user mode service
Service Start Type:  auto start
Service Account:     LocalSystem
```

I dug deeper into **CentraStage**, which is legitimate enterprise IT software — it functions as a Remote Monitoring and Management (RMM) tool designed to let IT administrators manage networks, push patches, and execute commands remotely. Because it features powerful, built-in system management capabilities, threat actors frequently abuse it as a stealthy backdoor.

<img width="923" height="549" alt="SS-Answer-07" src="https://github.com/user-attachments/assets/2a43f06f-684a-449b-b305-991bfb4a97af" />

---

## Task 8 — What was the timestomped modified timestamp set on the RMM executables?

### Investigation

Timestomping is an anti-forensics technique where an attacker alters a file's timestamps to make it look older, helping it evade timeline-based detection. To investigate this, I used MFTECmd to parse the Master File Table (MFT), exported the results to a CSV file, and filtered the `ParentPath` column for CentraStage files. The file `Gui.exe` (located in `.\Program Files (x86)\CentraStage`) showed clear timestamp mismatches across two critical MFT attributes:

- **$STANDARD_INFORMATION ($SI / 0x10 attributes):** These handle standard creation and modification times (Created0x10 and LastModified0x10). They can be modified via normal Windows API calls, making them vulnerable to attacker manipulation.

- **$FILE_NAME ($FN / 0x30 attributes):** These record the kernel-level creation and rename timestamps (Created0x30 and LastModified0x30). They are written directly by the OS kernel and cannot be altered by standard user-mode API calls.

The divergence between the manipulated $SI timestamps (2/9/2026 07:56:40 UTC) and the ground-truth $FN kernel timestamps (3/4/2026 16:44:45 UTC) provides definitive proof of timestomping.

<img width="1323" height="206" alt="SS-Answer-08" src="https://github.com/user-attachments/assets/5519540b-3cbe-4222-88f1-09c96e51b04e" />

---

## Task 9 — What is the name of the company whose product is the RMM tool?

### Investigation

CentraStage is the legacy/agent name for **Datto RMM**, a product of **Datto, Inc.** CentraStage is widely used by managed service providers for legitimate remote IT management, which is precisely why threat actors abuse it. It's signed binaries and legitimate network traffic blend seamlessly with normal business activity, making it difficult to detect and are rarely blocked by firewalls.

---

## Task 10 — When was the malicious domain registered?

### Investigation

`PhishTool > URL` tab exposed the domain. Then I queried `pomi.digital` on VirusTotal and under `registration data`, got the following info:

```
Registration: 2026-02-20T01:06:05Z
Expiration:   2027-02-20T01:06:05Z
```

The domain was registered just 12 days before the phishing email was sent (2026-03-04). This is a classic threat actor pattern — freshly registered domains used for short-lived phishing campaigns before being burned and abandoned.

<img width="1274" height="365" alt="SS-Answer-10" src="https://github.com/user-attachments/assets/4ebc56cf-8c44-4a24-9802-4d09cd8ff374" />

---

## Task 11 — What is another name for the initial executable?

### Investigation

In VirusTotal's *Details tab* for the premium.exe hash, the Names section lists all filenames under which this binary has been submitted:

<img width="1294" height="132" alt="SS-Answer_11" src="https://github.com/user-attachments/assets/57f07072-58fb-4805-ba9a-d5b76e62fb09" />

---

<img width="1324" height="591" alt="share-Ach" src="https://github.com/user-attachments/assets/168e9084-830e-4aa6-870e-5010dd038a70" />


🔗 [HTB Achievement](https://labs.hackthebox.com/achievement/sherlock/2413013/1212)

---

*Tags: DFIR, Cybersecurity, Hack The Box, Malware Analysis, Incident Response, PhishTool, Windows Event Log Analysis, MFT Analysis, Digital Forensics*

