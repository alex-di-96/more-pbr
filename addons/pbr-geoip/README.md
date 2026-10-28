# pbr-geoip — country-based routing companion for pbr

Routes traffic to whole countries via a chosen pbr interface by destination
IP, complementing pbr's domain-based (dnsmasq.nftset) routing: it also
catches services of the target countries hosted on generic TLDs
(.com/.io/...), where DNS-based TLD routing cannot help.

## How it works

```
LAN client traffic → mangle_prerouting
    ├─ jump pbr_geoip                ← this addon (evaluated first)
    │    ├─ mark already set → return
    │    ├─ fib daddr type local → return   (router-bound traffic untouched)
    │    └─ daddr ∈ @geoip_<cc> → mark = pbr mark of the target interface
    └─ jump pbr_prerouting           ← pbr (marked packets skip policies)
```

- Country CIDR lists are fetched from ipdeny.com (IPv4 + IPv6).
- Sets live inside `table inet fw4` (`/etc/nftables.d/20-pbr-geoip-sets.nft`),
  the marking chain is installed from
  `/usr/share/nftables.d/ruleset-post/25-pbr-geoip.nft` — both are re-applied
  automatically on every fw4 reload/boot, and `25-` sorts before pbr's
  `30-pbr.nft`, so only mark *values* are referenced, never pbr's chains.
- The fw mark for the target interface is resolved from the running pbr via
  `ubus call luci.pbr getInitStatus`, falling back to `pbr.config.uplink_mark`.
- Live updates go through `nft -f -` without an fw4 reload, so pbr's
  dnsmasq-populated nft sets are not flushed by the nightly cron.
- Stale sets of removed countries are garbage-collected.

## Files

| File | Destination |
|------|-------------|
| `pbr-geoip-update` | `/usr/bin/pbr-geoip-update` (0755) |
| `luci.pbrgeoip` | `/usr/share/rpcd/ucode/luci.pbrgeoip` |
| `luci-pbrgeoip.json` | `/usr/share/rpcd/acl.d/luci-pbrgeoip.json` |

## Configuration

`/etc/config/pbr_geoip`:

```
config pbr_geoip 'global'
	option enabled '1'
	option countries 'ru by kz'   # ipdeny.com 2-letter zones
	option interface 'wan'        # pbr interface to route matches via
	option src_addr ''            # optional source subnets; empty = all
	option ipv6 '1'
```

Cron (daily refresh):

```
40 4 * * * /usr/bin/pbr-geoip-update >> /var/log/pbr-geoip.log 2>&1
```

## rpcd API (`luci.pbrgeoip`)

- `getStatus` — config, per-country prefix counts, live per-set hit
  counters from the `pbr_geoip` chain, last update timestamp.
- `getPolicyHits` — per-policy packet counters aggregated by rule comment
  across pbr's chains (requires pbr `nft_rule_counter '1'`).
- `update` — run the updater in the background.
- `setConfig` — validated write of countries/interface/src_addr/enabled.
- `setCounters` — toggle pbr's `nft_rule_counter`/`nft_set_counter` with a
  clean service restart (works around the stale-set EEXIST failure on
  counter definition changes).

A matching LuCI panel (status chips, hit badges, settings dialog) ships in
the companion luci-app-pbr overlay.

## Requirements

- pbr ≥ 1.2.4 (ucode backend; `luci.pbr` ubus for mark resolution)
- firewall4/nftables, dnsmasq-full for pbr's own nftset support
- OpenWrt 23.05+ (tested on 25.12 / nft 1.1.6)
