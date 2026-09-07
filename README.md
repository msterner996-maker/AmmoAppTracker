# Armory

An offline log for firearms, ammunition, magazines, range trips, chronograph strings
and NFA paperwork. One HTML file, no build step, no account, no network. Everything
you enter stays in this device's browser storage.

Built from `Firearms_Inventory_v2.xlsm`. This is the prototype half of the plan: get
the data model right and the data entered on a real keyboard, then port to Kotlin
with the export file as the handoff.

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app. No personal data in it. |
| `armory-import.json` | Your inventory, extracted from the spreadsheet. Import once. |
| `manifest.json`, `sw.js`, `icon.svg` | Needed for installing to the home screen. |

`index.html` deliberately contains no serials or counts, so it is safe to put on a
host. Your data lives only in `armory-import.json` and, after import, in the
browser's storage on your device.

## Getting it running

**On the desktop first.** Open `index.html` in Chrome, go to Home, tap **Import my
data**, and pick `armory-import.json`. Do your cleanup here, where there's a
keyboard, then export a backup for the phone. 143 magazines and 32 ammunition lines
are miserable to fix on a phone.

**On the phone.** Two routes:

*Quick, no hosting.* Put `index.html` on the phone, open it in Chrome, import a
backup you exported from the desktop. Works, but there's no home-screen icon and
Chrome treats local-file storage as more disposable than a hosted origin, so export
often.

*Proper, recommended, and required for sync.* Serve the four support files over https and open that URL in
Chrome, then use **Add to home screen**. You get an icon, a full-screen app with no
browser chrome, offline caching through the service worker, and storage on a real
origin that Chrome is far less willing to evict. GitHub Pages works — a repo with
just these files and nothing of yours in them. Any static host or a Tailscale-reachable
box at home works the same way.

*Local server, if you'd rather not host anything:*

```
cd folder-with-these-files
python3 -m http.server 8080
```

Then reach it from the phone at `http://<your-computer-ip>:8080`. Note that
`Add to home screen` and the service worker need https or localhost, so plain http
over the LAN gives you the app but not the installed experience.

## Your three browsers

| Where | Verdict |
|---|---|
| Chrome on the Pixel | Best case. Install to the home screen. |
| Firefox on the PC | Works fully, but lives in a tab. See the caveat below. |
| Chrome on the iPad | **Don't.** Use Safari and add it to the home screen. |

**Firefox on the desktop** runs the offline cache and keeps storage normally, but it
does not install web apps, so don't go looking for an install button. Keep it as a
pinned tab. The one thing to check: if Firefox is set to clear cookies and site data
on exit, it takes this log with it. Add an exception for the site, or lean on sync and
exports. Firefox also puts a permission prompt in front of the persistent-storage
request, so the app asks once and remembers your answer rather than nagging.

**Chrome on the Pixel** is the good case. Menu, then Install app, and you get a
full-screen icon, offline caching, and storage Chrome is reluctant to clear.

**The iPad needs care.** Every browser on iPadOS is WebKit underneath, Chrome
included. WebKit deletes localStorage, IndexedDB and service worker registrations
after seven days of *browser use* without you interacting with the site, and Chrome on
iPad cannot run the offline cache at all. Web apps added to the home screen are exempt
from that purge, and only Safari can add them. So on the iPad: open the address in
Safari, Share, **Add to Home Screen**, and launch it from that icon. The app detects
where it is running and tells you this on the Data screen, and stops warning once it
sees it is running standalone.

Two consequences worth knowing. A home screen web app on iOS keeps its data separate
from Safari, so you sign in there once on its own. And if the iPad does get purged,
sync means you lose the sign-in and the client ID rather than the log, since the log
comes back from OneDrive.

**Setting up each browser** is one tap: on a device that already works, Data >
**Copy setup link for another device**. The link carries the client ID, so on the new
browser you only connect your Microsoft account. Keep that link somewhere; it is also
the recovery path if the iPad is ever wiped. The client ID is not a secret, and the
link contains no account details and no tokens.

**Storage mode** is shown on the Data screen as persistent or best effort, with a
button to ask for persistence. On WebKit the request has to be repeated each launch,
which the app does automatically.

## How it works

**Round counts.** Each firearm carries a receiver lifetime total plus a barrel with
its own count and two reset-able counters, one for barrel cleaning and one for deep
cleaning. Swapping a barrel retires the old one with its count intact and starts the
new one fresh, while the receiver total keeps climbing.

**Range trips replace the paste ritual.** Start a trip, add a line per firearm and
ammunition pairing, and pick which loaded magazines you're taking. After the trip,
enter what came home. Fired equals brought minus returned. Closing out does all of
it in one transaction: deducts from boxed and loose, sets each magazine to what's
left in it, removes the empties, and adds the rounds to every counter on the
firearms involved. If anything would push a count below zero it refuses and tells
you which line to check, instead of writing a negative like the spreadsheet did.
One tap of undo reverses the whole thing.

**Packing magazines for a trip** groups identical magazines together, so you set
"3 of these" with a stepper rather than ticking indistinguishable boxes. A partly
used magazine has a different round count, so it forms its own group and stays
visible instead of hiding among the full ones. The firearm and ammunition pickers
are grouped by caliber.

**Loading magazines** is the loose-to-loaded migration. Pick the ammunition, the
magazine type, rounds each and how many, and the rounds move out of boxed or loose
and into that many magazine records. It won't let you load more magazines than you
own or more rounds than a magazine holds. Unloading sends the rounds back.

**Service milestones** come from your AR schedule at 5,000-round steps through
20,000, measured against receiver rounds. When you pass one it shows on Home until
you mark it done.

**Weight counting** uses your recorded tares (bang box 176 g, plastic can 489 g, grey
Leo bag 40 g). Weigh the full container, give it the weight of ten rounds, and it
estimates a count with an honest range rather than a falsely precise number. Weigh
twenty or fifty rounds instead of ten to tighten it.

**Velocity** gives you average, standard deviation and extreme spread per string, not
just the average the spreadsheet computed. Export produces JSON with shots and stats
per string for MSDope.

## Fix these first

The spreadsheet didn't record everything the app wants, so the import made some
choices you should overrule:

- **26 ammunition lines have every round filed as loose.** The sheet kept one
  combined unloaded figure, so there was nothing to split boxed from loose with. Fix
  the ones you care about under Ammo, using *Add or remove*.
- **14 blocks of loaded rounds, 1,405 in total, are in unassigned magazines.** The
  sheet tracked loaded rounds as a bare number per ammunition type with no magazine
  attached, so there was no way to know it was five Pmags rather than one 160-round
  container. They are flagged "no type" under Mags. Open one and use **Split into
  separate magazines**: pick the type and rounds each, and it becomes the right number
  of real magazines with any remainder in a partial. Round counts are preserved
  exactly. Setting magazine capacities first makes this a two-tap job per block.
- **22 of 38 magazine types have no capacity**, which is worth fixing first because
  capacity feeds the loader default, the overfill check, and the splitter because it was never in the sheet
  except inside a few names. Windowed Pmag, TMAG, CZ, Surefeed, Lancer, Steel, ATF
  Mag, Circle 10 and others.
- **`.22lr Wolf` was showing 42 rounds negative loaded.** Clamped to zero and flagged
  on the line. Worth an actual recount.
- **Cleaning intervals were guessed** at 500 rounds for barrel and 1,000 for deep
  cleaning on every firearm, since the sheet tracked counters but never a threshold.
  Set real numbers per firearm, or 0 to switch a reminder off.
- **CZ75 and 2011 were added with zero counts.** They appear in the sheet's range trip
  table but had no row in the firearms list, so they have no serial and no history.
- **The "Bergara Stock" row at 1,739 rounds** is sitting in the Bergara's retired
  barrels with a note. I couldn't tell whether that's an earlier barrel, an earlier
  stock, or the receiver total. Correct or delete it.
- **8 firearms have no serial** and none of the 14 velocity strings have dates.

## OneDrive sync

Keeps every device in step through a single file in your own OneDrive, at
`Apps/Armory/armory.json`. There is no server in the middle and nothing to pay for.
Register the app once, then Data > **Set up OneDrive sync** walks you through it and
shows the exact redirect URI to paste.

The `Files.ReadWrite.AppFolder` scope confines the app to its own folder. It cannot
see or touch anything else in your OneDrive.

**Use a personal Microsoft account.** A work tenant's OneDrive is administrable by
that organisation, with audit logging and eDiscovery reach. Serial numbers do not
belong there.

**When it syncs.** Changes push a couple of seconds after you make them. It pulls
when you open the app, when you switch back to it, when the connection returns, and
every five minutes while it is open. Nothing ever waits on the network: log a trip
with no signal and it goes up when you are back on data.

**Expect an occasional sign-in.** A browser refresh token is fixed at 24 hours and
cannot be extended, so after a day or two away you will tap through a sign-in. This
is a browser limitation, not a setting. The native build gets 90-day tokens and the
nag disappears.

**Conflicts are never resolved silently.** Every save bumps a revision counter, and
the copy in OneDrive carries the revision it was written at. If OneDrive moved on
while this device also had unsent changes, sync stops and Home shows a warning. You
get both versions side by side with device name, timestamp, firearm count, rounds on
hand, trips and lifetime rounds fired, and three choices: keep this device, take
OneDrive, or export both to files first. Taking the OneDrive copy is undoable.

Uploads use an If-Match check, so if another device writes between this device's
check and its upload, the write is refused rather than landing on top. That turns
into a pull or a conflict prompt, never a silent overwrite.

Name each device under Data > **Name device** so the comparison is readable. The name
is per device and is not synced.

## Backups

Sync is not a backup: it faithfully copies a mistake to every device. Keep exporting.

Data > **Export a backup** any time; it writes `armory-YYYY-MM-DD.json` with
everything in it. That same file imports on any device running the app, and it's the
format the Kotlin port will read. Get in the habit after each range trip, since
browser storage is durable but not a guarantee. The app asks Chrome for persistent
storage on first run, which helps and is not a promise.

Undo keeps the last 8 changes and survives a reload. It is not a backup.

## Notes for the port

- The whole state is one JSON object; every screen renders from it and every action
  mutates it and saves. Straight mapping to Room entities: firearms with an embedded
  current barrel and a retired-barrel list, ammunition, magazine types, magazine
  loads, trips with entries, velocity strings, NFA items, containers, schedules.
- The closeout transaction in `ACT["closeout-go"]` is the piece worth porting
  carefully. It validates every stock delta before writing anything, then applies
  stock, magazines and firearm counters together.
- Undo is whole-state snapshots rather than reversible operations. That's cheap here
  and would be worth keeping.
- Bump `CACHE` in `sw.js` whenever you edit `index.html`, or the service worker will
  keep serving the old build offline.
- Auth is the OAuth 2.0 authorization code flow with PKCE, spoken directly to the
  consumer endpoint. MSAL was left out on purpose: around 200 KB from a CDN would
  have broken the promise that the app opens with no network. The same app
  registration works for the native build, which can use MSAL properly and gets the
  longer token lifetime.
- The revision counter and the If-Match upload are the two pieces that make sync
  safe. Port both.
