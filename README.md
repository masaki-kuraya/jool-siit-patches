# jool 4.1.13 patches for iptables-mode SIIT translating a self-owned address

Two patches against [Jool](https://github.com/NICMx/Jool) 4.1.13 (as packaged in
Debian 13 / `jool-dkms` `jool-tools` 4.1.13-1.1), needed to run a **stateless SIIT
(RFC 7915) gateway that translates to/from an IPv4 address the translator itself owns**,
selecting only one TCP port via iptables so the box's own SSH keeps working.

Use case: a $5 VPS receives IPv4 clients on its own `A.B.C.D:443`, SIIT-translates
them to an IPv6 backend, and preserves the client IP (embedded as `<pool6>::<v4>`).
Port 22 must NOT be translated, so netfilter mode (which grabs every packet to the
address) is unusable — iptables mode is required to scope translation to `dport 443`.

## 0001 — enable NETFILTER_XTABLES (iptables instance framework)

**This looks like a Debian packaging bug.** The kernel ships
`CONFIG_NETFILTER_XTABLES=m`, but Debian's jool build does not pass
`-DNETFILTER_XTABLES`, so the compiled module silently lacks the iptables/xtables
instance framework. `jool_siit instance add --iptables` then fails with
"Netfilter is the only available instance framework."

The patch adds, to the three module `ccflags-y` lines, a define that is a no-op
when the kernel option is absent:

```
$(if $(CONFIG_NETFILTER_XTABLES),-DNETFILTER_XTABLES)
```

`jool-tools` (userspace) must also be rebuilt with the same define so the CLI knows
the framework exists (`DEB_CPPFLAGS_MAINT_APPEND=-DNETFILTER_XTABLES dpkg-buildpackage`).

## 0002 — let an explicit EAMT entry override the implicit local-address refusal

Jool refuses to translate an address that belongs to a local interface (a safety
measure so the translator doesn't eat its own traffic). But when the EAMT maps the
GUA to the translator's *own* IPv4 (`.241`), both directions hit this refusal and
are silently dropped (`JSTAT46_DST` on 4→6, `JSTAT64_DST` on 6→4).

Since iptables mode already scopes what reaches Jool (only `dport 443`), the implicit
protection is redundant here, and an explicit admin EAMT entry should win. The patch:

- `addrxlat_siit46` (4→6): consult the EAMT **before** `must_not_translate`.
- `addrxlat_siit64` (6→4): on an EAMT hit, return `ADDRXLAT_CONTINUE` immediately,
  bypassing the `must_not_translate` check at the `success:` label.

This is arguably a **feature request** for upstream rather than a bug: "an explicit
EAM entry should take precedence over the implicit local-interface refusal."

## Applying

```sh
cd jool-4.1.13/                          # apt-get source jool, or the git tree
patch -p1 < 0001-enable-netfilter-xtables.patch
patch -p1 < 0002-eamt-overrides-local-address-refusal.patch
# then dkms remove/install for the module, and rebuild jool-tools with the XTABLES define
```

## Status

Running in production on a Debian 13 / kernel 6.12 Nanode since 2026-08-30.
Verified: real client IPv4 reaches the backend (appears as `<pool6>::<v4>`),
SSH on port 22 unaffected, survives reboot and DKMS kernel-upgrade rebuilds.

Not yet submitted upstream. 0001 → Debian bug; 0002 → Jool feature discussion.
