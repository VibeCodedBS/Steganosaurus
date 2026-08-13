# Roadmap

Ideas for Steganosaurus, roughly in the order I'd build them. Effort estimates assume someone already familiar with the codebase; "an hour" means an hour, not a sprint.

The guiding principle: this is for the privacy hobbyist who would never open a terminal to run `outguess`, but *would* happily spend an evening poking at something that's honest about what it does. Easy to start, honest about limits, and you learn something.

---

## Build order

Everything marked ✅ has been picked. The order below isn't preference — it's dependencies. Three sequencing rules drive it:

- **Tests before crypto changes.** Shipping the suite first means every later change is checked against every earlier image.
- **Format spec after the format settles.** Writing it before the payload layout stops moving means writing it twice.
- **Translation last.** Every string still churning is a string translators have to redo.

| # | Item | Effort | Blocked by |
|---|---|---|---|
| 1 | ✅ Version byte in the header | 30 min | — |
| 2 | ✅ Ship the test suite + CI | 1 h | — |
| 3 | ✅ Plain-language threat model | 2 h | — |
| 4 | ✅ "Save this page and use it offline" button | 30 min | — |
| 5 | ✅ Flag the format tell (jpg in → png out) | 10 min | — |
| 6 | ✅ Surface the stripped metadata / warn on GPS | 2 h | — |
| 7 | ✅ Passphrase generator | 1 h | — |
| 8 | ✅ LSB visualiser | half day | — |
| 9 | ✅ Make the dino react | 2 h | — |
| 10 | ✅ Hide any file — images, keyfiles, PDFs | half day | 1 |
| 11 | Reed–Solomon error correction | half day | 1 |
| 12 | ✅ Duress passphrase | 1 day | 11 |
| 13 | ✅ Format spec + Python CLI | 1 day | 1, 10, 12 |
| 14 | ✅ Translation | 1 day + translators | most of the above |
| 15 | ✅ Surviving JPEG | a weekend, honestly more | 11 |

Items 2–9 are all independent and none take more than half a day. That's a satisfying first run: by the end of it the tool is honest about what it strips, teaches you what it's doing, generates its own passphrases, has a dinosaur that blinks, and can't silently break in future.

**Note on 11:** Reed–Solomon isn't a feature anyone asked for, but both the duress passphrase and JPEG survival collapse without it. It's the one unglamorous dependency on the list.

---

## Do this first (30 minutes, prevents real pain)

### Add a version byte to the payload header

The header currently goes `magic · salt · iv · length · ciphertext`. There's no version field. The moment anything about that layout changes — and everything in the next section changes it — every image made with the old build becomes undecodable, with no graceful error. Just a confusing "no message found".

One byte after the magic, checked on decode, with a real message when it doesn't match:

> *This image was made with a newer version of Steganosaurus.*

Do this before shipping any other feature on this list. It costs nothing now and is unfixable later.

---

## The next three

### 1. Hide any file — images, keyfiles, PDFs

Currently the payload is UTF-8 text. Making it arbitrary bytes is a modest change to a format that's already length-prefixed:

```
version · type · filename-length · filename · [compressed] file bytes
```

Notes from thinking it through:

- **Compress before encrypting.** `CompressionStream('deflate-raw')` is native in browsers — no library. Text, keyfiles and many PDFs shrink a lot; already-compressed formats (JPEG, PNG, most modern PDFs) barely move. Compress first, then encrypt — never the other way round, since compressing ciphertext does nothing and compressing *after* can leak length information about the plaintext.
- **Capacity is the real constraint.** At 3 bits per pixel you get `w × h × 3 / 8` bytes. A 12 MP phone photo holds about **4.5 MB** — plenty for a keyfile or a modest PDF. A 400×400 avatar holds 60 KB, which is a text file and nothing more. The capacity meter already exists; it just needs to speak in file sizes.
- **Image-in-image has a trick:** the *payload* image can be re-encoded to JPEG first. It's cargo, not carrier, so lossy compression is fine and buys you an order of magnitude. Hiding a 4 MB photo fails; hiding the same photo at JPEG quality 80 succeeds easily.
- **Reveal needs a download button** and a preview when the payload is itself an image.

**Effort:** half a day. **Payoff:** the biggest single capability jump available. "Put your PGP key inside your profile picture" becomes a one-liner.

### 2. Duress passphrase

Two messages in one image under two different words. The real one under yours, an innocuous one under a word you can hand over. No way to prove the second exists.

This works because the scatter positions derive from the passphrase, so the two payloads land in essentially unrelated places. **But they will collide**, and this is the part that's easy to get wrong:

- With two payloads of *n* bits each in a carrier of *N* slots, expect roughly `n²/N` collisions. Ten thousand bits each in a one-megabit carrier is ~100 corrupted bits — not rare, guaranteed.
- AES-GCM is all-or-nothing. A single flipped bit fails the auth tag and the whole message is lost. So a naive implementation produces a decoy that simply never decodes.
- **The fix:** write the decoy first, the real message second (real wins every collision), and protect the decoy with error correction so it tolerates the damage. Which means this feature depends on Reed–Solomon (below) — build that first.
- The capacity meter must not betray the hidden payload, and encoding time shouldn't visibly differ.

**Effort:** a day, plus the error-correction work. **Payoff:** the most genuinely interesting feature on this list, and real plausible deniability rather than the decorative kind.

### 3. Surviving JPEG

The honest weakness: any re-compression destroys the message, which rules out most of the ways people actually send pictures.

Fixing it means abandoning raw pixel LSBs and embedding in **quantised DCT coefficients** instead — the values JPEG itself stores — then writing out a JPEG. Skip zero coefficients (changing them is both visible and detectable), spread across coefficient positions, and wrap everything in **Reed–Solomon** so a run of damaged bits doesn't take the message with it.

Being straight about what this does and doesn't buy you:

- ✅ Survives being re-saved at similar quality — email pipelines, cloud drives, Discord "send as file", most CMSes.
- ⚠️ *Might* survive a platform's re-encode, depending on their quality setting.
- ❌ **Will not survive resizing.** Nothing will. Instagram and WhatsApp resize aggressively, and no steganography that hides in a specific coefficient grid survives having the grid resampled. Anyone promising "survives social media" is selling something.

Needs a JPEG encoder you control — `jpeg-js` or mozjpeg via WASM — which is the first real dependency the project would take on, and it costs the "zero external requests, single file" property unless you inline it.

**Effort:** a weekend, honestly more. **Payoff:** the real *outguess*-tier upgrade, and the difference between "email this carefully" and "just send it".

---

## Supporting work these depend on

### Reed–Solomon error correction

Needed by both the duress passphrase and JPEG survival, and worth having alone — it means light damage degrades the message instead of destroying it. Well-trodden ground with good small implementations. **Half a day.**

### Format spec + a tiny Python CLI

Write the payload format down, then implement it independently. Three things fall out: people can script it, they can verify your work, and they can decode their images in ten years when this page is gone. That longevity is itself a privacy property — and an independent implementation is the strongest trust signal a project like this can offer. **A day.**

---

## Make it harder to detect

### Adaptive embedding

Bits currently land anywhere, including flat sky, where a ±1 change is statistically loud. Modern steganography (WOW, S-UNIWARD) hides only in busy regions — edges, foliage, texture, noise. Weighting the scatter by local variance is a simplified version of the same idea and the biggest real security upgrade available. **A day or two.**

### Matrix coding

Embed *k* bits while changing roughly one pixel, instead of one bit per change. Fewer modifications, less to detect. Fiddly maths, large payoff. **A day.**

### Length padding

Nobody can find your bits, but the *number* of altered pixels leaks roughly how long the message is. Padding to fixed size buckets hides that. Trade-off worth stating: pad too hard and you touch more pixels, which cuts the other way. **An hour.**

---

## Protect people from their own mistakes

This section is probably worth more real-world privacy than the crypto section.

### Never reuse an image that exists elsewhere

Hide a message in a stock photo or your own Instagram post and anyone can diff yours against the original — the altered pixels light up instantly. This defeats everything else in this document. It deserves a prominent warning, and the fix is delightful: **camera capture on mobile**, one HTML attribute, so you shoot a fresh photo with no counterpart anywhere in the world. **An hour.**

### Surface the metadata already being stripped

Drawing to canvas silently discards EXIF, so the tool *already* removes GPS coordinates, camera serials and timestamps. Say so — "removed: location, device, date" — and warn loudly when the original had GPS. Most people have no idea their holiday photos carry their home address. **Two hours.**

### Flag the format tell

`photo.jpg` in, `photo.png` out is itself suspicious. Worth saying out loud. **Ten minutes.**

### Passphrase generator

The encryption is only as strong as the word chosen, and people choose badly. Two modes, side by side — type your own, or roll the dice:

- **Type your own,** with a live strength estimate, so the field stops being a silent trap.
- **Roll the dice** for a diceware phrase. The EFF long wordlist is 7,776 words — about 12.9 bits per word, so six words is ~77 bits of entropy. Unguessable, and still sayable down a phone line.
- Use `crypto.getRandomValues`, never `Math.random`. This is the one place in the app where a lazy PRNG would quietly undo everything.
- The wordlist adds 40–60 KB inline. Worth it; it must not be fetched.
- Whichever mode, say plainly: **write it down, there is no recovery.**

**An hour.**

### Keyfile instead of a passphrase

Use a *file* as the secret — a song you both own, a specific photo, a book cover. "The key is our wedding photo" is more memorable and far stronger than anything someone will actually type. Hash the file to derive the key. **An hour.**

### An opsec cheat sheet

Short, plain, practical: don't send the image and the word through the same channel; don't reuse a published picture; a fresh photo from your own camera is the strongest carrier. The tool is the easy part — using it without leaking is the part nobody explains. **An hour of writing.**

---

## Make it trustworthy, not just trusted

### Publish a hash of `index.html`, and let people check it

A hosted page can be swapped at any moment — the fundamental weakness of browser crypto. Signed releases plus a "verify this matches" line goes a long way. **An hour.**

### A prominent "save this page and use it offline" button

The strongest version of this tool never touches the network again. The property already exists — it's one self-contained file — people just need telling. **Half an hour.**

### A plain-language threat model

One page: this protects you from someone scrolling your camera roll; it does not protect you from a forensics lab holding the original image. Being conspicuously honest about limits is what separates a real privacy tool from snake oil, and most projects won't do it. **Two hours.**

### Ship the test suite and run it in CI

A full suite already exists — ten engine tests plus a click-through end-to-end in headless Chromium — it just wasn't committed. In the repo with a CI workflow, any future change to the crypto gets caught before it silently breaks every image ever made. **An hour.**

---

## The reason people stay

### LSB visualiser

The cheapest way to make the whole idea click. Take bit 0 of every colour channel and amplify it to full brightness — that layer, normally invisible, becomes a picture in its own right.

- On an ordinary photo it looks like TV static.
- On a **naive** stego image — sequential embedding, the way most tutorials do it — you can often *see* the payload: a block of structure in the top-left corner, ending abruptly where the message ran out.
- On this tool's output it stays static, everywhere, because the bits are scattered across the whole frame. The visualiser is what turns "trust me, scattering matters" into something you can look at.

Add a side-by-side diff against the original — altered pixels lit up — and you've also shown, viscerally, why reusing a photo that exists online is fatal.

Worth building standalone rather than waiting for the full lab below: it's a fraction of the work and delivers most of the "oh, *that's* what's happening" moment. **Half a day.**

### Steganalysis lab

The standout idea. Run real detection attacks against your own output — chi-square, sample-pair analysis, RS analysis — and score it honestly:

> *This image would survive casual inspection but fail a forensic check.*

Show the LSB plane. Show the diff against the original, so people can *see* why reusing a published photo is fatal. A steganography tool that grades its own output and teaches you the attacks against it doesn't exist in friendly form. It makes the project educational, credible, and endlessly pokeable. **A weekend, and worth it.**

### Carrier scoring

Drop in five photos, get them ranked as hiding places — noise, texture, size, flat regions. Teaches the concept by doing rather than explaining. **A few hours.**

### Make the dino react

Small, and it's most of the personality.

- **Blink** on a randomised timer, so he reads as alive rather than looping.
- **Chew** while concealing — jaw moving, a slight squash and settle. He's swallowing your message; let him look like it.
- **Look pleased** when a message comes out. Blush, a little bounce.
- Optionally **doze off** after a few minutes idle.

One rule: gate all of it behind `prefers-reduced-motion`, which the page already respects. Animation that ignores that setting isn't charming, it's an accessibility bug. **Two hours.**

### Nerd panel

Pixels changed, capacity used, entropy before and after, the derivation chain from word to key. Hobbyists love seeing the machinery. **An hour.**

---

## Hooks — try it in ten seconds, tell a friend

### Practice images

Sample photos with a known secret word, so someone can hide-then-reveal without uploading anything of their own. Nearly every stego tool fails at "I don't have a suitable picture handy." **Half an hour.**

### Round-trip proof before download

Decode the output and confirm the message survives *before* handing over the file, then a "try revealing it" button that passes the image straight to the other tab. Someone who watches it work once actually believes it. **An hour.**

### Challenge mode

Hide a message, post the image publicly, let people try to crack it. Puzzle-hunt and ARG people would run with this immediately. **An afternoon.**

### Public key in your avatar

One-click template: drop in a PGP key or contact details, get back a profile picture usable anywhere. Your key travels with your face. *(Depends on arbitrary-file support.)* **An hour.**

---

## Reach

### Delivery preflight

Pick where you're sending it — Signal, WhatsApp, Gmail, Discord, iMessage, AirDrop — and find out whether the message will survive. The single most common reason hidden messages silently vanish, fixed by a small built-in compatibility table. Nothing else does this. **A couple of hours, mostly research.**

### Browser extension

Right-click any image on the web → "check for a message." Turns idle curiosity into a habit. **A weekend.**

### Other carriers, audio first

WAV files hide data beautifully and nobody expects it. "I hid a message in a song" is a better story than "I hid a message in a photo." **A weekend.**

### Translation

This kind of tool matters most to people not reading English, and it's a page of strings. **A day, plus translators.**

### Accessibility pass

Screen reader labels, keyboard-only operation, focus management. Privacy tools should not be for the sighted and mousing only. **Half a day.**

---

## Deliberately not doing

**Self-destructing or expiring messages.** Unenforceable client-side. The recipient has the file and the key; nothing can stop them keeping either. Shipping it would be a lie.

**"Survives Instagram."** Resizing resamples the pixel grid the data lives in. No amount of cleverness recovers from that.

**Anything server-side.** No accounts, no uploads, no analytics, no "share a link" backend. The entire trust story is that there's nothing to trust — one file, doing arithmetic on your own machine. Every server-side feature is a promise someone has to believe. Keeping this list empty is a feature.
