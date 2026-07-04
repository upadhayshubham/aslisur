# Music Upgrade Pipeline — Low Level Design v4

## Overview

Automated pipeline to upgrade an MP3 library (~3000 files, ~12GB) to AAC 256kbps.
Source: lossless FLAC from public torrent trackers via Lidarr + Prowlarr + qBittorrent.
Output: `~/Music-HQ/` (flat folder, deduplicated filenames). Originals in `~/Music/` untouched.

**Core invariant**: files never move. `file_path` is a stable, permanent key.
**Resumable**: every phase can be killed and restarted safely. All state lives in PostgreSQL.

---

## Goals

- Scan `~/Music/`, record every MP3 in PostgreSQL with UUID anchor
- Correct metadata tags via beets acoustic fingerprinting (AcoustID + MusicBrainz)
- Download lossless FLAC per-album via Lidarr → extract only the matching track by MusicBrainz Recording ID
- Convert FLAC track → AAC 256k `.m4a`, save to `~/Music-HQ/`
- Originals untouched

## Non-Goals

- Transcoding existing MP3s (no quality gain from lossy→lossy)
- YouTube / streaming as source (quality ceiling too low)
- Moving or renaming files in `~/Music/`
- Real-time monitoring (batch only)
- Multi-user / multi-machine setup

---

## Tech Stack

### Runtime

| Component | Technology | Version | Install |
|---|---|---|---|
| Language | Python | 3.11+ | system |
| Database | PostgreSQL | 15.x | Docker |
| DB driver | psycopg2-binary | 2.9+ | pip |
| Audio metadata | mutagen | 1.47+ | pip |
| Fingerprinting + tagging | beets | 1.6+ | pip |
| Chromaprint binary | fpcalc | Latest | `brew install chromaprint` |
| Audio conversion | FFmpeg | 6.x (Homebrew build) | `brew install ffmpeg` |
| Containerisation | Docker Desktop | Latest | docker.com |
| Config loading | python-dotenv | 1.0+ | pip |

> **FFmpeg encoder note**: Homebrew FFmpeg uses the native `aac` encoder (built-in to FFmpeg).
> `libfdk_aac` is higher quality but requires a non-default build. Native `aac` at 256k is
> transparent for most listeners and avoids build complexity. Use native `aac`.

### ARR Stack (Docker containers)

| Container | Image | Port (host) | Purpose |
|---|---|---|---|
| PostgreSQL | `postgres:15` | 5432 | Pipeline database |
| Gluetun | `qmcgaw/gluetun` | — | VPN tunnel (Surfshark WireGuard) |
| qBittorrent | `lscr.io/linuxserver/qbittorrent` | 8080 (via Gluetun) | Download client |
| Prowlarr | `lscr.io/linuxserver/prowlarr` | 9696 | Indexer manager |
| Lidarr | `lscr.io/linuxserver/lidarr` | 8686 | Music automation |

### External Services

| Service | Used by | Rate limit | Handled by |
|---|---|---|---|
| AcoustID API | beets (chroma plugin) | 3 req/sec | beets internally |
| MusicBrainz API | beets + Lidarr | 1 req/sec | beets + Lidarr internally |
| Public trackers | Prowlarr | Varies by tracker | 10s delay between album searches in Phase 3 |

---

## Network Architecture

```
MacBook host
│
├── Python scripts        (run directly on host)
├── PostgreSQL            (Docker, port 5432 exposed to host)
├── Lidarr                (Docker, host network mode, port 8686)
├── Prowlarr              (Docker, host network mode, port 9696)
│
└── Gluetun container     (VPN — Surfshark WireGuard)
    └── qBittorrent       (runs inside Gluetun network namespace)
                          (host accesses qBittorrent UI at localhost:8080 via Gluetun port forward)
                          (Lidarr talks to qBittorrent via Docker network: http://qbittorrent:8080)

Traffic flow for downloads:
  Lidarr → tells qBittorrent to download → qBittorrent → Gluetun VPN → public trackers
  VPN kill switch: if Gluetun VPN drops, qBittorrent loses network entirely (cannot leak)
  Phase 3 Python script → talks only to Lidarr API (localhost:8686) — never to qBittorrent directly
```

---

## PostgreSQL Schema

### Connection

```
host:     localhost
port:     5432
database: music_pipeline
user:     pipeline
password: defined in .env
schema:   music
```

### Schema

```sql
CREATE SCHEMA IF NOT EXISTS music;

-- pg15 has gen_random_uuid() built-in, no extension needed
-- If on pg13/14: CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

### Status Enum

```sql
CREATE TYPE music.track_status AS ENUM (
    'ingested',             -- Phase 1 done: in DB, raw tags read
    'needs_manual_review',  -- beets confidence below threshold OR beets total miss
    'verified',             -- Phase 2 done: all three MBIDs populated, ready for download
    'queued',               -- Lidarr AlbumSearch triggered
    'downloading',          -- Torrent active in qBittorrent (item in Lidarr queue)
    'converting',           -- FFmpeg running
    'done',                 -- AAC in ~/Music-HQ/, complete
    'failed',               -- Download attempt failed, retry_count < MAX_RETRY_COUNT
    'convert_failed',       -- FFmpeg failed; temp FLAC kept for manual inspection
    'permanent_failed',     -- retry_count >= MAX_RETRY_COUNT OR 48h elapsed
    'file_missing'          -- file_path no longer exists on disk at Phase 2 sync time
);
```

### Table: `music.tracks`

```sql
CREATE TABLE music.tracks (
    id                UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    file_path         VARCHAR(1000) UNIQUE NOT NULL,  -- absolute path, never changes
    filename          VARCHAR(500)  NOT NULL,          -- original filename without extension
    file_size_mb      NUMERIC(8,2),
    duration_sec      INTEGER,                         -- read from mutagen at ingest

    -- Raw tags from MP3 at ingest time (never overwritten after ingest)
    raw_title         VARCHAR(500),
    raw_artist        VARCHAR(500),
    raw_album         VARCHAR(500),
    raw_year          SMALLINT,                        -- integer year only; NULL if tag absent/unparseable
    raw_bitrate_kbps  SMALLINT,

    -- Corrected tags written by beets into the MP3 file, then read back by Phase 2
    verified_title    VARCHAR(500),
    verified_artist   VARCHAR(500),
    verified_album    VARCHAR(500),
    verified_year     SMALLINT,
    beets_confidence  NUMERIC(4,3),                   -- 0.000–1.000; NULL if beets total miss

    -- MusicBrainz IDs — populated in Phase 2, used in Phase 3
    recording_mbid       UUID,   -- MusicBrainz Recording ID (beets: mb_trackid) — matches FLAC track tag
    release_group_mbid   UUID,   -- MusicBrainz Release Group ID (beets: mb_releasegroupid) — used as foreignAlbumId in Lidarr
    artist_mbid          UUID,   -- MusicBrainz Artist ID (beets: mb_artistid) — used as foreignArtistId in Lidarr
    -- NOTE: 'foreignAlbumId' in Lidarr API = MusicBrainz Release Group MBID, NOT Release MBID
    -- beets field mb_albumid = Release ID (NOT Release Group ID) — do NOT use for Lidarr
    -- beets field mb_releasegroupid = Release Group ID — use THIS for Lidarr

    -- Phase 3 tracking
    lidarr_album_id   INTEGER,                         -- Lidarr internal album ID (from GET /api/v1/album)
    batch_no          INTEGER,
    status            music.track_status NOT NULL DEFAULT 'ingested',
    flac_temp_path    VARCHAR(1000),                   -- full path to matched FLAC track file
    aac_final_path    VARCHAR(1000),                   -- full path in ~/Music-HQ/
    error_message     TEXT,
    retry_count       SMALLINT DEFAULT 0,
    download_queued_at TIMESTAMPTZ,                    -- when AlbumSearch was last triggered

    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Table: `music.batches`

```sql
CREATE TABLE music.batches (
    id           SERIAL PRIMARY KEY,
    batch_no     INTEGER  NOT NULL,
    phase        SMALLINT NOT NULL CHECK (phase IN (1, 2, 3)),
    total_files  INTEGER  NOT NULL,
    processed    INTEGER  NOT NULL DEFAULT 0,
    failed       INTEGER  NOT NULL DEFAULT 0,
    status       VARCHAR(20) NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending', 'in_progress', 'done')),
    started_at   TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,

    UNIQUE (batch_no, phase)  -- prevents duplicate batch rows on re-run
);
```

### Triggers and Indexes

```sql
-- Auto-update updated_at on any row change
CREATE OR REPLACE FUNCTION music.fn_update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_tracks_updated_at
BEFORE UPDATE ON music.tracks
FOR EACH ROW EXECUTE FUNCTION music.fn_update_updated_at();

-- Query indexes
CREATE INDEX idx_tracks_status              ON music.tracks(status);
CREATE INDEX idx_tracks_batch_no            ON music.tracks(batch_no);
CREATE INDEX idx_tracks_release_group_mbid  ON music.tracks(release_group_mbid);
CREATE INDEX idx_tracks_recording_mbid      ON music.tracks(recording_mbid);
CREATE INDEX idx_tracks_lidarr_album_id     ON music.tracks(lidarr_album_id);
CREATE INDEX idx_batches_phase_status       ON music.batches(phase, status);
```

### Status State Machine

```
[Phase 1]
   │
   ▼
ingested ──────────────────────────────────────────────► needs_manual_review
   │                                                              │
   │  [Phase 2]                               (manual CLI:        │
   │                                    [a]accept / [m]manual /   │
   │                                    --accept-low-confidence)  │
   ▼                                                              │
verified ◄────────────────────────────────────────────────────────┘
   │
   │  [Phase 3]
   ▼
queued  (Lidarr AlbumSearch triggered, download_queued_at set)
   │
   ▼
downloading  (item visible in Lidarr queue)
   │
   ├──► failed ──► [retry_count < MAX] ──► queued (re-trigger after 24h)
   │         └──── [retry_count >= MAX OR >48h since queued] ──► permanent_failed
   │
   ▼
converting  (FFmpeg subprocess running)
   │
   ├──► convert_failed  (FFmpeg error or duration mismatch; FLAC kept)
   │
   ▼
done  (AAC verified, aac_final_path set)

Special:
   any phase ──► file_missing  (file_path not found on disk at Phase 2 time)

On Phase 3 script startup:
   downloading (stale >2h) ──► queued  (reset stale stuck downloads)
```

---

## Phase 1 — Ingest

### Purpose

Scan `~/Music/` recursively. For every `.mp3` file: assign UUID, read ID3 tags via mutagen,
write row to `music.tracks`. No external calls. Fully offline. Idempotent.

### Inputs

- `~/Music/` directory (path from `MUSIC_DIR` in `.env`)

### Outputs

- `music.tracks` — one row per MP3, all `status = ingested`
- `music.batches` — one row per batch, `phase = 1`

### Batch Size

100 files per batch. No external calls — batching for resume safety only.

### Processing Logic

```python
# Pseudocode
on startup:
    incomplete_batch = SELECT * FROM music.batches
                       WHERE phase=1 AND status='in_progress'
                       ORDER BY batch_no LIMIT 1
    if found:
        resume_from_batch_no = incomplete_batch.batch_no
    else:
        resume_from_batch_no = 0

all_mp3s = sorted list of all .mp3 files under MUSIC_DIR (os.walk, recursive)
batches  = chunk(all_mp3s, 100)

for batch_no, files in enumerate(batches, start=1):
    if batch_no < resume_from_batch_no:
        continue  # already done

    INSERT INTO music.batches (batch_no, phase, total_files, status, started_at)
    VALUES (batch_no, 1, len(files), 'in_progress', NOW())
    ON CONFLICT (batch_no, phase) DO UPDATE SET status='in_progress', started_at=NOW()

    for file_path in files:
        if SELECT 1 FROM music.tracks WHERE file_path = file_path:
            continue  # idempotent skip

        tags = mutagen.File(file_path, easy=False)  # use ID3 directly, not EasyID3
        row = {
            file_path:        file_path,
            filename:         Path(file_path).stem,
            file_size_mb:     os.path.getsize(file_path) / 1_048_576,
            duration_sec:     int(tags.info.length) if tags else None,
            raw_title:        tags['TIT2'].text[0] if 'TIT2' in tags else None,
            raw_artist:       tags['TPE1'].text[0] if 'TPE1' in tags else None,
            raw_album:        tags['TALB'].text[0] if 'TALB' in tags else None,
            raw_year:         int(str(tags['TDRC'])[0:4]) if 'TDRC' in tags else None,
            raw_bitrate_kbps: tags.info.bitrate // 1000 if tags else None,
            status:           'ingested'
        }
        INSERT INTO music.tracks (...) VALUES (...)

    UPDATE music.batches SET status='done', processed=N, failed=M, completed_at=NOW()
    WHERE batch_no=batch_no AND phase=1
```

### Tag Frame Names (mutagen ID3)

| Field | ID3 frame | Mutagen access |
|---|---|---|
| Title | `TIT2` | `tags['TIT2'].text[0]` |
| Artist | `TPE1` | `tags['TPE1'].text[0]` |
| Album | `TALB` | `tags['TALB'].text[0]` |
| Year | `TDRC` | `str(tags['TDRC'])[0:4]` → cast to int |
| Bitrate | info field | `tags.info.bitrate // 1000` |
| Duration | info field | `tags.info.length` (seconds, float) |

### Error Handling

| Error | Action |
|---|---|
| `os.walk` permission denied on subfolder | Log warning + skip subfolder, continue |
| File unreadable / corrupted | Log + skip file, `failed++` in batch |
| mutagen parse error | Store `filename` only, all tag fields NULL, `status = ingested` |
| DB write error | Retry 3× with 1s backoff, then log + skip |
| Duplicate `file_path` (ON CONFLICT) | Skip — already ingested |

---

## Phase 2 — Tag Correction

### Purpose

Run beets acoustic fingerprinting on `~/Music/`. beets corrects ID3 tags in the MP3 files
using AcoustID + MusicBrainz. Then Phase 2 script reads the corrected tags + MBIDs from
beets' library.db and writes them into our `music.tracks` DB.

### Sub-phase 2A — beets run

#### Idempotency check

```python
# Phase 2 script checks beets library.db before running beets
beets_db_path = Path("~/.config/beets/library.db").expanduser()
conn = sqlite3.connect(beets_db_path)
count = conn.execute("SELECT COUNT(*) FROM items").fetchone()[0]
if count > 0:
    log("beets library already populated ({count} items). Skipping beets run.")
else:
    run_beets()
```

#### beets command (exact)

```bash
beet -c ~/.config/beets/config.yaml import -q ~/Music
```

- `-c` — explicit config path (avoids ambient config interference)
- `import` — the correct subcommand for adding/tagging an existing library
- `-q` — quiet mode: **SKIPS** files that need user input (low confidence, ambiguous match).
  Does NOT auto-accept low-confidence matches. Only strong matches (distance <= strong_rec_thresh)
  are auto-accepted. Everything else is skipped with no tag write.

> **beets `import` vs `update`**: `import` is correct here even for an existing library.
> `beet update` only refreshes existing beets library entries, not fingerprint-match.
> `beet import -q` with `copy:no move:no` fingerprints, matches, and writes tags in-place.

#### beets config (`~/.config/beets/config.yaml`)

```yaml
directory: ~/Music
library: ~/.config/beets/library.db
plugins: chroma
# acousticbrainz plugin intentionally omitted — AcousticBrainz API shut down Nov 2022

import:
  copy: no          # never copy files
  move: no          # never move files
  write: yes        # write corrected ID3 tags to MP3 file in place
  timid: no         # do not ask for confirmation on strong matches
  quiet: yes        # SKIPS files below confidence threshold (does NOT auto-accept them)
                    # quiet_fallback defaults to 'skip' — do not change this
  log: ~/.config/beets/logs/beets_import.log   # logs all skipped files (review these for needs_manual_review)

match:
  strong_rec_thresh: 0.10   # match distance <= 0.10 → auto-accept (lower = better)
  medium_rec_thresh: 0.25   # distance 0.10–0.25 → skipped in quiet mode
  # distance > 0.25 → also skipped in quiet mode
  # ALL skipped files → no tags written → Phase 2 sets status = needs_manual_review
```

> **beets quiet mode confirmed behavior** (official docs):
> Only files with match distance <= `strong_rec_thresh` are auto-accepted.
> All other files are skipped entirely — no ID3 tags written, no MBID.
> `quiet_fallback: skip` (the default) means skipped files remain unmodified.
> Do NOT set `quiet_fallback: asis` — that imports files with original broken tags, bypassing matching.

> **beets confidence storage**: confidence is stored in beets' own `library.db`
> (`items.comp` column — lower value = better match, 0.0 = perfect). It is NOT written
> to the MP3 file tag. Phase 2 script reads it from `library.db` by matching on `path`.
> Our DB stores it as `beets_confidence = 1.0 - items.comp` (inverted to 0–1 scale where
> 1.0 = perfect).

#### beets subprocess (Phase 2 script)

```python
import subprocess, sys

proc = subprocess.Popen(
    ["beet", "-c", str(beets_config_path), "import", "-q", str(MUSIC_DIR)],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True
)
with open("logs/phase2_beets.log", "w") as logfile:
    for line in proc.stdout:
        sys.stdout.write(line)   # live terminal output
        logfile.write(line)      # persist to log
proc.wait()
if proc.returncode != 0:
    raise RuntimeError(f"beets exited with code {proc.returncode}")
```

Expected runtime: 2–6 hours for 3000 files (AcoustID API at 3 req/sec).

### Sub-phase 2B — Python sync

Reads beets results into `music.tracks`. Runs after beets finishes (or is skipped).

#### Inputs

- `music.tracks` — rows with `status = ingested`
- MP3 files (now have corrected ID3 tags including MusicBrainz IDs written by beets)
- `~/.config/beets/library.db` — beets SQLite DB

#### Outputs

- `music.tracks` — `status = verified` or `status = needs_manual_review`
- `verified_*` fields populated
- `recording_mbid`, `release_group_mbid`, `artist_mbid`, `beets_confidence` populated

#### Batch Size

50 files per batch.

#### Tag field names for MBIDs (mutagen on MP3 file after beets tags it)

beets writes MusicBrainz IDs as ID3 TXXX frames:

| Field | Mutagen frame key |
|---|---|
| MusicBrainz Recording ID | `TXXX:MusicBrainz Recording Id` |
| MusicBrainz Release ID (album) | `TXXX:MusicBrainz Album Id` |
| MusicBrainz Artist ID | `TXXX:MusicBrainz Artist Id` |

Access pattern:
```python
from mutagen.id3 import ID3
tags = ID3(file_path)
recording_mbid = tags.get('TXXX:MusicBrainz Recording Id')
recording_mbid = recording_mbid.text[0] if recording_mbid else None
```

#### beets library.db query (for confidence + MBIDs)

```sql
-- beets library.db schema (SQLite)
-- Relevant mb_ fields:
--   mb_trackid         = MusicBrainz Recording ID  → our recording_mbid
--   mb_releasegroupid  = MusicBrainz Release Group ID → our release_group_mbid (used for Lidarr foreignAlbumId)
--   mb_albumid         = MusicBrainz Release ID (NOT used for Lidarr — do not confuse with Release Group)
--   mb_artistid        = MusicBrainz Artist ID → our artist_mbid
--   comp               = match distance (0.0 = perfect, higher = worse)

SELECT mb_trackid, mb_releasegroupid, mb_artistid, comp, title, artist, album, year
FROM items
WHERE path = ?   -- absolute path, same as our file_path
```

Confidence conversion: `beets_confidence = round(1.0 - comp, 3)`

> **mb_albumid vs mb_releasegroupid**: beets `mb_albumid` is the MusicBrainz Release ID (specific
> pressing/edition). Lidarr's `foreignAlbumId` is the Release Group ID (the abstract album,
> all editions grouped). Always use `mb_releasegroupid` for Lidarr — using `mb_albumid` will
> return empty results from Lidarr's `GET /api/v1/album?foreignAlbumId=`.

#### Processing Logic

```python
for each batch of 50 tracks where status = ingested:
    for each track:
        if not os.path.exists(track.file_path):
            UPDATE status = 'file_missing'
            continue

        tags = ID3(track.file_path)
        recording_mbid = tags.get('TXXX:MusicBrainz Recording Id')?.text[0]

        if recording_mbid is not None:
            beets_row = beets_db.execute(
                "SELECT mb_trackid, mb_albumid, mb_artistid, comp, title, artist, album, year "
                "FROM items WHERE path = ?", (track.file_path,)
            ).fetchone()

            if beets_row:
                UPDATE music.tracks SET
                    recording_mbid       = beets_row.mb_trackid,         -- Recording ID (for FLAC matching)
                    release_group_mbid   = beets_row.mb_releasegroupid,  -- Release Group ID (for Lidarr foreignAlbumId)
                    artist_mbid          = beets_row.mb_artistid,        -- Artist ID (for Lidarr foreignArtistId)
                    beets_confidence     = 1.0 - beets_row.comp,
                    verified_title       = beets_row.title,
                    verified_artist      = beets_row.artist,
                    verified_album       = beets_row.album,
                    verified_year        = beets_row.year,
                    status               = 'verified'
            else:
                # MBID in file but not in beets DB — shouldn't happen normally
                UPDATE status = 'needs_manual_review', error_message = 'MBID in file but missing from beets DB'
        else:
            # beets did not tag this file (below confidence threshold or total miss)
            # try to read partial data from beets DB anyway (for confidence score display in manual review)
            beets_row = beets_db.execute("SELECT comp FROM items WHERE path = ?", ...).fetchone()
            beets_confidence = 1.0 - beets_row.comp if beets_row else None
            UPDATE status = 'needs_manual_review', beets_confidence = beets_confidence

    update music.batches
```

### Manual Review CLI (`phase2_manual_review.py`)

#### Mode 1: One-by-one review

```bash
python phase2_manual_review.py
```

For each `needs_manual_review` row:
```
File:       /Users/you/Music/01 - Bohemian Rhapsody.mp3
Raw tags:   Artist: Queen | Album: A Night at the Opera | Title: Bohemian Rhapsody
Beets guess: Artist: Queen | Album: A Night at the Opera | Confidence: 0.18 (LOW)
---
[a] Accept beets guess
[s] Skip (keep as needs_manual_review)
[m] Manual — enter artist + album name → lookup MusicBrainz API for MBIDs
[q] Quit
```

On `[m]`: script calls `https://musicbrainz.org/ws/2/recording?query=...&fmt=json`,
shows top 3 results, user picks one, script writes MBIDs to DB and `status = verified`.

#### Mode 2: Bulk accept

```bash
python phase2_manual_review.py --accept-low-confidence --min-confidence 0.15
```

- Selects `needs_manual_review` rows where `beets_confidence >= 0.15` AND `beets_confidence IS NOT NULL`
- Rows where `beets_confidence IS NULL` (total beets miss) are never auto-accepted — must go through Mode 1
- Writes `status = verified` for all accepted rows
- Logs every auto-accepted track to `logs/phase2_auto_accepted.log` with artist/album/confidence

### Error Handling

| Error | Action |
|---|---|
| `file_path` not found on disk | `status = file_missing`, continue |
| beets did not tag file (below threshold) | `status = needs_manual_review` |
| beets total miss (not in library.db at all) | `status = needs_manual_review`, `beets_confidence = NULL` |
| beets.db file not found | Halt with message: "beets library not found — run Phase 2 first" |
| mutagen ID3 parse error on file | Log + `status = needs_manual_review` |
| MusicBrainz API fails (manual mode) | Retry after 60s, max 3 attempts |

---

## Phase 3 — Download and Convert

### Purpose

For each `status = verified` track: add artist+album to Lidarr, trigger FLAC download,
extract the specific track matched by `recording_mbid` from the downloaded album folder,
convert to AAC 256k, save to `~/Music-HQ/`. Process one album at a time.

### Inputs

- `music.tracks` — rows with `status = verified`
- Lidarr REST API at `http://localhost:8686`
- `~/pipeline/downloads/` — where Lidarr/qBittorrent saves completed downloads

### Outputs

- `~/Music-HQ/{deduped_filename}.m4a` for each track
- `music.tracks` — `status = done`, `aac_final_path` set
- Temp FLAC album folder deleted after all needed tracks extracted

### Startup Reset (stale state recovery)

```python
# On every Phase 3 startup:
UPDATE music.tracks
SET status = 'queued', error_message = 'Reset from stale downloading state on restart'
WHERE status = 'downloading'
  AND download_queued_at < NOW() - INTERVAL '2 hours'
# Handles crash during active download — will re-trigger search
```

### Batch Strategy

Process one album at a time:
1. Group all `verified` tracks by `release_group_mbid`
2. For each unique album: run full cycle (Lidarr add → download → extract → convert all needed tracks → cleanup)
3. Only start next album after current is fully `done` or `permanent_failed`
4. 10s sleep between album searches (public tracker rate safety)

### Lidarr API Sequence

All requests include header: `X-Api-Key: {LIDARR_API_KEY}`

#### Step 1: Add artist (idempotent)

```
POST http://localhost:8686/api/v1/artist
Content-Type: application/json
X-Api-Key: {LIDARR_API_KEY}

{
  "foreignArtistId": "{artist_mbid}",  -- MusicBrainz Artist ID; field name is 'foreignArtistId' NOT 'mbId'
  "qualityProfileId": 1,               -- "FLAC Only" profile (configured in Lidarr UI, get ID from GET /api/v1/qualityprofile)
  "metadataProfileId": 1,              -- get from GET /api/v1/metadataprofile
  "rootFolderPath": "/music",          -- container path (required field)
  "monitored": true,
  "addOptions": {
    "searchForMissingAlbums": false    -- do NOT search entire discography
  }
}

Success: 201 Created → body contains { "id": <lidarr_artist_id>, "foreignArtistId": "...", ... }
Already exists: 400 with body containing "ArtistExistsValidator"
  → handle: GET /api/v1/artist?foreignArtistId={artist_mbid} to fetch existing record
```

#### Step 2: Find album by MusicBrainz Release Group ID

```
GET http://localhost:8686/api/v1/album?foreignAlbumId={release_group_mbid}
-- IMPORTANT: foreignAlbumId = MusicBrainz RELEASE GROUP ID (not Release ID)
-- beets field: mb_releasegroupid  (NOT mb_albumid which is Release ID)

Success: 200 → array; take first element → { "id": <lidarr_album_id>, ... }
Empty array: Lidarr hasn't indexed artist's discography yet
  → wait 30s, retry GET up to 5× with 30s gap
  → if still empty after 5 retries: status = permanent_failed, error_message = 'Album not found in Lidarr'
```

#### Step 3: Monitor album

```
PUT http://localhost:8686/api/v1/album/monitor
Content-Type: application/json
X-Api-Key: {LIDARR_API_KEY}

{
  "albumIds": [{lidarr_album_id}],   -- array of Lidarr integer IDs, NOT MusicBrainz MBIDs
  "monitored": true
}

Success: 202
Store lidarr_album_id in music.tracks for all tracks belonging to this album.
```

#### Step 4: Trigger search

```
POST http://localhost:8686/api/v1/command
Content-Type: application/json
X-Api-Key: {LIDARR_API_KEY}

{
  "name": "AlbumSearch",
  "albumIds": [{lidarr_album_id}]    -- undocumented in OpenAPI spec; confirmed by community SDKs (golift/starr, pyarr)
}

Success: 201 → { "id": <command_id>, "status": "queued" }
Update all tracks for this album: status = 'queued', download_queued_at = NOW()
```

### Polling Loop

```python
while True:
    time.sleep(60)  # poll every 60 seconds

    # Check Lidarr queue for this album
    queue = GET /api/v1/queue   # returns all active downloads
    album_items = [item for item in queue if item['albumId'] == lidarr_album_id]

    if not album_items:
        # Item not in queue at all — two possibilities:
        # 1. Download completed (Lidarr removes completed items from queue)
        # 2. No torrent found (Lidarr never queued it)

        # Check if download folder has appeared
        album_folder = find_album_folder_in_downloads(release_group_mbid)
        if album_folder:
            # Download complete — proceed to extraction
            UPDATE status = 'downloading' for all album tracks  -- mark as arrived
            break

        # Check timeout
        if (NOW() - download_queued_at) > DOWNLOAD_TIMEOUT_HOURS * 3600:
            retry_count += 1
            if retry_count >= MAX_RETRY_COUNT:
                UPDATE status = 'permanent_failed'
            else:
                re-trigger AlbumSearch, reset download_queued_at
            break
    else:
        # Item in queue — get status
        item = album_items[0]
        if item['status'] in ('downloading', 'queued'):
            UPDATE status = 'downloading' for all album tracks
            continue  # keep polling
```

### Finding the Downloaded Album Folder

Lidarr saves completed downloads to:
```
~/pipeline/downloads/{Artist Name}/{Album Name} ({Year})/
```

The exact folder name depends on Lidarr's naming convention. Discovery approach:
```python
def find_album_folder_in_downloads(release_group_mbid, lidarr_album_id):
    # Ask Lidarr for the album's expected path
    album = GET /api/v1/album/{lidarr_album_id}
    artist_name = album['artist']['artistName']
    album_title = album['title']
    album_year  = album['releaseDate'][0:4]

    # Search downloads folder for matching subfolder
    pattern = f"{DOWNLOADS_DIR}/{artist_name}*/{album_title}*"
    matches = glob.glob(pattern)
    if matches:
        return matches[0]

    # Fallback: search by modification time (most recently completed)
    # Only used if name match fails
    all_subdirs = [d for d in Path(DOWNLOADS_DIR).rglob('*') if d.is_dir()]
    recent = sorted(all_subdirs, key=lambda d: d.stat().st_mtime, reverse=True)
    return recent[0] if recent else None
```

### FLAC Track Extraction

FLAC files use Vorbis comment tags (not ID3). MusicBrainz Recording ID is stored as:

| Tag | Vorbis comment key |
|---|---|
| MusicBrainz Recording ID | `MUSICBRAINZ_TRACKID` |

```python
from mutagen.flac import FLAC

def find_matching_flac(album_folder, target_recording_mbid):
    for flac_file in Path(album_folder).rglob("*.flac"):
        tags = FLAC(str(flac_file))
        # Vorbis comment keys are case-insensitive; mutagen normalizes to uppercase
        mbid = tags.get('MUSICBRAINZ_TRACKID', [None])[0]
        if mbid and mbid.lower() == target_recording_mbid.lower():
            return str(flac_file)
    return None  # no match found
```

If no match found:
- Log which album folder was scanned and what MBIDs were found
- `status = failed` (will retry — maybe re-download gets different torrent with proper tags)
- Leave album folder intact for retry

### Output Filename Deduplication

Output folder `~/Music-HQ/` is flat. Filename collisions are possible (two different artists
with a track named `01 - Love.mp3`).

Dedup strategy:
```python
def output_filename(track):
    # Prefer verified artist if available, else raw artist
    artist = track.verified_artist or track.raw_artist or "Unknown"
    base = f"{artist} - {track.filename}"  # e.g. "Queen - 01 - Bohemian Rhapsody"
    path = Path(OUTPUT_DIR) / f"{base}.m4a"

    if path.exists():
        # Already exists — check if it's ours (same UUID via extended attribute or DB lookup)
        existing = SELECT aac_final_path FROM music.tracks WHERE aac_final_path = str(path)
        if existing and existing.id != track.id:
            # True collision — append UUID suffix
            path = Path(OUTPUT_DIR) / f"{base}_{str(track.id)[:8]}.m4a"
    return str(path)
```

### FFmpeg Conversion

```python
import subprocess

cmd = [
    "ffmpeg",
    "-i", flac_temp_path,
    "-c:a", "aac",         # native FFmpeg AAC encoder (Homebrew build default)
    "-b:a", "256k",
    "-movflags", "+faststart",  # valid for .m4a (MP4 container); moves moov atom to front
    "-map_metadata", "0",  # copy all tags from FLAC to AAC
    "-y",                  # overwrite output if exists
    aac_output_path
]

result = subprocess.run(cmd, capture_output=True, text=True)
if result.returncode != 0:
    log(f"FFmpeg stderr: {result.stderr}")
    UPDATE status = 'convert_failed', error_message = result.stderr[:500]
    return  # keep FLAC

# Post-conversion verify
if not Path(aac_output_path).exists():
    UPDATE status = 'convert_failed', error_message = 'Output file not created'
    return

aac_size = Path(aac_output_path).stat().st_size
if aac_size == 0:
    Path(aac_output_path).unlink()
    UPDATE status = 'convert_failed', error_message = 'Output file is 0 bytes'
    return

# Duration check
from mutagen.mp4 import MP4
aac_duration = MP4(aac_output_path).info.length
if abs(aac_duration - track.duration_sec) > 3:
    Path(aac_output_path).unlink()
    UPDATE status = 'convert_failed', error_message = f'Duration mismatch: original={track.duration_sec}s aac={aac_duration:.1f}s'
    return

# All checks pass
UPDATE status = 'done', aac_final_path = aac_output_path
```

### Temp FLAC Cleanup

After all needed tracks from an album are extracted and converted:
```python
# Only delete album folder if ALL tracks from that album are done or permanent_failed
pending = SELECT COUNT(*) FROM music.tracks
          WHERE release_group_mbid = ? AND status NOT IN ('done', 'permanent_failed', 'convert_failed')
if pending == 0:
    shutil.rmtree(album_folder)
    log(f"Cleaned up album folder: {album_folder}")
```

### Error Handling

| Error | Action |
|---|---|
| Lidarr API unreachable | Pause 5 min, retry indefinitely |
| Artist add 400 (already exists) | GET existing artist, continue to Step 2 |
| Artist MBID not found in Lidarr | Log + `status = permanent_failed` — MusicBrainz ID not in Lidarr's metadata |
| Album not found after 5×30s retries | `status = permanent_failed` |
| No torrent found (no queue item, no folder, timeout) | `retry_count++`; requeue after 24h wait |
| `retry_count >= MAX_RETRY_COUNT` | `status = permanent_failed` |
| 48h since `download_queued_at` | `status = permanent_failed` |
| No FLAC with matching `MUSICBRAINZ_TRACKID` | `status = failed`, retry |
| FFmpeg non-zero exit | `status = convert_failed`, keep FLAC, log stderr |
| Duration mismatch > 3s | `status = convert_failed`, delete bad AAC, keep FLAC |
| VPN drops mid-download | qBittorrent loses network via Gluetun kill switch; download pauses; Phase 3 sees stuck queue item; 48h timeout fires |
| Output filename collision | Append `_{uuid8}` suffix |

---

## Docker Compose (`docker-compose.yml`)

```yaml
version: "3.9"

services:

  postgres:
    image: postgres:15
    container_name: music_pipeline_pg
    environment:
      POSTGRES_DB: music_pipeline
      POSTGRES_USER: pipeline
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    environment:
      VPN_SERVICE_PROVIDER: surfshark
      VPN_TYPE: wireguard
      WIREGUARD_PRIVATE_KEY: ${WIREGUARD_PRIVATE_KEY}
      SERVER_COUNTRIES: Netherlands
    ports:
      - "8080:8080"   # qBittorrent WebUI exposed via Gluetun
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"   # all traffic via Gluetun VPN
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Asia/Kolkata
      WEBUI_PORT: 8080
    volumes:
      - ./qbt-config:/config
      - ./downloads:/downloads
    depends_on:
      - gluetun
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    network_mode: host
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Asia/Kolkata
    volumes:
      - ./prowlarr-config:/config
    restart: unless-stopped

  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    network_mode: host
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Asia/Kolkata
    volumes:
      - ./lidarr-config:/config
      - ~/Music:/music
      - ./downloads:/downloads
    restart: unless-stopped
```

> **Lidarr → qBittorrent connection**: Lidarr (host network) connects to qBittorrent via
> `http://localhost:8080` (Gluetun forwards this port). Configure in Lidarr UI:
> Settings → Download Clients → qBittorrent → Host: `localhost`, Port: `8080`.

---

## Lidarr UI Setup (one-time, before Phase 3)

```
1. Open http://localhost:8686

2. Settings → Profiles → Quality Profiles → Add:
   Name: FLAC Only
   Allowed: FLAC
   Rejected: MP3, AAC, OGG, OPUS, WMA, ALAC

3. Settings → Download Clients → Add → qBittorrent:
   Host: localhost
   Port: 8080
   Category: lidarr   (qBittorrent saves to /downloads/lidarr/)

4. Settings → Indexers → (Prowlarr sync):
   Settings → General → copy API key
   Prowlarr → Settings → Apps → Add Lidarr → paste API key
   (Prowlarr will push all configured indexers to Lidarr automatically)

5. Settings → General → Security → copy API key → put in .env as LIDARR_API_KEY
```

> With category `lidarr`, qBittorrent saves downloads to `~/pipeline/downloads/lidarr/`.
> Update `DOWNLOADS_DIR` in `.env` to `~/pipeline/downloads/lidarr`.

---

## Volume Mounts Summary

| Container | Host path | Container path | Purpose |
|---|---|---|---|
| postgres | `./pgdata` | `/var/lib/postgresql/data` | DB files |
| lidarr | `~/Music` | `/music` | Read-only source library |
| lidarr | `./downloads` | `/downloads` | FLAC downloads target |
| lidarr | `./lidarr-config` | `/config` | Lidarr config + DB |
| prowlarr | `./prowlarr-config` | `/config` | Prowlarr config |
| qbittorrent | `./downloads` | `/downloads` | Same host path as Lidarr |
| qbittorrent | `./qbt-config` | `/config` | qBittorrent config |

---

## Project Folder Structure

```
~/pipeline/                           ← working directory for all scripts
├── phase1_ingest.py                  # Phase 1: scan ~/Music/, write to DB
├── phase2_sync.py                    # Phase 2: run beets subprocess + sync to DB
├── phase2_manual_review.py           # CLI: review needs_manual_review rows
├── phase3_download.py                # Phase 3: Lidarr API + poll + extract + convert
├── db.py                             # psycopg2 connection pool + query helpers
├── config.py                         # python-dotenv loader, typed config constants
├── log.py                            # shared logging setup — setup_logger(name)
├── schema.sql                        # Full PostgreSQL DDL (run once to init DB)
├── .env                              # All secrets — never commit to git
├── .gitignore                        # includes: .env, pgdata/, downloads/, *.log
├── docker-compose.yml
├── logs/
│   ├── phase1.log
│   ├── phase2.log
│   ├── phase2_beets.log              # live beets subprocess stdout
│   ├── phase2_auto_accepted.log      # --accept-low-confidence audit trail
│   └── phase3.log
├── lidarr-config/
├── prowlarr-config/
├── qbt-config/
├── pgdata/                           # PostgreSQL data files
└── downloads/                        # FLAC temp folder — Lidarr writes here
    └── lidarr/                       # qBittorrent category subfolder
```

---

## `.env` File (never commit)

```bash
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=music_pipeline
DB_SCHEMA=music
DB_USER=pipeline
DB_PASSWORD=<strong random password>

# Lidarr
LIDARR_URL=http://localhost:8686
LIDARR_API_KEY=<from Lidarr UI Settings → General → Security>

# Paths (absolute paths, no ~ expansion)
MUSIC_DIR=/Users/<you>/Music
OUTPUT_DIR=/Users/<you>/Music-HQ
DOWNLOADS_DIR=/Users/<you>/pipeline/downloads/lidarr

# Gluetun (also referenced in docker-compose.yml)
WIREGUARD_PRIVATE_KEY=<from Surfshark dashboard → Manual setup → WireGuard>

# Pipeline behaviour
BEETS_CONFIDENCE_THRESHOLD=0.25     # used by --accept-low-confidence default
DOWNLOAD_TIMEOUT_HOURS=48
MAX_RETRY_COUNT=3
STALE_DOWNLOAD_RESET_HOURS=2        # reset 'downloading' rows older than this on startup
PHASE1_BATCH_SIZE=100
PHASE2_BATCH_SIZE=50
PHASE3_ALBUM_SEARCH_DELAY_SEC=10    # delay between Lidarr album searches
POLL_INTERVAL_SEC=60                # Lidarr queue poll interval
```

---

## Logging

### Library

Python standard `logging` module. No third-party dependency.

### Log Format

All phases use the same format:

```
%(asctime)s | %(levelname)-8s | %(name)-20s | %(message)s
```

Example output:
```
2026-07-05 14:23:01,412 | INFO     | phase1_ingest        | [batch=3] ingested 100/100 files
2026-07-05 14:23:02,001 | WARNING  | phase1_ingest        | [file=/Users/you/Music/bad.mp3] mutagen parse error — storing filename only
2026-07-05 14:23:03,100 | ERROR    | phase3_download      | [track_id=abc123] FFmpeg failed: exit code 1
```

### Setup (`log.py` — shared by all scripts)

```python
import logging
import logging.handlers
import sys
from pathlib import Path

LOG_DIR = Path(__file__).parent / "logs"
LOG_DIR.mkdir(exist_ok=True)

def setup_logger(name: str) -> logging.Logger:
    logger = logging.getLogger(name)
    logger.setLevel(logging.DEBUG)          # capture everything; handlers filter by level

    fmt = logging.Formatter(
        "%(asctime)s | %(levelname)-8s | %(name)-20s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )

    # Console handler — INFO and above only
    console = logging.StreamHandler(sys.stdout)
    console.setLevel(logging.INFO)
    console.setFormatter(fmt)

    # File handler — DEBUG and above, rotating 10MB × 5 backups
    log_file = LOG_DIR / f"{name}.log"
    file_handler = logging.handlers.RotatingFileHandler(
        log_file, maxBytes=10 * 1024 * 1024, backupCount=5, encoding="utf-8"
    )
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(fmt)

    logger.addHandler(console)
    logger.addHandler(file_handler)
    return logger
```

Usage in each script:
```python
from log import setup_logger
logger = setup_logger("phase1_ingest")   # or "phase2_sync", "phase3_download", etc.
```

### Log Files

| File | Written by | Content |
|---|---|---|
| `logs/phase1_ingest.log` | `phase1_ingest.py` | Scan progress, skips, errors |
| `logs/phase2_sync.log` | `phase2_sync.py` | beets subprocess trigger, MBID sync progress |
| `logs/phase2_beets.log` | `phase2_sync.py` | Raw beets stdout (line-by-line live tail) |
| `logs/phase2_manual_review.log` | `phase2_manual_review.py` | Each accept/skip/manual action |
| `logs/phase2_auto_accepted.log` | `phase2_manual_review.py` | Audit trail for `--accept-low-confidence` |
| `logs/phase3_download.log` | `phase3_download.py` | Lidarr API calls, poll events, extract, convert |

All files rotate at 10MB, keep 5 backups. `phase2_beets.log` is plain write (no rotation —
beets runs once; file is overwritten on re-run).

### Log Levels

| Level | When to use |
|---|---|
| `DEBUG` | Every file processed, every DB row written, every API response body |
| `INFO` | Batch start/end, status transitions, counts, phase milestones |
| `WARNING` | Recoverable issues: mutagen parse fail, file skip, retry attempt |
| `ERROR` | Non-recoverable for this item: FFmpeg fail, DB retry exhausted, MBID mismatch |
| `CRITICAL` | Pipeline halt: beets.db not found, DB unreachable, config missing |

### What Gets Logged (per phase)

#### Phase 1
```python
logger.info("[batch=%d] starting — %d files", batch_no, len(files))
logger.debug("[file=%s] ingested — title=%r artist=%r bitrate=%d", file_path, title, artist, bitrate)
logger.warning("[file=%s] mutagen parse error — %s", file_path, e)
logger.warning("[file=%s] unreadable — skipping", file_path)
logger.error("[file=%s] DB write failed after 3 retries — %s", file_path, e)
logger.info("[batch=%d] done — processed=%d failed=%d", batch_no, processed, failed)
```

#### Phase 2 — sync
```python
logger.info("beets library has %d items — skipping beets run", count)
logger.info("beets library empty — starting beets import subprocess")
logger.info("[beets] exited with code %d", proc.returncode)
logger.debug("[track=%s] recording_mbid=%s release_group_mbid=%s confidence=%.3f", filename, rec, rg, conf)
logger.warning("[track=%s] no MBID in file — status=needs_manual_review", filename)
logger.warning("[track=%s] file not found at path — status=file_missing", filename)
logger.info("[batch=%d] sync done — verified=%d manual_review=%d missing=%d", batch_no, v, m, mi)
```

#### Phase 2 — manual review
```python
logger.info("[manual] user accepted: %s → %s (confidence=%.3f)", filename, artist_album, conf)
logger.info("[manual] user skipped: %s", filename)
logger.info("[manual] user entered manually: %s → recording_mbid=%s", filename, mbid)
logger.info("[auto-accept] %s | %s | confidence=%.3f → verified", artist, album, conf)
```

#### Phase 3
```python
logger.info("[album=%s] starting — %d tracks need download", release_group_mbid, n)
logger.debug("[lidarr] POST /api/v1/artist — artist_mbid=%s → status=%d", artist_mbid, resp.status_code)
logger.debug("[lidarr] GET /api/v1/album?foreignAlbumId=%s → lidarr_album_id=%d", rg_mbid, album_id)
logger.info("[album=%d] AlbumSearch triggered — download_queued_at=%s", lidarr_album_id, ts)
logger.debug("[poll] album_id=%d queue_status=%s elapsed=%.0fs", album_id, status, elapsed)
logger.info("[album=%d] download complete — folder=%s", lidarr_album_id, folder)
logger.warning("[album=%d] no FLAC with matching recording_mbid=%s — status=failed", album_id, rec_mbid)
logger.info("[track=%s] FFmpeg started — input=%s output=%s", filename, flac_path, aac_path)
logger.error("[track=%s] FFmpeg failed (exit=%d) — stderr: %s", filename, rc, stderr[:300])
logger.error("[track=%s] duration mismatch — original=%.1fs aac=%.1fs", filename, orig, aac)
logger.info("[track=%s] done — aac_path=%s", filename, aac_path)
logger.info("[album=%d] cleanup — removed folder=%s", lidarr_album_id, folder)
logger.warning("[startup] reset %d stale downloading tracks → queued", n)
```

### Console vs File

Console (`INFO`+): shows progress — batch counts, phase milestones, warnings, errors.
File (`DEBUG`+): full audit trail — every file touched, every API call, every DB write, every tag extracted.

Running `tail -f logs/phase3_download.log | grep -E "ERROR|WARNING|done"` gives live failure monitoring without debug noise.

---

## Dependency Installation (complete)

```bash
# System tools (macOS)
brew install chromaprint ffmpeg

# Python dependencies
pip install mutagen "beets[chroma]" requests psycopg2-binary python-dotenv

# Verify fpcalc available (required by beets chroma plugin)
fpcalc -version

# Verify FFmpeg has AAC encoder
ffmpeg -encoders 2>/dev/null | grep -E "^\s*A.* aac"
# Expected output line: A....D aac                  AAC (Advanced Audio Coding)
```

---

## schema.sql (complete, run once)

```sql
CREATE SCHEMA IF NOT EXISTS music;

CREATE TYPE music.track_status AS ENUM (
    'ingested',
    'needs_manual_review',
    'verified',
    'queued',
    'downloading',
    'converting',
    'done',
    'failed',
    'convert_failed',
    'permanent_failed',
    'file_missing'
);

CREATE TABLE music.tracks (
    id                UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    file_path         VARCHAR(1000) UNIQUE NOT NULL,
    filename          VARCHAR(500)  NOT NULL,
    file_size_mb      NUMERIC(8,2),
    duration_sec      INTEGER,
    raw_title         VARCHAR(500),
    raw_artist        VARCHAR(500),
    raw_album         VARCHAR(500),
    raw_year          SMALLINT,
    raw_bitrate_kbps  SMALLINT,
    verified_title    VARCHAR(500),
    verified_artist   VARCHAR(500),
    verified_album    VARCHAR(500),
    verified_year     SMALLINT,
    beets_confidence  NUMERIC(4,3),
    recording_mbid       UUID,   -- MusicBrainz Recording ID (beets: mb_trackid)
    release_group_mbid   UUID,   -- MusicBrainz Release Group ID (beets: mb_releasegroupid) — for Lidarr foreignAlbumId
    artist_mbid          UUID,   -- MusicBrainz Artist ID (beets: mb_artistid)
    lidarr_album_id   INTEGER,
    batch_no          INTEGER,
    status            music.track_status NOT NULL DEFAULT 'ingested',
    flac_temp_path    VARCHAR(1000),
    aac_final_path    VARCHAR(1000),
    error_message     TEXT,
    retry_count       SMALLINT      NOT NULL DEFAULT 0,
    download_queued_at TIMESTAMPTZ,
    created_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE TABLE music.batches (
    id           SERIAL      PRIMARY KEY,
    batch_no     INTEGER     NOT NULL,
    phase        SMALLINT    NOT NULL CHECK (phase IN (1, 2, 3)),
    total_files  INTEGER     NOT NULL,
    processed    INTEGER     NOT NULL DEFAULT 0,
    failed       INTEGER     NOT NULL DEFAULT 0,
    status       VARCHAR(20) NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending', 'in_progress', 'done')),
    started_at   TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    UNIQUE (batch_no, phase)
);

CREATE OR REPLACE FUNCTION music.fn_update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_tracks_updated_at
BEFORE UPDATE ON music.tracks
FOR EACH ROW EXECUTE FUNCTION music.fn_update_updated_at();

CREATE INDEX idx_tracks_status           ON music.tracks(status);
CREATE INDEX idx_tracks_batch_no         ON music.tracks(batch_no);
CREATE INDEX idx_tracks_release_group_mbid ON music.tracks(release_group_mbid);
CREATE INDEX idx_tracks_recording_mbid     ON music.tracks(recording_mbid);
CREATE INDEX idx_tracks_lidarr_album_id    ON music.tracks(lidarr_album_id);
CREATE INDEX idx_batches_phase_status    ON music.batches(phase, status);
```

---

## Useful Queries

```sql
-- Progress overview
SELECT status, COUNT(*) FROM music.tracks GROUP BY status ORDER BY COUNT(*) DESC;

-- Permanent failures
SELECT filename, verified_artist, verified_album, error_message, retry_count
FROM music.tracks WHERE status = 'permanent_failed';

-- Still needs manual review
SELECT filename, raw_artist, raw_album, beets_confidence
FROM music.tracks WHERE status = 'needs_manual_review'
ORDER BY beets_confidence DESC NULLS LAST;

-- Phase 3 in-flight
SELECT filename, verified_artist, status, download_queued_at, retry_count
FROM music.tracks WHERE status IN ('queued', 'downloading')
ORDER BY download_queued_at;

-- Batch progress for Phase 3
SELECT batch_no, total_files, processed, failed, status FROM music.batches
WHERE phase = 3 ORDER BY batch_no;
```
