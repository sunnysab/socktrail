# Agent Note: Batch socket refreshes without unbounded subscriber queues

Status: implemented

## Problem

A refresh can publish more than 1024 flow frames immediately after a large handshake. A healthy subscriber still reading the handshake exhausts the frame-count queue and is disconnected repeatedly. This fix existed as local commit `2fa3b7e` but is absent from upstream `3ab29f2`.

## Decision

The capture loop batches one refresh into one queue entry. Each subscriber also accounts for queued and in-flight bytes atomically, starting with a 16 MiB budget. A larger individual batch raises that subscriber's budget to twice that batch's size, so a large handshake and one comparable update fit. Slow subscribers are disconnected, never silently skipped or allowed to block capture. Short writes close the subscriber. Upstream's explicit frame-kind filtering remains intact.

## Alternatives considered

- Increasing the frame-count limit is smaller but only moves the disconnect threshold and allows much more memory retention.
- Blocking the capture loop preserves output but causes packet loss and stalls other subscribers.
- Dropping frames silently leaves the consumer with incomplete counters and connection state.

## Consequences

Healthy readers survive refresh bursts. Memory is bounded relative to the largest batch, not by a strict universal 16 MiB ceiling, and the codec plus batch buffers temporarily retain additional copies. A disconnected client must reconnect and consume a new handshake. `TestStreamRefreshBatchKeepsGreetedClient`, `TestStreamClientBoundsQueuedBytes`, and the existing stats-only tests cover the queue and filtering behavior.
