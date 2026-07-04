# aslisur

Automated pipeline to upgrade an MP3 library to AAC 256kbps using lossless FLAC sources.

**aslisur** (Hindi: _असली सुर_ — "the real note") fetches lossless FLAC from public torrent
trackers via Lidarr, extracts the exact track matched by MusicBrainz Recording ID, converts
to AAC 256k, and saves to a separate output folder. Originals untouched.

---

## How it works

```
~/Music/ (MP3s)
    │
    ├── Phase 1 — Ingest
    │   Scan all MP3s, read ID3 tags, assign UUID, write to PostgreSQL
    │
    ├── Phase 2 — Tag Correction
    │   Run beets acoustic fingerprinting (AcoustID + MusicBrainz)
    │   Populate MusicBrainz IDs needed for download
    │   Low-confidence files → manual review CLI
    │
    └── Phase 3 — Download & Convert
        Lidarr searches for FLAC album via Prowlarr + public trackers
        Extract matching track by MusicBrainz Recording ID
        FFmpeg: FLAC → AAC 256k .m4a
        Save to ~/Music-HQ/

~/Music-HQ/ (AAC 256k output, originals untouched)
```

---

## Stack

| Layer | Technology |
|---|---|
| Language | Python 3.11+ |
| Database | PostgreSQL 15 (Docker) |
| Fingerprinting | beets + Chromaprint (AcoustID) |
| Download | Lidarr + Prowlarr + qBittorrent |
| VPN | Gluetun (Surfshark WireGuard) |
| Conversion | FFmpeg (native AAC encoder, 256kbps) |

---

## Status

> Design phase complete. Build not started.

See [`docs/LLD.md`](docs/LLD.md) for the full low-level design — schema, API sequences,
phase logic, logging, Docker Compose, and all edge cases.
