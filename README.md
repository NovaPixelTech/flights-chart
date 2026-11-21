# Cheap Flights Finder

Web app that finds the **cheapest direct flight** and the **cheapest flight with at most
2 stops** (stopover ≤ 3 hours) between two cities/airports for a vacation of a given length.
Each trip can be priced as a **single round-trip ticket** or as **two separately-booked
one-ways** (the cheapest one-way out plus the cheapest one-way back — often cheaper, because it
can mix airlines), or **both at once** and shown in their own sections so the two booking styles
are never conflated.
Supports country inputs (expands to **every** commercial airport in that country — all of them
are searched), explicit airport lists (`DUS, FRA, HAM`), flexible date windows, stay-length
ranges, open-jaw (different return airport), a **departure-weekday filter** (e.g. only depart on
Mondays), an optional **direct-only (nonstop)** filter, an optional **maximum ticket price**
(the most you will pay, in the selected currency), and an optional **in-country stopover comparison**
(the cheapest self-transfer via a major airport of your destination country).
Every result lists the **outbound and the return** flight (airport, `dd.mm.yyyy` date with the
weekday, times, airline) plus the number of stay days. The results also carry a small card that
**cross-checks the cheapest options on Google Flights (where Gotogate is listed), Kiwi.com and
Skyscanner**. Any search can also be turned into an **automation** that re-runs itself on your own
schedule, notifies you when each run's results list is ready, and keeps an archive of every run.
Any result set — a manual search or an archived automation run — can be **parked as a saved
search**, kept in its own section and grouped by destination, departure month and the date it was
searched.

By default a search is **exhaustive**: every resolved airport pair, every stay length and **every
departure date that fits the window** is queried, so the cheapest option reported really is the
cheapest on offer (or the cheapest at or under your maximum price, when one is set) — no sampling
that could step over the cheap Sunday. That takes a while on the free reader, so a run is meant to
happen **in the background**: keep using your machine, the tab icon spins and the tab title shows
the progress, and a **browser notification** tells you when the results are ready. Stopping early
needs a deliberate **two-click confirmation**, and an interrupted run can be **resumed** after a
reload; a free reader API key removes the wait.

**Live flight prices. Completely static. No backend, no accounts, no API keys, no credit card.**
It runs entirely in the browser on GitHub Pages.

## How it works (no server!)

GitHub Pages can only serve static files, and a browser can't fetch Google Flights directly
(CORS + it's a JavaScript app). So instead:

```
┌──────────────────────────────────────────────────────────────────┐
│  Your browser (GitHub Pages)                                     │
│                                                                  │
│  1. resolve city/country → IATA          (client-side)           │
│  2. build Google Flights' ?tfs= query     (protobuf encoder)      │
│  3. ask a public "reader" to render the   ──────► r.jina.ai       │
│     Google Flights results page                    (free, keyless)│
│  4. parse prices from the returned HTML   (flight parser)         │
│  5. render the tables                     (client-side)           │
└──────────────────────────────────────────────────────────────────┘
```

Everything runs in `docs/app.js`. It is a self-contained JavaScript port of the
[`fast-flights`](https://github.com/AWeirdDev/flights) library:

- **Protobuf encoder** — builds the `?tfs=` parameter Google Flights expects
  (byte-identical to the Python library; verified against known values).
- **Reader fetch** — asks a public reader service (`https://r.jina.ai/`, free and
  key-less) to fetch and render the Google Flights results page server-side, so CORS and
  Google's JavaScript don't matter. The reader returns the rendered HTML with CORS enabled.
  The same query works on any Google domain, so if the reader blocks one
  (e.g. *"anonymous access to www.google.com blocked"*), the app automatically tries the
  next domain in `FLIGHT_GOOGLE_HOSTS` until it gets real flight data.
- **Parser** — extracts the flight data (`AF_initDataCallback` / `ds:1` payload) from that
  HTML and maps it to itineraries, prices and airlines. It reads **every** itinerary list in
  the payload (the nonstop list *and* the connecting/"best" list), because the cheapest flight
  is frequently only in one of them (the `fast-flights` library reads just one, so it can miss
  the cheapest). Verified to produce the same results as the Python library, plus the cheaper
  options it misses.
- **Reader rate limiter** — the key-less reader serves roughly **20 lookups per rolling minute**
  per visitor and answers `HTTP 429` beyond that (one lookup = one rendered Google Flights page).
  Instead of firing until it breaks, every reader request is admitted through a rolling-window
  queue (`READER_MAX_PER_MINUTE`, default 18/min) and a `429` adds a short penalty and is retried
  with patience, so a big sweep finishes instead of failing. The progress line shows the wait, the
  elapsed time, the plan and the estimated total; a free reader API key removes the wait almost
  entirely.
- **Long runs are survivable** — an exhaustive sweep can run for a long time, so results are
  checkpointed to `localStorage` as they arrive and offered back after a reload (**Resume**,
  which re-runs only the unfinished lookups and restores what was already found). A **Stop**
  button ends a run at any time and renders the partial results — it takes a **second click** to
  confirm (the first click only arms it and offers *Keep searching*), so a long sweep can't be
  killed by a stray tap. Sweep lookups are also cached in memory, so re-running a search never
  pays the reader twice for the same lookup.
- **Built for the background** — a sweep is designed to be left alone: the favicon spins while it
  runs and turns into a green check when it finishes, the tab title carries the live lookup count
  and then the finished result (it reverts to normal once you return to the tab), and a
  **browser notification** (permission is requested when you press *Search*, and can be triggered
  again from ⚙️ settings) announces the finished search with the cheapest option found. Because
  browsers throttle hidden tabs hard, an inaudible ~20 Hz keep-alive tone plays while the tab is
  hidden (it is ~-55 dBFS, i.e. silent on any speaker or headphone) so the sweep keeps its full
  pace; both the notification and the keep-alive can be turned off in ⚙️ settings.
- **A catchy animated icon** — the brand mark is an animated paper plane (SVG in the header, a
  static `docs/favicon.svg` for the tab when idle, and a canvas-painted animated plane/check
  favicon while a search runs). Animation only runs while a search is active, and it is disabled
  under `prefers-reduced-motion`.
- **Cross-check on other sites** — under the results a card carries **one row per booking way**
  (round-trip, and two separate one-ways), each built from the **cheapest option found** for that
  way (its route, dates and one adult). Every row links **Google Flights** — whose own result list
  is where the travel agents, **Gotogate** included, appear under *Booking options* — plus
  **Kiwi.com** and **Skyscanner** (prefilled searches), and **Gotogate** itself (home page only: it
  has no prefillable search URL — see
  [Cross-check on other sites](#cross-check-on-other-sites-google-flights-kiwicom-skyscanner-gotogate)).
  These are plain outbound links: nothing is fetched from those sites, no key is used, and the app
  still makes **only** the reader call described above.
- **Automations** — the **Automations** card turns the current search into a schedule: give it a
  name, pick an interval (15/30 minutes, hours or days) and an optional daily time window, and the
  app re-runs **the main form's criteria**, sends a **browser notification headed by the cheapest
  flight found** as each run's results list is ready, and **archives each run's full results list**
  (the last 24 per automation, each one openable on its own screen with a *← Back* button, and
  exportable). Because the page is 100% static, runs happen **while the tab/installed app is open**
  — an interrupted run continues from its saved progress when you reopen, and a slot missed while it
  was closed is replayed **once**, not N times. See [Automations](#automations).
- **Saved searches (favorites)** — the **Saved searches** card parks whole result sets: press
  **☆ Save this search** above the results (or **☆ Save this run** on an archived automation run)
  and the flights, prices and the criteria that produced them are kept in `localStorage`. They are
  grouped **destination → departure month → search date**, and each one can be opened to **show its
  results** exactly as they came back or **re-run the identical criteria** in one click. See
  [Saved searches (favorites)](#saved-searches-favorites).

> **No secret is embedded and nothing is exposed.** The reader request and the (already
> public) Google Flights page are the only network calls.

## Use it

Open the GitHub Pages URL, type a country/city/airport, pick your dates and stay, and search.
You can also list several airports or cities directly, separated by commas or spaces —
e.g. `DUS, FRA, HAM` (or `DUS FRA HAM`) — and every listed airport is searched.

Two input rules keep the form from being filled in wrongly:

- **The From/To fields are capitals-only.** Whatever you type (or paste) is turned into CAPITAL
  letters as you go — `dus, fra` becomes `DUS, FRA`, `germany` becomes `GERMANY` — and the same
  normalisation is applied to the submitted search, so resolution, hints and lookups always see
  one casing. (`text-transform: uppercase` also keeps the *display* capital for autofill/paste,
  and the mobile keyboard opens in caps mode.)
- **The two date fields are the two ends of one window, so they cannot contradict each other.**
  Picking a departure date (*Date window from*) immediately re-fills the return end
  (*Date window to*) with **departure + the longest stay**, and the return calendar then only
  offers days **after** the departure — earlier days *and* earlier months are greyed out and
  cannot be selected. Symmetrically, the departure calendar is capped by the return end, and a
  return end you picked yourself is never overwritten while it is still valid. So a departure of
  `01.12.2026` gives a return end of `08.12.2026` with a 7-day maximum stay — never a month or a
  date before the departure.

## The date window covers the whole trip (default)

**Date window applies to → *Whole trip* (default).** Every flight listed both **departs and
returns inside the window you set**. The stay length therefore decides the return date, and a
departure that could not return in time is neither searched nor listed — so a result can never
fall outside the dates you asked for. Example: window `06.12.2026 – 13.12.2026`, stay 6–7 days →
only `06.12` (stay 6 and 7) and `07.12` (stay 6) can still return by `13.12`; `08.12` onward is
impossible and is skipped (which also saves reader quota). The result header states this in plain
language, e.g. *"Trip window: every listed flight departs and returns inside … so departures run
…"*.

**Date window applies to → *Departure date only*.** Opt in to the looser rule: the outbound must
be inside the window, but the return may land after it. Use it when you only care about when you
leave.

If a stay cannot fit the window at all (e.g. a 20-day stay in a 8-day window), the app says so
instead of quietly returning nothing, and suggests widening the window, shortening the stay, or
switching to *Departure date only*.

## Departure weekday filter (outbound)

**Departure day of week** restricts the search to outbound departures that leave on the weekdays
you tick (e.g. **Mon** for "cheapest flight that departs on a Monday"). Tick several (e.g. Fri +
Sat + Sun) to allow any of them; leave them all unchecked for any day (the default). It is one
more criterion alongside the route, stay length, date window, direct-only and ticket type — all
of them apply together, and the returned price is still the cheapest that satisfies the whole
set.

Because it removes departure dates before anything is looked up, it also makes a big sweep
faster (only the chosen weekdays are queried). If no departure on the chosen weekday fits the
window and stay length, the app says so and suggests another weekday, a wider window, a shorter
stay, or *Departure date only*. The result header and the plain-language note both state which
weekdays were searched.

## Maximum ticket price

**Max ticket price (optional)** is the most you are willing to pay, entered as a number **in the
selected currency** (e.g. `250` with EUR selected). Leave it blank or set it to `0` for no limit.
It is one more criterion alongside the route, dates, stay length, direct-only and ticket type —
every one of them still applies, and a flight is listed only if it satisfies **all** of them
*and* costs **at or under** your limit.

The cap is compared against the **total price of each listed booking option**: Google's round-trip
total for a round-trip ticket, or the summed price of the cheapest outbound plus the cheapest
return for a two-separate-one-ways (or open-jaw) combo. It is not a per-leg or per-adult limit.
All prices are for **one adult**, and no currency conversion is done — the number is interpreted
in whatever currency the search was run with.

It applies everywhere: the headline picks, the ranked list, the per-date and per-airport tables,
the booking-style comparison card, browser notifications, archived automation runs and saved
searches. The result header shows the cap and how many over-cap options were hidden, and the
plain-language note explains what was filtered. If nothing fits, the app says so and suggests
raising the cap or loosening the other criteria. Saved searches and automations store the cap with
their criteria, so re-running one restores it automatically.

**Does it make searches faster? Not the main sweep.** Unlike the weekday filter, a price cap
cannot tell the app to skip an airport/date lookup: the price of a slot is only known **after**
that slot has been looked up, so every route and date still has to be fetched to learn whether it
is under your limit. The cap does save work **after** prices are in — it trims the extra
**return-direction** lookups spent on flights that would be displayed (fewer over-cap flights are
displayed, so fewer return details are fetched), and it can skip the return lookup inside a
one-way/open-jaw task when the cheapest outbound already exceeds the cap. In practice the main
grid sweep dominates the runtime, so a price cap is a **result filter**, not a meaningful speed-up
— use a free reader API key, a narrower window or the weekday filter if you want a faster run.

## Cheapest flight via an in-country stopover

Tick **Also compare a stopover inside the destination country** when your chosen city is not the
only airport worth flying into. For example, flying to Athens can be much cheaper if you land at
Thessaloniki (SKG) and take a short internal flight down to ATH. The app then looks for:

1. the cheapest **direct** flight from your origin airport(s) to a **major airport of the
   destination country** (a *gateway*), and
2. an **internal flight** from that gateway to your final destination airport,

with both directions handled the same way on the way home. The two are combined into one
**self-transfer** itinerary with one stopover in the destination country, and reported as a
**third comparison** alongside the round-trip and two-one-way sections. The cheapest valid
combination wins — subject to every other criterion, including the maximum price.

- **Gateways:** the top **5 major airports** of the destination country (excluding your final
  airport), taken from the same country→airports table used for country inputs. If the country is
  not in that table, the option simply produces no stopover results.
- **Connection rule:** the internal flight must leave at least **2 hours** (and at most 24 hours)
  after the inbound lands at the same gateway. Arrival and departure are both in gateway-local
  time, so no time-zone conversion is needed.
- **Extra lookups:** each stopover task makes four one-way lookups per gateway (two per direction).
  To keep this bounded it is sampled on a few spread-out dates for the top airport pairs; it is an
  extra comparison, not part of the main grid's coverage claims.
- **Two separate tickets:** these are independent bookings, usually on different airlines. Bags are
  generally **not checked through**, and a delayed inbound can make you miss the internal flight —
  the traveller carries that risk. The result is labelled as such and is not linked as a single
  round-trip on the price-comparison card.
- **Per-gateway table:** when the option is on, a **Per in-country gateway (best found)** table is
  shown just above *Per departure airport (best found)*. Every gateway tried gets a row (including
  ones with no valid connection), with the direct leg, the internal leg, each leg's price and the
  connection time in both directions, plus the gateway's combined total. Rows used by the cheapest
  itinerary are highlighted and marked *Included in cheapest*; others read *Not used* or
  *No valid connection*. This makes it easy to see *why* a particular gateway won.

The option is stored as a criterion in automations and saved searches, so re-running either
restores it. It is a static-app (`docs/`) feature only; the legacy Python backend does not have it.

## Cross-check on other sites (Google Flights, Kiwi.com, Skyscanner, Gotogate)

The app prices from Google Flights, and Google's own result list is also where the travel agents —
**Gotogate** among them — appear: select a flight there and its **Booking options** list them. So
the results carry a **"Compare on other sites"** card with **one row per booking way**, each row
built from the **cheapest option the sweep found** for that way (its route, dates and one adult):

| Link | What it does |
| --- | --- |
| **Google Flights** (`round-trip`) | Opens Google's result list for the exact round-trip search — the same `?tfs=` protobuf the app itself fetched, e.g. `https://www.google.com/travel/flights?tfs=…&hl=en&curr=EUR`. |
| **Google Flights** (`out` / `back`) | A *two separate one-ways* or *open-jaw* cheapest option is two one-way tickets, so it gets one link per leg, each with that leg's route and date. |
| **Kiwi.com** | Opens a prefilled search: `https://www.kiwi.com/deep?from=DUS&to=ATH&departure=2026-12-06&return=2026-12-12&adults=1&lang=en&currency=EUR`. For a one-way (or each leg of a two-one-way/open-jaw option) the `return=` parameter is omitted, which makes Kiwi search a one-way. |
| **Skyscanner** | Opens a prefilled search: `https://www.skyscanner.net/transport/flights/dus/ath/261206/261212/?adults=1&adultsv2=1`. Dates are `YYMMDD` path segments; a one-way uses a single date segment; the direct-only filter adds `preferdirects=true`. |
| **Gotogate** | Opens `https://www.gotogate.com/` — its **home page**, with no prefilled search. |

**Why Gotogate has no prefilled link.** Gotogate (like its siblings Mytrip/BudgetAir, all part of
Etraveli Group) is a JavaScript single-page app whose search is driven by a **server-issued
selection key** obtained through a GraphQL call — the only URL parameters its client code reads are
affiliate/UTM ones, never route or dates. Every candidate deep link was tested against the live
site and fails: `/searching-for-flights?...` answers
`{"message":"Search parameters invalid. No flight-routes requested."}`, and `/start`, `/result`,
`/flights/DUS/ATH`, `/rf/flights/dus-ath` all return `404`. On top of that, Gotogate serves
**`HTTP 403 Access Denied`** (Akamai) to non-browser clients, so a static page could never query it
directly anyway. A plain home-page link is the honest maximum — you re-enter the route and dates
there. The **Google Flights** links above are the practical way to reach Gotogate's price, since
Gotogate is one of the booking options Google lists.

**No key, no scrape, no server.** Every link is an ordinary outbound link opened in a new tab
(`target="_blank"` + `rel="noopener noreferrer"`). Nothing is fetched from those sites, so the
app's only network call remains the reader request described above, and the page stays 100 %
static. The card's own note states that prices on those sites are shown in **their** currency and
may differ from Google's.

> **If you need Gotogate's price shown *inside* the app**, the only route is a server-side proxy
> with a paid API (e.g. SerpApi's Google Flights `booking_options`, which lists OTA prices such as
> Gotogate/Mytrip/BudgetAir/Kiwi for a pinned itinerary). That needs a key and a backend, so it
> does **not** belong in the static GitHub Pages app — it would fit the legacy Flask backend
> (`app.py`) instead.

## Automations

An **automation** is a saved search plus a schedule: it re-runs itself, tells you when a run
finishes (browser notification + on screen), and keeps that run's full results list in an archive
you can browse or export.

Open the **Automations** card below the search form and:

1. Fill the search form with the criteria you want (route, stay, date window, ticket type,
   direct-only, maximum price, in-country stopover, currency, …). This is the **only** place criteria are chosen: an
   automation always uses the main form's criteria exactly as they stand when it is saved.
2. In the Automations card set a **name**, an **interval** (15/30 minutes, hours, or days) and —
   optionally — an **“only run between”** band, e.g. `08:00`–`22:00` so it never fires at night.
3. Press **Save automation using current main form criteria**.

What each automation offers:

| Option | Meaning |
| --- | --- |
| **Name** | What the automation is called in the list and in every notification. |
| **Run every** | How often it re-runs: minutes, hours or days. |
| **Only run between** (optional) | Keeps runs inside a daily time band; a due run outside the band waits for the next band start. |
| **Notify me when each run's results are ready** | One browser notification per completed run, headed by the **cheapest flight found — price and route** — with the run number, dates and airline. |

Each automation keeps the **last 24 runs**. Every archived run stores the **full results list**
plus a summary (cheapest round-trip and two-one-ways, lookups used, coverage), so re-opening it
renders exactly what that run found. The completed list is built to be scanned: each run's row is
headed by the **cheapest flight that run found** (price, route, dates, airline), the cheapest run of
the automation carries a **CHEAPEST** tag, and the card shows a *cheapest so far* line. Clicking a
run opens **its own results screen** — never mixed into the main results — with a
**← Back to main screen** button that returns you to the search form or the Automations card, ready
for a single search or a new automation. **Export** downloads an automation and its archive as JSON,
**Import JSON** merges a file back in, and deleting an automation removes its archive.

**Runs continue where they left off.** Progress is checkpointed while a run sweeps, so locking the
phone or switching apps leaves the run going and the progress still on screen; if the page is closed
or reloaded, the interrupted run **picks up from its saved progress** when the app is opened again
(the lookups that already finished are skipped) and its progress is shown in the Automations badge
and in the run screen.

**The honest limitation.** This page is 100% static — there is no server to schedule anything — so
an automation runs **only while the tab (or the installed app) is open**, reusing the same
keep-alive and notification machinery as a manual search. If the tab was closed past a run time the
missed slot is **not** replayed N times: the automation is simply due and runs **once** as soon as
the page is next opened. On iPhone/iPad the page must be installed to the home screen
(*Share → Add to Home Screen*, iOS 16.4+) for notifications to appear. Automations also share a
single reader quota and run **one at a time** — a manual search politely stops a running automation
first. **Run now** and **Pause** act immediately: pressing **Run now** on an automation stops whatever
search is in flight (a manual search or another automation's run) and starts the chosen automation as
soon as the engine is free, and pressing **Pause** on the automation that is currently running stops
that run too, so it no longer holds up the queue. (A truly always-on schedule would need a keyed
backend or a CI cron, which this project deliberately avoids.)

Archives live in your browser's `localStorage`. If it fills up, the oldest archived runs are pruned
automatically, so **export first** if you want to keep them.

## Saved searches (favorites)

A **saved search** is a whole result set parked exactly as it came back — every flight, price and
the criteria that produced it (including any maximum price or stopover option) — so you can look at it again later,
or re-run it with the identical parameters. It is the "I want to keep this one" side of a search,
next to the Automations "run it again on a schedule" side.

**Parking a search.** Finish a search (or open an archived automation run) and press the save
button above the results:

- **☆ Save this search to Saved searches** — on the main results toolbar, for a manual search.
- **☆ Save this run** — on an archived automation run's own screen.

The whole result set is copied into the **Saved searches** card below the form, along with the
exact criteria. The button then reads **★ Saved to Saved searches** and is disabled, so the same
on-screen result set cannot be parked twice; the next search clears it.

**How they are grouped.** Saved searches are filed the way you would look for them:

1. **Destination country / city** — a single resolved city if the search named one, otherwise the
   single resolved country (e.g. *Greece*), otherwise the raw input. The group's subtitle lists the
   airport codes actually searched and the country.
2. **Departure month** — the month the outbound date window starts in (e.g. *December 2026*).
3. **Search date** — the day you made the search (e.g. *Sun 15.11.2026*).

So the same route searched on different days, or aimed at different months, stays easy to tell
apart. **Every level of that tree collapses and expands** — click the destination, the month/year
or the search-date heading to fold that branch away — and each level starts open, ordered
**newest-first**, so the most recent searches are always at the top and on screen. A **filter box**
narrows the list by destination, month, search date, source or criteria text.

**Opening one.** Every saved search offers:

- **Show results** — opens it on its own full-screen page, rendering the stored list exactly as it
  came back, with a **← Back to main screen** button. Prices are a snapshot: live prices move on.
- **Search again (same criteria)** — restores the exact criteria into the form (maximum price
  included) and starts a **new** search right away, so a saved search and a hand-made one behave
  identically from there on.
- **Delete** — removes just that saved search (asks first).

**Managing them.** **Export all (JSON)** downloads every saved search as a JSON file, **Import
JSON** merges a file back in (duplicate ids are re-keyed so nothing is overwritten), and **Delete
all** empties the list (with a confirmation). The header badge shows how many are saved.

Saved searches live in your browser's `localStorage` (this browser only, no account, nothing sent
anywhere). At most **40** are kept — the newest — and if storage fills up the **oldest saved
searches are pruned automatically**, so **export first** if you want to keep them. The user-facing
storage and analytics note lives in `docs/privacy.html`.

## Install it on your phone's home screen

The app ships home-screen icons, so "Add to Home Screen" gives you a real app tile with the brand
mark instead of a screenshot:

- **iPhone / iPad (Safari).** Open the site, tap **Share** → **Add to Home Screen** → **Add**.
  It launches full-screen (no Safari bars) and uses `apple-touch-icon.png` (180×180 @3x, plus
  152×152 and 120×120 variants) — the opaque, full-bleed plane mark in `docs/`.
- **Android (Chrome).** Open the site, tap **⋮** → **Add to Home screen** / **Install app**. It
  uses `docs/manifest.json` (`icon-192.png`, `icon-512.png`, standalone display).
- **Desktop.** Chrome/Edge "Install" and pinned tabs use the same icons.

> **One platform limit, stated plainly:** iOS and Android do **not** animate home-screen icons —
> the tile is always a static image, so no web app can animate there. The animated plane (header
> logo + spinning/check tab favicon) therefore lives *inside* the app, and the home-screen tile
> uses a matching static frame of the same artwork. On iOS, notifications additionally only work
> for the installed home-screen app (iOS 16.4+), not for a plain Safari tab — another reason to
> install it.

## Publish on GitHub Pages

The included workflow `.github/workflows/deploy-pages.yml` deploys `docs/` on every push to
`main`. In the repository go to **Settings → Pages → Source: "GitHub Actions"**.

Your site is live at `https://<user>.github.io/<repo>/`.

## Configuration (all optional)

`docs/config.js`:

| Setting | Default | Purpose |
| --- | --- | --- |
| `FLIGHT_READER` | `https://r.jina.ai/` | The reader/fetcher prefix. |
| `FLIGHT_READER_KEY` | `""` | Optional reader API key — only **raises rate limits**; never required. |
| `FLIGHT_GOOGLE_HOSTS` | `["www.google.co.uk", "www.google.ca", …]` | Google domains tried in order until real flight data is returned. |
| `MAX_SEARCHES` | `0` (exhaustive) | Lookup ceiling per search (each lookup = one reader request and covers one airport pair × one stay length × one departure date). **`0` means exhaustive**: every airport pair, every stay length and **every departure date that fits the window** is swept, so the reported cheapest option really is the cheapest on offer — time is not a shortcut. A positive value caps the sweep; airport pairs and stay lengths are still always all covered (the cap is a **floor** for coverage), and the remaining budget goes to departure dates, spread evenly across the window. Hard cap: 50000. |
| `MAX_SEARCHES_SAFETY` | `1500` | Hard ceiling applied in exhaustive mode, so a pathological grid (whole country → whole country, month-long, wide stay range) can't queue tens of thousands of requests. When it bites, the result header says exactly what was covered and how to raise it. |
| `READER_MAX_PER_MINUTE` | `18` | How many reader lookups may be admitted per rolling minute. The key-less reader starts answering `HTTP 429` somewhere above 20, so requests are queued under this rate instead of being sent and failing. |
| `RETURN_FETCH_LIMIT` | `16` | Minimum number of return-direction lookups spent on the displayed flights. It grows with the lookup ceiling (up to a fifth of it, max 200), since return lookups share the same per-minute quota as the date sweep. |
| `SEARCH_CONCURRENCY` | `3` | How many reader requests may be in flight at once (the rate limiter still paces them). |
| `SEARCH_TIME_BUDGET_MS` | `0` | Hard stop for a search in milliseconds. **`0` = no time limit** (only the Stop button ends a run); partial results are shown either way. |
| `UMAMI_WEBSITE_ID` | `""` | Optional [Umami Cloud](https://cloud.umami.is) website ID for anonymous, cookieless analytics. **Blank disables analytics completely** — no request is made. |

Visitors can override the reader, add their own key, change the lookup cap, and turn the
**finish notification** or the **background keep-alive** on and off from the in-page
**⚙️ Data source settings** panel (saved in their browser only). The same panel shows whether
notifications are granted, blocked or unsupported, with a button to ask again.

## Analytics (optional, free)

The static app can report anonymous, cookieless usage stats through
[Umami Cloud](https://cloud.umami.is) — the free Hobby plan includes 100K events/month,
one website and 6 months of retention.

1. Create a website in Umami and copy its **website ID**.
2. Set `window.UMAMI_WEBSITE_ID` in `docs/config.js`.
3. Deploy. No backend, account system or build step is involved.

Tracked: page views, visitors, countries, referrers and devices, plus the events
`search-started`, `search-completed`, `search-error` (category only), `outbound-click`
(Google Flights / Kiwi / Skyscanner / Gotogate) and `saved-search` (action only: `save`, `open`,
`rerun` or `delete`). Origin, destination, dates and prices are never sent. Leave the ID blank to
disable analytics entirely. The user-facing note lives in `docs/privacy.html`.

## Local development

Any static server works — there is no build step:

```bash
python -m http.server 8000 --directory docs
# then open http://127.0.0.1:8000/
```

## Limitations

- **No official Google Flights API exists.** This uses a public reader to render the same
  page a browser would, then parses it. Google may occasionally show a consent/verification
  page, and the free reader is rate-limited or may block a given Google domain for a while —
  the app reports both clearly and falls back to other Google domains. Retrying usually works.
- **Complete airport coverage.** Every airport the input resolves to is searched — a country
  expands to all of its commercial airports (e.g. Germany → 30 airports) and **every**
  origin→destination pair is looked up, so the real cheapest departure airport is found even
  when it isn't the country's main hub. Multiple airports/cities can also be listed directly
  (`DUS, FRA, HAM`, `DUS FRA HAM`, `Frankfurt, Munich`). Cities that have several hubs (London,
  Paris, New York, …) expand to all of them.
- **Accuracy first: the whole grid, by default.** One *route* = an airport pair × a stay length;
  one *lookup* = a route on one departure date. With the default ceiling (`0`) the app sweeps the
  **entire grid** — every pair, every stay length, **every departure date that fits the window** —
  so the cheapest option it reports really is the cheapest on offer. For the reference case (5
  German airports → ATH/SKG, 2 stay lengths, December, whole-trip window) that is 490 lookups ≈
  **25–30 minutes** key-less (fewer than the looser departure-only rule, because departures that
  could not return inside the window are skipped); the run is meant to happen in the
  **background** — progress is checkpointed, a reload offers to **resume**, and a browser
  notification reports the finished search. "**Sampled days**" is the explicit fast path: 4
  spread-out days, no deep pass.
  If the grid is bigger than the budget (a positive ceiling, or the `MAX_SEARCHES_SAFETY` cap),
  coverage is trimmed **in departure dates only** — never airport pairs or stay lengths — and the
  priority order is: (1) **broad pass** over every route on spread-out dates, then (2) **deep
  pass** over the remaining dates for the routes that looked cheapest (a route whose whole
  remaining calendar fits gets every date; otherwise it gets a share, so one badly-ranked route
  can't swallow the budget; missing dates are visited in **bisection order** so partial coverage
  is spread evenly). The result header always states exactly what was covered
  (`Coverage: N/M airport pairs · N/M departure dates · N/M stay lengths`), and when everything
  fitted it states that the sweep was **exhaustive** and that the cheapest option listed is the
  cheapest on offer (or the cheapest at or under your maximum price, when a cap is set). Raise the
  lookup budget in ⚙️ settings to cover more; paste a free reader API key to make any of it much
  faster.
- **Round-trip detail.** Google's round-trip page returns its round-trip total price but lists
  only the **outbound** itinerary. The app therefore fetches the **return direction separately**
  (one extra lookup per return date, de-duplicated) and shows the cheapest return on that date
  for the same airline when available. Return lookups share the per-minute quota, so they are
  spent on the flights actually displayed — the headline picks, the top of the ranked list and
  the per-airport/per-date cheapest — up to `RETURN_FETCH_LIMIT`. Rows beyond that still show the
  return **date** and say that times/airline are unavailable. The price shown is always Google's
  own round-trip total. Open-jaw totals are estimated from two separate one-ways.
- **Every flight shows both directions** in a fixed order: departure airport → departure date →
  departure &amp; arrival time → airline → stay days (outbound), then the same for the return.
  All dates are shown as `dd.mm.yyyy` **preceded by the abbreviated weekday** (e.g.
  `Sun 06.12.2026`, in your browser's language), which makes the "cheap Sunday" jumps in a price
  sweep easy to spot.
- **Direct-only.** Tick *Direct flights only (nonstop)* to restrict every lookup (outbound and
  return) to nonstop itineraries.
- **Maximum price filters results, not the sweep.** A max-price cap removes over-cap tickets from
  every results section, but prices are only known after a slot has been looked up, so it cannot
  reduce the number of airport/date lookups in the main grid. It does avoid some **return-detail**
  lookups (fewer displayed flights, and one-way/open-jaw return directions are skipped when the
  cheapest outbound already busts the cap). If no ticket fits, the cap is applied to an otherwise
  complete sweep, so raising it does not require re-running to discover new cheap dates.
- **In-country stopovers are self-transfers, and cost extra lookups.** The comparison samples a few
  dates and the top 5 gateways of the destination country; each stopover task spends four one-way
  lookups per gateway, so it lengthens a run. It needs a ≥2-hour same-day connection and only works
  for countries in the built-in gateway table. The two legs are separate tickets — no through-
  checked bags and no protection if the inbound is delayed.
- **Background searches have browser limits.** A sweep keeps its full pace while the tab is
  hidden on desktop browsers (helped by the inaudible keep-alive audio, which you can switch off).
  Phones throttle or **suspend** background tabs entirely — iOS stops JavaScript in a backgrounded
  tab, so the run pauses there and continues when you return to it (nothing is lost: progress is
  checkpointed, and the *Resume* offer skips what was already done). Notifications also need the
  page to be added to the home screen on iOS.
- **Automations run while the page is open.** A scheduled run reuses the keep-alive + notification
  path above, so it only fires while the tab (or installed app) is open. A run interrupted by a
  locked phone or a reload **continues from its saved progress** on the next open; a slot missed
  entirely while the page was closed is replayed **once** on the next open, never N times.
  Automations run **one at a time** on the shared reader quota (a manual search stops a running
  automation first). Each automation's archive (last 24 runs) is stored in `localStorage` —
  **export it** before clearing site data.
- **Saved searches are a snapshot, in this browser only.** A saved search keeps the results exactly
  as they came back (live prices will have moved on since) plus the criteria, in `localStorage` —
  no account, nothing sent anywhere. At most **40** are kept and the **oldest are pruned
  automatically** when storage fills up, so **export first** if you want to keep them. **Search
  again (same criteria)** re-runs the stored criteria as a fresh search; only **Show results**
  replays the old list.
- Prices are live estimates only — not bookable through this app. Booking-site comparisons
  (Google Flights, Kiwi.com, Skyscanner, Gotogate) are **outbound links only** — see
  [Cross-check on other sites](#cross-check-on-other-sites-google-flights-kiwicom-skyscanner-gotogate);
  no OTA price is fetched or shown inline (that would need a keyed backend API).
- City resolution works for major cities (and expands to all of a city's hubs, e.g. London →
  LHR/LGW/STN/LCY); you can also list airports directly (`DUS, FRA, HAM`). An exact IATA code
  always resolves to that single airport.

## Optional: self-hosted Python backend (legacy)

Earlier versions of this app used a Flask backend (`app.py`) that scrapes Google Flights with
`fast-flights` and serves JSON over CORS. That path still works if you prefer to run your own
backend (Docker, Vercel, a VPS, …) — see `Dockerfile`, `requirements.txt` and `vercel.json` —
but it is **no longer required**: the GitHub Pages app above is fully self-contained.

### ⚠️ The Python backend lacks "ticket type", "departure weekday", "max price" and stopovers

The static app (`docs/`) gained a **Round-trip vs. two separate one-ways** comparison, a
**departure-weekday filter**, an optional **maximum ticket price** and an optional **in-country
stopover comparison**, but `app.py` + `templates/index.html` were intentionally left untouched (it
is the legacy path). **Saved searches** are likewise a static-app feature only: they need no backend
at all (they are `localStorage` in `docs/app.js`), so the legacy `templates/index.html` does not have
them. If you ever want the Flask backend to match the other two, here is exactly what to change —
the static app is the reference implementation, so mirror it rather than inventing a new design.
(For the weekday filter specifically: accept a `depart_days` array of `0`–`6` weekday indices in
`api_search()`, and when building the sampled outbound dates, skip any date whose `date.weekday()`
is not in that set — mirroring `runSearch()` in `docs/app.js`. For the price cap: accept a
`max_price` number in `api_search()`, treat blank/`0` as no cap, and drop any record whose total
`price` exceeds it *after* the sweep but before summaries/rendering — mirroring `withinPriceCap()`
in `docs/app.js`. For the stopover option: accept `via_country`, derive the top 5 gateways of the
destination country, combine a direct leg with an internal leg in each direction subject to a
≥2-hour same-day connection, and tag the record `ticket_type: "stopover"` — mirroring
`searchCountryStopover()` in `docs/app.js`.)

**Concept.** A *round-trip ticket* is a single booking priced from Google's round-trip response.
*Two separate one-ways* is the sum of the cheapest one-way in each direction (often two different
airlines, so it can be cheaper). They are different products and must never be added or compared
as if they were the same thing.

**Backend (`app.py`)**

- Read a new request field in `api_search()`: `ticket_type` ∈ `"roundtrip" | "oneway" | "both"`
  (default `"roundtrip"` so existing callers are unaffected).
- For each `(origin, dest, outbound, stay)` slot the loop already computes, keep the existing
  round-trip call for `roundtrip`/`both`, and additionally, for `oneway`/`both`, call
  `search_flights(o, d, outbound, None, …)` and `search_flights(d, o, ret, None, …)` (i.e. two
  **one-way** searches — `return_date=None`). Sum the two cheapest, tag the merged record with
  `ticket_type: "oneway"` (or `"open_jaw"` for the existing open-jaw estimate), and append it.
  A one-way combo is **two** `search_flights` calls, so count it against `MAX_SEARCHES` as two.
- Tag round-trip records with `ticket_type: "roundtrip"`.
- In the JSON response, add `ticket_type`, plus `cheapest_roundtrip` / `cheapest_roundtrip_direct`
  and `cheapest_oneway` / `cheapest_oneway_direct` (filter `all_results` by `ticket_type`, then by
  nonstop), and `has_roundtrip_results` / `has_oneway_results`, mirroring
  `docs/app.js` → `runSearch()` (search for `roundTripOpts`, `oneWayOpts`).
- "Direct" for a one-way combo means **both** legs are nonstop (`stops == 0` and
  `return_stops == 0`), matching `docs/app.js` → `_isDirectOption()`.

**Frontend (`templates/index.html`)**

- Add a `ticket_type` `<select>` next to the existing `open_jaw` control (same three options,
  default `both`), include it in the `fetch("/api/search", …)` body, and render the two sections
  ("Round-trip ticket — one booking" / "Two separate one-way tickets — two bookings") plus the
  Type column, exactly like `renderResults()` / `_ticketLabel()` in `docs/app.js`.

**Caveat.** `app.py` runs synchronously under a hard `MAX_SEARCHES` (default 40) and
`SEARCH_TIME_BUDGET_S` (default 100s) budget on a free host; `"both"` roughly doubles the work,
so consider leaving the backend default at `"roundtrip"` and letting the client opt in.

## Example

- From: `New York` or `JFK`
- To: `London` or `LHR`
- Vacation days: `7`
- Optional departure date
