# Reservations — installable web app (PWA)

A single-user reservation book. Clients and bookings are stored on the phone
itself using IndexedDB. No server, no account, no internet needed after the
first load.

## Files

| File | What it does |
|---|---|
| `index.html` | The whole app — markup, styles, and logic in one file |
| `manifest.json` | Makes it installable, sets the icon and colours |
| `sw.js` | Service worker — caches the app so it opens offline |
| `icon-192.png`, `icon-512.png`, `icon-512-maskable.png` | Home-screen icons |

## Putting it on your phone

The service worker and "Add to Home Screen" only work over **HTTPS**. Opening
`index.html` by double-clicking it will not install. Pick any static host:

- **GitHub Pages** — push these files to a repo, Settings → Pages → deploy from
  `main` branch, root folder. Free, HTTPS included.
- **Netlify Drop** — drag the folder onto netlify.com/drop. No account needed
  for a test.
- **Your own hosting** — upload the folder to any directory. Nothing to install
  on the server; these are static files.

### Install

- **Android (Chrome)** — open the URL, menu ⋮ → *Add to Home screen* → *Install*.
- **iPhone (Safari only)** — open the URL, Share button → *Add to Home Screen*.
  Chrome on iOS cannot install it; use Safari.

On iPhone, installing to the home screen is not optional. Safari deletes stored
site data after about seven days without a visit. Installed apps are exempt from
that rule. Do not skip this step.

## Local testing

```bash
cd this-folder
python3 -m http.server 8080
```

Then open `http://localhost:8080`. `localhost` counts as a secure origin, so the
service worker registers normally there.

## Things you will want to change

| What | Where |
|---|---|
| Default country code | `COUNTRIES` array in `index.html` — first entry is the default |
| Add or remove countries | Same array, format `["+966","🇸🇦"]` |
| Default booking length | `value="${r ? r.duration : 60}"` in `resSheet` |
| Colours | The `:root` block at the top of the `<style>` section |
| Any wording, either language | The `T` object — `T.en` and `T.ar` |
| Cache version after an edit | `const CACHE = "reservations-v1"` in `sw.js` — bump it, or phones keep serving the old copy |

That last row is the one people forget. If you change the app and your phone
still shows the old version, you didn't bump the cache name.

## Changes in v9

**Morning reminder via the phone's Calendar.**

A button adds the next **14 days** to the phone's calendar as one summary event
per day that has bookings:

- Fires at **6:00 am** on the day itself
- Title shows the count — `حجز واحد اليوم` / `حجزان اليوم` / `٣ حجوزات اليوم`
  (Arabic has a dual form, so 2 is not "٢ حجوزات")
- Details list each booking: time — client — service
- Cancelled bookings are left out; empty days produce no event

Two places to tap it: a card in the Backup tab, and an amber banner at the top
of the Upcoming tab that appears **only** when bookings have changed since the
last export.

### Why a calendar event and not a push notification

A web push needs a server to send it, and that server would have no idea what
the bookings are — they live only on the phone. The Calendar app already does
this job, for free, offline, and with no permissions to grant.

### Implementation notes

- The `.ics` is written by hand because the format is unforgiving: CRLF line
  endings, lines folded at **75 octets** (counted in bytes, not characters —
  Arabic is multi-byte and folding by character length produces a file readers
  reject), and `\` `;` `,` and newlines escaped.
- `DTSTART` is floating local time, so 6:00 am means 6:00 am wherever the phone
  is, with no timezone table in the file.
- Each day's `UID` is `resday-YYYYMMDD@reservations.app` and `SEQUENCE`
  increments every export, so re-adding **updates** the existing events rather
  than creating duplicates.
- Staleness is detected by fingerprinting the bookings in range, which catches
  deletions — a timestamp alone would not.
- Delivery uses the Web Share sheet when the browser supports sharing files
  (the route that reaches Calendar on iOS) and falls back to a download.

Verified against the independent `node-ical` parser, not just self-checks.

### Needs testing on the real iPhone

The share-sheet path cannot be tested here. If tapping the button does not
offer "Calendar" on his phone, say so and I will change how the file is handed
over.

## Changes in v8

Readability pass, from the first real user's feedback: text too small and too
thin.

- **Everything is bigger and heavier.** Base size 16px → 17px, body weight 400 →
  500, headings and names 600 → 700. The small sizes were the real problem —
  nothing is below 0.78rem now (was 0.68rem, about 11px).
- **Muted text is darker** (`--ink-soft` #5A6672 → #475563) for contrast. Thin
  grey text reads as faint even at a larger size.
- **Text size control** in the Backup tab: Normal / Large / Largest. It changes
  the root font size (17 / 19 / 21px) and every size in the stylesheet is in
  `rem`, so the whole interface scales together. The choice is saved with the
  language preference and survives reload.
- Two fixes the larger type exposed: the day rail was a fixed 104px and broke
  "07:00 ص" across two lines (now `7.2rem`, so it scales with the text), and the
  Gregorian and Hijri dates now sit on their own lines instead of wrapping
  unpredictably.

If the user still wants it bigger, raise the three values in the
`html[data-scale]` rules at the top of the stylesheet.

## Changes in v7

**Fixed daily slots.** The day now has exactly four bookable times, one client
each:

| Slot | Time |
|---|---|
| Morning 1 / صباح ١ | 7:00 |
| Morning 2 / صباح ٢ | 9:00 |
| Evening 1 / مساء ١ | 16:00 |
| Evening 2 / مساء ٢ | 18:00 |

- The time dropdowns are gone. The booking form shows four buttons; tap one.
- Before you save, a taken slot is already marked red with the name of the
  client holding it, so most conflicts never reach the save button.
- Saving into a taken slot is **refused**, not warned. The message names the
  client and gives a button that opens their booking, so you can remove it.
- A cancelled booking frees its slot. Done and no-show do not.
- The day header lists all four slots, with used ones struck through.
- A booking stored at any other time (made before this version) keeps its own
  "Other time" chip in the form, so editing it never silently moves it.

To change the hours, edit the `SLOTS` array near the top of the script. Nothing
else needs touching.

## Changes in v6

- **Past-date warning.** Saving a reservation dated before today with status
  "upcoming" now asks first, and on confirm records it as "done" instead. A past
  date with any other status is coherent, so it saves without a prompt. This
  catches a mistyped date, which would otherwise vanish — the Upcoming tab
  filters past bookings out, so the mistake would be invisible.
- **Month mode on the Upcoming tab.** A toggle switches between:
  - *Upcoming* — everything ahead, grouped Today / Tomorrow / This week / Later.
  - *By month* — one month at a time, every booking in it, past and future, all
    statuses. Cancelled ones are dimmed and struck through so a cancellation
    leaves a visible trace. The header shows the month and the booking count.
  The ‹ › arrows in the top bar step months in month mode and days elsewhere.
  Month stepping is anchored to day 1, so moving from the 31st never skips a
  short month.

## Changes in v5

- **Fixed: the time dropdowns were unusable on a phone.** Date and time shared
  one flex row, so the three time selects split half the screen between them.
  After padding and the native dropdown arrow there was no room left for the
  digits — all three rendered as a bare arrow. Date and time now each get a
  full-width row, and the selects carry `min-width:0` so a flex child shrinks
  instead of clipping its own text. Verified at 412px wide: 121px per select.

## Changes in v4

- **Searchable client picker in the booking form.** The dropdown is replaced by
  a type-to-filter box matching name or phone digits, showing both on each row.
  Picking a client collapses it to a single confirmed row.
- **Phone search matches any format.** Both sides are reduced to digits with a
  leading zero dropped, so `0501234567`, `501234567`, `+966501234567` and
  `966 50 123 4567` all find the same client. Applies to the clients tab and the
  booking picker.
- **Phone is required** when saving a client, and still format-validated.
- **Phone is unique.** A duplicate blocks the save, names the client who already
  holds that number, and offers a button to open that record.
- **Location links show their address.** A Google Maps link is decoded to the
  address inside it rather than printed raw. Short `maps.app.goo.gl` links carry
  no readable text and resolve server-side, so those show "Shared location"
  instead. Every subtitle is clamped to one line regardless.
- **`updatedAt` on every record.** Invisible today; it is what last-write-wins
  sync would need, and adding it now avoids migrating existing data later.

### Known limitation

Adding a new client from inside the booking form still discards the reservation
you were typing. Add the client first, then book.

## Changes in v3

- **Upcoming tab** — every future reservation in one scrolling list, grouped
  Today / Tomorrow / This week / Later. Tap a row to edit it; the green button
  goes straight to WhatsApp. The day view is still there for working one day.
- **Day header** — weekday name, a "today" badge when it applies, the Gregorian
  date, and the Umm al-Qura Hijri date. If a browser does not resolve the
  `islamic-umalqura` calendar, the Hijri part is hidden rather than shown from a
  different Hijri variant that could be a day off.
- **Time entry** — hour, minute (00/15/30/45) and AM/PM as three dropdowns, so
  12-hour format is guaranteed on every device instead of following the phone's
  locale. A reservation saved on an off-interval minute (10:20) keeps that
  minute as an extra option, so editing it never moves the time.
- **Location links** — a location that is a full `http://` or `https://` URL now
  opens directly; typed text still goes to Google as a search. Nothing else is
  ever used as a link.
- **Clearer date error** — names the missing date instead of a generic message.
- **Desktop layout** — content capped at 480px and centred above 600px wide.

To change the minute interval, edit the `[0,15,30,45]` arrays in `resSheet`.

## What it does

- One day per screen, bookings pinned to a time rail, arrows to move between days
- Upcoming tab listing everything ahead, grouped by how soon it is
- Tap the date to jump back to today
- Clients: name, phone, location, notes — searchable by name or number
- Phone numbers are normalised to international format on save, because
  `wa.me` links fail silently on badly formatted numbers
- WhatsApp, call, and map links on every booking
- Overlap detection — warns before saving a clashing booking, and flags existing
  clashes on the day view
- Statuses: upcoming, done, cancelled, no-show
- Full English/Arabic switch including right-to-left layout
- Export to a JSON file, restore by merging a file back in

## What it does not do, and why

- **No reminder notifications.** Reliable web push needs a push server and a
  VAPID key pair, and on iOS it only works for an installed app on 16.4+. Not
  worth it for v1 when you said reminders were nice-to-have.
- **No sync between devices.** Single-device by design. Two phones means a
  backend, accounts, and conflict resolution — a different project.
- **No recurring bookings.** Each reservation is a separate record.
- **Adding a new client from inside the reservation form discards the
  reservation you were typing.** The form is replaced, not stacked. Add the
  client first, then book. Fixing this properly needs a sheet stack.

## Storage protection (Backup tab)

On load the app calls `navigator.storage.persist()`, asking the browser to mark
its data as persistent so it is not cleared to reclaim space. The Backup tab
reports the real answer rather than assuming one:

- **Granted** — Chrome usually grants this to an installed app.
- **Not granted** — Safari usually refuses. The card turns amber.
- **Unknown** — the browser does not implement the Storage API.

On iPhone, if you are in a Safari tab rather than the installed app, the Backup
tab shows an amber card telling you to install it. That warning is accurate, not
decorative: Safari deletes IndexedDB, localStorage and service worker
registrations after 7 days without a visit, and only home-screen web apps are
exempt from that rule.

The export card also tracks when you last exported and turns amber after 7 days.

## The risk you are accepting

Browser storage is not permanent storage. Clearing site data, "Clear browsing
data", or deleting the installed app removes everything with no recovery. The
export button exists for exactly this reason. Use it weekly and keep the file
somewhere that isn't the phone.
