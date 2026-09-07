# Installation Guide — Sentry Access Automation

Script file: `sentry-access-automation.user.js`

## 1. Install Tampermonkey

You need the Tampermonkey extension/app for your browser first.

| Platform | Browser | Where to get Tampermonkey |
|---|---|---|
| Desktop | Chrome / Edge / Brave | Chrome Web Store → search "Tampermonkey" |
| Desktop | Firefox | Firefox Add-ons → search "Tampermonkey" |
| Android | Firefox for Android | Firefox → Menu → Add-ons → search "Tampermonkey" (real Tampermonkey add-on works here) |
| Android | Kiwi Browser | Chrome Web Store works directly in Kiwi (Chromium-based) → search "Tampermonkey" |
| iOS / iPadOS | Safari | App Store → install the **Userscripts** app, then enable it under Settings → Safari → Extensions (Tampermonkey itself isn't available on iOS Safari) |

## 2. Add the script

**Option A — paste it in directly**
1. Click the Tampermonkey icon → **Dashboard**.
2. Go to the **Utilities** tab (or the **+** icon) → **Create a new script**.
3. Delete the placeholder template code.
4. Open `sentry-access-automation.user.js`, copy its entire contents, and paste it in.
5. Save (Ctrl+S / Cmd+S, or File → Save).

**Option B — import the file**
1. Tampermonkey Dashboard → Utilities tab → **Import from file**.
2. Select `sentry-access-automation.user.js`.
3. Confirm the install prompt.

## 3. Verify it's active

1. Navigate to any page under `https://sentry-apac.com/`.
2. Click the Tampermonkey icon in the toolbar — you should see **"Sentry Access Automation"**
   listed and enabled (toggle is blue/on), with three menu commands underneath:
   - Open Sentry Menu
   - Toggle Debug Logging
   - Toggle Auto Submit
3. A round orange-bordered floating button (FAB) should appear in the bottom-right corner of
   the page.

If the FAB doesn't appear or the script isn't listed in the Tampermonkey icon's popup, the
script didn't match/load on that page — double check the URL starts with
`https://sentry-apac.com/` and that the script is enabled (not just installed) in the dashboard.

## 4. First run

1. Tap/click the FAB (or use "Open Sentry Menu" from the Tampermonkey icon) to open the room
   menu.
2. Pick a room, then tap **Enter** or **Leave**.
3. Watch the status panel (bottom-left) step through Load Form → Direction → Data Center →
   Park/Campus → Building → Room → Access Control Location → Document Association.
4. Once all steps show ✔, review the form and click **Submit** yourself (unless you've turned
   on Auto Submit in Settings).

If a step fails, use **Retry** to re-attempt it, or **Continue manually** to finish the form by
hand from that point.

## 5. Mobile-specific notes (Firefox for Android)

- Debugging without a desktop: tap the Tampermonkey icon in Firefox's toolbar on the page in
  question — if "Sentry Access Automation" and its menu commands are listed, the script loaded
  correctly and any issue is with a specific step, not the install.
- Full console access: connect the phone via USB, enable USB debugging on the phone, then on a
  desktop Firefox go to `about:debugging#/setup`, connect the device, and inspect the tab
  running the page to see `console.log` output (enable **Debug logging** in the script's
  Settings first to get useful output).

## 6. Adding a room from the FAB (no script editing)

Open the menu and scroll to **Add Room** under Settings. Fill in:

- Data Center (e.g. `MY88`, or a new code to start a new datacenter)
- Campus (e.g. `MY88_C01`) — must match what the real Sentry portal's Campus drawer expects;
  it's used both as the search text for that step and as the menu's display label
- Building (e.g. `A`)
- Room (e.g. `A1-99`)
- Access Control Location — optional door number, defaults to 1

Tap **Add Room**. It's saved on-device and shows up in the menu immediately, grouped the same
way as built-in rooms. Duplicate room codes are rejected. Each added room gets its own **Edit**
and **Delete** buttons next to Enter/Leave: Edit reloads its values into the form (now titled
"Edit Room") so you can change any field and tap **Save Changes** (or **Cancel** to back out);
Delete removes it immediately. Use **Clear added rooms** in Settings if you'd rather wipe all of
them at once.

## 7. Adding/adjusting rooms or datacenters in the script (advanced)

Rooms, buildings, and per-room overrides also live in the `CONFIG` object near the top of the
script, for anyone who'd rather edit the source directly (e.g. to make a room permanent across
devices, since the Add Room form only saves locally on the device you used it on).
To add a room, add its code to the right building's array under `CONFIG.tree` (nested
datacenter → campus → building → room codes). If that room's Access Control Location isn't the
default door 1, add an entry to `CONFIG.doorOverrides`.

To add a whole new datacenter, add a new top-level key under `CONFIG.tree` with its own
`campuses` map — there's a commented example right above `tree` in the script showing the
shape to copy. Confirm the campus/building codes against the real Sentry portal first, the same
way MY88's values were confirmed. Once added, the new datacenter's rooms show up in the menu
automatically, grouped separately from MY88.

After editing, save in the Tampermonkey editor and reload the target page for changes to take
effect.

## 8. Updating the script

Tampermonkey does not hot-reload edits. After changing the script (whether editing directly in
Tampermonkey or re-pasting an updated version):
1. Save in the Tampermonkey editor.
2. Refresh the Sentry page you're testing on.
