# Tharangam Radio

Spin the globe, land on a city, hear its radio. A website (installable as an app) and an
Android app built from the same code, with a station checker that keeps dead links out.

```
web/        the website (plain HTML/CSS/JS - no build step)
checker/    station checker (Python, standard library only) + tests
app/        Android app (Capacitor 8 wrapper around web/)
.github/    automation: twice-daily check + deploy, Android build
```

## How dead links are kept out

| When | What happens |
|---|---|
| Twice a day (00:17 and 12:17 UTC) | Every station is opened and the first few KB of audio are read. Each failure is retried twice before it counts. |
| A station fails a check | It is **hidden** from the site and app at once, so listeners never see it. It comes back as soon as it passes again. |
| Failing continuously for **72 hours** (and at least 5 checks) | It is **removed** (moved to a "graveyard"). |
| Weekly | Removed stations are re-checked; any that came back are restored. After 90 days they are forgotten. |

The checker treats these as dead, which simple "does the URL respond?" checks miss:
web pages that return 200 instead of audio, empty streams, redirect loops, `.pls`/`.m3u`
playlists whose streams are dead, and empty HLS playlists. It handles old Shoutcast
servers (`ICY 200 OK`) that most HTTP libraries reject.

## Setup (about 15 minutes)

### 1. Website + automatic checks (free, GitHub)
1. Create a GitHub repository named `tharangam-radio` and upload this folder.
2. **Settings → Pages → Source: GitHub Actions.**
3. **Settings → Actions → General → Workflow permissions: Read and write.**
4. **Actions → "Check stations & deploy" → Run workflow.** The first run pulls ~12,000
   popular stations from [radio-browser.info](https://www.radio-browser.info), checks them
   all (10–25 minutes), and publishes the site at
   `https://YOUR-USERNAME.github.io/tharangam-radio/`.
5. Edit `web/js/config.js`: replace `YOUR-GITHUB-USERNAME` in `remoteDataUrl` and `siteUrl`.

Each run's summary (Actions tab) lists how many stations passed, were hidden and removed.

### 2. Android app
- **Easiest:** Actions → "Build Android app" → Run workflow → download the APK from the
  run's Artifacts and install it.
- **Android Studio:** `cd app && npm install && npm run sync && npm run open`.
- **Play Store:** create an upload keystore, add the four `ANDROID_KEYSTORE_*` secrets
  listed at the top of `build-android.yml`, and the same workflow builds a signed `.aab`.
  Change `appId` in `app/capacitor.config.json` *before* your first Play upload — it can
  never change afterwards.

### 3. Your own stations
Add hand-picked stations (e.g. Malayalam stations missing from radio-browser) to
`checker/custom_stations.json` — see `custom_stations.example.json`. `"featured": true`
stations are listed first in their city and get 6 days instead of 3 before removal.

### Run locally
```
python checker/check_stations.py --seed-limit 300 --state state/state.json --out web/data
cd web && python -m http.server 8000      # open http://localhost:8000
python -m unittest checker/test_checker.py -v
```

## Problems found and fixed before release

1. **https sites can't play http streams.** Browsers block "mixed content", and many
   stations still stream over plain http. The checker now tests an https version of every
   http stream and stores it when it works. The website only lists stations it can play;
   the Android app plays both (cleartext + mixed content allowed), and the site's settings
   tell visitors how many extra stations the app has.
2. **Geo-blocked stations would be wrongly deleted.** The checker runs from one country
   (GitHub's US servers). If our check fails but radio-browser's own checkers (in several
   countries) say the station works, it is kept.
3. **A checker-side network outage would wipe the list.** If more than half of all
   stations fail in one run, the run is treated as our problem and no failures are counted.
4. **GitHub silently disables scheduled workflows after 60 days** without commits. The
   workflow makes an empty "keepalive" commit when needed.
5. **Repository bloat.** Committing a multi-megabyte station list twice a day would bloat the repository fast.
   The checker's memory lives on a separate `data` branch as one replaced commit.
6. **Android 14 crash in background playback.** The media-session plugin starts a
   media-playback foreground service but doesn't request the
   `FOREGROUND_SERVICE_MEDIA_PLAYBACK` permission Android 14 requires; added to the app.
7. **CORS breaks playback.** `crossorigin` on the audio element would make most streams
   fail (stations don't send CORS headers); removed.
8. **Security.** Station names, logos and links come from a public database, so all text
   is escaped and only `http(s)` links are used (blocks `javascript:` links).
9. **Pausing live radio** now stops the download (saves mobile data) and reconnects to the
   live point on resume instead of playing stale audio.
10. **Windows uploads lose `gradlew`'s executable bit**, breaking cloud builds; the build
    workflow fixes it.

Tested: 19 checker tests (playlists, ICY, redirects, timeouts, HTML-as-audio, the full
hide → 72h removal → restore cycle, outage safety valve) and browser tests on phone and
desktop sizes (play, dead-station handling, next station, search, saved, Malayalam UI,
Android lock-screen controls simulated). Not testable here, so do it once on a real phone:
the Android build itself, and background playback with the screen off for 30+ minutes.

## Suggestions to make it better

**Before launch**
- **Pick the name and check trademarks.** "Tharangam" (Malayalam for *wave*) is a
  placeholder. Don't copy Radio Garden's name, look or the "green dots" — that's why this
  design uses lamp-light gold on a night sea.
- **Privacy policy + Play data-safety form.** Required by Google Play. Easy here: no
  accounts, no tracking; saved stations stay on the phone.
- **Respect stations' wishes.** Add a contact email so a broadcaster can ask to be removed,
  and keep a `blocklist` (radio-browser data is community-sourced).

**Product**
- **Listener reports:** when a station fails in the app, send an anonymous "failed" ping
  (e.g. a free Cloudflare Worker). Many reports → hide it before the next check.
- **Curated collections:** "Malayalam film songs", "Christian devotional", "News from
  Kerala", "Gulf Malayalam stations". Curation is what makes users stay.
- **Song titles:** many streams send "now playing" (ICY metadata). Showing it needs a small
  server proxy; a big upgrade.
- **Android Auto and widgets**, plus a home-screen "last station" button.
- **Chromecast / Bluetooth car** display already works through the media notification.
- **Second checker location** (e.g. a free Oracle Cloud VM in India) to further cut false
  "dead" results for Indian stations.

**Growth and money**
- Share links open the exact station (`#s=…`) — add Android App Links so they open the app.
- A one-time "supporter" in-app purchase or donations fits better than ads on a radio app
  (ads over audio annoy listeners).
- Submit to F-Droid as well as Play; radio fans use it a lot.

## Credits and licences
Station directory: radio-browser.info (public domain). World map: Natural Earth via
world-atlas (ISC). Map engine: MapLibre GL JS (BSD-3). HLS: hls.js (Apache-2.0).
Font: Anek Malayalam (OFL-1.1). Detailed map tiles: OpenFreeMap / © OpenStreetMap.
