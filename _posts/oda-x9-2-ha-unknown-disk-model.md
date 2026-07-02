# The 7.68 TB SSD That Wouldn't Provision: An ODA X9-2-HA vs. 12.1.0.2 Standoff

We recently stood up a new **Oracle Database Appliance X9-2-HA** to host a legacy application whose databases are still on **Oracle 12.1.0.2**. What should have been a routine `create-appliance` turned into a three-week detour through disk metadata, an Oracle SR, a version catch-22, and — in the end — a physical disk swap. I'm writing it down because the "official" fix didn't actually work, and the thing that did work isn't obvious until you understand *why* the official fix failed.

If you're provisioning an X9-2-HA (or any DE3-24C-based ODA) with newer SSDs and you're pinned to an older ODA release, this one's for you.

## The setup

Two nodes, `scodanode01` / `scodanode02`, one shared DE3-24C shelf. We were deliberately deploying **ODA 19.16**, and that "deliberately" matters: our source databases are 12.1.0.2, and **19.16 is the last ODA release that ships the 12.1.0.2 clone** and supports provisioning 12.1 databases. Anything newer drops it. So 19.16 wasn't a lazy choice — it was a hard requirement handed to us by the application.

Provisioning ran fine right up until storage:

```
odacli describe-job -i 36f4db14-...

Status:  Failure
Message: DCS-10001:Internal error encountered: OAK-10005:Disk with unknown
         model number 'MSCAC2DD2ORA7.6T' found on the system. Cause: Failed
         to match above model number with any model number in oakMetadata.
         Action: Check product documentation for supported disk models.

...
Grid home creation            ...  Success
Extract GI clone              ...  Success
Storage discovery             ...  Failure
Discovering DiskGroup         ...  Failure
```

Everything up to and including the GI clone extraction succeeded. It died the moment `oakd` tried to enumerate the shelf disks. The culprit is right there in the message: a disk model string, **`MSCAC2DD2ORA7.6T`**, that OAK doesn't recognize.

## What the error actually means

ODA keeps a list of the disk models it knows how to handle in a metadata config file. On an X9 that file is:

```
/opt/oracle/oak/conf/oakMetadataConf_X9.xml
```

When `create-appliance` gets to storage discovery, `oakd` reads the model string off every physical disk and checks it against the `SsdDiskModelSupported` list in that XML. If the disk reports a model that isn't on the list, OAK refuses to claim it and throws OAK-10005.

We confirmed the disks were physically healthy and visible — the SAS paths and both shelf expanders were fine:

```
fwr_exp_util list
0 Controller [/SYS/MB/PCIE9/SAS2] => Expander[... Secondary ... "E0"]
1 Controller [/SYS/MB/PCIE2/SAS3] => Expander[... Primary   ... "E0"]
```

and `odaadmcli show storage` (when `oakd` was up) showed six Samsung 7.68 TB SSDs:

```
Total number of PDs: 24
  /dev/sdc  SAMSUNG  SSD  7681gb  slot: 24  exp: 2  MSCAC2DD2ORA7.6T
  /dev/sdd  SAMSUNG  SSD  7681gb  slot: 25  exp: 2  MSCAC2DD2ORA7.6T
  ... (six disks, slots 24-29)
```

So the hardware was fine. This was purely a *software doesn't recognize this part number* problem. The disks were a newer manufacturing revision than 19.16's metadata baseline knew about.

## The "official" fix — which didn't hold

We opened an SR, and Oracle came back with a clean-sounding action plan: just add the missing model string to the supported list. Their steps:

1. Back up `oakMetadataConf_X9.xml` on **both** nodes.
2. `chmod 644` the file to make it writable.
3. Add the new model under `SsdDiskModelSupported` — there are **two** sections in the file, both need it:

```xml
<!-- Before -->
<SsdDiskModelSupported ...>
  <Description>SSD disk model supported by OAK</Description>
  <Value>MS9AC2DD2SUN7.6T</Value>
  <Value>HPCAC2DH2ORA7.6T</Value>
  <Value>HPSAC2DH2ORA7.6T</Value>
</SsdDiskModelSupported>

<!-- After -->
<SsdDiskModelSupported ...>
  <Description>SSD disk model supported by OAK</Description>
  <Value>MS9AC2DD2SUN7.6T</Value>
  <Value>HPCAC2DH2ORA7.6T</Value>
  <Value>HPSAC2DH2ORA7.6T</Value>
  <Value>MSCAC2DD2ORA7.6T</Value>   <!-- our disk -->
</SsdDiskModelSupported>
```

4. `chmod 444` to restore permissions.
5. `perl /opt/oracle/oak/onecmd/cleanup.pl -f` on both nodes, then re-provision.

On paper this is airtight: the model string is the only thing missing, so add it and move on. We did exactly that, cleaned up, and re-ran `create-appliance`.

It got *further*. And then it failed differently:

```
odacli describe-job -i d1c94360-...

Status:  Failure
Message: DCS-10001:Internal error encountered: OAK-10029:Fail to validate
         the correct number of disk partitions. Cause: OAK failed to create
         a disk partition on one or more disks. Action: Run cleanup and retry
         again. If issues persist contact Oracle support.

Storage discovery      ...  Failure
Discovering DiskGroup  ...  Failure
```

This is the important moment, so it's worth being precise about it. The metadata edit **worked** — OAK-10005 was gone, the model was accepted, and discovery moved past name-matching into *partitioning* the disks. But partitioning then failed with OAK-10029.

The lesson: **adding a model to the XML allow-list is necessary but not sufficient.** Recognizing a disk's name is one thing; correctly partitioning it needs the deeper storage layers to actually understand that specific drive's geometry and characteristics. Whitelisting the string got us past the bouncer at the door and straight into a wall we couldn't edit our way through. The metadata file is text; the layer that failed next isn't.

## The real problem: a version catch-22

Back on the phone with Oracle, the picture got clear — and uncomfortable:

- The `MSCAC2DD2ORA7.6T` SSD is **only supported from ODA 19.18 onward.** Full stop. 19.16 will never partition it correctly, no matter what you put in the XML.
- But our databases are **12.1.0.2**, which is **only available in ODA 19.16 and earlier.**

Read those two lines together and you have a genuine standoff:

> The **disk** needs 19.18 or newer.
> The **database** needs 19.16 or older.
> On this hardware, as-shipped, those two requirements don't overlap.

Oracle's suggestion was to go to 19.18 and then install a 12.1.0.2 home manually, outside `odacli`, against the 19c Grid Infrastructure. That's a legitimate, supported path (19c GI does support 12.1 databases) — but it's a lot of manual surface area: node pinning, a software-only 12.1 home, ASM disk-group compatibility to worry about (`compatible.rdbms` is irreversible!), and databases that `odacli` will never see or manage. Workable, but heavy.

We wanted a way to keep the appliance fully ODA-managed on 19.16. Which meant the version constraint had to give somewhere — and the one place it could give was the **hardware**.

## The fix that actually worked: swap the disks

Look again at that "Before" XML block Oracle sent us. The very first entry in the already-supported list is:

```
<Value>MS9AC2DD2SUN7.6T</Value>
```

That's *also* a 7.68 TB SSD — same capacity class as our unsupported Samsung drive — and 19.16 already knows it natively. In other words, a 7.68 TB SSD that works on 19.16 does exist; we just had the wrong revision in the shelf.

Our hardware partner at **Eclipsys** ran this down and sourced disks carrying the **`MS9AC2DD2SUN7.6T`** label — the model 19.16 supports out of the box. We physically swapped them into the shelf, ran a clean `cleanup.pl`, and re-provisioned.

This time storage discovery didn't just pass — it kept going all the way through the stack:

```
odacli describe-job -i 9379ae94-...

Status:  Success

Storage discovery             ...  Success
Grid stack creation           ...  Success
Disk group 'RECO' creation    ...  Success
Register Scan and Vips ...     ...  Success
ACFS File system 'DATA' ...    ...  Success
Restart Zookeeper and DCS Agent ... Success
```

Clean success on **19.16** — appliance fully deployed, GI up, disk groups created, SCAN and VIPs registered. And the thing we did all of this to protect was right there waiting for us:

```
odacli list-availablepatches

ODA Release Version  Supported DB Versions    Available DB Versions   Supported Platforms
19.16.0.0.0          12.1.0.2.220719          12.1.0.2.220719         Bare Metal
```

`12.1.0.2.220719` available on bare metal. Exactly what we needed.

## What I'd tell the next person

**The model string in the error is the whole story — read it literally.** OAK-10005 names the exact disk model that isn't supported. Don't go chasing cabling, expanders, or controllers; check that specific string against `SsdDiskModelSupported` in `oakMetadataConf_X9.xml`.

**Editing the XML is a diagnostic, not a cure.** It'll tell you whether name-matching is your *only* problem. If you edit it and provisioning dies at a *different* step (for us, OAK-10029 partition validation), that's your signal the disk genuinely isn't supported on that release and no amount of config editing will save you. Treat the metadata hack as a probe, and don't ship a hand-edited metadata file into production — it gets wiped by the next reimage or patch anyway.

**"Supported from 19.18" and "12.1.0.2 only on ≤19.16" is a real, common trap.** Newer ODAs ship with current-revision disks, but plenty of us are pinned to older releases for old database versions. When the disk and the database pull in opposite directions, you have three levers: move the database version up, decouple the DB from ODA tooling (manual home on a newer release), or move the *disk* revision down to one the older release supports. For us, the disk swap was by far the least invasive — no manual homes, no `compatible.rdbms` landmines, no loss of `odacli` management.

**Lean on your hardware partner.** The winning move here wasn't a clever command — it was knowing that a supported-revision 7.68 TB SSD existed and getting hold of it. Our Eclipsys contact turning up `MS9AC2DD2SUN7.6T` drives is what closed this out.

**Always `cleanup.pl` between attempts.** By the time storage fails, GI is already extracted and users/networks exist. A straight retry trips over that partial state. `perl /opt/oracle/oak/onecmd/cleanup.pl -f` on both nodes, every time, before the next run.

---

*Environment: ODA X9-2-HA, ODA software 19.16, Oracle Linux 7.9, target database 12.1.0.2. Internal management addresses and identifiers have been genericized. Your mileage — and your disk revisions — may vary.*
