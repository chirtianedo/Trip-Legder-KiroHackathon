# Trip-Legder-KiroHackathon
Turn phone screenshots and GPS traces into structured field-research trips.

# Trip Ledger

**Turn phone screenshots and GPS traces into structured field-research trips.**

Built for [Kiroween 2026](https://kiroween.devpost.com/) · Category: **Frankenstein** · Built with [Kiro](https://kiro.dev)



---

## The problem

In Abuja, informal cabs routinely carry more people than the vehicle is rated for. A sedan built for 4 to 5 occupants often runs with 7, including two passengers in the front seat. Fares and waiting times also differ a lot between **park** cabs (which wait to fill before leaving) and **along** cabs (which pick up passengers on the road).

Measuring this properly takes trip-level data: route, boarding type, fare, headcount, times and vehicle. The hardest part to collect is **time**. An observer riding a cab cannot stop to log timestamps while working, travelling or holding a bag, so manual logs end up late and unreliable.

## What Trip Ledger does

Trip Ledger rebuilds each trip afterwards from evidence the observer's phone already holds:

- a Google Maps screenshot taken when boarding and another when alighting
- an optional GPS trace from a background logger
- the file's own name and metadata

From that evidence it extracts times, ETA and distance, pairs boarding with alighting, and asks the observer only for what a phone cannot see (headcount, fare, vehicle) in a short follow-up form. A rules engine then measures overloading against rated capacity, and a dashboard turns the trips into research data.

Two principles run through the whole project:

1. **Every captured value stores its source and a confidence level.** The app never silently guesses.
2. **Uncertain values go to a review queue**, where a person confirms or corrects them with one tap.

## Why this is a Frankenstein

Five things that were never designed to work together, stitched into one tool:

| Piece | Role |
|---|---|
| Google Maps screenshots | Evidence of boarding time, ETA and distance |
| Background GPS traces | Evidence of when the vehicle actually moved and stopped |
| OCR on screenshot regions | Reads the status-bar clock and the Maps ETA panel |
| Vehicle capacity rules | Rated seats per model, used to measure overloading per trip |
| An installable Android web app with an offline outbox | Lets observers capture on ordinary phones with unreliable data |

## What is new here

Related tools exist for parts of this: smartphone travel diaries that detect trips and ask the person to confirm them (for example MEILI and MoveSmarter), Android apps for on-board transit data collection (Flocktracker), rider-collected GPS mapping of informal routes (Digital Matatus), and generic offline survey tools (KoboToolbox, ODK). Trip Ledger builds on the same prompted-recall idea.

What it adds:

- **Screenshots as an evidence source.** Google's ETA is a no-waiting, no-stops estimate, so the gap between ETA and actual duration exposes waiting-to-fill time and stops on the road.
- **Overload measured per trip against rated seats**, including the front-seat split, which is the signature of Abuja overloading.
- **Field-study ergonomics.** Capture costs one screenshot; everything else happens later.



| Module | Status |
|---|---|
| `trip-schema`: canonical schema, CSV import and export, capacity table | Planned |
| `evidence-intake`: upload, Android share target, offline outbox, de-duplication | Planned |
| `time-extraction`: filename, status-bar clock, Maps ETA and distance | Planned |
| `trip-reconstruction`: pair boarding with GPS movement, incomplete-trip state | Planned |
| `rules-engine`: overload, fare per head and per km, ETA gap, wait time | Planned |
| `review-and-dashboard`: follow-up form, review queue, dashboard, demo mode | Planned |



## How it works

```
 Android phone                               Server (Python / Django)
 -------------                               ------------------------
 Maps screenshot  --+
 GPX trace        --+--> offline outbox --> 1. Intake        (hash, de-duplicate)
 one-tap buttons  --+    (queued locally,   2. Extraction    (filename, status-bar
                          uploaded when                       clock, Maps ETA/distance)
                          online)            3. Reconstruction (pair boarding and
                                                                 alighting, trim GPX)
                                             4. Rules engine  (overload, fares, gaps)
                                             5. Review queue  (low-confidence values)
                                             6. Dashboard + CSV export
```

### Time sources, in order of trust

1. Timestamp in the screenshot **filename**
2. **Status-bar clock** read from the screenshot itself
3. File **last-modified** time (low confidence, because sharing can reset it)

The upload time on the server is never used as event time. If two sources disagree, the trip is flagged for review.

### Two boarding types

| Type | Meaning |
|---|---|
| Park | Fixed fare, cab waits at a park until full, then departs |
| Along | Cheaper, picks up passengers on the road, capped below full until a seat frees |

## Data model

The canonical schema follows the columns of the original field log, so real trips can be imported unchanged. Every captured field also gets `<field>_source` and `<field>_confidence`.

| Group | Fields |
|---|---|
| Trip | trip ID, observer ID (pseudonymous, for example `OBS-01`), date, day of week |
| Route | axis / route, pickup point (named landmark, not coordinates), boarding type |
| Times | expected time of arrival (Maps ETA), time of movement (departure), time dropped (arrival), wait time before motion, actual trip duration |
| Distance | distance covered (km), flagged if Maps and GPS disagree |
| Vehicle | brand / model, year |
| Load | front and back seat passengers at entry and closing, total passengers at entry and closing |
| Road | number of stops, passengers picked on the road |
| Fare | price of trip (NGN) |

Derived fields: `rated_passenger_seats`, `overload_count`, `front_overload`, `fare_per_head`, `fare_per_km`, `eta_gap_min`.

### Definitions that every number depends on

- **Rated passenger seats** means seats excluding the driver. A 5-seat sedan has 4.
- **Overload count** = `max(0, passengers − rated_passenger_seats)`.
- **Front overload** = `max(0, front passengers − rated front passenger seats)`.
- **Wait time** = departed − boarded.
- **ETA gap** = actual duration − Google's ETA.

The capacity table is editable data, not code, and each entry cites its source.

## Privacy and ethics

- **No raw screenshots or GPX files are committed or exported.** A hook blocks them from being committed.
- **GPS traces are trimmed to the trip window**, derived times are stored, and the raw trace is discarded. Traces can reveal where an observer lives or works.
- Screenshots are cropped to the regions that matter, then discarded after extraction.
- No plate numbers, faces, or exact coordinates. Pickup points are named landmarks.
- Observer identifiers are pseudonymous, and **CSV export strips them by default**.
- Observers consent before using the app, and the field protocol is in [`docs/field-protocol.md`](docs/field-protocol.md).


## License

[MIT](LICENSE). 
