# The disk was fine. The software just refused to believe it existed.

Field notes from a new ODA X9-2-HA that fought us for three weeks.

---

We got a new Oracle Database Appliance X9-2-HA to replace an aging box. The plan was boring on purpose: image it at ODA 19.16, run `create-appliance`, move our 12.1.0.2 databases across, done.

It was not done. It wasn't done for three weeks.

Writing it up because the "official" fix Oracle handed us didn't actually work, and the thing that finally did isn't obvious until you understand why the official one failed. If you're standing up an X9-2-HA with recent disks while stuck on an older ODA release, maybe this saves you a couple of those weeks.

## First, why we were on 19.16 at all

This part is load-bearing, so bear with me.

Our source databases are 12.1.0.2. And 19.16 happens to be the *last* ODA release that ships a 12.1.0.2 clone — after that Oracle drops it, and `odacli` won't provision a 12.1 database, full stop. So 19.16 wasn't laziness or "we hadn't gotten around to patching." The application pinned us there. Remember that, because it turns into the whole problem later.

## Where it broke

Networks, users, GI clone extraction — all green. Then storage:

```
odacli describe-job -i 36f4db14-...
Status:  Failure
Message: OAK-10005: Disk with unknown model number 'MSCAC2DD2ORA7.6T' found on
         the system. Failed to match above model number with any model number
         in oakMetadata.
...
Extract GI clone      ...  Success
Storage discovery     ...  Failure
Discovering DiskGroup ...  Failure
```

June 3rd. First failure. The message is doing you a favor if you actually read it: it names the exact disk model that's the problem — `MSCAC2DD2ORA7.6T`.

The disks themselves were completely fine, by the way. Both shelf expanders were happy (`fwr_exp_util` showed the Primary and Secondary IOMs of the DE3-24C, both named "E0"), and `odaadmcli show storage` listed six Samsung 7.68 TB SSDs sitting in slots 24–29. Nothing wrong with the hardware. ODA just didn't recognize the part number.

## What OAK-10005 is really telling you

ODA carries a list of disk models it knows how to deal with, in an XML config. On an X9 it's here:

```
/opt/oracle/oak/conf/oakMetadataConf_X9.xml
```

During storage discovery `oakd` reads the model string off each disk and checks it against the `SsdDiskModelSupported` list in that file. Our disk's string wasn't in the list. That's the entire cause of OAK-10005 — the disks were a newer manufacturing revision than 19.16's metadata had ever heard of.

## The fix Oracle gave us (spoiler: it wasn't enough)

Opened an SR. Oracle came back quickly with a tidy-looking plan — just add the model string to the allow-list. Back up the XML on both nodes, `chmod 644`, add one line, `chmod 444`, cleanup, retry:

```xml
<SsdDiskModelSupported ...>
  <Value>MS9AC2DD2SUN7.6T</Value>
  <Value>HPCAC2DH2ORA7.6T</Value>
  <Value>HPSAC2DH2ORA7.6T</Value>
  <Value>MSCAC2DD2ORA7.6T</Value>   <!-- the line we added -->
</SsdDiskModelSupported>
```

(Watch out — there are *two* of these blocks in the file. Both need the edit.)

Made the change on both nodes, ran `cleanup.pl -f`, kicked off `create-appliance` again. And it got further! Then it fell over somewhere new:

```
odacli describe-job -i d1c94360-...
Status:  Failure
Message: OAK-10029: Fail to validate the correct number of disk partitions.
         OAK failed to create a disk partition on one or more disks.
```

June 4th. Different error, same wall.

This is the bit worth slowing down for. The edit *did* do something — OAK-10005 was gone, the model got accepted, discovery walked right past the name check and into partitioning the disks. And then partitioning died.

So here's what that sequence actually tells you: getting the name onto the allow-list is necessary, but nowhere near sufficient. Recognizing a disk by its string is one thing. Laying down partitions on it correctly needs the layers underneath to genuinely understand that drive — and those layers aren't in a text file you can edit. We'd talked our way past the front door and hit a wall in the next room. You can't `vi` your way through that one.

Which, honestly, we should have half-expected. If a config edit were all it took, the disk wouldn't need a newer ODA release in the first place.

## The actual problem: two requirements that don't overlap

Back on the phone with Oracle, and now it got clear and kind of annoying:

- `MSCAC2DD2ORA7.6T` is supported from **ODA 19.18 onward.** Not before. No amount of XML edits changes that.
- Our databases are **12.1.0.2**, which only exists on ODA **19.16 and earlier.**

Sit those two facts next to each other:

- disk wants 19.18+
- database wants 19.16 or older

There's no version that satisfies both. On this hardware, as it shipped, you can't have the disk and the database happy at the same time.

Oracle's out was to go to 19.18 and install the 12.1.0.2 home by hand, outside `odacli`, against the 19c Grid Infrastructure. That's real and supported — 19c GI does run 12.1 databases — but it's a pile of manual work: pin the cluster nodes, build a software-only 12.1 home yourself, sweat the ASM `compatible.rdbms` attribute (which you can never lower once it's set), and end up with databases `odacli` doesn't know exist. Doable. Not what we wanted for a box that's supposed to be appliance-managed.

We wanted to stay on 19.16, fully ODA-managed. So the version conflict had to break somewhere, and the only piece with any give left was the hardware itself.

## The fix that worked: change the disks, not the config

Go back and look at the "before" list Oracle sent us. First entry, already supported:

```
<Value>MS9AC2DD2SUN7.6T</Value>
```

That's *also* a 7.68 TB SSD. Same capacity, different revision — and 19.16 already knows it natively. Meaning: a 7.68 TB drive that works on 19.16 exists. We just had the wrong flavor of it in the shelf.

Our hardware guy at Eclipsys chased this down and got us disks labelled `MS9AC2DD2SUN7.6T` — the model 19.16 supports out of the box. Swapped them into the enclosure, ran a clean `cleanup.pl -f`, re-provisioned.

June 23rd. This time it just... kept going:

```
odacli describe-job -i 9379ae94-...
Status:  Success

Storage discovery              ...  Success
Grid stack creation            ...  Success
Disk group 'RECO' creation     ...  Success
ACFS File system 'DATA' ...     ...  Success
Restart Zookeeper and DCS Agent ... Success
```

Full clean deploy on 19.16. And the whole reason we put ourselves through this was sitting right there:

```
odacli list-availablepatches
19.16.0.0.0   12.1.0.2.220719   12.1.0.2.220719   Bare Metal
```

12.1.0.2 available on bare metal. Which is all we ever wanted.

## Things I know now that I didn't three weeks ago

Read the model string in the OAK-10005 message and take it at face value. It's naming the disk that isn't supported. Don't go poking at cables or expanders — check that one string against `SsdDiskModelSupported` and you've basically diagnosed it.

The XML edit is a test, not a cure. It's genuinely useful for one thing: finding out whether name-matching is your *only* problem. Edit it, retry, and if you die at a different step — for us, OAK-10029 partitioning — that's your answer. The disk isn't really supported on this release and config surgery won't save you. Also, don't leave a hand-edited metadata file on a production box; the next reimage or patch wipes it anyway.

"New disk, old release" is a trap a lot of people are going to hit. Fresh ODAs come with current-revision drives, and plenty of us are stuck on older software for old database versions. When the disk and the database want opposite things, you've got three levers: push the database version up, decouple it from ODA (manual home on a newer release), or drop the disk revision down to something the old release supports. For us the disk swap was miles less painful than the manual-home route — no `compatible.rdbms` traps, nothing falling outside `odacli`.

Your hardware partner might be the actual MVP. The winning move wasn't a clever command. It was someone knowing a supported-revision 7.68 TB SSD existed and getting it into our hands. Thanks Eclipsys.

And run `cleanup.pl -f` on both nodes between every single attempt. By the time storage fails, GI's already extracted and the users and networks exist, so a naked retry just trips over the leftovers.

---

*ODA X9-2-HA, ODA 19.16, OL 7.9, target DB 12.1.0.2. Internal addresses and IDs genericized. Your disk revisions may not match mine — that's kind of the whole point.*
