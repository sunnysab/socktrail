# Agent Note: Deduplicate unchanged socket refreshes before encoding

Status: implemented

## Problem

Four uncommitted files on `feat/systemd-collector` implement flow-refresh fingerprinting. Upstream `3ab29f2` still serializes every idle connection each refresh. The optimization is useful for a collector configured with `--refresh 1s`, but the original tests cover only top-level fields, duplicate the map-hashing implementation, and do not exercise stats-only filtering. Unbounded suppression also prevents a consumer from rediscovering an idle row after cache eviction.

## Decision

The sink keeps a content fingerprint and last-published time per flow ID, only for sockets publishing flow frames. It hashes each wire field directly before encoding, including nested process, container, GeoIP, evidence, DNS and TCP data. Maps use order-independent folding. The changes marker is excluded from content without mutating the caller's row. Only periodic refreshes are deduplicated; change events always publish. Stdout and change logs retain their previous semantics.

An unchanged row is republished once at least one minute has elapsed since its previous publication, when the normal refresh scheduler next visits it. This bounds collision-related delays and allows bounded consumer caches to rediscover idle rows. Initial subscriber handshakes remain full snapshots. Fingerprints are removed when rows leave the displayed view. Stats-only sockets and sockets without subscribers do no hashing.

The four original files form one logical change and stay together with the added edge-case tests. Their batching dependency is ported separately in [the backpressure fix](../bug-fix/2026-10-23-stream-backpressure.md). The original stash is retained as a recovery copy.

## Alternatives considered

- Encoding each row and comparing encoded bytes is much smaller and follows schema changes automatically, but still incurs the dominant serialization and allocation cost. MessagePack map iteration order also makes raw bytes unsuitable for direct equality without canonicalization.
- Keeping shallow row copies and using deep equality retains mutable shared maps and can compare a row with its already-mutated previous state. Deep copies would add allocation and lifecycle overhead.
- Suppressing unchanged fingerprints indefinitely saves slightly more traffic but can leave evicted consumer rows missing forever; a periodic full refresh is retained instead.

## Consequences

The manual fingerprint must evolve with the wire schema. Recursive reflection-based tests mutate every nested wire leaf and compare decoded MessagePack objects, rather than order-dependent encoded map bytes; reflection is never used in capture. A 64-bit fingerprint is not collision-free, so it is only an optimization with periodic resynchronization, never an identity or security check. Change events are not suppressed by a collision.

On this host, a populated-row microbenchmark measures approximately 5.6–5.9 microseconds and 103 allocations for encoding versus 0.72–0.78 microseconds and one allocation for an unchanged refresh. This excludes row construction and is not an end-to-end CPU claim. Tests cover changes markers, forgetting IDs, map-order independence, frame-kind filtering, idle sockets, bare output and periodic resynchronization. No prior active note covers deduplication; the existing stream-output design is updated in place and the related backpressure note is linked, not replaced.
