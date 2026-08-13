# Roadmap

What's planned for Steganosaurus, and what deliberately isn't.

Nothing here has dates attached. It's a hobby project, built in evenings, and a roadmap with deadlines on it would just be a roadmap that's wrong. The order below is real though — it reflects what depends on what.

**Everything on this page is planned, not built.** What exists today is described just below.

---

## Where things stand

The current version does one thing and does it properly: it hides an encrypted text message inside an image, entirely in your browser.

- Your secret word is stretched into a key with PBKDF2-SHA256, and the message is encrypted with AES-256-GCM before it goes anywhere near a pixel.
- The encrypted bits are scattered pseudo-randomly across every red, green and blue least-significant bit in the picture, in an order derived from that same word. Without it, you don't know which bits to even look at.
- A wrong word and an image carrying nothing are indistinguishable. There's no way to probe an image to find out whether it holds something.
- No uploads, no server, no accounts, no analytics, and not one external request. It's a single HTML file that works offline.

**Known limits, stated plainly:** re-compression destroys the message, so the file has to arrive exactly as it left. That rules out most chat apps and social platforms today — see *Surviving JPEG* below. And while the scattering defeats casual inspection, a forensic analyst holding the original image can often tell that *something* was embedded. They still can't read it.

---

## How to read the sizes

| Tag | Means |
|---|---|
| `small` | An hour or two |
| `medium` | Half a day to a day |
| `large` | A weekend or more |

Anything marked **good first issue** is self-contained, needs no cryptography knowledge, and won't collide with other work.

---

## Phase one — foundations and honesty

Small, independent pieces. None of them depend on each other, and together they make the tool safer to use and easier to trust. This is where a new contributor should start.

### Add a version byte to the payload header `small`

The header currently reads `magic · salt · iv · length · ciphertext`. There's no version field.

That matters more than it sounds. The moment the layout changes — and most of phase two changes it — every image made with the older build becomes undecodable, with no useful error. Just a misleading "no message found".

One byte, checked on decode, with a real message when it doesn't match: *"This image was made with a newer version of Steganosaurus."*

This is the single highest-priority item on the page. It costs half an hour now and cannot be fixed retroactively.

### Ship the test suite and run it in CI `small` · **good first issue**

A full suite already exists — ten tests against the encoding engine plus a click-through end-to-end run in a headless browser — it just hasn't been committed. Once it's in the repository with a CI workflow, any future change to the cryptography gets caught before it silently breaks every image anyone has ever made.

### A plain-language threat model `small` · **good first issue**

One page, no jargon: this protects you from someone idly scrolling your camera roll. It does not protect you from a forensics lab holding the original photograph.

Being conspicuously honest about limits is what separates a real privacy tool from snake oil, and most projects won't do it.

### A "save this page and use it offline" button `small` · **good first issue**

The strongest version of this tool is one that never touches the network again — and that's already true, it's one self-contained file. People just need telling, prominently.

### Say what's being stripped from your photo `small`

Drawing an image to a canvas silently discards its EXIF data. The tool is therefore *already* removing GPS coordinates, camera serial numbers and timestamps from every picture that passes through it.

That should be visible — "removed: location, device, date" — with a loud warning when the original contained GPS. Most people have no idea their holiday photos carry their home address.

### Flag the format tell `small` · **good first issue**

Put `photo.jpg` in, get `photo.png` back. That change is itself a little suspicious, and it's worth saying out loud rather than letting people discover it the hard way.

### A passphrase generator `small`

The encryption is only ever as strong as the word someone chooses, and people choose badly. Two modes, side by side:

- **Type your own,** with a live strength estimate, so the field stops being a silent trap.
- **Roll the dice** for a diceware phrase. The EFF long wordlist carries about 12.9 bits of entropy per word, so six words is roughly 77 bits — unguessable, and still sayable down a phone line.

The wordlist gets embedded in the page like everything else; it must never be fetched. And whichever mode you use, the tool should say plainly: write it down, there is no recovery.

### An LSB visualiser `medium`

The cheapest way to make the whole idea click.

Take the last bit of every colour channel and amplify it to full brightness. That layer — normally invisible — becomes a picture in its own right:

- On an ordinary photo, it looks like television static.
- On a **naively** hidden message, the kind most tutorials teach, you can often *see* the payload: a block of structure in the corner, stopping abruptly where the message ran out.
- On this tool's output it stays static everywhere, because the bits are scattered across the whole frame.

It turns "trust me, the scattering matters" into something you can look at. Pair it with a diff against the original — altered pixels lit up — and it also shows, viscerally, why reusing a photo that already exists online is fatal.

### Make the dinosaur react `small` · **good first issue**

Small, and it's most of the personality. Blink on a randomised timer so he reads as alive rather than looping. Chew while concealing — he's swallowing your message, let him look like it. Look pleased when a message comes back out.

One rule: all of it gated behind `prefers-reduced-motion`, which the page already respects. Animation that ignores that setting isn't charming, it's an accessibility bug.

---

## Phase two — bigger capabilities

### Hide any file — images, keyfiles, PDFs `medium`

*Depends on the version byte.*

Today the payload is text. Making it arbitrary bytes turns the tool into something quite different: hide a photo inside a photo, a PDF, a key file.

How it'll work:

- The payload gains a type, a filename, and optional compression. Browsers now do deflate natively, so text and key files shrink considerably before encryption. Already-compressed formats won't move much.
- **Capacity becomes the real constraint.** Three bits per pixel means a 12-megapixel phone photo holds about 4.5 MB — plenty for a key file or a modest document. A 400×400 avatar holds 60 KB, which is a text file and nothing more.
- There's a nice trick for hiding images: the payload picture can be re-encoded to JPEG first. It's cargo rather than carrier, so lossy compression is fine, and it buys an order of magnitude.

Once this lands, "put your public key inside your profile picture" becomes a one-click template.

### Reed–Solomon error correction `medium`

Nobody asks for error correction. But two of the most interesting features on this page collapse without it, and on its own it means light damage degrades a message instead of destroying it outright.

### A duress passphrase `medium`

*Depends on error correction.*

Two messages in one image, under two different words. The real one under yours; an innocuous one under a word you could hand over. No way for anyone to prove a second exists.

This works because the hiding positions are derived from the passphrase, so two payloads land in essentially unrelated places. The subtlety is that they *will* overlap — and AES-GCM is all-or-nothing, so a single corrupted bit means a message that never decodes at all. The fix is to write the decoy first, let the real message win every collision, and protect the decoy with error correction. Hence the dependency.

Done properly this is real plausible deniability rather than the decorative kind.

### A format specification and a small Python CLI `medium`

*Depends on the format settling first.*

Write the payload format down, then implement it a second time, independently.

Three things fall out. People can script it. They can check the browser version actually does what it claims. And they can still decode their images in ten years, when this page is long gone. That longevity is itself a privacy property — and an independent implementation is the strongest trust signal a project like this can offer.

---

## Phase three — reach

### Surviving JPEG `large`

The honest weakness: any re-compression destroys the message, which rules out most of the ways people actually send pictures.

Fixing it means giving up raw pixel bits and hiding in the frequency-domain coefficients that JPEG itself stores, wrapped in error correction so a run of damage doesn't take the message with it.

What that would and wouldn't buy:

| | |
|---|---|
| **Yes** | Survives being re-saved at similar quality — email, cloud drives, sending as a file attachment, most content systems. |
| **Maybe** | A platform's own re-encode, depending entirely on their quality setting. |
| **No** | **Resizing.** Nothing survives resizing. Instagram and WhatsApp resample aggressively, and no technique that hides in a specific coefficient grid lives through having that grid rebuilt. |

Anyone promising you "survives social media" is selling something.

This is also the first feature that would need a real third-party dependency — a JPEG encoder — which costs the project its single-file, zero-external-requests property unless it's embedded directly.

### Translation `medium`

This kind of tool matters most to people who aren't reading English, and the interface is only a page of strings. Deliberately scheduled late: every string still changing is a string translators would have to redo.

### An accessibility pass `medium`

Screen reader labels, keyboard-only operation, proper focus management. Privacy tools shouldn't be for the sighted and mousing only.

### A browser extension `large`

Right-click any image on the web, check whether it's carrying a message. Turns idle curiosity into a habit.

### Other carriers, starting with audio `large`

WAV files hide data beautifully and nobody expects it. "I hid a message in a song" is a better story than "I hid a message in a photo."

---

## Under consideration

Good ideas without a slot yet. Opinions welcome — open an issue if one of these matters to you.

**Harder to detect.** Bits currently land anywhere, including flat sky, where a small change is statistically loud. Modern techniques hide only in busy regions — edges, foliage, texture. There's also a coding trick that embeds several bits per pixel changed rather than one, which means far less to detect. Between them, these are the biggest genuine security upgrades available.

**A steganalysis lab.** Run real detection attacks against your own output and get an honest score: *this would survive casual inspection but fail a forensic check.* A steganography tool that grades its own work and teaches you the attacks against it doesn't really exist in friendly form.

**Carrier scoring.** Drop in five photos, see them ranked as hiding places. Teaches the concept by doing rather than explaining.

**A key file instead of a passphrase.** Use a file as the secret — a song you both own, a specific photo. "The key is our wedding photo" is more memorable, and far stronger, than anything most people would type.

**Delivery preflight.** Choose where you're sending the image and find out whether the message will survive the trip. The most common reason hidden messages silently vanish.

**Practice images.** Samples with a known secret word, so anyone can try hiding and revealing in ten seconds without uploading a photo of their own.

**Round-trip proof.** Decode the output and confirm the message survives *before* handing over the file.

**Challenge mode.** Hide a message, post the image publicly, let people try to crack it.

**A nerd panel.** Pixels changed, capacity used, and the derivation chain from word to key, for people who like seeing the machinery.

**An opsec cheat sheet.** Don't send the image and the word through the same channel. Don't reuse a picture that exists online. A photo you just took yourself is the strongest carrier there is. The tool is the easy part; using it without leaking is the part nobody explains.

---

## Deliberately not doing

**Self-destructing or expiring messages.** Unenforceable. The recipient holds the file and the key, and nothing can stop them keeping either. Shipping it would be a lie told with a progress bar.

**"Survives Instagram."** Resizing resamples the pixel grid the data lives in. No amount of cleverness recovers from that, and claiming otherwise would get someone's message lost when it mattered.

**Anything server-side.** No accounts, no uploads, no analytics, no link-sharing backend. The entire trust story is that there's nothing to trust: one file, doing arithmetic on your own machine, with nothing to intercept and no logs to subpoena. Every server-side feature is a promise someone has to take on faith. Keeping this list empty is a feature.

---

## Contributing

Issues and pull requests welcome. The **good first issue** items above are self-contained and need no cryptography background.

Two things to know before touching the encoding:

1. **Don't change the payload layout without adding the version byte first.** Every image ever made with this tool depends on that format.
2. **The scatter salt is a constant, not a label.** Change one character of it and every previously encoded image stops decoding, permanently.

There's no build step, no dependencies and no `node_modules`. Open `index.html`, edit it, refresh.
