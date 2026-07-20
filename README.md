# SpoolTrack

Filament inventory for the print farm. A single static `index.html`, same shape
as Price My Print and QuoteBench, launched from PrintGrange with the same Google
sign-in.

## What it does

- **Scan a spool** with the USB barcode scanner. It's a keyboard wedge, so
  SpoolTrack listens for the fast keystroke burst anywhere on the page — you
  never have to click into a box first.
- **Look the spool up** in the [Open Filament
  Database](https://github.com/OpenFilamentCollective/open-filament-database):
  brand, material, series, colour name, hex, traits, spool weight, diameter,
  density, nozzle/bed temps.
- **Learn unknown barcodes.** See "Barcode coverage" below — most scans won't
  hit. When one misses, you search and pick the filament once, and SpoolTrack
  binds that barcode to it permanently in your own database. The next scan of
  that spool model is instant.
- **Track individual spools** — each with its own cost, vendor, purchase date,
  order ref, and condition (sealed / open / empty). Cost is optional, so
  existing stock can go in without it.
- **See it all** as colour-coded periodic-table tiles driven by the database's
  hex codes, with light and dark themes.

## Barcode coverage — read this before you expect scanning to just work

The open database has no barcode lookup endpoint, and more importantly, most
records carry no barcode at all. Measured against the live dataset
(22,362 purchasable sizes):

| | |
|---|---|
| Rows with any scannable ID | **16.6%** |
| Rows with a GTIN | 12.0% |

And it's very uneven by brand:

| Brand | Coverage |
|---|---|
| ROSA3D | 99% |
| FormFutura | 84% |
| Bambu Lab | 78% |
| Spectrum | 69% |
| Fiberlogy | 60% |
| Polymaker | 33% |
| SUNLU, eSUN, Overture, ELEGOO, Protopasta, PolyLite, Eryone, Nebula | **0%** |

That's why the learn-once alias flow exists. Scanning Bambu spools mostly works
out of the box; scanning SUNLU never will until you teach it. After a few weeks
of intake your own alias table is the thing carrying most scans.

## How the catalog works

There's no barcode endpoint to call, so SpoolTrack pulls the whole bulk dump
once, flattens it, and queries it locally:

1. `GET /api/v1/index.json` — a tiny meta file with a `version` string.
2. If the version changed (or nothing is cached), `GET /json/all.json` —
   11.6 MB of JSON that transfers as **~3.0 MB** gzipped. `access-control-allow-origin: *`,
   so the browser can fetch it directly.
3. Flatten brands → materials → filaments → variants → sizes into one row per
   purchasable size (22,362 rows, ~29 ms), build a barcode index and a search
   index, and store it in IndexedDB.

After the first load it's offline-capable and lookups are instant. The version
check means normal startups transfer a few hundred bytes, not 3 MB.

Two data quirks are handled deliberately:

- A filament's `brand_id` doesn't always resolve; the join falls back through
  its material. Without that, brands come back "Unknown".
- Tiles group on what a spool *is* (brand + series + colour + hex + weight +
  diameter), not on the catalog UUID, because the database carries ~4,100 rows
  that describe the same product under different UUIDs. Grouping by UUID would
  split one filament across two tiles. This also merges manual entries with
  catalog ones.

## Data model

Everything is per-user under `users/{uid}/`:

- `spools/{id}` — one document per physical spool, with a denormalised snapshot
  of the filament so inventory still renders if the catalog changes underneath.
- `aliases/{barcode}` — learned barcode bindings, keyed by the normalised code
  so a rescan is one direct read. UPC-A and its EAN-13 form are both indexed.

## Setup

1. **Deploy.** Push this folder to a `SpoolTrack` GitHub repo with Pages on, so
   it serves from `https://andyschwartz.github.io/SpoolTrack/` — the same origin
   as the other modules, which is what lets one sign-in cover them all.
2. **Firestore.** If the `price-my-print` project doesn't have Firestore enabled
   yet, enable it (Firebase console → Build → Firestore Database → Create).
3. **Rules.** Merge the `spools` and `aliases` blocks from `firestore.rules`
   into the project's existing rules. Don't replace the file wholesale — the
   other modules share this project and would lose access.
4. PrintGrange already links to it; the card shows up once Pages is live.

## Things worth adding later

- Remaining-weight tracking per spool, for "nearly empty" and reorder alerts.
- Contributing learned barcodes back upstream as PRs to the open database.
- Pulling filament cost into Price My Print / QuoteBench so quotes use real
  per-gram cost instead of a typed estimate.
