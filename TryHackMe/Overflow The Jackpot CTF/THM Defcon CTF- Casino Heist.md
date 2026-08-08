# Casino Heist — Forensics 

Machine got flagged for acting weird late at night, and someone pulled the full packet capture before killing it.

```
unzip attachment-1785961082183.zip
```

![unzip](../Screenshots/Overflow%20The%20Jackpot%20CTF/01-unzip-pcap.png)

Got `stolen_jackpot.pcapng` out of it.

First thing I do with any pcap is get a quick protocol breakdown so I know what I'm dealing with before diving in blind.

```
tshark -r stolen_jackpot.pcapng -q -z io,phs
```

![protocol hierarchy](../Screenshots/Overflow%20The%20Jackpot%20CTF/02-protocol-hierarchy.png)

701 frames total, mostly TCP, and only 8 HTTP frames — but one of those HTTP frames is a single ~52KB data blob. That's not a normal page load, something got transferred.

Pulled every file that went over HTTP out of the capture to see what that blob actually was.

```
tshark -r stolen_jackpot.pcapng --export-objects "http,extracted_http" -q
ls -la extracted_http
```

![extracted http objects](../Screenshots/Overflow%20The%20Jackpot%20CTF/03-extracted-http-objects.png)

Four files came out: `admin`, `cms`, `flag` (all 469 bytes each) and `stealer` at a chunky 20MB. Ran `file` on all of them — the three small ones turned out to be identical generic 404 pages (someone probing paths that didn't exist), and `stealer` came back as a stripped 64-bit ELF binary. That's the getaway car the challenge briefing was talking about.

The 404s were a dead end, so I moved on to figuring out where the actual stolen data went. HTTP only showed the download of `stealer`, no upload traffic — so if something got exfiltrated, it went out over a different connection. Listed all the TCP conversations in the capture to check.

```
tshark -r stolen_jackpot.pcapng -q -z conv,tcp
```

![tcp conversations](../Screenshots/Overflow%20The%20Jackpot%20CTF/04-tcp-conversations-4444.png)

Found two short TCP streams going to `172.20.0.3:4444` — classic reverse shell / listener port. That's not HTTP traffic, that's raw TCP, which lines up with the brief saying the loot "went straight over the wire."

Followed both streams in ASCII to see what actually got sent.

```
tshark -r stolen_jackpot.pcapng -q -z follow,tcp,ascii,172.20.0.1:49496,172.20.0.3:4444
tshark -r stolen_jackpot.pcapng -q -z follow,tcp,ascii,172.20.0.1:49500,172.20.0.3:4444
```

![tcp stream ascii flag.jackpot](../Screenshots/Overflow%20The%20Jackpot%20CTF/05-tcp-stream-ascii-flagjackpot.png)

Both streams start with the filename `flag.jackpot` followed by a chunk of binary garbage — encrypted data. So the stealer grabbed a file called `flag.jackpot`, encrypted it, and shipped it out over that port 4444 connection. Now I needed the key.

The briefing said the tool was "built in a hurry" and the key would be "sitting in plain sight" — so the obvious move is to crack open the stealer binary itself. Installed pyinstxtractor-ng through pipx (regular pip install got blocked by Kali's externally-managed-environment thing) and ran it against the binary.

```
pipx install pyinstxtractor-ng
pyinstxtractor-ng extracted_http/stealer
```

![pyinstxtractor unpack](../Screenshots/Overflow%20The%20Jackpot%20CTF/06-pyinstxtractor-unpack.png)

It's a PyInstaller-packed Python binary — explains the 20MB size. Extraction spit out a bunch of files, and one stood out as the real entry point: `stealer.pyc`, matching the binary's own name.

Ran `strings` on it to see if the key was sitting there in plain text.

```
strings -n 6 stealer_extracted/stealer.pyc
```

![strings raw key/iv candidates](../Screenshots/Overflow%20The%20Jackpot%20CTF/07-pyc-strings-raw-keyiv.png)

Two strings jumped out immediately: `J4ckp0tH4ck3rKeys` and `Iv_For_Exf1ltr8!z` — clearly a key and an IV. But I noticed other strings nearby were slightly corrupted (`Cipherr`, `Paddingr` instead of `Cipher`/`Padding`), which meant raw `strings` was bleeding extra characters into some of these — so I didn't trust those values as-is.

To get a clean read, I wrote a small Python script using the `xdis` library to properly parse the bytecode and dump the actual string constants instead of relying on a raw byte scan.

```python
import sys
from xdis.load import load_module

def walk(co, seen=None):
    if seen is None:
        seen = set()
    if id(co) in seen:
        return
    seen.add(id(co))
    for const in co.co_consts:
        if isinstance(const, str) and len(const) >= 4:
            print(repr(const))
        elif isinstance(const, bytes) and len(const) >= 4:
            print(repr(const))
        elif hasattr(const, 'co_consts'):
            walk(const, seen)

path = sys.argv[1]
result = load_module(path)
co = next(x for x in result if hasattr(x, 'co_consts'))
walk(co)
```

```
python3 dump_consts.py stealer_extracted/stealer.pyc
```

![clean key and iv extracted](../Screenshots/Overflow%20The%20Jackpot%20CTF/08-clean-key-iv-extracted.png)

That confirmed it — the real key is `J4ckp0tH4ck3rKey` and the real IV is `Iv_For_Exf1ltr8!`, both clean 16-byte strings, no trailing junk like the raw strings dump showed. Also saw `172.20.0.3` and `.jackpot` in there, matching what we already found on the wire.

Now I needed the actual ciphertext bytes. Pulled the same TCP stream again but in hex this time, since ASCII mode mangles binary data.

```
tshark -r stolen_jackpot.pcapng -q -z follow,tcp,hex,172.20.0.1:49496,172.20.0.3:4444
```

![tcp stream hex ciphertext](../Screenshots/Overflow%20The%20Jackpot%20CTF/09-tcp-stream-hex-ciphertext.png)

Clean structure: `flag.jackpot\n` as a plaintext header (13 bytes), followed by exactly 48 bytes of ciphertext — 3 full AES blocks, so AES-CBC made sense.

Got AI to help me put together a quick decrypt script with the key, IV, and ciphertext bytes I'd pulled off the wire.

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b'J4ckp0tH4ck3rKey'
iv  = b'Iv_For_Exf1ltr8!'

ciphertext = bytes.fromhex(
    "9bcc341f8374f7d031a5a0ee46635013"
    "13b15466b184e33a3f295efd0cd1b4f5"
    "c64b48dd831bbca7ec4423e3782f8fe5"
)

cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext = cipher.decrypt(ciphertext)
plaintext = unpad(plaintext, 16)
print(plaintext)
```

```
python3 decrypt.py
```

![flag decrypted](../Screenshots/Overflow%20The%20Jackpot%20CTF/10-flag-decrypted.png)

Padding checked out clean and out popped the flag:

```
THM{J@ckP05_R0bb3ry_VIA_pYth0nN}
```

