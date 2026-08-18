# Agent Note: History pagination bounds browser replay without splitting messages

Status: implemented

English | [中文](2026-08-04-large-history-pagination-call-stack.zh.md)

## Problem

A finalized assistant message can reference hundreds of thousands of streamed chunks through `sourceEventSeqs`. History pagination found the message group's first event with `Math.min(event.seq, ...sourceEventSeqs)`, so a valid session could exceed the JavaScript engine's function-argument limit and make `session.history` fail with HTTP 500. Even without that exception, counting only append-origin messages let one tail page carry hundreds of raw chunk and tool events, making browser replay expensive enough to prevent a conversation from opening.

## Decision

Pagination scans `sourceEventSeqs` and updates the earliest sequence number one element at a time. Gateway configuration gives a history page default interactive replay limits of 128 raw events, 128 KiB of serialized raw-event data, and 128 KiB of serialized `HistoryEntry` data after host-computed tool views join the response. A single derived view that exceeds the final-entry limit is omitted while its durable event remains. When the requested message page exceeds either remaining limit, pagination advances to the next complete message group until the suffix fits. A single newest complete message group may exceed a raw-event limit, because splitting its provenance would make replay incomplete.

The response continues to use a contiguous raw event range and reports `hasMore` when the replay bound advances the start. `maxMessages` remains a maximum rather than a guarantee, so a bounded page can return fewer messages than requested.

Regression coverage rejects multi-argument minimum calls, keeps every provenance event with its finalized message, proves a chunk-heavy tail advances at a whole-message boundary before it reaches the browser, and covers a tool presenter whose view is larger than its raw result.

## Alternatives considered

- **Raise the JavaScript stack or argument limit** — rejected: the limit is engine- and deployment-dependent, and array expansion still makes valid history depend on an unrelated runtime ceiling.
- **Truncate `sourceEventSeqs` during pagination** — rejected: this could cut a page inside a message and violate replay grouping.
- **Count messages only** — rejected: streamed chunks and tool events do not consume message quota, so that rule does not bound browser work.
- **Cap streamed chunk count at the provider boundary** — rejected: providers may legitimately emit long streams, and pagination must handle every valid session representation.

## Consequences

- Large provenance arrays no longer make history pagination throw solely because of their length.
- A normal page is bounded by raw event count, serialized event bytes, and serialized rendered entries, preventing chunk streams or presenter views from becoming an unbounded initial browser replay.
- A caller may receive fewer messages than its requested maximum, and a lone oversized completed message still has to be replayed as one group.
