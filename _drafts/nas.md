---
layout: post
title: My Network Attached Storage server build
categories:
    - System administration
---
# TODO
* Disk space budgeting
* Learn how to do multiple shares with many users FreeNAS

# Data organization
* Media (movies, tv shows, music, etc.): Snapshot every hour / day, no offsite backup
* Personal data (photos, documents, whatever): Snapshot every hour / day, offsite backup
* Network drive (AFP/NFS/SMB): Used for time machine, scratch data, and so on and other non-backed up stuff like that. No snapshot, no offsite backup

* Torrent is TDB but a temporary idea is to have inprogress downloads stored in
  the transmission jail dataset and move completed torrents to Media/Torrents
  for user triage.
  The big question is of course if users will do the triage.

# Budget

| What                                       | Qty. | Unit price | Sub total|
|--------------------------------------------|:----:|-----------:|---------:|
| HP Proliant Micro Gen 8 with Celeron 1610T | 1    | 300 CHF    | 300 CHF  |
| Linksys powerline adapter                  | 1    |  60 CHF    |  60 CHF  |
| Western Digital RED 3TB hard drive         | 4    | 115 CHF    | 460 CHF  |
| Kingston KVR1333D3E9S (8Go)                | 2    |  80 CHF    | 160 CHF  |
| Ethernet patch cable                       | 2    |  10 CHF    |  20 CHF  |
| Bootable USB key                           | 1    |  20 CHF    |  20 CHF  |

_Total:_ 1020 CHF

## Comparison with Synology
TODO
