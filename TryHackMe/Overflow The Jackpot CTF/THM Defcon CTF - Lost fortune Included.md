# Lost Fortune Included

Easy web box, target was `10.113.146.22`. Started with a full port scan to see what's actually running.

```
nmap -sV -sC -p- --min-rate 1000 10.113.146.22
```
![nmap scan](../Screenshots/Overflow%20The%20Jackpot%20CTF/01-nmap-scan.png)

Just SSH and Apache 2.4.58 on Ubuntu. The http-title said "VILLAGE ARCHIVE TERMINAL v2.3", so that's where the app lives.

Pulled up the page in the browser to see what we're dealing with.

![kiosk homepage](../Screenshots/Overflow%20The%20Jackpot%20CTF/02-kiosk-homepage-doc-param.png)

A little kiosk that serves two documents through `?doc=<filename>`, with a note that only `.pdf` and `.png` are allowed. Classic file-serving pattern, felt like LFI territory right away.

First thing I tried was plain path traversal:

```
curl -s "http://10.113.146.22/?doc=../../../../etc/passwd"
```

Got rejected with "Only .pdf and .png village documents may be viewed." So there's an extension check happening before any file read. Fine, let's stop guessing blind and just read the app's own source instead, using the `php://filter` wrapper to base64-encode it so it doesn't get executed:

```
curl -s "http://10.113.146.22/?doc=php://filter/convert.base64-encode/resource=index.php" | tail -1 | base64 -d
```
![leaked index.php source](../Screenshots/Overflow%20The%20Jackpot%20CTF/03-leaked-index-php-source.png)

And there it is. The whitelist check only runs if `strpos($doc, '://')` is false — meaning any value using a stream wrapper skips the `.pdf`/`.png` check entirely and gets passed straight to `readfile()`. The traversal filter is also a single-pass `str_replace('../', '', $doc)`, but that didn't even matter once the wrapper path was open.

Tested it against `/etc/passwd` to confirm arbitrary file read:

```
curl -s -i "http://10.113.146.22/?doc=php://filter/resource=/etc/passwd"
```
![arbitrary file read via php filter](../Screenshots/Overflow%20The%20Jackpot%20CTF/04-etc-passwd-arbitrary-read.png)

Full passwd file back, no extension needed, no traversal needed. Whitelist's completely bypassed for wrapper syntax.

From there just guessed the flag would sit in the web root:

```
http://10.113.146.22/?doc=php://filter/resource=/var/www/flag.txt
```
![flag file downloaded through the browser](../Screenshots/Overflow%20The%20Jackpot%20CTF/05-flag-download-browser.png)

Grabbed it straight through the browser and opened it up.

![flag.txt opened](../Screenshots/Overflow%20The%20Jackpot%20CTF/06-flag-txt-opened.png)

```
THM{wr4pp3rs_sk1p_th3_wh1t3l1st}
```

Root cause is basically a security check that only ever looks at one branch of the logic. Whoever built this whitelisted `.pdf`/`.png` for normal filenames but never accounted for PHP happily accepting stream wrapper URIs through the same function, and the code was written to just skip the check entirely when it sees `://` instead of rejecting it. Once you spot the exemption, the actual bypass takes zero effort.
