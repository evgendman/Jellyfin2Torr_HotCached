# HotCached technical specification

[English](SPECIFICATION.md) | [Русская версия](SPECIFICATION.ru.md)

**Status:** agreed design for the planned MVP. Any item marked *validation required* must be tested before it is treated as implemented behavior.

## 1. Purpose and non-goals

HotCached is a local Linux service running beside TorrServer. It maintains a small, event-driven pool of torrents in a libtorrent session and downloads only pieces covering configured HEAD and TAIL byte ranges of selected media files. TorrServer remains responsible for streaming, buffering, and seeking; HotCached helps only by acting as another BitTorrent peer with useful pieces already available.

Non-goals: replacing TorrServer, tracking playback position, processing PlaybackProgress/PlaybackStop, prefetching arbitrary seek offsets, downloading complete media intentionally, watching STRM directories, depending on Tiramisu, or modifying TorrServer internals.

## 2. Components and responsibilities

- **Jellyfin:** emits ItemAdded and PlaybackStart webhooks; its API/source resolution is used to identify the actual selected media source.
- **TorrServer:** playback endpoint and independent streaming engine. HotCached must not add/remove torrents through TorrServer's API.
- **libtorrent 2.1.x:** single BitTorrent session; metadata, peer discovery, piece priorities, piece verification, per-torrent limits, resume data.
- **HotCached:** webhook receiver, source resolver, cache policy, queue manager, eviction, accounting, configuration, diagnostics.
- **SQLite:** durable application state and queue membership.
- **One SSD cache root:** shared physical storage. Queue names are database state, not directory names.

## 3. Event flow

### ItemAdded

1. Validate webhook and resolve the Jellyfin item/source.
2. Establish that it is a STRM source pointing to the configured TorrServer.
3. Parse infohash and file index from a supported playback URL.
4. If the torrent is not already tracked, admit it to PREWARM after capacity checks.
5. Obtain torrent metadata, calculate required pieces, start managed downloads.

If source identity cannot be established reliably, ignore the event and log the reason. Never infer torrent identity from a movie title or library folder.

### PlaybackStart

1. Resolve the MediaSource actually selected for playback, using MediaSourceId where available.
2. Verify that the source points to the configured TorrServer.
3. Extract infohash and file index.
4. If already ACTIVE, update last_played_at only.
5. If in PREWARM, promote it to ACTIVE without creating a duplicate torrent handle.
6. Otherwise admit it directly to ACTIVE.
7. Evict older entries if needed, then recalculate per-torrent bandwidth limits.

PlaybackProgress and PlaybackStop are ignored in the MVP.

**Validation required:** the exact Jellyfin API call(s) needed to resolve the selected MediaSource for both webhook types must be confirmed against the installed Jellyfin version. If the source is ambiguous or unavailable, fail closed and ignore it.

## 4. Source identity and URL parsing

Configuration defines one canonical TorrServer base URL, for example `http://192.168.0.14:8097`. Normalize scheme, hostname, and port and compare the source URL's origin. Validate the expected TorrServer playback path; do not accept an arbitrary URL merely because its host matches.

For a confirmed URL shaped like `/play/{hash}/{id}`, parse:
- `infohash`: normalized lowercase hexadecimal infohash, validating expected length/encoding;
- `file_index`: non-negative integer identifying the torrent file.

Ignore unrelated paths, other hosts/ports, malformed hashes, invalid indices, and non-STRM local media. Query strings must not corrupt parsing.

For PlaybackStart, prefer the selected MediaSource rather than the library item's default path. ItemAdded has no selected playback source, so use a verified item/source lookup or read the STRM content if the path is accessible. If neither method can confirm the URL, do not prewarm.

## 5. Queue policy

Two application-level queues:

| Setting | ACTIVE | PREWARM |
|---|---:|---:|
| max_items | 20 | 20 |
| max_cache_bytes | 5 GiB | 5 GiB |
| download_bandwidth | 15 Mbit/s | 5 Mbit/s |

Global HotCached download cap: 20 Mbit/s.

Bandwidth is divided evenly among torrents in the same queue and recalculated whenever membership changes. No unused bandwidth is borrowed across queues in the MVP. Configure a session-wide cap as a safety limit in addition to per-torrent limits. Confirm exact libtorrent units and behavior in the selected build.

All managed torrents use explicit application management (`auto_managed = false`). libtorrent's active-download auto-management limits must not choose queue membership.

### Promotion and idempotency

- ItemAdded adds a new eligible torrent to PREWARM only if it is not tracked.
- ItemAdded for an existing ACTIVE/PREWARM torrent does not duplicate it or refresh its age.
- PlaybackStart for PREWARM promotes the existing record/handle to ACTIVE.
- PlaybackStart for ACTIVE updates last_played_at.
- PlaybackStart for an unknown torrent adds it to ACTIVE.
- A torrent's most recent actual playback is more important than its library-added time.

## 6. Cache capacity and eviction

Each queue independently enforces both item count and cached payload bytes. Before admission, determine the required pieces and estimate their full piece payload (including boundary pieces). If the entry cannot fit, evict least-recently-used entries in the target queue until it can fit, or until the queue is empty. If a single entry exceeds the byte budget, do not admit it; log a clear reason. Never download a partial piece merely to fit a budget.

ACTIVE LRU order uses last_played_at, with added_at as deterministic fallback. PREWARM order uses added_at. Eviction is reactive; no TTL and no periodic cleanup are required in MVP.

Eviction must remove the libtorrent handle and its HotCached-owned storage/resume files and database records, but must never alter TorrServer's own torrent list, cache, or data.

## 7. Torrent identity and multiple files

One libtorrent torrent handle and one physical torrent payload per unique infohash. Queue capacity is counted by unique infohash, not by STRM item.

A tracked torrent may have multiple media targets (different file_index values). For every target, calculate the pieces covering its HEAD and TAIL ranges; the torrent's desired piece set is the union of all target sets. Shared pieces are counted once. PlaybackStart for another file in the same torrent refreshes recency and adds that file's required pieces without creating another handle.

Suggested relational model:
- `torrents`: one row per infohash, queue, added_at, last_played_at, cached_bytes, metadata/resume state and diagnostics;
- `media_targets`: one row per (infohash, file_index), file size, piece length and target metadata.

The agreed minimum fields (infohash, file_index, queue, added_at, last_played_at, cached_bytes) must remain represented. If a normalized schema is used, file_index belongs to media_targets while queue/recency/cached_bytes belong to torrents.

## 8. Piece selection

Defaults:
- `head_bytes = 64 MiB`
- `tail_bytes = 16 MiB`
- `piece_priority = 1`

For file size S:
- HEAD byte range: [0, min(head_bytes, S))
- TAIL byte range: [max(0, S - tail_bytes), S)

Map ranges through libtorrent's file/piece mapping APIs. Build a set of unique piece indices covering both ranges. After metadata arrives, set all torrent pieces to priority 0 and each required piece to priority 1. Reapply priorities after restore or after adding another target.

BitTorrent pieces are indivisible: download a whole piece if any part intersects a required range. Small-file overlaps naturally deduplicate. Do not introduce resolution-specific, bitrate-specific, or file-size-scaled cache sizes in MVP.

A multi-file torrent has pieces spanning file boundaries; a piece required by one target may contain bytes belonging to another file. Account for the actual piece payload, not merely the target file's byte range.

## 9. Cache accounting and storage

`cached_bytes` is the sum of payload bytes for unique required pieces that are completed and verified, counted once per torrent. Do not use sparse file apparent length, full media file size, or HEAD+TAIL nominal range sum as a substitute.

Maintain a piece-completion view from libtorrent and reconcile it on startup. Do not count incomplete pieces as cached. Distinguish:
- logical payload budget per queue;
- actual filesystem free space.

Suggested safety default: `minimum_free_disk_bytes = 10 GiB`. If the disk reserve would be violated, reject PREWARM admission. ACTIVE admission may evict existing ACTIVE entries first, but must never violate the reserve. Exact admission behavior must be tested.

Use one configured cache root, e.g. `/mnt/hotcached`. Avoid queue-specific directories. Safe per-infohash storage subdirectories are acceptable. Promotion changes database state only; it must not move files.

## 10. Metadata and peer discovery

A URL supplies infohash and file index, not necessarily torrent metadata. Add by infohash, then acquire metadata from available peers. Enable suitable peer discovery (DHT/LSD/PEX as appropriate to the torrent and library settings).

**Validation required:** metadata discovery may not work for every private torrent. Document supported cases and fail gracefully when metadata cannot be obtained.

**Critical integration test:** prove that TorrServer can connect to HotCached for the same infohash and request cached pieces. Test LAN discovery and local-peer behavior. If necessary, implement explicit peer connection only after confirming how TorrServer exposes its BitTorrent listen endpoint. Do not assume same-host placement guarantees a connection.

HotCached must not intentionally become an unrestricted public seed. Test upload behavior and local-peer access. Any upload limits must preserve TorrServer's ability to obtain cached pieces.

## 11. Persistence and restart recovery

Use SQLite transactions for queue and target changes. Store libtorrent resume data and enough metadata to reconstruct the session after a restart.

Startup reconciliation:
1. load configuration and database;
2. inspect cache/resume files;
3. restore valid torrent handles;
4. reconcile verified pieces and cached_bytes;
5. reapply piece priorities and bandwidth limits;
6. evict oldest records if updated configuration makes limits smaller.

Missing/corrupt resume data must not crash the service. Do not delete files outside HotCached's cache root.

## 12. Configuration

Proposed file: `/etc/hotcached/config.toml`.

```toml
[jellyfin]
url = "http://127.0.0.1:8096"
api_token = "CHANGE_ME"

[webhook]
bind = "127.0.0.1"
port = 9120
path = "/webhook"
auth_token = "CHANGE_ME"

[torrserver]
url = "http://192.168.0.14:8097"

[cache]
path = "/mnt/hotcached"
head_bytes = "64MiB"
tail_bytes = "16MiB"
piece_priority = 1
minimum_free_disk_bytes = "10GiB"

[global]
download_bandwidth = "20Mbit/s"

[active]
max_items = 20
max_cache_bytes = "5GiB"
download_bandwidth = "15Mbit/s"

[prewarm]
max_items = 20
max_cache_bytes = "5GiB"
download_bandwidth = "5Mbit/s"
```

Tokens are secrets: keep the file readable only by root and the service account. Never log tokens. The parser must validate values, normalize units, reject negative limits, and ensure ACTIVE + PREWARM bandwidth does not exceed the configured global cap.

## 13. Local diagnostics and service

Suggested local endpoints:
- `GET /health`: process/DB/session health;
- `GET /status`: queue counts, bytes and limits;
- `GET /cache`: per-torrent and per-target details.

Useful per-torrent fields: infohash, queue, target file indices, cached_bytes, required/completed piece counts, last_played_at, current download rate, configured limit, peers, last error.

Bind diagnostics to loopback by default or protect them with authentication. The webhook must validate an authentication token and enqueue work quickly; metadata discovery must not block webhook acknowledgement.

Initial deployment target:
- executable under `/opt/hotcached/`;
- config under `/etc/hotcached/`;
- state under `/var/lib/hotcached/`;
- cache under `/mnt/hotcached/`;
- systemd unit `hotcached.service`;
- dedicated least-privilege `hotcached` service account.

## 14. Error handling and observability

Handle invalid webhook payloads, Jellyfin API failures, missing MediaSource, foreign TorrServer, invalid hash/index, metadata timeout, no peers, disk reserve violation, SQLite failures, libtorrent errors, and corrupt resume data.

Log event type, Jellyfin ItemId where available, infohash, file index, queue, action, and error. Never log API/auth tokens. Resolver failures must not affect playback.

## 15. Acceptance requirements

See [TEST_PLAN.md](TEST_PLAN.md). The essential end-to-end condition is that a real TorrServer playback can obtain a previously downloaded piece from HotCached. All queue, piece-selection, capacity, bandwidth, restart, and source-filtering tests must also pass.
