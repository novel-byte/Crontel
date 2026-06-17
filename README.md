# Crontel

A download scheduler that runs your downloads when the network is quiet.

**Status: not yet implemented.** This repository holds the design and the
intended behaviour. The code is not written yet.

## The problem

Downloads are usually left to run whenever you happen to start them. That is
fine on a quiet line, but it is not fine on a metered one, and it is not fine
either when a large download competes with everything else you are doing at
the same time.

The fix is unglamorous: run the downloads at night, when there is less
traffic, and keep the rest of the day for the work that needs the bandwidth
now.

Crontel is a small daemon that watches the sources you care about, keeps a
queue of what needs fetching, and works through it on a schedule — either the
schedule you give it, or one it suggests from what it has observed.

## What it does

**Watches sources.** You point it at things you want downloaded: files behind
a URL, releases that appear over time, feeds that grow. It checks them and
notices when something new is there.

**Queues the work.** Anything that needs fetching goes into a queue with a
priority, so the file you need now can jump ahead of the archive you are
filling in the background.

**Manages the queue.** A download that fails is retried with a backoff rather
than being abandoned. A download that is interrupted is resumed rather than
started again. A file that is already present is not fetched twice.

**Runs at the right time.** The schedule can be set by hand, or left to
Crontel, which watches how much of the connection is in use and picks the
quietest window it can find.

## The schedule

The schedule is the part worth explaining, because it is the reason this
exists.

**By hand.** You say when. A cron expression, a fixed hour, or a window like
"between 1am and 5am". Simple and predictable.

**Suggested.** Crontel keeps a running picture of how busy the connection
has been, and works out when the quiet periods are. It looks at the last few
weeks rather than only yesterday, because yesterday may not be
representative. The suggestion changes as the picture improves, and you can
always override it.

The reasoning is simple. If the link is rarely busy before 2am and often busy
at 9pm, the answer is obvious, and no cleverness is needed. The harder case
is a link that is busy at unpredictable times, and there the suggestion
settles on the window that is most often quiet rather than the one that was
quiet last time.

**Pausing.** If the schedule is set and something urgent needs the bandwidth
now, the queue pauses and picks up where it left off.

## The queue

Each entry records where the file comes from, where it is going, how big it
is, and how far along it is.

**Priority.** An entry can be raised to the front of the queue. A download
that has not moved in a while can be raised automatically, so a file that
keeps failing does not block the rest.

**Retry.** A failed download is retried, with the interval growing between
attempts. The attempts stop after a limit, and the entry is marked as failed
rather than retried forever.

**Resume.** A partial download is kept, and the next attempt continues from
where it stopped rather than starting again.

**Dedup.** Two entries for the same file resolve to one. If a file is
already present and complete, it is not fetched.

## The configuration

One file, in TOML:

```toml
[schedule]
mode = "suggested"        # or "fixed"
window = "01:00-05:00"    # used when mode = "fixed"

[[sources]]
name = "epub-catalog"
url = "https://example.org/catalog.json"
schedule = "daily"        # daily, weekly, hourly, or a cron expression
into = "Books"

[[sources]]
name = "magazine"
url = "https://example.org/issues/latest.pdf"
schedule = "weekly"
into = "Reading/magazines"
priority = "low"

[queue]
max_parallel = 2
retries = 5
retry_backoff = "exponential"
```

The shape of it is deliberately plain. A queue, a set of sources, and a
schedule.

## How it is meant to run

As a service, on the machine that holds the files. A systemd timer, a
launchd agent, or a cron entry that starts it and lets it sleep between
runs.

Nothing here needs a server, a database, or an account. The state is a
directory of partial downloads and a small file of queue metadata, both of
which are plain files you can read, back up, and delete.

## What this is not

- **Not a download accelerator.** It does not split a file across
  connections or run in parallel segments. The queue is about *when* to
  download, not how fast.
- **Not a torrent client.** No peer discovery, no swarming.
- **Not a browser.** It fetches over HTTP and nothing else. A source that
  needs a session, a login, or scripting is out of scope.
- **Not clever about traffic.** It schedules around the observed load on your
  own connection. It cannot see what anyone else is doing, and it does not
  try to.

## Licence

MIT.
