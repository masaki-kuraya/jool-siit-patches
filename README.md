# jool patches for iptables-mode SIIT translating a self-owned address

Patches against [Jool](https://github.com/NICMx/Jool) to run a **stateless SIIT
(RFC 7915) gateway that translates to/from an IPv4 address the translator itself owns**,
selecting only one TCP port via iptables so the box's own SSH keeps working.

Use case: a $5 VPS receives IPv4 clients on its own `A.B.C.D:443`, SIIT-translates
them to an IPv6 backend, and preserves the client IP (embedded as `<pool6>::<v4>`).
Port 22 must NOT be translated, so netfilter mode (which grabs every packet to the
address) is unusable — iptables mode is required to scope translation to `dport 443`.

## Which patches you need depends on the Jool version

| Problem | 4.1.13 | 4.1.15 |
|---|---|---|
| **A. iptables mode unavailable** (`Netfilter is the only available instance framework`) | needs `0001-...-4.1.13.patch` | **fixed upstream** (no patch) |
| **B. EAMT to a self-owned address is refused** (`Address is subnet-scoped or belongs to a local interface`) | needs `0002-...-4.1.13.patch` | still needs `0002-...-4.1.15.patch` |

**Recommendation: use Jool 4.1.15** (latest stable) so you only need patch B.

### Problem A — NETFILTER_XTABLES not defined (Debian 4.1.13 packaging regression)

4.1.13's build lost the `-DNETFILTER_XTABLES` define, so `jool_siit instance add
--iptables` fails with "Netfilter is the only available instance framework." This is
[Jool #424](https://github.com/NICMx/Jool/issues/424) / [#432](https://github.com/NICMx/Jool/issues/432),
**fixed upstream in v4.1.14** (commit `518790de`). On 4.1.13, `0001-...-4.1.13.patch`
re-adds the define to the three module `ccflags-y` lines; you must also rebuild
`jool-tools` with the same define. On 4.1.15+ this is unnecessary.

### Problem B — explicit EAMT loses to the implicit local-address refusal

Jool refuses to translate an address that belongs to a local interface. When the EAMT
maps the backend GUA to the translator's *own* IPv4, both directions are silently
dropped (`JSTAT46_DST` on 4→6, `JSTAT64_DST` on 6→4).

This is **not fixed for our case even in 4.1.15**. [Jool #223](https://github.com/NICMx/Jool/issues/223)
reworked the implicit denylist to "allow if the address is on an interface **as a /32**",
which helps *secondary* /32 addresses — but a cloud VPS's primary address is typically
a **/24**, so it is still refused. The patch makes an explicit EAMT hit take precedence
over the implicit refusal, in both `addrxlat_siit46` (4→6) and `addrxlat_siit64` (6→4).

Since iptables mode already scopes what reaches Jool (only `dport 443`), the implicit
protection is redundant here. Arguably a **feature request** for upstream: "an explicit
EAM entry should override the implicit local-interface refusal (regardless of prefix length)."

## Applying (Jool 4.1.15, recommended)

```sh
cd jool-4.1.15/                                       # apt-get source jool, or git
patch -p1 < 0002-eamt-overrides-local-address-refusal-4.1.15.patch
# rebuild the DKMS module: dkms remove jool/4.1.15 -k $(uname -r); dkms install jool/4.1.15
# NOTE: to reload, flush the iptables rule that references JOOL_SIIT FIRST — otherwise the
#       module refcount stays >0 and rmmod silently no-ops, leaving the OLD module loaded.
```

## Status

Running in production on a Debian 13 / kernel 6.12 Nanode, Jool 4.1.15, since 2026-08-30.
Verified: real client IPv4 reaches the backend (appears as `<pool6>::<v4>`), SSH on
port 22 unaffected, survives reboot and DKMS kernel-upgrade rebuilds.

Not yet submitted upstream. Problem A is already fixed (4.1.14+). Problem B → Jool
feature discussion (#223 follow-up: primary /24 addresses still can't be EAMT targets).
