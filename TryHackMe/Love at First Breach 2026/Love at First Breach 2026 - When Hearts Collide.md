# When Hearts Collide

**Room:** [Love at First Breach 2026 – When Hearts Collide](https://tryhackme.com/room/lafb2026e1)

An MD5 collision attack the app "matches" you with a dog by comparing MD5 hashes instead of actually comparing images, so if you can forge a file with the same MD5 as the target dog photo, you get matched instantly without ever needing the real image content to match.

![When Hearts Collide](../Screenshots/Love%20at%20First%20Breach%202026/01_room_banner.png)

---

## Recon

Let's navigate to the URL and see what's there.

![Matchmaker homepage](../Screenshots/Love%20at%20First%20Breach%202026/02_matchmaker_homepage.png)

There's a hint on the page that tells us about MD5:

> "Matchmaker keeps up with the universe by comparing your photo's MD5 hash to every doggo snapshot that wanders through the site. If your hash is identical to a pup's, that's our cue: the algorithm wags its glittering tail and declares you a match."

So the matching isn't based on the actual image content at all it's purely comparing **MD5 hashes**. That's the vulnerability right there.

![MD5 hint on the page](../Screenshots/Love%20at%20First%20Breach%202026/03_md5_hint.png)

---

## Testing the Upload

There's an option to upload images, so let's try uploading some random stuff first.

![Uploading a random image](../Screenshots/Love%20at%20First%20Breach%202026/04_random_upload_no_match.png)

But this image doesn't match, since its MD5 hash has nothing in common with any dog in the database.

If we click on "browse a random picture," it navigates to a dog image.

![Target dog picture](../Screenshots/Love%20at%20First%20Breach%202026/05_target_dog_image.png)

Let's download it first and upload it on the page.

![Uploading the same dog image](../Screenshots/Love%20at%20First%20Breach%202026/06_duplicate_upload_detected.png)

The app recognizes it as an exact duplicate it already has on file and tells us there's no need to upload it again confirming the check is purely hash-based, not a fresh visual comparison each time.

---

## Building the Collision

Then I checked the MD5 of `dog.jpg` that I downloaded from the browser:

```
md5sum dog.jpg
a15ec1ecaef0eac2d8a9be79d1d51296  dog.jpg
```

![MD5 of dog.jpg](../Screenshots/Love%20at%20First%20Breach%202026/07_md5sum_of_dog.png)

After some research, I installed `fastcoll` (an MD5 collision tool). Since MD5 is cryptographically broken, we can use `fastcoll` to generate two different files that share an identical MD5 hash:

```bash
./fastcoll dog.jpg
md5sum md5_data1 md5_data2
mv md5_data1 1.jpg
mv md5_data2 2.jpg
```

![fastcoll generating two files with the same MD5](../Screenshots/Love%20at%20First%20Breach%202026/08_fastcoll_collision_generated.png)

After this, both images have the exact same MD5 hash. I uploaded one of them and got the flag.

![Match complete - flag](../Screenshots/Love%20at%20First%20Breach%202026/09_match_complete_flag.png)

---

## Flag

```
THM{hash_puppies_4_all}
```

