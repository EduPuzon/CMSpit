# CMSpit — TryHackMe Writeup

**Room:** [TryHackMe - CMSpit](https://tryhackme.com/room/cmspit)
**Difficulty:** Medium
**Category:** Web Exploitation, CMS Vulnerability, Privilege Escalation

---

## Overview

This writeup documents my process of enumerating, exploiting, and fully compromising the **CMSpit** machine on TryHackMe. The box runs a CMS vulnerable to a CSRF-based information disclosure, which I chained into an admin account takeover via password reset abuse, followed by remote code execution through a malicious file upload, and ultimately privilege escalation to root through a MongoDB credential leak and a sudo misconfiguration.

**Attack chain summary:**
1. Nmap scan to identify open ports/services
2. Intercepted login request in Burp Suite and identified a CSRF token vulnerability
3. Leaked user data via the CMS login flow
4. Abused the password reset flow to hijack the admin account
5. Uploaded a PHP web shell through the admin panel to gain RCE
6. Caught a reverse shell as `www-data`
7. Found MongoDB credentials on the box, pivoted to a low-privileged user
8. Found a sudo misconfiguration / GTFOBins entry leading to a root shell

---

## Reconnaissance

Started with a standard Nmap scan to identify open ports and running services on the target.

```bash
nmap -sC -sV -oN nmap_scan.txt <TARGET_IP>
```

![Nmap Scan](nmap.png)

---

## Web Enumeration

Port 80 was open, hosting the CMS application. Navigating to the site brought me to a login page.

![Website Landing Page](Cockpit.png)

With no credentials available, I needed to inspect the login flow more closely — this meant firing up Burp Suite to intercept the traffic.

---

## Vulnerability Discovery — CSRF / Info Leak

Intercepting the login request in Burp Suite, I noticed the request included a CSRF token that stood out.

![Burp Suite Login Intercept](Burp.png)

Researching this behavior led me to a known vulnerability affecting this CMS, detailed here:

🔗 **Reference:** [anquanke.com/post/id/241113](https://www.anquanke.com/post/id/241113)

This CSRF flaw allows an attacker to leak backend user data (usernames and password hashes) via a crafted login/auth request.

![CSRF Payload](Burpesuite.png)


---

## Exploitation — Admin Password Reset

With the leaked account information, the next step was hijacking the password reset flow to take over the admin account.

**Step 1 — Request a password reset token**

Using the extracted username, a password reset request was triggered to generate a token.
![token](Burp(1).png)

**Step 2 — Swap the endpoint**

The key to this exploit: instead of submitting the token to `POST /auth/resetpassword`, the request is redirected to `POST /auth/newpassword`. This bypasses the intended validation flow and allows direct extraction of the reset token.

```
POST /auth/resetpassword   →   POST /auth/newpassword
```


**Step 3 — Extract user account data**

Using the modified endpoint, I was able to extract full account details — including the username, password hash, API key, and reset token.


**Step 4 — Reset the password**

With the valid reset token in hand, I reset the admin account's password and successfully logged in.

![Leaked User Info](Burpesuite(1).png)

---

## Gaining RCE — Web Shell Upload

Now authenticated as admin, I located a file manager / upload feature within the CMS panel that allowed arbitrary file creation.


I crafted a simple PHP web shell payload and uploaded it as `shell.php`.

![shell](shell.png)
![Web Shell Upload](import_shell.png)

Confirmed code execution by browsing directly to the uploaded shell:

```bash
curl http://<TARGET_IP>:80/shell.php
```
---

## Reverse Shell

With confirmed RCE, I crafted a reverse shell payload pointing back to my attacking host on port `4447` and triggered it through the web shell.

```bash
# Listener on attacker machine
nc -lnvp 4447
```


Successfully caught a shell as the web server user:

![Reverse Shell Payload](reverse_shell.png)

---

## Privilege Escalation — User

While enumerating the compromised web server, I discovered a MongoDB instance running locally with no authentication configured.

```bash
mongo
show dbs
use <database>
show collections
db.user.find()
```

![MongoDB Enumeration](Password.png)

The database contained plaintext/hashed credentials for a local system user, which I used to `ssh` into the box and escalate from `www-data` to a standard user account.

---

## Privilege Escalation — Root

Checking sudo privileges for the current user revealed a binary that could be run as root without a password.

```bash
sudo -l
```

![Sudo Permissions](Sudo.png)

Cross-referencing the binary against **GTFOBins** confirmed it could be abused for privilege escalation.

![GTFOBins Reference](exiftool.png)

In addition to the GTFOBins path, the sudo-permitted binary was also vulnerable to a **known remote code execution (RCE) CVE**, which provided an alternative and more reliable escalation route.

https://github.com/UNICORDev/exploit-CVE-2021-22204

![Sudo RCE Vulnerability](CVE-2021-2204.png)


Executing the vulnerable binary as root via sudo triggered code execution in the root context, dropping me into a root shell.

```bash
sudo /path/to/vulnerable-binary <malicious-file>
```

![exploit](Exploitation.png)

**Root flag captured — box fully compromised.** ✅

![root](root.png)
---

## Summary

| Stage | Technique |
|---|---|
| Recon | Nmap port/service scan |
| Initial Foothold | CSRF-based user data leak |
| Account Takeover | Password reset endpoint manipulation (`/auth/resetpassword` → `/auth/newpassword`) |
| RCE | Malicious file upload via admin file manager |
| Shell Access | PHP reverse shell over port 4447 |
| Privilege Escalation (User) | Unauthenticated MongoDB credential leak |
| Privilege Escalation (Root) | Sudo misconfiguration + GTFOBins / RCE CVE |

---

## Lessons Learned / Remediation

- **CSRF tokens must be tied to authenticated sessions** and validated server-side; predictable or leakable tokens defeat the purpose of the protection.
- **Password reset endpoints should never expose account data** (hashes, API keys, tokens) in the response body — return only generic success/failure messages.
- **File upload/creation features must enforce strict file-type validation** and execute uploaded content outside the webroot, or disable script execution in upload directories entirely.
- **Databases should never be left without authentication**, even on "internal" services — lateral movement from a low-privileged shell is trivial otherwise.
- **Sudo rules should follow least privilege.** Avoid granting NOPASSWD access to binaries capable of file read/write or code execution unless absolutely necessary, and keep third-party binaries patched against known CVEs.

---

*Writeup for educational purposes only, completed on TryHackMe's CMSpit room in an isolated lab environment.*
