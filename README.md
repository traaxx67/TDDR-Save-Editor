# Construct 3 Save Editor (Browser Console Script) For TDDR

A quick console script for editing values (like money/currency) in the save data of offline **Construct 3** games — originally built for *Trapper: Drug Dealing RPG*, but it should work for any Construct 3 game since they all use the same IndexedDB save format.

> ⚠️ **For editing your own save files in single-player/offline games only.** This does not work against online/multiplayer games or anything with server-side validation — Construct 3's save data lives entirely in your browser's IndexedDB, so this only touches data that's already local to your machine.

## How it works

Construct 3 games store save data in IndexedDB (visible in DevTools under **Application → Storage → IndexedDB**), usually in a database named something like `c3-savegames-<gameid>` with an object store called `keyvaluepairs`. This script opens that database, lets you pick a save key, searches the save object for a number matching your current in-game balance, and overwrites every match with a new value.

## Step 1 — Enable DevTools (if needed)

Some Construct 3 exports ship with DevTools disabled. If Control + F12 doesn't work:

1. Find the game's local files (e.g. `package.json` inside the game's install folder).
2. Open `package.json` and find the line:
   ```json
   "enable-devtools": false
   ```
3. Change it to `true` and save.
4. Relaunch the game.

## Step 2 — Find your save

1. Open DevTools → **Application** tab → **IndexedDB**.
2. Look for a database named like `c3-savegames-<gameid>` → `keyvaluepairs`.
3. Note the exact **key** (save name) shown in the table — you'll need to type it exactly.

## Step 3 — Run the script

Open the **Console** tab and paste in [`edit-save.js`](./edit-save.js) (or copy it from below), then follow the prompts:

```javascript
(async function () {
  const dbName = prompt("IndexedDB database name:", "c3-savegames-54xrl0uia1r");
  const storeName = prompt("Object store name:", "keyvaluepairs");
  const key = prompt("Save key/name to edit (exactly as shown in the table):");
  if (!dbName || !storeName || !key) return console.log("Cancelled.");

  const dbReq = indexedDB.open(dbName);
  dbReq.onsuccess = () => {
    const db = dbReq.result;
    const tx = db.transaction(storeName, "readwrite");
    const store = tx.objectStore(storeName);
    const getReq = store.get(key);

    getReq.onsuccess = () => {
      const raw = getReq.result;
      if (raw === undefined) return console.log("No record found for key:", key);

      const isString = typeof raw === "string";
      let data;
      try {
        data = isString ? JSON.parse(raw) : raw;
      } catch (e) {
        return console.log("Couldn't parse as JSON — raw value:", raw);
      }

      console.log("Loaded save data (expand to inspect):", data);

      const oldVal = parseFloat(prompt("Current money amount (as shown in-game), to search for:"));
      const newVal = parseFloat(prompt("New money amount:"));
      if (isNaN(oldVal) || isNaN(newVal)) return console.log("Cancelled or invalid numbers.");

      const matches = [];
      (function walk(obj, path) {
        if (obj && typeof obj === "object") {
          for (const k in obj) walk(obj[k], path.concat(k));
        } else if (typeof obj === "number" && Math.abs(obj - oldVal) < 0.01) {
          matches.push(path.slice());
        }
      })(data, []);

      if (matches.length === 0) {
        return console.log("No exact numeric match found. Expand 'data' above in the console and look for the field manually — money is sometimes stored as a string.", data);
      }

      console.log(`Found ${matches.length} matching field(s) at:`, matches);

      function setPath(obj, path, value) {
        let cur = obj;
        for (let i = 0; i < path.length - 1; i++) cur = cur[path[i]];
        cur[path[path.length - 1]] = value;
      }
      matches.forEach(p => setPath(data, p, newVal));

      const toStore = isString ? JSON.stringify(data) : data;
      store.put(toStore, key).onsuccess = () => console.log("Save updated — reload/relaunch the game to see it take effect.");
    };
  };
  dbReq.onerror = (e) => console.error("Failed to open database:", e);
})();
```

## Notes

- **Back up first.** Run `store.get(key)` on its own before editing and copy the logged output somewhere, in case you want to revert.
- **If the search finds nothing**, the value might not be a plain number — some games store stats as padded strings (e.g. a value like `"*save name*     5233209.803920033"` found in the `c3-localstorage-<gameid>` database instead of `c3-savegames-<gameid>`). In that case, inspect `data` in the console and edit/`store.put()` that field directly.
- **Database/store names vary per game.** The prompts default to the `Trapper` values but you can type in whatever your target game's DevTools shows under Application → IndexedDB.

## Disclaimer

This is a personal save-editing tool for offline games you own. It doesn't bypass any online protections, DRM, or anti-cheat — it only edits local browser storage. Use responsibly and don't use it against multiplayer or live-service games.

# Much Love From Keta.lol/com & Claude Sonnet 5
