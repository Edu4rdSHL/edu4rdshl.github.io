---
layout: post
title: 'Rethinking the GNOME clipboard issues'
date: 2026-05-26 23:35 -0500
categories: ["gnome", "rust", "clipboard"]
tags: ["gnome", "rust", "clipboard", "wayland", "gjs", "performance"]
author: edu4rdshl
image:
  path: /2026-05-26/strata.jpg
  alt: 'The Strata clipboard manager logo'
excerpt: 'GNOME clipboard managers stutter as history grows. Strata fixed it with a Rust daemon, a thin GJS extension, lazy loading, and pre-decoded thumbnails.'
---

## Introduction

A clipboard manager is one of those tools you don't think about until you don't have it. It's a basic feature and QoL enhancement on everyone's routine, and yet, some desktop environment(s) doesn't offer one by default (we all know about what I'm talking about but I love that DE for everything else). You copy something, copy something else, and then realize you needed the first thing. Having the history there is great. What is not great is when the history starts to cost you frames.

I have used several clipboard managers on GNOME over the years, and they all eventually did the same thing: the desktop would hitch. A small stutter when I copied a screenshot. A longer one when I opened the history and it had grown to a few hundred entries. Search that lagged behind my typing. None of it was a dealbreaker on day one, but it got worse the more I used the tool, which is exactly backwards from what you want. Some worakrounds included disabling image history, limiting a lot the amount of text that they can keep in history, limiting the number of entries and more.

I have been thinking about this problem since a long time. It was not an easy work as Gnome doesn't expose any of the `wlr-data-control` protocols, so you are forced to implement this on a extension, which carry all of the previously mentioned problems. This weekend, I decided to write Strata, a clipboard manager that solves these problems using several mechanisms that are not present on any of the other implementations. The code is at [github.com/Edu4rdSHL/Strata](https://github.com/Edu4rdSHL/Strata), and the whole point of this post is the one thing I had to get right: the stutter is not a tuning problem you fix with a faster loop or a smaller cache. It is architectural. If you want it gone, you have to move the work somewhere the compositor can't feel it.

## Where the stutter comes from

GNOME Shell extensions run in GJS, which is single-threaded, and that single thread is shared with the entire compositor. The same loop that draws your cursor, animates your workspaces, and composites every window is the loop your extension runs on. There is no background thread to hide in. Whatever your extension does synchronously, the desktop _does not_ do anything else while it happens.

That is fine for forwarding a click. It is not fine for the work a clipboard manager actually does. Hashing a payload to deduplicate it, running a SQLite query, decoding a pasted PNG so you can show a preview, building a thousand list rows when the history panel opens: every one of those is real CPU or I/O, and every millisecond of it is a millisecond the compositor is frozen. You see it as a hitch.

It gets worse as the history grows, which is the part that always bothered me. Loading the whole history into JavaScript memory, holding decoded images around, re-rendering a long list on every open, searching by walking the list: that is O(n) work, or worse, on the one thread that also has to draw the screen. A clipboard manager that is smooth with twenty items and janky with two thousand is not really smooth. It is just empty.

This is the pattern most GNOME clipboard tools share. The work lives in the extension, on the compositor thread, and the only knobs you get are how much work and how often. Strata starts from a different premise: get the work off that thread entirely, and the question of how much of it stops mattering.

## Two components, one boundary

Strata is two processes that talk over the session D-Bus, and you need both.

The first is `strata-daemon`, written in Rust on top of tokio and zbus. It owns everything heavy: a SQLite database (with WAL and an FTS5 full-text index), content deduplication, thumbnail generation, and search. It exposes all of that as a D-Bus service named `dev.edu4rdshl.Strata`.

The second is the GNOME Shell extension, `strata@edu4rdshl.dev`, written in GJS. It draws the top-bar panel, runs the search box, handles paste-back, and watches the clipboard for new content. That is all it does. It renders UI and forwards events. It never hashes, never decodes an image, never touches SQLite.

The topology is small:

```text
GNOME Shell (GJS)  --D-Bus-->  strata-daemon  -->  SQLite (~/.local/share/strata)
                                     |
                                     +-->  thumbnails (~/.cache/strata)
```

The cost of this split is one IPC hop per operation. In practice that is cheap, because the session bus is shared-memory fast on the same machine, and as you'll see the extension almost never waits on a reply in any path that matters. What you buy for that hop is the entire premise of the project: the expensive work runs in a separate process, on a thread pool, and the compositor cannot feel it no matter how long it takes.

The extension also supervises the daemon. On enable it spawns the binary and watches it, respawning with exponential backoff if it dies, and giving up only after a rapid crash loop. If a daemon is already running (for example, one you started as a systemd user service), the extension detects it and reuses it instead of spawning a second copy.

## Never block the ingest path

The hottest path in a clipboard manager is the moment you copy something. Get this wrong and every single copy costs you a hitch. The rule in Strata is simple: the extension fires and forgets.

When the selection owner changes, the extension checks the MIME type against a strict allowlist, reads the bytes, and calls `SubmitItemAsync(mime, rawBytes)`. Then it returns. It does not `await` the result. The copy is recorded as far as the extension is concerned the moment the call is dispatched, and the compositor goes back to its frame.

A couple of details make that call cheap. The bytes go over D-Bus as a raw byte array (`ay`), so there is no encoding step in JavaScript and no decode step in Rust. And a 50 ms debounce sits in front of it, because a surprising number of applications write the clipboard several times for a single copy, and there is no reason to record each of those.

On the other side of the bus, the daemon does the actual work, but it does it where it belongs. Every database operation runs inside `spawn_blocking`, off the async reactor:

```rust
let conn = self.conn.clone();
tokio::task::spawn_blocking(move || {
    let guard = conn.lock();          // poison-recovering wrapper
    db::upsert_item(&guard, ...)      // hash, dedup, thumbnail, prune
}).await?
```

Inside that closure the daemon hashes the payload with blake3 for deduplication, performs an atomic upsert keyed on the content hash (a unique index means copying the same thing twice just updates the timestamp instead of creating a duplicate row), decodes and thumbnails the content if it's an image, and prunes the history back to its limit. None of that runs on the D-Bus reactor, and obviously none of it runs on the compositor. The reactor stays responsive while disk and CPU work happen on the blocking pool, and the desktop never knew a thing happened.

## Lazy loading, so history size stops mattering

This is the part I cared about most, because "smooth until the history grows" was the exact failure I was trying to kill. Strata has two independent lazy layers, and between them the size of your history stops being something the UI has to pay for.

The first is paginated metadata. The panel asks for history with `GetHistory(offset, limit)`, and what comes back is metadata only: the id, the MIME type, a short text preview truncated to about 200 characters in SQL, a timestamp, and a flag saying whether the item has a thumbnail. It does not return the full content. A page of large text items costs a few kilobytes of JSON instead of megabytes. On the Rust side these come straight off a `created_at DESC` index with `LIMIT` and `OFFSET`, so a page stays an O(log n) lookup no matter how big the table is. The panel loads one page when you open it and another only when you scroll within 200 px of the bottom. The full table never lives in JavaScript.

The schema is built around that access pattern:

```sql
CREATE TABLE clipboard_history (
    id              TEXT PRIMARY KEY,
    mime_type       TEXT NOT NULL,
    content_text    TEXT,             -- text payloads
    content_blob    BLOB,             -- binary payloads
    thumbnail_blob  BLOB,             -- pre-decoded PNG, ~200 px
    content_hash    TEXT NOT NULL,    -- blake3 of the raw bytes
    created_at      INTEGER NOT NULL
);
CREATE INDEX        idx_created_at ON clipboard_history (created_at DESC);
CREATE UNIQUE INDEX idx_hash       ON clipboard_history (content_hash);
```

Search is the second path, and it does not walk the list. There is an FTS5 full-text index over the text content, so `SearchHistory(query)` is an index lookup, not a scan. It returns the matching set, and the panel pages through that snapshot exactly the same way it pages through the recent view. The tokenizer ignores diacritics and matches on prefixes, so searching `cafe` finds `café` and typing half a word finds the whole thing. Because the FTS5 table uses external content, the text itself is stored once in the base table and the index holds only the inverted data.

## Previews and thumbnails for fast rendering

Images are where a clipboard list usually falls apart, because decoding a full-resolution screenshot to draw a small preview is exactly the kind of work that has no business running on the compositor thread. Strata never does it there.

The daemon decodes and resizes each image to a roughly 200 px PNG once, at ingest, on the blocking pool, and stores that thumbnail in the database. The UI never decodes a full-resolution image, ever. By the time the panel needs to draw an image row, the expensive part already happened, in another process, at copy time.

And even the thumbnail is not handed over until it's needed. `GetHistory` does not return image bytes. Each image row renders immediately with a placeholder icon, and then the UI does one of two things: if `~/.cache/strata/thumbnails/<id>.png` already exists it loads straight from disk, and if it doesn't it calls `GetThumbnail(id)` once, writes the PNG to that cache file, and loads it from there. A given thumbnail is fetched at most once per session; after that, reopening the panel reads it from the page cache. The practical effect is that scrolling past a thousand image rows costs zero D-Bus traffic for everything outside the viewport. You only pay for what's on screen.

The rendering itself is paced too. Rows are added to the list in chunks of 20 through `GLib.idle_add`, so even a full page is built across several idle ticks instead of in one blocking burst, and a frame always has room to land in between. The search box has a 150 ms debounce, and there's an epoch counter so that if you keep typing, results from an older query that arrive late are simply dropped instead of painting and then getting replaced.

## Paste-back without blocking either

Putting an item back on the clipboard is the one moment Strata needs the full content, and it fetches it lazily, only then. The panel calls `GetItemContent(id)`, which returns the MIME type and the raw bytes as `(s, ay)`. Raw bytes again, so there is no base64 to decode on the compositor thread.

From there it splits by type. Text goes through `St.Clipboard.set_text`. Binary content, including images, is wrapped in a memory-backed selection source and made the clipboard owner:

```js
const source = Meta.SelectionSourceMemory.new(mimeType, GLib.Bytes.new(bytes));
global.display.get_selection().set_owner(
    Meta.SelectionType.SELECTION_CLIPBOARD,
    source);
```

There's a security property that falls out of this design for free. No code path in Strata ever executes clipboard content. Nothing spawns a process, nothing opens a URI, and clipboard text is never fed through Pango markup. Items are rendered with plain labels and pasted back as opaque bytes. A hostile payload sitting in your history has nothing to grab, because the code that handles it does not interpret it.

## Installing it

If you want to try it, the [Install section in the README](https://github.com/Edu4rdSHL/Strata#install) has the up-to-date instructions and I'll keep them current there rather than here. The short version is that there are two pieces, the daemon and the extension, and you need both. On Arch there are AUR packages for each (stable and `-git` channels); on anything else you build from source with `make install-daemon` and `make install`, or run the daemon as a systemd user service. After that, enable the extension and log back in. The README spells out each path.

## Where this leaves us

The result is a clipboard manager that feels the same with fifty items as it does with several thousand, which is the whole thing I wanted. That isn't because the work is faster than what other does (which can be), but mainly because the work that is genuinely slow (hashing, image decoding, search, and moving full payloads around) never runs on the thread that draws the screen, and the UI only ever pulls the handful of kilobytes it needs to show what's currently visible. The history can grow as large as you let it and the panel does not care.

One more thing worth mentioning if you're not on GNOME: the daemon is desktop-agnostic. It speaks plain D-Bus and has a built-in `wl-clipboard-rs` monitor, so it runs standalone on wlroots compositors like Sway and Hyprland with a different front-end against the same interface. The GNOME Shell extension is just the front-end I happened to need.

If you want the details, the [ARCHITECTURE.md](https://github.com/Edu4rdSHL/Strata/blob/main/ARCHITECTURE.md) in the repo covers the storage schema, the FTS5 query construction, the concurrency model, and the security boundary in more depth than a blog post should. The code is GPL-3.0-or-later. If you try it and something stutters, I'd genuinely like to know, because that's not allowed in this project (really).

Happy Ctrl+c/Ctrl+v!
