# What BC's Permanent Clock Change Did to My Demo Database, One Output at a Time

I want to walk through this the way it actually happened, output by output, because the whole story is sitting in the outputs. If you run Oracle on a British Columbia host you are going to want to run the same handful of checks, and it really helps to know what each line is telling you before you panic or relax.

Here is the background in one breath. BC sprang the clocks forward on March 8, 2026, and that was the last time it will ever do it. No more fall-back. The province stays on UTC-7 all year now and calls it Pacific time. Sounds like good news, and for a normal person it is. But every server you own still has the old "fall back on the first Sunday of November" rule buried inside it, and unless something taught it otherwise, it is still planning to fall back on November 1. On a date that legally no longer has a fall-back.

So I went to poke at a demo box. Oracle Linux 8, a single-instance 19c Standard Edition 2 database, nothing special. Here is everything I ran and what each piece actually meant.

## First look: is this box even affected

```bash
[opc@demo ~]$ rpm -q tzdata
tzdata-2023c-1.el8.noarch
```

That one line told me most of the story already. The `tzdata` package is where Linux keeps every timezone rule in the world, and this copy is version `2023c`. The BC change did not show up in tzdata until version `2026b`. So this box is running rules that are three years too old to have ever heard of permanent Pacific time. Left to itself, it is going to do the old November thing.

An old version number on its own is not proof, though. I wanted to watch the box get November wrong with my own eyes, not just assume it from a package name. So I asked it a very pointed question.

```bash
[opc@demo ~]$ TZ='America/Vancouver' date -d '2026-12-01 20:00:00 UTC'
Tue Dec  1 12:00:00 PST 2026
```

I picked a moment safely past the November boundary, December 1 at 20:00 UTC, and asked what that instant looks like on a Vancouver wall clock. If the box knew about the BC change it would answer in permanent daylight time, UTC-7, which would be 13:00. Instead it came back `12:00 PST`, offset UTC-8. That `PST` is the giveaway. The box still believes BC drops back to standard time for the winter, the same as it did in 2025 and every year before. It never got the memo.

To see the rule itself instead of one sample point, I dumped the transition table.

```bash
[opc@demo ~]$ zdump -v America/Vancouver | grep 2026
America/Vancouver  Sun Mar  8 09:59:59 2026 UT = Sun Mar  8 01:59:59 2026 PST isdst=0 gmtoff=-28800
America/Vancouver  Sun Mar  8 10:00:00 2026 UT = Sun Mar  8 03:00:00 2026 PDT isdst=1 gmtoff=-25200
America/Vancouver  Sun Nov  1 08:59:59 2026 UT = Sun Nov  1 01:59:59 2026 PDT isdst=1 gmtoff=-25200
America/Vancouver  Sun Nov  1 09:00:00 2026 UT = Sun Nov  1 01:00:00 2026 PST isdst=0 gmtoff=-28800
```

Read the bottom two lines as a pair. At 08:59:59 UTC the clock is in PDT, offset `-25200` seconds, which is UTC-7. One second later, at 09:00:00 UTC, it flips to PST, offset `-28800`, UTC-8, and the local time drops from 01:59:59 back to 01:00:00. That is the fall-back, drawn out in black and white, still scheduled for November 1. This is exactly the thing BC legislated out of existence, and this box has it right there on the calendar. Seeing it like that is what took me from "probably a problem" to "yes, definitely."

## The bit that nearly caught me out

Before I patched anything I ran one more check, and I am glad I did, because this is the part of the whole exercise most likely to burn someone. I asked what timezone the database resource itself starts up with.

```bash
[oracle@demo ~]$ srvctl getenv database -d demo_cdb
demo_cdb:
ORACLE_UNQNAME=demo_cdb
TZ=Canada/Pacific
```

Look at that. `TZ=Canada/Pacific`. My shell, my `timedatectl`, and every test I had just run were all on `America/Vancouver`. But the database does not start in my shell. Grid Infrastructure starts it, and it hands the instance `Canada/Pacific`. So all that careful testing had been pointed at a zone the database does not actually use. If I had patched, retested `America/Vancouver`, seen a clean result and closed the ticket, the instance could still have been wrong and I would have walked away none the wiser.

Now, `Canada/Pacific` is an old alias that points at `America/Vancouver` underneath, so I fully expected the two to behave the same. But expecting is not checking, so I checked.

```bash
[oracle@demo ~]$ zdump -v Canada/Pacific | grep 2026
Canada/Pacific  Sun Mar  8 09:59:59 2026 UT = Sun Mar  8 01:59:59 2026 PST isdst=0 gmtoff=-28800
Canada/Pacific  Sun Mar  8 10:00:00 2026 UT = Sun Mar  8 03:00:00 2026 PDT isdst=1 gmtoff=-25200
Canada/Pacific  Sun Nov  1 08:59:59 2026 UT = Sun Nov  1 01:59:59 2026 PDT isdst=1 gmtoff=-25200
Canada/Pacific  Sun Nov  1 09:00:00 2026 UT = Sun Nov  1 01:00:00 2026 PST isdst=0 gmtoff=-28800
```

Same four lines, same fall-back. Good. Same broken rule, and it will take the same fix from the same package. The lesson I am keeping from this: on a Grid Infrastructure box, run `srvctl getenv` before you decide which zone to trust, because the instance's idea of its timezone and your shell's idea are not guaranteed to line up.

For the record, the listener `getenv` came back empty with no TZ line at all, which simply means it uses whatever the host is set to when it starts. Nothing to do there.

## The fix itself

One package update. That is the entire fix at this layer.

```bash
[opc@demo ~]$ sudo yum update tzdata -y
...
Upgrading:
 tzdata   noarch   2026c-1.0.1.el8_10   ol8_baseos_latest   549 k
...
Upgraded:
  tzdata-2026c-1.0.1.el8_10.noarch
Complete!

[opc@demo ~]$ rpm -q tzdata
tzdata-2026c-1.0.1.el8_10.noarch
```

The repo had `2026c`, which is a touch newer than the `2026b` where BC first landed. That is fine, better even. 2026c also carries a similar permanent-time change for Alberta, so if I ever point this at an Edmonton-zoned box it is already covered. The `rpm -q` afterward just confirms the box swapped `2023c` for `2026c`.

## Checking my work, on the right zone this time

```bash
[opc@demo ~]$ TZ='Canada/Pacific'    date -d '2026-12-01 20:00:00 UTC'
Tue Dec  1 13:00:00 MST 2026
[opc@demo ~]$ TZ='America/Vancouver' date -d '2026-12-01 20:00:00 UTC'
Tue Dec  1 13:00:00 MST 2026
```

Now December comes back as `13:00`, offset UTC-7, on both zone names. That is the permanent daylight time BC was after. The exact same instant that used to read 12:00 PST now reads 13:00, an hour later, because the box finally understands the clocks never fell back.

There is one thing here that will make you do a genuine double take, and it is worth slowing down on because it looks wrong and is not. The abbreviation says `MST`. Mountain Standard Time. The box did not move to Mountain. Ignore the letters and look at the offset: `-0700` is UTC-7, which is the correct BC time. What is going on is a quirk in how tzdata labels a zone that is now permanently parked on what used to be its summer offset. Once the summer and winter flag stops flipping, tzdata reaches for the generic name for that offset, and the generic name for UTC-7 happens to be MST. The time is right, the label is just cosmetic and a bit strange. The only place this could ever bite you is if something in your stack keys off the letters "PDT" or "PST" rather than the offset. Worth a grep if you have that kind of tooling.

And the transition table again, to be thorough.

```bash
[opc@demo ~]$ zdump -v Canada/Pacific | grep 2026
Canada/Pacific  Sun Mar  8 09:59:59 2026 UT = Sun Mar  8 01:59:59 2026 PST isdst=0 gmtoff=-28800
Canada/Pacific  Sun Mar  8 10:00:00 2026 UT = Sun Mar  8 03:00:00 2026 PDT isdst=1 gmtoff=-25200
Canada/Pacific  Sun Nov  1 08:59:59 2026 UT = Sun Nov  1 01:59:59 2026 PDT isdst=1 gmtoff=-25200
Canada/Pacific  Sun Nov  1 09:00:00 2026 UT = Sun Nov  1 02:00:00 2026 MST isdst=0 gmtoff=-25200
```

Compare that last line to the baseline. Before, it dropped to `gmtoff=-28800` and the clock jumped back to 01:00. Now it stays at `gmtoff=-25200`, the clock rolls from 01:59:59 straight on to 02:00:00, and life carries on. No repeated hour, no lost hour. The November fall-back is simply gone, which was the entire goal.

## Do not forget to restart

Here is the mistake that is genuinely easy to make. The files on disk are fixed now, but the database and the listener were started long before I ran that update, and a running process reads its timezone rules once at startup and then hangs onto them. So the instance is still carrying the old 2023c rules around in memory even though the disk is right. The only way it picks up the new rules is a restart.

```bash
[oracle@demo ~]$ srvctl stop database -d demo_cdb
[oracle@demo ~]$ srvctl start database -d demo_cdb

[grid@demo ~]$ srvctl stop listener
[grid@demo ~]$ srvctl start listener

[grid@demo ~]$ srvctl getenv database -d demo_cdb
demo_cdb:
ORACLE_UNQNAME=demo_cdb
TZ=Canada/Pacific
```

The `getenv` after the restart still shows `Canada/Pacific`, which is correct and exactly what I wanted. I was never trying to change which zone the instance uses, only to refresh the rules sitting behind that zone, and the restart does precisely that.

Then I logged in and looked at the time.

```sql
SQL> SELECT systimestamp FROM dual;

SYSTIMESTAMP
---------------------------------------------------------------------------
10-SEP-26 07.37.10.889011 AM -07:00
```

This is where people get confused, so let me be blunt about it. It says `-07:00` now, and it also said `-07:00` before I started any of this. Nothing visibly moved. That is not a failure. It is September, and BC is on UTC-7 in September under both the old rules and the new ones, so today looks identical either way. The fix does not show up today. It shows up on November 1, when this box will now hold at UTC-7 instead of sliding back to UTC-8. I did the work in September on purpose, so that November turns out to be a complete non-event.

## The database's own timezone file

One last thing I checked, because Oracle has a second, entirely separate timezone concept that catches people out constantly.

```sql
SQL> SELECT version FROM v$timezone_file;

   VERSION
----------
        38

SQL> SELECT property_name, SUBSTR(property_value,1,30) value
     FROM   database_properties
     WHERE  property_name LIKE 'DST_%'
     ORDER  BY property_name;

DST_PRIMARY_TT_VERSION         38
DST_SECONDARY_TT_VERSION       0
DST_UPGRADE_STATE              NONE
```

This is the database's internal DST file, and it is a completely different animal from the OS package I just patched. It only matters for one specific data type, `TIMESTAMP WITH TIME ZONE`, and only when that data is stored against a named region. It has nothing to do with SYSDATE or SYSTIMESTAMP, and nothing to do with the November problem I just fixed. I am spelling this out because it is the single most common mix-up on this topic. People see an old version number here and panic, or they patch this file and think they have solved the clock problem. Neither is right.

The version here is 38, which is old. But the two lines beneath it are the ones I actually care about. `DST_SECONDARY_TT_VERSION` is 0 and `DST_UPGRADE_STATE` is NONE, and together they mean the database is sitting cleanly on a single version with no half-finished upgrade left hanging. That is a healthy, quiet state. I left it at 38 on purpose. If this database were storing future-dated Pacific timestamps in that particular data type, I would plan a separate upgrade to the version that carries the BC change (that is DSTv46, built from the same 2026c data). But that is its own job on its own schedule, and it changes nothing at all about November.

## So what actually fixed November

Strip the whole thing down and it is simple. The November problem was the operating system's `tzdata`, full stop. SYSDATE and SYSTIMESTAMP come straight off the OS clock, so once the OS knows BC stays on UTC-7 and the database has been restarted to pick that up, the box is correct for November. The Oracle DST file is a side quest that only touches one data type. And the trap in the middle, the instance quietly running on `Canada/Pacific` while I sat there testing `America/Vancouver`, is the first thing I will check on every other box.

demo is done. Start to finish it was about ten minutes and one restart. The hardest part was mentally accepting that a clock showing the right time today can still be dead wrong about a date two months away.

---

*Written up from a live remediation on demo, September 10, 2026. If you decide the database DST file is in scope, confirm the DSTv46 steps against Oracle Support notes KB927784 and 412160.1 first.*
