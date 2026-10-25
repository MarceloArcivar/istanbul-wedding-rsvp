# Istanbul Wedding RSVP — project brief for Claude Code

## Who you're working with
- **Marcelo**, product manager based in Bonn. He builds personal apps to grow his technical skills and portfolio.
- **He does not write code.** You write all the code; he does the hands-on parts (GitHub, Google account, testing on his phone, sending links).
- He wants to understand the code **at a high level only**, not line by line.
- For anything he hasn't done before (GitHub Pages, Google Apps Script, deploying), give **detailed step-by-step instructions and explain why** each step matters. One step at a time, and wait for him to confirm before moving on when he has to click something.

## What we're building
A personal "can you come?" page for the wedding guests of Marcelo and his partner (shown as **"M & M"**).

- **Wedding:** Istanbul, Türkiye, **August or September 2027** (exact date not set).
- **Guests:** about 80 people, spread across continents, invited per invitation (solo, couple/plus-one, or family).
- **Goal:** learn how many of their closest people can make it to Istanbul on those dates. This is **not** the formal invitation and it does **not** ask about the date or the venue.
- **Reply deadline:** **8 November 2026**.
- **Sent via WhatsApp**, one personal link per invitation.

## Decisions already made (don't reopen without asking)
1. **One personal link per invitation** with a random code, e.g. `https://<user>.github.io/<repo>/#k7f3qx`. Never use names in links (guessable).
2. **Languages:** Spanish, English, Turkish. Each invitation has a default language; guests can switch with ES / EN / TR.
3. **Answers (4 options):**
   - `yes` — "Sí, seguro" / "Yes, definitely" / "Evet, kesinlikle"
   - `likely_yes` — "Probablemente sí" / "Probably yes" / "Büyük ihtimalle evet"
   - `likely_no` — "Probablemente no" / "Probably not" / "Büyük ihtimalle hayır"
   - `no` — "No será posible" / "Sadly, no" / "Maalesef hayır"
4. **Party size:** for invitations covering more than 1 person, when the answer is `yes` or `likely_yes`, the guest picks **how many will come** (1 … invited size).
5. Guests can **change their answer** later from the same link.
6. **Headcount** is shown as a range: confirmed (`yes`) up to confirmed + `likely_yes`. Plan with the upper number.

## Architecture
```
Guest's phone ──(opens link)──▶ GitHub Pages (static page, public repo)
       │                                  │
       │                    fetch guest by code / send answer
       ▼                                  ▼
                     Google Apps Script web app ("the messenger")
                                          │
                                          ▼
                     Google Sheet in Marcelo's Drive ("the replies list")
```
- **Page:** static HTML/CSS/JS on **GitHub Pages**. No build step, no framework.
- **Messenger:** **Google Apps Script** web app bound to the Sheet. Deployed as *Execute as: Me*, *Who has access: Anyone*.
  - `GET ?code=XXXX` → returns only that invitation: `{name, size, lang, status, count}`; unknown code → error.
  - `POST` `{code, status, count}` → validates (code exists, status in the 4 values, `0 ≤ count ≤ size`, count = 0 for `likely_no`/`no`), writes the reply, returns ok. Use `LockService` to avoid write clashes.
  - Send POST bodies as `text/plain` (JSON string inside) to avoid a CORS preflight, which Apps Script doesn't handle. Apps Script responses redirect to `googleusercontent.com`; `fetch` follows that automatically.
- **Sheet tabs:**
  - `Guests`: `code | name | size | lang | status | count | updated_at | message | whatsapp_link`
  - `Log`: every submission appended (timestamp, code, status, count), so changes are traceable.
  - `Summary`: formulas for people invited, yes, likely yes, likely no, no, no reply, and the headcount range.
- **The replies dashboard lives in the Sheet**, not on the public site. The preview's "Replies" view and the "Preview / See it as" bar must **not** ship.

## Privacy rules (important)
- The repo is **public** (GitHub Pages free plan). **Never commit the guest list, codes, phone numbers, the Sheet, or exports.** `guests/` is git-ignored except `guests.example.csv`.
- The page must **not contain the guest list**. It fetches only the one invitation matching the code.
- Codes: random, 6+ characters, unambiguous alphabet (no 0/O/1/l). Generate in the Sheet/Apps Script, not in the repo.
- Collect only: name on invitation, answer, count, timestamp. Nothing else (Marcelo is in Germany, GDPR mindset: minimal data).
- The Apps Script web app URL will be in the page; that's fine. It can only read one invitation by exact code and write validated answers.

## Design (approved — see `design/approved-preview.html`)
Open `design/approved-preview.html` in a browser: it is the **approved look and copy**, with sample guests and a local-only demo of replies. Reuse its CSS, structure and texts; replace its sample data and localStorage with the real fetch/POST.
- **Top:** white sky band with "NOS CASAMOS / WE'RE GETTING MARRIED / EVLENİYORUZ" and the **M & M** monogram (navy, gold italic "&"), then the Istanbul skyline illustration `assets/istanbul-skyline.jpg` fading into the navy page.
- **Couple photo** `assets/couple.jpg` in an **arched frame** (mosque-window shape) with a **blue ring** (`#4f7cae`), overlapping the top of the cream card.
- **Blue flourish** divider under the photo; small **tulip** divider inside the card.
- **Card:** cream paper (`#fbf6ec`), navy text, fonts **Bodoni Moda** (display) + **Jost** (body) from Google Fonts.
- **Page background:** navy `#232f41`.
- Must look great **on phones first** (guests open it from WhatsApp).
- The approved preview embeds the images as base64; in the real site load them from `assets/` instead.

## Copy (final, in the preview)
English version of the main text:
> We're celebrating our wedding in Istanbul in August or September 2027. We know it's a long trip, and we'd love to know which of our closest people might be able to join us in Istanbul on those dates.

Bottom note: "This isn't the formal invitation yet. We'll send all the details once everything is confirmed."
Spanish and Turkish versions are in the preview's `T` object. Keep all three in sync when copy changes.

## Open items
- [ ] Partner's name (monogram currently "M & M"; alt text says "Marcelo and his partner").
- [ ] Real guest list: name on invitation, size, language (Marcelo will provide; goes only into the Sheet).
- [ ] Higher-resolution couple photo (current is 360×540, compressed).
- [ ] Confirm the skyline illustration's license allows use on a public page.
- [ ] Decide whether the gold "&" and gold diamond next to the date stay gold or become blue.

## Next steps (suggested order)
1. Set up the GitHub repo and push this folder (check `.gitignore` first). Turn on GitHub Pages.
2. Turn the preview into the production `index.html` + `assets/` (no preview bar, no dashboard, no sample data); show a friendly message for missing/unknown codes and a loading state.
3. Create the Google Sheet (tabs above) and the Apps Script; deploy the web app; put its URL in the page config.
4. End-to-end test with 2–3 fake invitations from Marcelo's phone (all 4 answers, changing an answer, party counts).
5. Generate per-guest WhatsApp messages in the Sheet (language-specific text + link, and a `https://wa.me/?text=` link) so Marcelo can send them quickly.
6. Add Open Graph tags (title, description, `og:image` of the skyline) so the WhatsApp link shows a nice preview card.
7. Load the real guest list into the Sheet, run a final test, send.
