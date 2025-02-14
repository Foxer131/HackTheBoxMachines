# Write up for Sea
Sea is a easy machine that uses a WonderCMS to get a reverse shell and after enumeration we can pivot using ssh and get root flag that way

## Start
As always let's begin with an nmap scan
```
nmap -sC -sV 10.10.11.28
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-02-14 00:54 EST
Nmap scan report for 10.10.11.28
Host is up (0.42s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e3:54:e0:72:20:3c:01:42:93:d1:66:9d:90:0c:ab:e8 (RSA)
|   256 f3:24:4b:08:aa:51:9d:56:15:3d:67:56:74:7c:20:38 (ECDSA)
|_  256 30:b1:05:c6:41:50:ff:22:a3:7f:41:06:0e:67:fd:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Sea - Home
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Results show only 2 ports open. Nmap didn't show, however http is redirecting into sea.htb.
Let's add that to our host file and access the page 

## Website 

![image](https://github.com/user-attachments/assets/e0fc2844-5327-4d27-8383-aa065f1fe8cc)

Nothing special at the bat of an eye. However running a ffuf scan we can get some more info about how the site is built
```
ffuf -u "http://sea.htb/FUZZ" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -e .md,.php -fs 199 --recursion -ic
```
Eventually you'll find two interesting hits

```
http://sea.htb/themes/bike/version
http://sea.htb/themes/bike/README.md
```

In these links we find out the site uses WonderCMS on version 3.2.0.
Doing a quick google search we find CVE-2023-41425
This is a cross-site-scripting exploit, so we need to send it to someone. Luckily on the main page we can find a contact page under
```
http://sea.htb/contact.php
```
A poc of the CVE can be found here
```
https://github.com/insomnia-jacob/CVE-2023-41425?tab=readme-ov-file
```
Hopefully the admin will click the link and you'll get a shell.
Next, simply look for the database.js file and get a password hash.

Use mode 3200 to crack it on hashcat with following command
```
hashcat /usr/share/wordlist/rockyou.txt hash -m 3200
```

There's only user amay under the home folder. Since this is a easy machine we can use the password mychemicalromance to log in via ssh and get the user flag.

### Root tomorrow
