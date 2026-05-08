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
