# Love Letter Locker

**Room:** [Love at First Breach 2026 – Love Letter Locker](https://tryhackme.com/room/lafb2026e2)

An IDOR vulnerability that lets you read other users' private love letters just by changing a number in the URL.

![Love Letter Locker](../Screenshots/Love%20at%20First%20Breach%202026/01_room_banner.png)

---

## Recon

Let's do an nmap scan on the target IP:

```
nmap -sC -sV -p- -T4 10.112.137.43
```

![Nmap scan](screenshots/02_nmap_scan.png)

After doing the nmap scan I found 2 open ports which are:

- **22/tcp** — SSH
- **5000/tcp** — HTTP 

---

## Exploring the Web App

Let's check port 5000 and see what we can do there.

![Port 5000 homepage](screenshots/03_port_5000_homepage.png)

Make an account here and check what we see.

![Account created](screenshots/04_account_created.png)

After login we can see that there is a **Tip from Cupid** that says:

> "Every love letter gets a unique number in the archive. Numbers make everything easier to find."

That's a good hint as it tells about the IDOR vulnerability. Let's check where it actually is.

![My Letters - Tip from Cupid](screenshots/05_my_letters_cupid_tip.png)

---

## Finding the IDOR

I write a letter just to check what we get from it.

![Write a love letter](screenshots/06_write_love_letter.png)

Once the letter is saved, I open it and look at the URL — it contains the number **3**.

```
http://10.112.137.43:5000/letter/3
```

![Letter with ID in URL](screenshots/07_letter_id_in_url.png)

Let's try to change it and check what happens.

When I change the ID to **1**, it gives us the flag — meaning letter #1 belongs to another user, and there is no access control checking whether the logged-in user actually owns that letter ID.

```
http://10.112.137.43:5000/letter/1
```

![IDOR - flag in another user's letter](screenshots/08_idor_flag.png)

Letter #1 turns out to belong to another user ("Gonz0") titled *"To my secret Valentine"* — and it contains the flag directly in the message body.

---

## Flag

```
THM{1_c4n_r3ad_4ll_l3tters_w1th_th1s_1d0r}
```

---

## Vulnerability Summary

| | |
|---|---|
| **Vulnerability** | Insecure Direct Object Reference (IDOR) |
| **Location** | `/letter/<id>` endpoint |
| **Root Cause** | Letter IDs are sequential and predictable, and the server does not verify that the requesting user owns the letter before returning it |
| **Impact** | Any authenticated user can read any other user's private love letter by simply incrementing/decrementing the ID in the URL |
| **Fix** | Enforce server-side ownership checks (verify `letter.user_id == current_user.id` before returning data) and/or use non-sequential, unguessable identifiers (UUIDs) |
