# Roadmap / Implementation status

[English](README.md) | [Русская версия](README.ru.md)

**Project status: specification stage. No implementation or end-to-end test is claimed yet.**

## Project goal

Build a small local service that keeps a bounded set of TorrServer torrents warm in a libtorrent session. It downloads only pieces covering configurable ranges at the beginning and end of selected files, so TorrServer may obtain those pieces from a local BitTorrent peer. Playback remains entirely under TorrServer's control.

## Agreed MVP decisions

- Jellyfin events: `ItemAdded` → PREWARM; `PlaybackStart` → ACTIVE.
- Ignore `PlaybackProgress` and `PlaybackStop`.
- Identify the actual selected source by the configured TorrServer origin and playback URL, not by library or directory names.
- ACTIVE: 20 entries, 5 GiB, 15 Mbit/s.
- PREWARM: 20 entries, 5 GiB, 5 Mbit/s.
- Global HotCached download cap: 20 Mbit/s.
- Divide each pool's bandwidth evenly across its torrents; do not transfer unused pool bandwidth.
- Per media file: first 64 MiB and last 16 MiB; needed pieces priority 1, all others 0.
- One physical cache root; queue membership is stored in SQLite.
- No TTL or periodic eviction; evict reactively when an event needs capacity.
- No Tiramisu, no STRM directory watcher, and no changes to TorrServer's streaming logic.
- Use explicit torrent management (`auto_managed = false`).

## Implementation roadmap

### Phase 0 — Integration probes

- [ ] Confirm which fields the installed Jellyfin webhook supplies for `ItemAdded` and `PlaybackStart`.
- [ ] Determine how to resolve the **actually selected** media source, including STRM sources and multiple versions.
- [ ] Confirm the configured TorrServer URL parser can safely extract infohash and file index from the real playback URL.
- [ ] Verify whether a local TorrServer can discover and connect to the libtorrent peer; determine whether explicit peer connection is needed.
- [ ] Test metadata discovery for a hash-only torrent and identify limitations for private torrents.
- [ ] Verify libtorrent 2.1.x packaging/bindings on the target Ubuntu system.

### Phase 1 — BitTorrent core

- [ ] Create one libtorrent session and add a torrent by infohash.
- [ ] Receive metadata and validate the requested file index.
- [ ] Map HEAD/TAIL byte ranges to unique piece indices.
- [ ] Set all pieces to priority 0 and required pieces to priority 1.
- [ ] Disable auto-management and implement explicit start/remove behavior.
- [ ] Confirm that only required pieces are downloaded and verified.

### Phase 2 — Storage and accounting

- [ ] Implement one cache root and safe torrent storage paths.
- [ ] Track completed required pieces and calculate actual `cached_bytes`.
- [ ] Protect the filesystem with a minimum-free-space threshold.
- [ ] Save and restore libtorrent resume data and reconcile it with SQLite after restart.
- [ ] Test removal of data without touching TorrServer's own state.

### Phase 3 — State, queues, and bandwidth

- [ ] Add SQLite schema and migrations.
- [ ] Implement ACTIVE/PREWARM membership and idempotent event processing.
- [ ] Implement PREWARM → ACTIVE promotion without duplicate torrent handles or re-download.
- [ ] Enforce both item-count and cached-payload limits.
- [ ] Implement LRU eviction and recalculate per-torrent limits after every membership change.
- [ ] Enforce per-pool budgets and a global 20 Mbit/s cap.

### Phase 4 — Jellyfin integration

- [ ] Add a small authenticated local webhook endpoint.
- [ ] Implement event validation, logging, and asynchronous processing.
- [ ] Resolve the actual selected media source for PlaybackStart.
- [ ] Resolve ItemAdded sources without assuming every STRM belongs to this TorrServer.
- [ ] Ignore ordinary local media, other servers, malformed URLs, and ambiguous sources safely.
- [ ] Add deduplication and retry-safe processing.

### Phase 5 — Operations and diagnostics

- [ ] Add TOML configuration with documented defaults.
- [ ] Add `/health`, `/status`, and `/cache` local diagnostic endpoints.
- [ ] Add a systemd unit and least-privilege service account.
- [ ] Document install, upgrade, backup, logs, and recovery.
- [ ] Add unit and integration tests to CI where feasible.

### Phase 6 — End-to-end acceptance

- [ ] Verify Jellyfin → HotCached event flow.
- [ ] Verify HotCached and TorrServer are in the same swarm and TorrServer can obtain a cached piece from HotCached.
- [ ] Verify ordinary and foreign media sources are ignored.
- [ ] Verify HEAD/TAIL-only downloading, small-file overlap, and piece-boundary behavior.
- [ ] Verify queue promotion, LRU eviction, and bandwidth limits under concurrent downloads.
- [ ] Verify restart recovery and cache accounting.
- [ ] Measure whether the cache improves startup/seek behavior on representative files.

## Validation gates (not yet verified)

1. **Jellyfin source resolution:** webhook data may not itself contain a directly usable TorrServer URL. The implementation must reliably resolve the selected MediaSource without assuming `Item.Path` is sufficient.
2. **Metadata discovery:** infohash-only add operations require peers that can provide metadata. Private torrents or tracker-specific restrictions may limit discovery.
3. **Local peer path:** TorrServer must actually connect to HotCached and request its cached pieces. Being on the same host does not by itself guarantee peer discovery.
4. **Cache accounting:** sparse-file apparent size is not the cache budget. Account for completed, verified pieces and separately protect real disk space.
5. **Tail usefulness:** 16 MiB is the initial agreed default; container indexes and seek behavior must be tested, not assumed.
6. **Bandwidth semantics:** confirm libtorrent's per-torrent and global limit behavior, including units and local-peer exceptions, in the selected build.

## Definition of MVP complete

MVP is complete only when the acceptance tests in [TEST_PLAN.md](TEST_PLAN.md) pass on the target host and a real TorrServer playback can demonstrably obtain at least one pre-cached piece from HotCached. Unit tests alone are not sufficient.
