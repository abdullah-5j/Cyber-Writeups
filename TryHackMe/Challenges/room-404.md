# Hacker Holidays — The Byte Lotus Hotel

TryHackMe, Web category, Very Easy. Briefing was pretty on the nose about it too — something about a room "not on the floor plan" on port 8080, and a night-shift dev who shipped more than the website. Objective was just to dump the exposed source and grab the flag.

Target: `10.114.128.65:8080`

## Poking the site first

Hit it with curl before doing anything else, just to see what's actually running.

```
curl -i http://10.114.128.65:8080/
```

Response headers say `Server: Werkzeug/3.0.1 Python/3.12.3` which means this is a Flask app being served straight off the dev server, not behind anything real. Footer on the page also says "guest experience platform · build staging" so yeah, this is clearly not meant to be public-facing in this state.

![homepage](../Screenshots/byte-lotus/site-homepage.png)

Just a hotel landing page otherwise, nothing on it directly.

## Directory brute force

Ran gobuster against it to see if there's anything not linked from the nav.

```
gobuster dir -u http://10.114.128.65:8080 -w /usr/share/wordlists/dirb/common.txt -x html,txt,py,json,js -t 50
```

![gobuster hit](../Screenshots/byte-lotus/gobuster-dirsearch-git-hit.png)

Bunch of timeouts in the wordlist that don't matter, but `.git/HEAD` came back 200. That's the actual git folder sitting exposed on the live server, not just a reference to it somewhere. Checked it:

```
curl -s http://10.114.128.65:8080/.git/HEAD
```

```
ref: refs/heads/main
```

Yeah that's a real, browsable `.git` directory. Dev never removed it from staging before this got exposed.

## Dumping the repo

Grabbed git-dumper for this since it walks the whole `.git` structure over HTTP and rebuilds the repo locally.

```
pipx install git-dumper
```

(had the usual PEP 668 externally-managed-environment fight with pip first, pipx just sidesteps it)

```
git-dumper http://10.114.128.65:8080/.git/ ./byte-lotus-git
```

![git dumper](../Screenshots/byte-lotus/git-dumper-clone.png)

Pulled everything down and checked out the working tree fine. Only one commit in the whole repo actually — checked the reflog and it's a single "initial Byte Lotus guest platform" commit from a `night-shift` dev, so no digging through history needed, just look at what's in the tree.

```
cd byte-lotus-git && ls -la
```

Three files: `app.js`, `index.html`, `README.md`. Read the README since that's usually where people leave notes to themselves.

```
cat README.md
```

![readme flag](../Screenshots/byte-lotus/readme-flag.png)

And there it is, sitting right in the README under a comment that says "remove before launch" — which obviously didn't happen.

**Flag:** `THM{byt3_l0tus_n3v3r_f0rg3ts}`

Classic exposed `.git` folder leak, dev committed a staging note with the flag baked in and never scrubbed it before pushing this to something reachable. Room lived up to its briefing, the "room not on the floor plan" was literally just the `.git` directory the whole time.
