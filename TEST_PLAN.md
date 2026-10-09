# HotCached test and acceptance plan

[English](TEST_PLAN.md) | [Русская версия](TEST_PLAN.ru.md)

**Status:** planned tests. Nothing in this document implies that a test has already passed.

## Test strategy

Use three layers:
1. **Unit tests** — URL parsing, source classification, piece selection, byte accounting, queue transitions, eviction, config validation.
2. **Component tests** — SQLite persistence, libtorrent metadata/piece behavior, webhook/API resolver.
3. **End-to-end tests** — Jellyfin webhook → HotCached → libtorrent → local TorrServer peer.

Record environment details with every integration run: Ubuntu/kernel, Jellyfin and webhook plugin versions, TorrServer version/settings, libtorrent version, test torrent privacy/public status, piece length, and network topology.

## A. Source filtering

- [ ] Ordinary local MKV/MP4 playback creates no cache entry.
- [ ] STRM pointing to a different host/port is ignored.
- [ ] Same host but wrong service/path is ignored.
- [ ] Valid STRM pointing to configured TorrServer is accepted.
- [ ] STRM stored on SMB/network storage is handled if Jellyfin can resolve its selected source; storage location does not determine ownership.
- [ ] Multiple versions/media sources: only the actually selected source controls the decision.
- [ ] Missing or ambiguous source is ignored and logged.
- [ ] Invalid hash, malformed URL, invalid/negative file index and unexpected path are rejected.
- [ ] Query parameters and URL encoding do not corrupt valid parsing.

## B. Webhook behavior

- [ ] Valid ItemAdded for our source admits PREWARM.
- [ ] Valid PlaybackStart for an unknown source admits ACTIVE.
- [ ] PlaybackStart for PREWARM promotes the existing record.
- [ ] PlaybackStart for ACTIVE updates last_played_at.
- [ ] Duplicate/retried webhooks do not create duplicate records or handles.
- [ ] ItemAdded repeated for an existing item does not refresh its age.
- [ ] PlaybackProgress and PlaybackStop do not alter cache state.
- [ ] Invalid authentication and malformed JSON are rejected.
- [ ] Webhook acknowledgement is fast even while metadata discovery is pending.

## C. Piece selection

Test with controlled torrent metadata and files of varied sizes, piece lengths, and file offsets.

- [ ] Large single-file torrent selects only pieces intersecting first 64 MiB and last 16 MiB.
- [ ] Every unneeded piece has priority 0; every needed piece has priority 1.
- [ ] A piece intersecting a range boundary is included whole.
- [ ] A file smaller than HEAD + TAIL produces a deduplicated set.
- [ ] A file smaller than HEAD is covered by the HEAD range without invalid offsets.
- [ ] A file exactly at a range boundary is handled correctly.
- [ ] One piece larger than HEAD or TAIL is still included whole.
- [ ] Multi-file torrent maps file ranges to global piece indices correctly.
- [ ] Pieces shared by two media targets are counted only once.
- [ ] Adding a second target adds required pieces without removing pieces needed by the first target.
- [ ] Priorities are re-applied after session restore.

## D. Cache accounting and disk protection

- [ ] cached_bytes increases only when a required piece is complete and verified.
- [ ] Incomplete pieces are not counted.
- [ ] Shared pieces across targets/torrent entries are not double-counted.
- [ ] Restart reconciliation produces the same verified-payload count.
- [ ] Sparse file apparent size is not used as cached_bytes.
- [ ] Admission is refused or eviction performed before violating the minimum disk reserve.
- [ ] A single target that cannot fit the configured queue byte budget is rejected safely.
- [ ] Eviction removes only files inside HotCached's configured root.

## E. Queue policy and LRU

- [ ] New eligible ItemAdded enters PREWARM.
- [ ] New eligible PlaybackStart enters ACTIVE directly.
- [ ] PREWARM → ACTIVE promotion does not create another handle or discard cached pieces.
- [ ] A repeated PlaybackStart refreshes ACTIVE recency.
- [ ] PREWARM duplicate ItemAdded does not refresh added_at.
- [ ] ACTIVE eviction selects least-recently-played entry.
- [ ] PREWARM eviction selects oldest added entry.
- [ ] Count and byte limits are independently enforced.
- [ ] If necessary, queue contains fewer than max_items rather than violating byte budget.
- [ ] Emptying a queue does not transfer its bandwidth budget to the other queue.
- [ ] Membership changes trigger recalculation for every torrent in both affected queues.
- [ ] A torrent appearing under multiple file indices still counts as one queue entry.

## F. Bandwidth

- [ ] One ACTIVE torrent receives a 15 Mbit/s configured cap; two receive 7.5 Mbit/s each; N entries divide the 15 Mbit/s budget.
- [ ] PREWARM equivalents divide 5 Mbit/s.
- [ ] Session cap is configured at 20 Mbit/s.
- [ ] Config parser converts Mbit/s to libtorrent's expected units correctly.
- [ ] Observed rates are measured over a long enough interval and interpreted as approximate, not instantaneous guarantees.
- [ ] Local-peer exceptions and upload limits are tested separately.
- [ ] Under simultaneous ACTIVE/PREWARM traffic, configured pool limits are respected within expected protocol/measurement tolerance.

## G. Metadata, peers, and TorrServer interoperability

- [ ] Public test torrent metadata can be obtained from infohash alone under supported conditions.
- [ ] Metadata timeout/no-peer behavior is bounded and logged.
- [ ] Document results for private torrents; do not claim universal support without evidence.
- [ ] HotCached and TorrServer join the same swarm for the same infohash.
- [ ] TorrServer can obtain a piece that HotCached has completed and verified.
- [ ] A piece not yet cached is not reported as available.
- [ ] Upload/peer limits do not prevent TorrServer from retrieving cached pieces.
- [ ] Compare TorrServer peer lists/logs and libtorrent peer state to establish the transfer path.
- [ ] Measure playback start and end-of-file/seek behavior before and after cache warm-up on representative media.

This section is a release gate: **without a demonstrated piece transfer to TorrServer, the central project assumption is not validated.**

## H. Restart and resilience

- [ ] Restart preserves queue membership and last_played_at.
- [ ] Resume data restores valid pieces and priorities.
- [ ] Missing/corrupt resume data does not crash the service.
- [ ] SQLite and filesystem state disagreement is reconciled safely.
- [ ] Configuration with reduced limits triggers orderly eviction at startup.
- [ ] Jellyfin/TorrServer temporarily unavailable does not crash the process.
- [ ] A libtorrent error is logged and does not corrupt unrelated queue entries.

## I. Security and operations

- [ ] API tokens are never logged or returned by diagnostic endpoints.
- [ ] Webhook authentication is enforced.
- [ ] Default bind address is loopback.
- [ ] Service runs as a dedicated least-privilege user.
- [ ] The process cannot delete or modify TorrServer data.
- [ ] `/health`, `/status`, and `/cache` return useful, bounded diagnostics.
- [ ] systemd restart and shutdown behavior is tested.

## Release checklist

- [ ] Unit tests pass.
- [ ] Integration tests pass on the target host.
- [ ] Real TorrServer receives a verified cached piece from HotCached.
- [ ] ACTIVE/PREWARM and global bandwidth caps are validated.
- [ ] Disk-budget behavior and restart recovery are validated.
- [ ] README and roadmap reflect only confirmed results.
