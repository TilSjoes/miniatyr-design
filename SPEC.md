# How Miniatyr-Norge works

*A design description, October 2026. Miniatyr-Norge is a one-person test by DONTPANIC AS: Norway's transport
infrastructure in 3D, as exactly as the open data allows, with a Lego-style skin. It is not official and not for
travel. Not affiliated with or endorsed by the LEGO Group.*

This document explains how it works: the data, the rules that turn the data into a landscape, the way one continuous
Norway is streamed to a browser, what moves on it and how far each moving thing can be trusted, and how it was built.
The code is not published. The ideas are below.

---

## 1. The one idea: rules, not hand work

Nothing in Miniatyr is placed by hand. Every decision is a **rule over the open data**: which level a road is on,
which way a carriageway runs, where a tunnel enters the hill, whether a road is closed to cars, where a lake is flat,
how tall a tree stands. A new area is then just a box to bake. An error is always one of two things: **the data's**
or **a rule's**. Both can be found, counted and fixed once, for the whole country.

That also decides how faults are fixed. When a viewer reports "the train runs inside the hill", nobody edits that
hill. A check is written that finds *every* place where it happens, and the rule that caused it is changed. Section 7
shows this in practice: most of what the first wave of viewers found became a country-wide check within a day or two.

The data's own disagreements, such as a road the survey puts underground while the laser puts it on a bridge, are
counted during every bake. They are published on the page as a report, *Funn i dataene* ("findings in the data"),
for the people who own the data.

## 2. The data

Everything is open public data. Miniatyr pays nothing for data licences.

| What | From | Licence | How fresh |
|---|---|---|---|
| Terrain (ground and surface) | Kartverket, national laser scanning (DTM/DOM, 1-2 m) | CC BY 4.0 | baked |
| Roads in 3D: geometry, lanes, levels, bridges, tunnels, speed limits, bans | Statens vegvesen, NVDB | NLOD | baked |
| Traffic counts | Statens vegvesen, Trafikkdata | NLOD | baked typical week, today's counts live |
| Road situations, webcams, road weather, travel times | Statens vegvesen, DATEX II | NLOD | live |
| Railways, tram and metro track, buildings, names | OpenStreetMap contributors | ODbL | baked |
| Seabed | EMODnet Bathymetry | CC BY 4.0 | baked |
| Trains, buses, ferries, boats: positions, timetables, real time | Entur | NLOD | live |
| Rail disruptions: closures, works, strikes | Bane NOR (SIRI SX) | NLOD | live, every 5 min |
| Ships | Kystverket's open AIS stream | NLOD | live |
| Aircraft | adsb.lol (ADS-B); flight data from Avinor | ODbL; Avinor's terms | live |
| Weather, snow depth | MET Norway; seNorge snow grid | NLOD / CC BY 4.0 | forecast live; snow grid every 3 h |

Each layer reads through an **adapter** with an id, a label, a licence and a live/baked flag. The page shows every
layer's source and licence. A source can be swapped without touching the rest, for example an open API for an
internal or more precise feed.

**Beyond transport.** The same map carries a set of other layers, each from its owner's open data:

| Layer | Source |
|---|---|
| The power grid; flows between price zones and the production mix | NVE, Statnett, ENTSO-E |
| Air quality per station | Miljødirektoratet |
| Population on a 250 m grid; commuting between municipalities | SSB |
| Flood, landslide and avalanche zones; road noise | NVE, Statens vegvesen |
| Snow depth; avalanche danger | seNorge, Varsom |
| Toll stations and a trip's cost; speed cameras | NVDB |
| Parking | Parkeringsregisteret |
| Chargers and fuel | NOBIL, OpenStreetMap |
| Freight trains | Bane NOR |
| Animals on the road; road webcams | DATEX II |
| Ski lifts and cable cars | OpenStreetMap |
| How far you get in 15-60 minutes | Valhalla on OpenStreetMap |

Each is a dataset on its own. The value is in putting them on **one grid, one clock and one map**, because then they
can be read *together*. A road closure can be set against the people who commute along it, a flood zone against the
roads and the homes in it, a cold evening against the grid's flows. Open data multiplies when it can be correlated. A
shared grid and adapters are what make adding the next source cheap (Section 12).

## 3. From data to landscape: the bake

A **bake** turns one box of Norway into files the browser can draw. It runs in Python (55 small programs, standard
library plus numpy where it pays) and makes these decisions, in order:

1. **Terrain** from the laser, at 2 m (4 m along corridors). The terrain service sometimes answers with a truncated
   picture. Failing blocks are split into quarters, all-sea blocks (known from the seabed data) are never asked for,
   and zeros outside Norway's border are not treated as sea.
2. **Roads** from NVDB's 3D centre lines.
   - **The level** comes first from NVDB's own objects (bridge, tunnel), then from the surveyed line's "medium", then
     from the laser.
   - **The terrain is carved** under every road at ground level, so the ground never covers a road.
   - **Direction comes from the lanes.** On divided roads the driving direction comes from NVDB's lane numbering (odd
     lanes run with the line, even against). It is not taken from the road's stated direction, which turned out to
     mislead.
   - **Other rules:** roads under construction are drawn only in x-ray, and pedestrian streets and roads closed to
     motor traffic are marked.
3. **Rail** from OpenStreetMap, with heights from the terrain. Tunnels are carved through the hill, with portals.
4. **Buildings** from their footprints and the laser's surface: the height and roof shape. Lego bricks are laid up
   close. **Trees** from the canopy tops in the laser, **lakes** where the laser water is flat, and the **seabed**
   from EMODnet.
5. **Checks** after every bake (Section 7), and the **findings** written for the report.

A baked box is a set of terrain pictures (heights in two colour channels, land cover in the third) and gzipped JSON
tiles of roads, rail, buildings, trees, counters and stations.

## 4. One continuous Norway: the pyramid

There are no areas to choose. You zoom from the whole country down to 2 m and pan anywhere.

- **One grid for everything:** UTM zone 33, origin at multiples of 2^18 m, so every baked tile lands exactly on a
  pyramid tile.
- **Terrain levels z5 to z13,** from 512 m cells down to 2 m, each tile 128 x 128 cells.
  - Coarser levels are made from finer ones by taking the lowest sample below, so a coarse level never lifts ground
    over a carved road.
  - Water takes the mean instead.
  - Gaps come from the national overview.
- **Data tiles** of 1 km. Where baked boxes overlap, each tile takes everything from one box: the finest, and the one
  farthest from its edge.
- **Detail grows where people look.**
  - The country is baked busiest first: city centres at 2 m, then the corridors along the main roads.
  - A viewer can press "Mer detalj her" (more detail here). That puts the 4 km square under the camera in a queue,
    most-asked first.
  - A night job bakes up to 20 squares when the machine is idle, rebuilds the pyramid and publishes it.

## 5. The page

There is no game engine and no framework: one web page, WebGL and about 90 small ES modules, built in order into one
file.

- **Shared state and hooks.** Each feature is a module, and new layers plug in through hooks without editing the
  core. Values that several modules change live on one shared state object.
- **The terrain is a quadtree.** It refines a tile when its error on screen exceeds a pixel budget. A parent hides
  the quarters its children draw, and skirts close the seams.
- **Data tiles load round the camera** and are built into meshes by the same builders a single area uses.
- **The world's origin follows the camera** every 150 km, because 32-bit floats are not enough for a country.
- **Speed:** 57 frames a second at 1080p, from the country view down to street level in Oslo, Bergen and Tromsø (on
  a desktop GPU). Phones get a lighter mode.
- **Lego is a switch:** the same landscape in bricks, or realistic, one click apart.

## 6. What moves, and how far to trust it

This is the most important section for anyone who uses the map to judge something. Every moving thing is in one of
four classes, and the page labels which.

| Class | What | Where it comes from | What can go wrong |
|---|---|---|---|
| **Live position** | trains with GPS, ferries and fast boats, ships (AIS), aircraft (ADS-B), buses where the operator shares GPS | the vehicle reports where it is | a fix freezes in a tunnel (the bus is carried on along the tunnel at its last speed); a train is shown on the neighbouring track at a station; operators' own data errors |
| **Timetable + real time** | metro, trams, most buses, trains without GPS | Entur's timetable and delays, laid along the real track or road between stops | where no path is found between two stops the vehicle is not drawn: a guess is not shown as a fact |
| **Modelled** | road traffic | the counters' typical week for that hour; elsewhere the road's yearly average spread over the day | it is a model of a typical day, not today's cars: it does not know about today's accident or closure unless the road data says so |
| **Measured or forecast** | weather, snow on the ground, disruptions | MET Norway, the seNorge snow grid (1 km, a day old), Bane NOR, Statens vegvesen | snow is blurred over 1 km; disruptions are shown as their two ends and a straight line, not along the track yet |

The rule behind the table is that **a guess is labelled or not drawn.** One example: the "Oslo metro train in Voss"
that the first post joked about. It was not a data error. One generic train model, a short unit with a pantograph,
was used for every train, and in the regional trains' colour at Voss it looked like a metro unit. Now there is a model
per kind of train, and each kind runs only on its own kind of track.

## 7. The traffic as a test instrument

The cars are not only decoration. They are a measuring instrument for the road network.

The Lego traffic follows the road data exactly: the lanes, the one-way rules, the junctions. So **where the data is
wrong, the cars show it.**

- Cars meet head-on where a carriageway has the wrong direction, or where two road pieces lie on top of each other.
- They get stuck at a one-way piece that leads nowhere.
- They circle a roundabout whose exits all point the wrong way.
- They drive through each other where nobody yields.

A monitor in the page counts each such event in 25 m cells, together with the NVDB references of the roads involved.
Trains are watched the same way: two trains in one place, a train jumping to a track not joined to its own, a metro
train on the main railway.

**Every night**, a machine in the garage opens the page headless at about 70 places across Norway (every detailed
area, and points along the corridors). It lets the traffic run and collects the hot spots. In the morning there is a
ranked list.

The faults are read as **patterns** and fixed as **rules**, never place by place. One night's examples (70 places,
61 600 simulated minutes):

- **40 278 head-on meetings.** The cause: at the end of a traffic island the two halves of the road join into one,
  and both directions were drawn on one line. The fix was a rule to keep each car on its own side of the opposite
  flow. Head-on meetings fell from 3.6 to 0.9 a minute at the worst places.
- **35 539 dead ends.** The road on was simply in a data tile not yet loaded. With the fix, Kristiansand went from 74
  a minute to 0, and as a side effect the traffic round the camera nearly doubled.
- **20 256 locked junctions.** Side roads facing a steady stream. A driver's accepted gap now shrinks with the wait,
  as it does for people.

The same instrument then caught a real data problem. Measured against OpenStreetMap, **62 % of OSM's one-way street
metres had no direction in our data**:

- NVDB draws many divided city streets as two separate lines, and the bake had read them as two-way.
- A rule from NVDB's lane numbering brought agreement in central Oslo to 98.8 %.
- The night survey's worst remaining head-on spots turned out to be exactly the roads OSM disagrees with. Two
  independent checks pointed at the same places.

The method in one sentence: **make the model behave the way the real world must, and every place where it can't is a
finding.**

## 8. Checks that run by themselves

- **After every bake:**
  - spikes at sea;
  - blocks of zeros (the terrain service's "swimming pools");
  - values at the encoding's limits;
  - stripes in the source terrain.
- **Rail against terrain:**
  - track that runs inside a hill;
  - banks that poke through a bridge deck;
  - tunnels that float above the ground.

  187 stretches were found and healed after viewers reported the Nordland line.
- **Road direction against OpenStreetMap,** where both claim a one-way street.
- **A smoke test before every deploy:** the page loaded headless, and the parts that have broken before checked one
  by one.
- **Health by results, not by processes,** every ten minutes: a file's age and size, a page's answer. A process that
  is "up" while writing nothing has happened more than once.

## 9. Serving it

- **Mostly static files:** the page, the pyramid tiles and the live files written by small collectors (ships,
  aircraft, disruptions, snow).
- **One small live server.** It sits between the page and the real-time sources (Entur, MET and the rest). Every
  viewer looking at the same area asks the same questions, so one answer is fetched and shared. The sources see one
  client however many people watch.
  - Upstreams are allow-listed.
  - Identical requests are merged.
  - If a source fails, the last good answer is served.
  - Keys and credentials stay on the server. **The browser talks only to its own server.**
- **The whole site also packs into an offline zip:** unzip on a laptop, double-click, no install.

## 10. What it does not do

The thread on the first post asked about AI and surveillance, so here is the plain answer:

- **No people.** What moves on the map is infrastructure and vehicles that broadcast their own position in public:
  trains and buses through Entur, ships through AIS, aircraft through ADS-B. Road traffic is a model, not real cars.
  There are no cameras of people. The road webcams shown are Statens vegvesen's public pictures of the road.
- **No tracking of viewers.** There is no login, no cookies, no analytics. Usage is counted as numbers per day, with
  no addresses.
- **"Find me" stays in your browser.** Your position is never sent, and never put in a shared link.
- **"Report an error"** sends a category, the place the camera looks at and an optional short note. It never sends a
  name, an address or an id, and reports are deleted after 90 days.

## 11. How it was built

One person and an AI coding assistant (Claude, by Anthropic):

- **The start:** 26 September 2026, online four days later.
- **By mid-October:** about 560 commits, about 12 600 lines of JavaScript and 9 300 of Python.

The person decided what to build, looked at the result and complained. The assistant wrote nearly all the code.

What made it work, as far as we can tell:

- **A working model in front of people early.** People exploring a map find missing data and odd results faster
  than any specification meeting. The first post's comments became checks within days (Section 7, and the list in
  the follow-up post).
- **Keep asking the "dumb" question.** Is this too complicated, is it slow, can we do it differently? Asked again and
  again, not once at the start.
- **Measure before and after.** Every rule change is an A/B at the same places and the same time of day, with the old
  rule one switch away (each rule has its own, e.g. `?oneway=0`).
- **Checks in code, not in someone's head** (Section 8).
- **Everything written down as it happens:** a running log of what was done, why, and what is left. The next session
  starts from the log, not from memory.

What it cost in AI use is not measured. The work ran on one subscription alongside other projects, and it hit the
plan's ceiling several times.

## 12. What we would keep in a rebuild

The code grew feature by feature, and a spec-driven rebuild ("phoenix build") is planned. From what is known today:

- **Keep:**
  - rules over data, with every rule switchable;
  - adapters per source;
  - one pyramid grid aligned to powers of two;
  - the traffic monitor and the night survey;
  - health by results;
  - one shared cache in front of every live source.
- **Change:**
  - one shared state object from the start, instead of names added as needed;
  - fewer, clearer controls (the panel grew one button at a time);
  - NVDB's duplicate ramp lines dropped in the bake, not worked around in the traffic.

*This list grows as the rebuild is planned.*

## 13. Limits

- **Terrain:** where the laser and the infrastructure disagree, a rule decides, and it is sometimes wrong. Bridges
  over old ground, new roads over old laser.
- **Road traffic:** a typical day, not today.
- **Detail:** only where baked. The rest of the country is coarser until someone asks.
- **The data:** its own quirks show up as they are. That is partly the point: the report lists them for the people
  who can fix them at the source.
