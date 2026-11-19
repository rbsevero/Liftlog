# LiftLog

Personal workout tracker. One file (`liftlog.html`), no build step, no account. Data lives in your browser's IndexedDB via Dexie.js; charts use Chart.js. Both load from cdnjs.

## Run locally
Serve the folder over HTTP (IndexedDB works best off `file://`):

    python3 -m http.server 8000   # then open http://localhost:8000/liftlog.html

## Deploy
Upload `liftlog.html` to any static host (Netlify Drop, GitHub Pages, Cloudflare Pages, Vercel). No server code is needed.
Data is per browser and per device. Export a backup regularly (Profile → Export backup).

## Data model (Dexie database `liftlog`, version 1)
| Table | Key | Contents |
|---|---|---|
| `w` workouts | `id`, index `start` | `{id, name, start, end, note, src, ex:[{id, n, note, sets:[{w, r, done, warm}]}]}` times are epoch ms; `w` is kg; `src` is `app`, `hevy` or `sample` |
| `e` exercises | `id` | `{id, name, m (muscle), q (equipment), custom?}` |
| `r` routines | `id` | `{id, name, items:[{id, n, sets, reps, w}]}` |
| `bw` body weight | `date` (YYYY-MM-DD) | `{date, t, kg}` |
| `iv` interval sessions | `id`, index `start` | `{id, start, end, name, work, rest, rounds, ex}` (database version 2) |
| `a` activities | `id`, indexes `start`, `type` | `{id, type: walk/run/cycle, start, dur (s moving), dist (m), elev (m), cal, steps, max (m/s), src: gps/manual/gpx, route:[[lat, lon, moving-seconds, altitude]]}` (database version 3) |
| `k` key-value | `k` | `a` = workout in progress, `g` = GPS activity in progress, `s` = settings, profile and goals |

Workouts store the exercise name (`n`) next to its id, so history stays readable if a library entry is renamed or deleted. Routines are separate from history. To move to Supabase/Postgres later, each table maps to one SQL table and `src` + `start|name` is the de-duplication key.

## Exercise info and suggested routines
Tap ⓘ next to any exercise (in the workout, the picker, or Profile → Browse exercise library) for instructions, secondary muscles, a muscle map and your own history. Routines → Suggested routines offers templates for muscle gain, strength, fat loss and general fitness. They are general starting points. Adding one copies it into your routines with weights taken from your last logged sets (blank if none).

## Calculations
- Working sets and volume count completed sets only. Warm-up sets are excluded unless you enable the option in Profile.
- Volume = Σ weight × reps. Estimated 1RM (Epley) = weight × (1 + reps / 30).
- Streak = consecutive calendar weeks (Mon–Sun) with at least one workout.
- Previous performance always comes from your stored workouts. Nothing is invented.

## Import from Hevy
Profile → choose a Hevy `.json` or `.csv` export. You get a preview with counts of new workouts, sets, duplicates and invalid records. Nothing is saved until you confirm, and existing workouts are never overwritten. Duplicates are detected by start time plus workout name. Exercise names are matched to the library ignoring brackets and punctuation (so `Bench Press (Barbell)` matches). Unmatched names become custom exercises. Weights are assumed to be kg.
The parser was written for common Hevy layouts without a real export to inspect. If your file is rejected or mis-read, share it and adjust `norm()` in the script.

## Known limitations
- Offline install needs self-hosting (see below). The hosted preview cannot register a service worker.
- Timed or distance sets with no reps (plank, cardio) are skipped on import and reported.
- Exercise "images" are generated muscle maps (front and back body, main and secondary muscles highlighted), not photos or movement demos. The built-in 39 exercises have written instructions. Custom exercises get an optional instructions field but no secondary muscles.
- Kilograms only. Charts show top weight and est. 1RM per workout, not per set.
- Not yet tested across browsers. Safari date parsing of Hevy CSV dates may reject some rows (they are reported as invalid).

## Activities (walking, running, cycling)
Activity tab: start live GPS tracking or log manually. Records, history filters, goals, achievements and calendar dots (strength, walk, run, cycle, interval) all read from the `a` table. Export a single route as GPX from its detail card, all activities as CSV from Profile, and import GPX files (type is guessed from the file or your average speed and can be changed afterwards). JSON backups include activities, intervals, routines, goals and profile. Older backups without these still restore. Existing databases upgrade from version 1 or 2 to 3 by adding stores only, so no records are touched.

### GPS: what to expect
- Permission is requested only when you tap Start GPS tracking. If denied, you get a message and manual logging.
- Fixes with accuracy worse than 40 m, jumps faster than a person or bike could move, and movement under 3 m are ignored. No fix for 15 s shows "signal lost".
- The unfinished activity is checkpointed to the device every 10 s. After an accidental reload it reappears paused, and you can resume, finish or discard it. Time while the page was closed is not counted.
- Browsers do not guarantee background GPS. Locking the screen or switching tabs can pause location updates and leave gaps. Keep the screen on (the app requests a wake lock where supported).
- The route is drawn as a line (SVG), not on map tiles. A tile map (Leaflet + OpenStreetMap) needs external stylesheets and images, which the hosted preview blocks. When self-hosting you can add Leaflet. The route data already stores latitude and longitude.
- Calories are rough estimates (kcal per kg per km by activity, using your latest body weight or 70 kg). Steps are manual entry only.

## Install as an app (self-hosted)
Put `liftlog.html`, `manifest.webmanifest`, `sw.js`, `icon-192.png` and `icon-512.png` in one folder on any HTTPS host (GPS and service workers need HTTPS or localhost). Open `liftlog.html` once online. The service worker then caches the page and the two libraries, so it opens offline. Use the browser's Install / Add to Home Screen option. Data still lives in that browser on that device.
