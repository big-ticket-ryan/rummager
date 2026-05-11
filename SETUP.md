# Rummager — Setup & Publish

Goal: get **https://big-ticket-ryan.github.io/rummager/** live so you (and your sister) can use Rummager from any phone over HTTPS.

You only do this once. After that, updates are a `git push`.

---

## One-time setup

### 1. Create the repo on GitHub

1. Go to https://github.com/new
2. **Repository name:** `rummager`
3. **Owner:** `big-ticket-ryan`
4. **Public** (required for free GitHub Pages).
5. Leave everything else default. Click **Create repository**.

### 2. Push the file (one-time, from this workspace)

From PowerShell in the foundation root:

```powershell
# Make a fresh local clone next to the foundation
cd C:\TestDev
git clone https://github.com/big-ticket-ryan/rummager.git
cd rummager

# Copy the app in as index.html so the URL is just /rummager/
Copy-Item C:\TestDev\ftoptix-foundation\local-temp\rummager.html .\index.html

# (Optional) drop the plan and this setup doc in too for posterity
Copy-Item C:\TestDev\ftoptix-foundation\local-temp\rummager-plan.md .\PLAN.md
Copy-Item C:\TestDev\ftoptix-foundation\local-temp\rummager-SETUP.md .\SETUP.md

git add .
git commit -m "Initial Rummager v1"
git push origin main
```

### 3. Enable GitHub Pages

1. In your browser go to https://github.com/big-ticket-ryan/rummager/settings/pages
2. Under **Build and deployment → Source**, choose **Deploy from a branch**.
3. **Branch:** `main`. **Folder:** `/ (root)`. Click **Save**.
4. Wait ~30–90 seconds. The page will show a green box with your URL:
   **https://big-ticket-ryan.github.io/rummager/**

That's it. Open it on your phone, "Add to Home Screen", you're done.

---

## Sharing with your sister

Just text her the URL: **https://big-ticket-ryan.github.io/rummager/**

She does NOT need a GitHub account. She does NOT need to install anything. Tap → opens in Chrome → "Add to Home Screen" if she wants it as an app icon.

Her data is independent of yours (separate browser localStorage). No conflicts possible.

---

## Updating later

Edit `local-temp/rummager.html` in the foundation, then:

```powershell
cd C:\TestDev\rummager
Copy-Item C:\TestDev\ftoptix-foundation\local-temp\rummager.html .\index.html -Force
git add index.html
git commit -m "Rummager update: <what changed>"
git push
```

GitHub Pages republishes automatically in ~30 seconds. **The URL never changes.**

---

## First-launch behavior (what to expect)

1. **First visit**: app loads, ~9 stops show on the map immediately (the ones with hand-corrected coords). You'll see a "Geocoding 56 preloaded stops…" toast at the bottom — it takes about 60 seconds to look up the rest (one per second, OSM rate limit). Coords are then cached in your browser, so this only happens once.
2. After that, all 65 markers stay put across visits.
3. Tap **📍 (header)** → grants location permission → live blue dot.
4. Tap **📍＋ (FAB, bottom-right)** → drops a pin at your current GPS, prefills the address, status defaults to **Visited**.
5. Tap **＋ (header)** → manual add by typing an address.
6. Tap any pin or list row → opens the detail sheet (status / note / photos / Navigate).

---

## Notes

- **Internet required on first load** of the page (to fetch Leaflet from CDN and tiles). After that, mostly works offline since browsers cache.
- **Permissions you'll be asked for:**
  - Location (for 📍 and 📍＋ features). Says "no" → manual add still works fine.
  - Camera (when you tap "+" inside the photo grid). Says "no" → can still pick from gallery.
- **Storage cap:** ~5 MB. Photos are auto-compressed (~200 KB each) so you can fit ~20+ photos before any warning.
- **No "reset" button.** If you ever want to wipe data, in Chrome: Settings → Site Settings → search `big-ticket-ryan.github.io` → Clear data.

---

## v2.1 — Group location sharing (Firebase)

Group sharing is **opt-in** and **off by default**. The app still works fully without it. To enable, do these one-time steps:

### 1. Create a Firebase project

1. https://console.firebase.google.com/ → **Add project** → name `rummager` → disable Analytics → Create.
2. Project overview → ⚙️ **Project settings** → **Your apps** → click `</>` (Web) → nickname `rummager-web` → **don't** enable Hosting → **Register app**.
3. Copy the `firebaseConfig` object shown (you can also find it later under Project settings → General → SDK setup and configuration → Config).

### 2. Enable Anonymous Auth + Realtime Database

- Left sidebar → **Build → Authentication** → Get started → **Sign-in method** → enable **Anonymous**.
- Left sidebar → **Build → Realtime Database** → Create database → US region → Start in **locked mode**.
- Open the **Rules** tab and paste:

  ```json
  {
    "rules": {
      "groups": {
        "$code": {
          ".read": "auth != null",
          "members": {
            "$uid": {
              ".write": "auth != null && auth.uid === $uid",
              ".validate": "newData.hasChildren(['name','lat','lon','updatedAt'])"
            }
          },
          "meta": { ".write": "auth != null" }
        }
      },
      "publicStops": {
        ".read": "auth != null",
        "$id": {
          ".write": "auth != null && ((!data.exists() && newData.child('createdBy').val() === auth.uid) || (data.exists() && data.child('createdBy').val() === auth.uid) || root.child('admins').child(auth.uid).exists())",
          ".validate": "newData.hasChildren(['addr','lat','lon','createdBy','createdAt']) && newData.child('addr').isString() && newData.child('addr').val().length <= 200 && newData.child('lat').isNumber() && newData.child('lat').val() >= 42.5 && newData.child('lat').val() <= 47.5 && newData.child('lon').isNumber() && newData.child('lon').val() >= -93.0 && newData.child('lon').val() <= -86.0",
          "flags": {
            ".write": "auth != null"
          }
        }
      },
      "admins": {
        ".read": "auth != null && root.child('admins').child(auth.uid).exists()",
        "$uid": { ".write": false }
      }
    }
  }
  ```

  Click **Publish**.

- Authentication → **Settings → Authorized domains** → add `big-ticket-ryan.github.io` if not already listed.

### 3. Paste the config into `index.html`

Find this line near the bottom of the script in `index.html`:

```js
const FIREBASE_CONFIG = null; // e.g. { apiKey: "...", authDomain: "...", databaseURL: "...", projectId: "..." }
```

Replace `null` with the object Firebase gave you, e.g.:

```js
const FIREBASE_CONFIG = {
  apiKey: "AIza…",
  authDomain: "rummager-xxxx.firebaseapp.com",
  databaseURL: "https://rummager-xxxx-default-rtdb.firebaseio.com",
  projectId: "rummager-xxxx",
  storageBucket: "rummager-xxxx.appspot.com",
  messagingSenderId: "…",
  appId: "1:…:web:…"
};
```

Commit and push. Group sharing is now live.

### 4. Using groups

- Tap **👥** in the header.
- Enter your display name.
- Tap **Create new group** → you'll get a 6-char code like `BLG-K9X`. Tap **Invite** to share via your phone's share sheet (or it's copied to clipboard as a fallback).
- Friends tap the invite link or open Rummager → 👥 → enter the code → **Join group**.
- Each member taps **Share my location** to start broadcasting. The blue banner across the top is always visible while you're sharing — tap **Stop** anywhere to stop. Sharing only stops on explicit Stop, GPS being turned off, or closing the tab.
- Members appear as colored dots with their initial on each other's maps, plus a name label.

### Privacy notes

- Sharing is OFF by default and requires an explicit tap.
- Stale members (no update for 5 minutes) drop off the map automatically.
- Anyone with the group code can read positions of members who are sharing — treat the code like a password.
- The Firebase config in the HTML is meant to be public; security comes from the Auth + Database rules above.

---

## v2.2 — Public stops + admin moderation

Anyone using Rummager can submit a public stop (a sale that wasn't on the flyer). All users see public stops on their map with a **purple pin + gold ring** and a yellow **NEW** badge until they tap and decide to **Save**, **Flag**, or **Ignore** it.

### Behavior

- **Add → Save & Share Public**: appears as a third button on the Add Stop sheet (between Save and Cancel). Writes the stop to `/publicStops/{id}` with a 7-day TTL.
- **Tap a public pin**: opens a sheet with the address, who shared it, age, and three actions:
  - **Save to my list** — copies into the user's localStorage as a regular manual stop.
  - **Flag** — increments a `flags` counter. When `flags ≥ 3`, the stop is hidden from everyone's map (3-strikes rule).
  - **Ignore** — local only; just removes the NEW badge for that user.
- The original poster also sees a **Delete (mine)** button.

### Admin (you)

- **Hidden trigger**: tap the `🛒 Rummager` title in the header **7 times within 5 seconds**. A password prompt appears. Password: the one you set during build (hashed in the source).
- After unlocking, an Admin sheet exposes **Wipe ALL public stops**.

### One-time admin bootstrap (required for the wipe button to actually work)

Server security rules only allow cross-user deletes if your auth UID is registered as an admin. Do this once:

1. Open the app on your usual phone/browser. Open the Admin sheet (tap title 7×, enter password). It shows **Your auth UID**: copy it.
2. In the Firebase console → **Realtime Database** → **Data** tab → at the root, hover over the root node → click `+` → key `admins` → value `{}`.
3. Click `+` on `admins` → key your UID (paste from step 1) → value `true`.
4. Done. The Wipe button now works from that device.

> **Caution**: anonymous Firebase UIDs are per-device + per-browser. If you clear browser data or switch devices, you'll get a new UID and lose admin powers until you re-add it. To make admin permanent, link your anonymous account to email/password in the Firebase Auth console (advanced).

### Cleanup behavior

- Each public stop carries `expiresAt = createdAt + 7 days`. Clients filter expired stops out client-side. They still occupy DB space until manually wiped.
- A future enhancement could auto-delete expired entries on app boot for any user (cheap, no Cloud Functions needed). Not implemented in v2.2.

### If it gets out of control

1. Open the app, tap title 7×, enter the password.
2. Tap **Wipe ALL public stops**. Done.
3. Optional: temporarily set the rule for `/publicStops/$id` `.write` to `false` in Firebase to disable all new posts while you investigate.

---

## Fredonia edition � operations notes

### Wiping leftover Belgium public sales (one-time)

The Fredonia rebuild keeps the same Firebase project (ummager-37958), so any old Belgium-area public sales already in /publicStops will appear if their lat/lon happens to fall inside the new Ozaukee bbox. To clear them:

1. Open the live app on a phone you control.
2. Tap the title bar **7 times** to unlock the admin sheet.
3. Use **Wipe ALL public stops** to clear /publicStops/* from RTDB.
4. (Optional) tighten the database rules below so future writes are bbox-validated.

### Recommended Realtime Database rules

In the Firebase Console ? Realtime Database ? Rules, replace the public-stops block with the tighter version below. It validates the new optional fields and constrains the bbox to Ozaukee County:

```json
{
  "rules": {
    "publicStops": {
      ".read": "auth != null",
      "15412": {
        ".write": "auth != null && (!data.exists() || data.child('createdBy').val() == auth.uid)",
        ".validate": "newData.hasChildren(['lat','lon','addr'])",
        "lat":          { ".validate": "newData.isNumber() && newData.val() >= 43.30 && newData.val() <= 43.65" },
        "lon":          { ".validate": "newData.isNumber() && newData.val() >= -88.10 && newData.val() <= -87.65" },
        "addr":         { ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length <= 200" },
        "name":         { ".validate": "newData.isString() && newData.val().length <= 80" },
        "sellerName":   { ".validate": "newData.isString() && newData.val().length <= 40" },
        "items":        { ".validate": "newData.isString() && newData.val().length <= 5000" },
        "area":         { ".validate": "newData.isString() && newData.val().length <= 60" },
        "createdBy":    { ".validate": "newData.isString() && newData.val() == auth.uid" },
        "createdByName":{ ".validate": "newData.isString() && newData.val().length <= 40" },
        "createdAt":    { ".validate": "newData.isNumber()" },
        "updatedAt":    { ".validate": "newData.isNumber()" },
        "pinVerified":  { ".validate": "newData.isBoolean()" },
        "hours":        { ".validate": "newData.hasChildren() || !newData.exists()" },
        "priceSlash":   { ".validate": "newData.hasChildren() || !newData.exists()" },
        "flags":        {},
        "":       { ".validate": false }
      }
    }
  }
}
```

### Map / bbox tuning

If Fredonia residents need a wider catchment, edit two constants near the top of index.html:

- FREDONIA_DEFAULT � initial map center.
- PUBLIC_BBOX � geofence used by both the tile-precache and the publish-sale validator.

### Realtime Database rules � usage stats block

Append this sibling block under the top-level rules object so the anonymous-user counters work:

```json
"publicStats": {
  ".read": "auth != null",
  "totals": { "": { ".write": "auth != null" } },
  "daily":  { "": { "": { ".write": "auth != null" } } },
  "online": {
    "$uid": {
      ".write": "auth != null && auth.uid == $uid"
    }
  }
}
```

(Backticks around $uid are markdown-escapes � use plain $uid in your real rules.)

Remove the row-cap fields if you start hitting Firebase Spark plan caps; an Ozaukee-sized community will fit comfortably.
