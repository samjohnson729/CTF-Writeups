# TryHackMe: DUNE
https://tryhackme.com/room/dune
## 1. Network Enumeration
Use `nmap` to run a quick port scan to see what services are running on the box:

```
$ nmap 10.10.76.115        
                           
Starting Nmap 7.80 ( https://nmap.org ) at 2024-01-22 19:09 CET
Nmap scan report for 10.10.76.115
Host is up (0.22s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 29.04 seconds
```
We see that this box is running an FTP, SSH, and HTTP servers. Let's take a look at the HTTP server.
## 2. Web Server
The home page of the web server gives a quick summary of Frank Herbert's sci-fi book DUNE, but doesn't appear to show any other useful information. Lets start `gobuster` to see if we can find any hidden web pages.

```
$ gobuster -u http://<ip-address> -w /usr/share/wordlists/dirb/common.txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://<ip-address>/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirb/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2024/01/22 19:19:01 Starting gobuster
=====================================================
/.hta (Status: 403)
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/images (Status: 301)
/index.html (Status: 200)
/robots.txt (Status: 200)
/server-status (Status: 403)
=====================================================
2024/01/22 19:20:31 Finished
=====================================================
```

We found an `/images` directory, but nothing there looks particularly interesting. We also found a `robots.txt` file. Let's check it out:

```
User-Agent: *
Disallow: /chapterhouse/
```

Okay, great! Let's check out `/chapterhouse`.

