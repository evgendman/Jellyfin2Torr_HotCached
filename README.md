# Jellyfin2Torr_HotCached

[English](README.md) | [Русская версия](README.ru.md)

**A local, event-driven partial BitTorrent cache for Jellyfin and TorrServer.**

HotCached is a planned companion service for a media server. It keeps a bounded set of recently watched and newly added TorrServer torrents active in a local libtorrent session, prefetching only selected pieces from the beginning and end of the relevant media files.

The goal is to make useful pieces available from a nearby BitTorrent peer without replacing TorrServer's own streaming, buffering, or seeking logic.

> **Project status: design and specification stage.** The repository currently documents the agreed architecture, implementation roadmap, and acceptance tests. There is no released or tested implementation yet.

## Why this project exists

TorrServer is already responsible for obtaining the pieces needed for playback. HotCached is not intended to duplicate that logic. Instead, it prepares a small local pool of useful pieces before they are needed.

When Jellyfin reports that a new item has appeared, HotCached may place its torrent in the **PREWARM CACHE**. When playback actually starts, that torrent becomes part of the higher-priority **ACTIVE CACHE**. For each registered media file, HotCached selects the pieces covering a configurable byte range at the start and at the end of the file. All other pieces are explicitly disabled for download.

If TorrServer later requests one of the cached pieces, it can obtain it through the ordinary BitTorrent protocol from HotCached. TorrServer does not need to know that the peer has a special caching role.

## Data flow

```text
Jellyfin Webhook
  ├── ItemAdded ───────► PREWARM CACHE
  └── PlaybackStart ──► ACTIVE CACHE
                              │
                       hotcached manager
                              │
                     one libtorrent session
                              │
                   selected HEAD/TAIL pieces
                              │
                       local BT peer
                              │
                         TorrServer
                              │
                           playback
```

The actual media source is checked against a configured TorrServer playback URL. Cache decisions must not depend on library names, directory names, or whether a STRM file is stored locally or on a network share.

## Two cache pools

| Pool | Purpose | Maximum entries | Payload budget | Download budget |
|---|---|---:|---:|---:|
| **ACTIVE CACHE** | Torrents whose media has actually started playing | 20 | 5 GiB | 15 Mbit/s |
| **PREWARM CACHE** | Newly added media not yet confirmed as played | 20 | 5 GiB | 5 Mbit/s |
| **Global cap** | Maximum download rate for HotCached as a whole | — | — | 20 Mbit/s |

Each pool's bandwidth is divided evenly across its entries. When the pool membership changes, per-torrent limits are recalculated. Unused capacity is **not** transferred from one pool to the other in the MVP.

The pools are application-level policy. Libtorrent does not need to know their names: HotCached controls membership, eviction, and per-torrent limits.

## What is cached

Default selection for each registered media file:

- first **64 MiB**;
- last **16 MiB**.

HotCached converts both byte ranges into the torrent's piece numbers and combines them into one unique set. Required pieces use libtorrent priority `1`; every other piece uses priority `0`. HEAD and TAIL have equal priority.

Pieces are indivisible in BitTorrent. If a boundary falls in the middle of a piece, the entire piece must be downloaded—even if that makes the actual payload slightly larger than the configured byte ranges. Small files are handled naturally because overlapping ranges produce no duplicate pieces.

Only the useful pieces are requested; HotCached does not intentionally download complete movies or season packs. The `cached_bytes` statistic tracks completed, verified pieces. A separate free-disk check protects the actual filesystem.

The chosen tail size is a configurable initial value, not a promise that every container's seek index fits in that range. Real-world container behavior is part of the validation plan.

## Event behavior

### `ItemAdded` → PREWARM

If a new Jellyfin item can be reliably identified as a STRM source pointing to the configured TorrServer, HotCached registers its media file as a PREWARM candidate and starts warming its selected pieces.

An ordinary local MKV/MP4, a STRM pointing to another server, or an unresolvable source is ignored.

### `PlaybackStart` → ACTIVE

PlaybackStart is the primary signal of actual user interest:

- a new torrent enters ACTIVE;
- a PREWARM torrent is promoted to ACTIVE;
- an existing ACTIVE torrent has its last-played time refreshed;
- a repeated webhook does not create a duplicate libtorrent torrent.

The least recently used entries are evicted as needed to respect the target pool's limits. There is no time-to-live timer or periodic cleanup task in the MVP.

`PlaybackProgress` and `PlaybackStop` are intentionally not used. HotCached does not follow playback position and does not try to prefetch seek targets.

## Responsibilities

| Component | Responsibility |
|---|---|
| **Jellyfin** | Reports `ItemAdded` and `PlaybackStart`; supplies information about the media source selected for playback |
| **TorrServer** | Continues to handle all playback, buffering, and seeking |
| **libtorrent 2.1.x** | BitTorrent peer activity, metadata, piece priorities, transfers, verification, and per-torrent download limits |
| **HotCached** | Source validation, the ACTIVE/PREWARM pools, promotion, LRU eviction, cache accounting, and bandwidth allocation |
| **SQLite** | Persistent application state across restarts |
| **SSD cache directory** | Shared storage for all cached torrent data; queue membership lives in the database, not in separate directories |

## Documentation

- [Roadmap and implementation status](ROADMAP.md)
- [Technical specification](SPECIFICATION.md)
- [Test and acceptance plan](TEST_PLAN.md)

Russian documentation is available in [README.ru.md](README.ru.md), [ROADMAP.ru.md](ROADMAP.ru.md), [SPECIFICATION.ru.md](SPECIFICATION.ru.md), and [TEST_PLAN.ru.md](TEST_PLAN.ru.md).

## Important design boundaries

- No Tiramisu dependency.
- No STRM-directory watcher.
- No changes to TorrServer's internal playback logic.
- No matching torrents by title: use the actual selected STRM/TorrServer URL and its infohash/file index.
- One libtorrent session; HotCached manages its torrent handles explicitly rather than using libtorrent auto-management.
- Queue limits count cache entries by unique torrent infohash; several media-file targets inside a multi-file torrent must share one torrent handle and one set of downloaded pieces.
- The repository separates **agreed design** from **verified implementation**. Items labelled as validation gates in the specification are not claimed to work until they pass tests.

## Planned platform

The initial target is a headless Linux service on the same machine as TorrServer, with a TOML configuration file, SQLite state, a systemd unit, and a local webhook endpoint.

The deployment details, actual Jellyfin source-resolution behavior, torrent metadata discovery, and local peer connectivity must be verified during implementation before a first usable release is declared.
