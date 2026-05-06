# Fixing yum repos after upgrading an Exadata Cloud Service VM from OL7 to OL8

> *A small post-upgrade gotcha that cost us a few minutes of head-scratching, written down so the next person on the team doesn't have to.*

We recently upgraded a couple of our ExaCS guest VMs from Oracle Linux 7.9 to 8.10 using the standard Leapp path. Reboot was clean, GI and DB came up fine, everything looked good. Then I ran `dnf repolist` and stared at the output for a minute before it clicked.

This post is about that minute, and what to do about it.

## The setup, before the upgrade

Before the upgrade, the box was on:

- **OS:** Oracle Linux 7.9
- **Exadata image:** `22.1.25.0.0.240710`

Repos were exactly what you'd expect on an OL7 ExaCS node:

```bash
[root@exaprd02-xxxxxx yum.repos.d]# pwd
/etc/yum.repos.d
[root@exaprd02-xxxxxx yum.repos.d]# ls -ltr
total 28
-rw-r--r-- 1 root root  260 Feb 25  2021 oracle-ol7.repo
-rw-r--r-- 1 root root  226 Jun  9  2021 virt-ol7.repo
-rw-r--r-- 1 root root 2587 Jun  9  2021 uek-ol7.repo
-r--r----- 1 root root  901 Jul 10  2024 Exadata-computenode.repo.sample
-rw-r--r-- 1 root root 4836 Oct 25  2024 oracle-linux-ol7.repo
-r-xr----- 1 root root 3169 Apr 14 14:39 RPM-GPG-KEY-oracle-ol8
```

`yum repolist` looked the way it should:

```text
ol7_addons/x86_64            Oracle Linux 7Server Add ons (x86_64)             568+413
ol7_latest/x86_64            Oracle Linux 7Server Latest (x86_64)         22,056+5,373
ol7_optional_latest/x86_64   Oracle Linux 7Server Optional Latest (x86_64) 15,656+3,273
repolist: 39,604
```

The only odd thing — and in hindsight, the clue — was that `RPM-GPG-KEY-oracle-ol8` was already sitting there. The upgrade tooling drops it in advance. I didn't think much of it at the time.

## After the upgrade

Post-Leapp, the box came up on:

- **OS:** Oracle Linux 8.10
- **Exadata image:** `25.1.11.0.0.251116`

I ran `dnf repolist`, expecting clean OL8 output. Instead:

```text
[root@exaprd01-xxxxxx ~]# dnf repolist
repo id                                       repo name
ol7_addons /x86_64                            Oracle Linux 8 Add ons (x86_64)
ol7_latest /x86_64                            Oracle Linux 8 Latest (x86_64)
ol7_optional_latest /x86_64                   Oracle Linux 8 Optional Latest (x86_64)
```

Read that carefully. The repo **IDs** still say `ol7_*`. The repo **names** say "Oracle Linux 8". If you only glance at the name column you'll think you're fine. You're not.

What's happening is that dnf is still loading the old `oracle-linux-ol7.repo` file. Oracle relabeled the names inside that file at some point so they say "Oracle Linux 8", which I assume was meant to be helpful but in practice just hides the problem. The IDs don't lie though — `ol7_*` means OL7 channels.

If you `dnf install` anything in this state, best case you pull from a confused channel layout, worst case you pull packages not built for OL8. Don't.

## Why Leapp leaves the old repo file behind

I checked, because I was annoyed. The short answer is: Leapp deliberately doesn't touch your repo files. On a generic OL system those files might point at ULN, an internal mirror, Spacewalk, an Artifactory proxy, whatever. If Leapp rewrote them automatically it'd break more environments than it'd fix.

So on ExaCS, the OL7 repo file just sits there after the upgrade. Nothing replaces it, nothing disables it. Until you do something about it, dnf has nothing else to read.

## The fix

Two steps. Move the OL7 repo file out of the way, drop in an OL8 one.

### Step 1 — get the OL7 file out of dnf's view

Don't `rm` it. Move it into a subdirectory inside `/etc/yum.repos.d/`. dnf only reads `*.repo` files at the top level of that directory, so a subfolder is invisible to dnf but the file is still there if you ever want it back.

```bash
[root@exaprd01-xxxxxx yum.repos.d]# mkdir backup_repos
[root@exaprd01-xxxxxx yum.repos.d]# mv oracle-linux-ol7.repo backup_repos/
[root@exaprd01-xxxxxx yum.repos.d]# dnf repolist
```

After the `mv`, `dnf repolist` returns nothing. That's the right intermediate state — confirms the OL7 file was the only thing dnf was reading.

### Step 2 — write `/etc/yum.repos.d/ol8.repo`

Six repos, which is what most ExaCS guest VMs need:

- `ol8_addons`
- `ol8_UEKR6`
- `ol8_UEKR6_RDMA` *(disabled by default — see notes)*
- `ol8_baseos_latest`
- `ol8_appstream`
- `ol8_codeready_builder`
- `ol8_oci_included`

The important bit is the `$ociregion` and `$ocidomain` variables in the baseurl. Those are populated automatically on OCI instances and resolve to your nearest regional mirror. That's why downloads later in this post hit ~100 MB/s — they're coming over the OCI internal network, not the public internet.

```ini
[ol8_addons]
name=Oracle Linux 8 Addons ($basearch)
baseurl=https://yum$ociregion.$ocidomain/repo/OracleLinux/OL8/addons/$basearch/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
gpgcheck=1
enabled=1
exclude="*.src"

[ol8_UEKR6]
name=Latest Unbreakable Enterprise Kernel Release 6 for Oracle Linux $releasever ($basearch)
baseurl=https://yum$ociregion.$ocidomain/repo/OracleLinux/OL8/UEKR6/$basearch/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
gpgcheck=1
enabled=1
exclude="*.src"

[ol8_UEKR6_RDMA]
name=Oracle Linux 8 UEK6 RDMA ($basearch)
baseurl=https://yum$ociregion.$ocidomain/repo/OracleLinux/OL8/UEKR6/RDMA/$basearch/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
gpgcheck=1
enabled=0
exclude="*.src"

[ol8_baseos_latest]
name=Oracle Linux 8 BaseOS Latest ($basearch)
baseurl=https://yum$ociregion.$ocidomain/repo/OracleLinux/OL8/baseos/latest/$basearch/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
gpgcheck=1
enabled=1
exclude="*.src"

[ol8_appstream]
name=Oracle Linux 8 Application Stream ($basearch)
baseurl=https://yum$ociregion.$ocidomain/repo/OracleLinux/OL8/appstream/$basearch/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
gpgcheck=1
enabled=1
exclude="*.src"

[ol8_codeready_builder]
name=Oracle Linux 8 CodeReady Builder ($basearch) - Unsupported
baseurl=https://yum$ociregion.$ocidomain/repo/OracleLinux/OL8/codeready/builder/$basearch/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
gpgcheck=1
enabled=1
exclude="*.src"

[ol8_oci_included]
name=Oracle Software for OCI users on Oracle Linux $releasever ($basearch)
baseurl=https://yum$ociregion.$ocidomain/repo/OracleLinux/OL8/oci/included/$basearch/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
gpgcheck=1
enabled=1
exclude="*.src"
```

A few things about the repos themselves:

- `baseos_latest` + `appstream` together replace what `ol7_latest` used to give you. OL8 split the package world into BaseOS (the OS itself) and AppStream (modular stuff — Python, Node, PHP, language runtimes). Need both.
- `UEKR6_RDMA` stays disabled. The RDMA fabric on ExaCS is handled by the bare-metal host, the guest doesn't need it. Don't flip it on unless you have a specific reason.
- `oci_included` matters more than people think. Several `python3-oci-cli` deps live there. Skip it and your installs fail in confusing ways.
- `codeready_builder` is officially "Unsupported" by Oracle. You'll need it eventually if you ever build anything against `-devel` packages that don't live in BaseOS or AppStream.

## Verify

`dnf repolist` again:

```text
[root@exaprd01-xxxxxx yum.repos.d]# dnf repolist
repo id                  repo name
ol8_UEKR6                Latest Unbreakable Enterprise Kernel Release 6 for Oracle Linux 8 (x86_64)
ol8_addons               Oracle Linux 8 Addons (x86_64)
ol8_appstream            Oracle Linux 8 Application Stream (x86_64)
ol8_baseos_latest        Oracle Linux 8 BaseOS Latest (x86_64)
ol8_codeready_builder    Oracle Linux 8 CodeReady Builder (x86_64) - Unsupported
ol8_oci_included         Oracle Software for OCI users on Oracle Linux 8 (x86_64)
```

`ol8_*` IDs across the board. Good.

For a stronger sanity check, install something cheap that pulls from multiple OL8 repos:

```bash
dnf install -y python3-oci-cli
```

On our run that pulled `python36-oci-cli-3.81.0-1.el8` plus nine `.el8` deps from `ol8_addons`, `ol8_appstream`, and `ol8_oci_included`, all in about a second from the regional mirror.

```bash
[root@exaprd01-xxxxxx yum.repos.d]# oci --version
3.81.0
```

Done.

## Gotchas

A few things I'd watch out for, especially on a multi-node cluster:

**Do this on every node.** Each compute node has its own `/etc/yum.repos.d/`. Leapp runs per-node, this fix is per-node.

**Don't do all nodes at once.** RAC rolling pattern, same as the upgrade itself. Repo changes alone aren't service-affecting, but a stray `dnf update` while things are misconfigured can be.

**Ignore `leapp-upgrade-repos-ol8.repo.save`.** It's been left there as a record. The `.save` extension means dnf doesn't load it. Don't rename it to `.repo` thinking it's the file you need — it isn't.

**Leave `Exadata-inactive.repo` and `Exadata-computenode.repo.sample` alone.** Oracle-managed. The `.sample` one isn't loaded by dnf anyway because of the extension, but don't delete it.

**GPG keys are already handled.** Your `ol8.repo` references `/etc/pki/rpm-gpg/RPM-GPG-KEY-oracle` and the upgrade tooling has set that up. If `dnf install` ever complains about GPG, look in `/etc/pki/rpm-gpg/`, not `/etc/yum.repos.d/`.

**Keep `backup_repos/` for a while.** A few weeks at least. If something blows up later and Oracle Support asks what your pre-fix state looked like, you've got the original file right there.

## Closing thought

The Leapp upgrade itself is well-trodden. The post-upgrade repo state is just one of those small things the official runbooks don't shout about loudly enough, and if you miss it every package operation afterwards is silently wrong.

If your team has an internal runbook for ExaCS upgrades, this is worth adding as a named post-upgrade step. Don't make the next person discover it the way I did.

---

*Hostnames in the output above have been redacted. Everything else is real.*
