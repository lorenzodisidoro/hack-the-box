# /etc/passwd file
To see the user and password stored in the system we can take a look a `/etc/passwd` that show a list of username, eg. as a follow

```sh
➜  ~ cat /etc/passwd
##
# User Database
#
# Note that this file is consulted directly only when the system is running
# in single-user mode.  At other times this information is provided by
# Open Directory.
#
# See the opendirectoryd(8) man page for additional information about
# Open Directory.
##
nobody:*:-2:-2:Unprivileged User:/var/empty:/usr/bin/false
root:*:0:0:System Administrator:/var/root:/bin/sh
...
```

usually the password are stored in the hash format into `/etc/shadow` file. 
```sh
root:$6$Ke02nYgo.9v0SF4p$hjztYvo/M4buqO4oBX8KZTftjCn6fE4cV5o/I95QPekeQpITwFTRbDUBYBLIUx2mhorQoj9bLN8v.w6btE9xy1:16431:0:99999:7:::
sys:$6$5H0QpwprRiJQR19Y$bXGOh7dIfOWpUb/Tuqr7yQVCqL3UkrJns9.7msfvMg4ZO/PsFC5Tbt32PXAw9qRFEBs1254aLimFeNM8YsYOv.:16431:0:99999:7:::
...
nagios:$6$P9zn0KwR$tgfvvFWJJ5FKmoXiP5rXWOjwoEBOEoAuBi3EphRbJqqjWYvhEM2wa67L9XgQ7W591FxUNklkDIQsk4kijuhE50:16632:0:99999:7:::
```

copy username and password into another file as a follow (eg. `passwords.txt`)
```sh
root:$6$Ke02nYgo.9v0SF4p$hjztYvo/M4buqO4oBX8KZTftjCn6fE4cV5o/I95QPekeQpITwFTRbDUBYBLIUx2mhorQoj9bLN8v.w6btE9xy1
sys:$6$5H0QpwprRiJQR19Y$bXGOh7dIfOWpUb/Tuqr7yQVCqL3UkrJns9.7msfvMg4ZO/PsFC5Tbt32PXAw9qRFEBs1254aLimFeNM8YsYOv.
```

yuo can crack these password using [John Th Ripper tool](https://www.openwall.com/john/) or something like this, for exemple running 
```
root@kali:~# john passwords.txt --show
```