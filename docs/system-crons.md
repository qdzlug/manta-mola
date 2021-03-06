---
title: Mola System Crons Overview
markdown2extras: tables, code-friendly, fenced-code-blocks
apisections:
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# Overview

The contents of this repo are joined with Mackerel and work with components
running on other systems to produce system-wide "crons".  Some of these system
tasks depend on others.  In the absence of an uber-coordinator (MANTA-2452),
we rely on time spacing to push tasks through the system.

# Dependency "Graph"

Here is a json representation of the dependency tree:

```
{
    "postgres": {
        "sql-to-json": {
            "storage-hourly-metering": null,
            "mako": {
                "audit": null,
                "cruft": null
            },
            "gc": {
                "gc-links": {
                    "moray-gc": null,
                    "mako-gc": null
                }
            }
        }
    }
}
```

And a description of each of those:

* postgres: Runs on a manatee async, dumps the manatee DB and uploads to Manta.
* sql-to-json: Runs as a Manta job to take the output from postgres and
  transform it to a json format that later jobs understand.
* hourly-metering: Runs as a Manta job, takes output from sql-to-json and
  computes storage metering.
* mako: Runs on each mako zone, dumps a recursive directory listing into Manta.
* audit: Runs as a Manta job, Takes the output from sql-to-json and mako,
  verifies that all objects in moray exist on makos.
* cruft: In not currently running in a cron, but looks exactly like audit.
* gc: Runs as a Manta job, takes output from sql-to-json to figure out what
  objects should be garbage collected.
* gc-links: Runs from the ops zone, verifies the gc Manta jobs and links output
  so that it is picked up by moray-gc and mako gc.
* moray-gc: Runs from the ops zone, takes output from gc/gc-links to find and
  clean up Manta delete log records.
* mako-gc: Runs on each Mako, takes output from gc/gc-create-links to find and
  tombstone dead objects.  Also removes object tombstoned some number of days
  ago (21 days as of this writing).

NOTE: There are three additional jobs scheduled in cron that don't aren't
listed here. They are hourly compute metering, hourly request metering and a
daily summary job. The hourly compute and request metering jobs run every hour
and meter for the previous hour. The daily summary jobs takes as input all of
the previous day's compute and request metering records (24 records each) and
the single storage metering record for a total of 49 records. Since the daily
summary job depends on the last hour of metering from the previous day, it is
run a few hours after midnight to allow those jobs to finish.

# Timeline

Each of the processes above is given some fixed time to complete work for the
day.  This is the "authoritative" timeline:

```
| 00 | 01 | 02 | 03 | 04 | 05 | 06 | 07 | 08 | 09 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 |
|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
|postgres |
|         |sql-to-json                  |
|                                       |storage-hourly-metering      |
|                                       |mako                         |
|                                                                     |audit
|                                                                     |cruft
|                                       |gc            |
|                                                      |gc-l|inks
|                                                           |moray-gc
|                                                           |mako-gc
|                   |(daily-metering)
|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| 00 | 01 | 02 | 03 | 04 | 05 | 06 | 07 | 08 | 09 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 |
```

Also see MANTA-2438.
