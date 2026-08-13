# TryHeartMe

**Room:** [Love at First Breach 2026 – TryHeartMe](https://tryhackme.com/room/lafb2026e5)

TryHeartMe is a Valentine's gift shop where every logged-in user carries a JWT that quietly tells the server who they are, how many credits they have, and critically what role they hold. The problem is that none of that information is actually protected. The token can be decoded, edited, and handed back to the server as if nothing happened, which turns a regular guest account into a staff account with a few clicks on [jwt.io](https://jwt.io).

![TryHeartMe](../Screenshots/Love%20at%20First%20Breach%202026/banner_tryheartme.png)

---

## Recon

First, we create an account so we can actually interact with the shop instead of browsing as a guest.

![Creating an account](../Screenshots/Love%20at%20First%20Breach%202026/02_create_account.png)

The shop itself is a simple Valentine's storefront roses, chocolates, strawberries, a love letter card each purchasable with in-app credits. Online top-ups are disabled, so credits aren't something you're meant to just buy your way into.

![TryHeartMe Valentines Shop](../Screenshots/Love%20at%20First%20Breach%202026/03_tryheartme_shop.png)

Once logged in and viewing a product page, the app conveniently displays our current session state right in the corner: `Credits: 0` and `Role: user`. As a fresh account, we have no credits and no elevated access.

![Logged in as a regular user with 0 credits](../Screenshots/Love%20at%20First%20Breach%202026/04_role_user_no_admin_access.png)

---

## Finding the Flaw

Since the app is clearly tracking role and credit state somewhere client-side, the natural place to look is the session cookie and sure enough, it's a JWT (JSON Web Token). Pulling that token into [jwt.io](https://jwt.io) and decoding it reveals the payload in plain, readable JSON:

```json
{
  "email": "abdullah@gmail.com",
  "role": "user",
  "credits": 0,
  "iat": 1786632692,
  "theme": "valentine"
}
```

![Decoding the JWT on jwt.io](../Screenshots/Love%20at%20First%20Breach%202026/05_jwt_decoded_role_field.png)

There it is — `"role": "user"` sitting right there in cleartext, along with our credit balance. A JWT is only as trustworthy as its signature; if the server doesn't properly re-verify that signature, then editing this payload and handing it back is just as good as being an actual admin. So we edit the `role` field from `"user"` to `"admin"`, let jwt.io re-sign the token, and swap it into our session cookie in place of the original.

---

## Escalating Privileges

After dropping the tampered token back in and refreshing, the difference is immediate — the corner tags now read `Credits: 5000` and `Role: admin`, and a brand new **Admin** link has appeared in the navigation bar that wasn't there before.

![Session now shows Role: admin and Credits: 5000](screenshots/06_role_admin_after_tampering.png)

Following that Admin link drops us straight into the **Admin Portal**, which openly states: *"Staff session detected. Staff can purchase the ValenFlag item."* There's a staff-only purchase panel with a single button **Open ValenFlag**.

![Admin Portal with the staff-only ValenFlag purchase](screenshots/07_admin_portal.png)

---

## Getting the Flag

Clicking through purchases the `ValenFlag` item using our now-admin session, and the app prints out a receipt confirming the order and redeeming a "voucher" which is really just the flag, sitting right there on the confirmation screen.

![Receipt showing the redeemed ValenFlag voucher](screenshots/08_valenflag_redeemed.png)

---

## Flag

```
THM{v4l3nt1n3_jwt_c00k13_t4mp3r_4dm1n_sh0p}
```

---

## Vulnerability Summary

| | |
|---|---|
| **Vulnerability** | Broken Access Control via JWT Tampering (Privilege Escalation) |
| **Location** | Session cookie / JWT used for authentication and authorization on the TryHeartMe shop |
| **Root Cause** | Authorization data (`role`, `credits`) is stored directly in the JWT payload, which is only base64url-encoded and not encrypted. The server trusts the claims inside the token without properly enforcing signature integrity, allowing the payload to be edited and re-signed (e.g. via jwt.io) and accepted as valid |
| **Impact** | Any authenticated user can escalate their own privileges from `user` to `admin`, unlocking staff-only functionality (the Admin Portal) and purchasing restricted items — in this case, redeeming the flag itself |
| **Fix** | Never trust client-controlled claims for authorization decisions without strict server-side signature verification using a strong, secret, server-only signing key. Sensitive state like role and balance should be looked up server-side from a trusted data store rather than read directly out of the token, and the server must reject tokens signed with anything other than its own known key/algorithm |
