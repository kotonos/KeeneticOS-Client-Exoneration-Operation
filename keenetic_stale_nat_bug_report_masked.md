# KeeneticOS Post-Reboot Static NAT / Port-Forwarding Stale-Target Bug
## Live Reproduction Report with Timestamped CLI/Log Evidence (Redacted)

> **Note:** Identifying information (public WAN IP address, KeenDNS/registered domain names, router hostname, and the target device's MAC address) has been masked/replaced with placeholder values in this version for safe public sharing. All private LAN IPs (`192.168.1.x`), third-party public IPs (Cloudflare edge nodes, scanning hosts), and technical details are unchanged and remain fully representative of the original evidence.

**Router:** Keenetic Extra DSL (KN-2111), hw_id `KN-2111`
**Firmware:** release `5.01.C.1.0-0`, title `5.1.1`, ndw4 `5.1.C.1.0`, region TR
**Reporter role:** Network owner / independent verification via direct CLI (Telnet, NDMS)
**Related public report:** [Keenetic Forum #26592](https://forum.keenetic.com/topic/26592-a-user-experience-keenetic-kernel-level-bug-post-soft-reload-conntracknat-chain-locks-port-forwarding-target-to-a-staleghost-ip-proven-with-arp-nat-table/)

---

## 1. Summary

A MAC-based static port-forwarding rule (`ip static tcp`) targeting a DHCP-reserved host continues to forward traffic to a **previous, stale/expired IP address** after a router reboot ("soft reload"), even after:

- the device successfully completes a fresh DHCP handshake and is assigned/confirmed on its **current, correct IP**,
- the kernel's IPv4 conntrack/route cache is explicitly flushed by the router's own `Network::InterfaceFlusher` during boot.

The rule only re-resolves the correct target IP when it is **manually disabled and re-enabled** (`no ip static tcp ...` / `ip static tcp ...`), which forces the `Network::StaticNat` module to re-register the rule. A plain reboot does **not** trigger this re-registration.

This was reproduced live, on request, on the affected router, with real (non-simulated) external TCP traffic from independent, geographically distributed hosts (Cloudflare edge network + several public scanning hosts) hitting the stale target during the bug window, and a **before/during/after** behavioral test (connection timeout vs. successful HTTP response) confirming both the failure and the fix.

---

## 2. Affected Configuration

```
known host IoT-Device <MAC_ADDR>
ip dhcp host <MAC_ADDR> 192.168.1.114
ip static tcp PPPoE0 80 <MAC_ADDR>
ip http proxy ev
    upstream http <MAC_ADDR> 80
    domain ndns
    ssl redirect
    security-level public
    timeout 86400
ppe software
ppe hardware
```

Notes:
- The forwarding target is defined **by MAC address**, not by static IP, relying on KeeneticOS to resolve the MAC to the host's *current* DHCP lease at rule-apply time.
- The device has a DHCP **static reservation** for `192.168.1.114`, but historically (and again during this test) briefly requests/attempts its old lease `192.168.1.101` before the DHCP server corrects it.
- `ppe hardware` (hardware NAT acceleration / hwnat) is enabled — see §6 for a related observation.
- A second mechanism, `ip http proxy ev` (KeenDNS/nginx reverse proxy), also targets the same MAC independently of the `ip static tcp` rule.

---

## 3. Test Method

1. Captured a "before" snapshot: `show ip neighbour`, `show ip nat tcp`.
2. Manually triggered `system reboot` (soft reload) via NDMS CLI, with owner's explicit consent.
3. Reconnected as soon as the router came back online (~135–195s uptime) and immediately captured:
   - `show system` (uptime confirmation)
   - `show ip neighbour`
   - `show ip nat tcp`
   - `show log`
4. Attempted a live TCP connection to the router's public IP on port 80 from an independent host (LAN-originated, hairpin/NAT-loopback path to the WAN IP) to observe real behavior.
5. Applied the known community workaround: `no ip static tcp PPPoE0 80 <MAC_ADDR>` followed by `ip static tcp PPPoE0 80 <MAC_ADDR>`.
6. Repeated the same connection test to confirm resolution.
7. Additionally configured the router's **built-in native packet capture** (`monitor > capture > interface <if>`, `output-directory`, `filter "tcp port 80"`, `direction in-out`) simultaneously on `PPPoE0` (WAN) and `Home`/`Bridge0` (LAN), writing to a dedicated ext4 USB partition, to allow full packet-level analysis independent of the summarized NAT table.

---

## 4. Timeline of the Reboot Window (system log, local time UTC+3)

```
I [Jul 18 05:37:09] ndm: Core::System::Clock: system time has been changed.
I [Jul 18 05:37:09] ndhcps: DHCPREQUEST received (STATE_SELECTING) for
                    192.168.1.101 from <MAC_ADDR> hostname "IoT-Device".
...
I [Jul 18 05:37:11] ndm: Network::InterfaceFlusher: flushed IPv4 conntrack and
                    route cache.
...
I [Jul 18 05:37:19] ndhcps: DHCPREQUEST received (STATE_INIT) for 192.168.1.101
                    from <MAC_ADDR> hostname "IoT-Device".
I [Jul 18 05:37:19] ndhcps: sending NAK to <MAC_ADDR>.
I [Jul 18 05:37:19] ndhcps: DHCPDISCOVER received from <MAC_ADDR>
                    hostname "IoT-Device".
I [Jul 18 05:37:19] ndhcps: making OFFER of 192.168.1.114 to <MAC_ADDR>.
I [Jul 18 05:37:19] ndhcps: DHCPREQUEST received (STATE_SELECTING) for
                    192.168.1.114 from <MAC_ADDR> hostname "IoT-Device".
I [Jul 18 05:37:20] ndhcps: sending ACK of 192.168.1.114 to <MAC_ADDR>.
```

**Key observation:** the device tried to renew its *old* lease (`.101`) twice, was correctly `NAK`'d, and was correctly re-assigned/confirmed on `.114` at `05:37:20`. DHCP itself behaved correctly.

Critically, **no `Network::StaticNat: static NAT rule has been added` event appears anywhere in this boot cycle's log.** Compare with the *previous* reboot (caused by an unrelated component install), where the exact same rule produced this log line on boot:

```
I [Jul 18 03:57:52] ndm: Network::Nat: a NAT rule added.
I [Jul 18 03:57:52] ndm: Network::Nat: a NAT rule added.
I [Jul 18 03:57:52] ndm: Network::StaticNat: static NAT rule has been added.
```

This strongly suggests the `Network::StaticNat` module does **not** always re-register/re-resolve the MAC→IP binding fresh on every boot — on at least one of the two observed reboots, the rule was carried over from a stale internal/serialized state rather than being freshly bound to the DHCP server's current lease table. This would explain why a manual `no ip static tcp` / `ip static tcp` toggle (which unambiguously forces a fresh `Network::StaticNat` registration) fixes the problem, while a reboot does not reliably do so.

---

## 5. Live NAT Table Evidence — Bug Active (captured within seconds of the ACK above)

```
show ip neighbour   (<MAC_ADDR>)
    address: 192.168.1.101   expired: yes   <- stale
    address: 192.168.1.114   expired: no    <- current, DHCP-confirmed, leasetime 193s

show ip nat tcp   (filtered to relevant rows; <WAN_IP> = router's public IP)
TCP  172.69.63.192   9633   <WAN_IP>  80   4
     192.168.1.101   80     172.69.63.192 9633  4      <- real Cloudflare edge node, forwarded to GHOST IP
TCP  172.69.63.193   13722  <WAN_IP>  80   4
     192.168.1.101   80     172.69.63.193 13722 4      <- same
TCP  160.200.30.10   40733  <WAN_IP>  80   1
     192.168.1.101   80     160.200.30.10 40733 1      <- external scanner, forwarded to GHOST IP
TCP  160.200.32.10   40819  <WAN_IP>  80   1
     192.168.1.101   80     160.200.32.10 40819 1
TCP  160.200.38.10   40820  <WAN_IP>  80   1
     192.168.1.101   80     160.200.38.10 40820 1
```

At the exact same moment, the router's own neighbour table (correctly) shows `192.168.1.114` as the live, non-expired host for that MAC. **Five independent real external sources were forwarded to a dead IP while the correct IP was already confirmed and active.**

### Behavioral confirmation (live connection test)

| State | Test | Result |
|---|---|---|
| Bug active | `GET / HTTP/1.1` to router's public IP:80 | **`TimeoutError`** — connection never established, no RST, no data (silent black hole at `.101`) |
| After toggling `no ip static tcp` / `ip static tcp` | Same test repeated | **`HTTP/1.1 500 Internal Server Error`** returned immediately — proves the packet now reaches the live device at `.114` |

---

## 6. Secondary Observation: Hardware NAT Acceleration Path

While captures were configured on `Home`/`Bridge0` (LAN) with `filter "tcp port 80"`, a LAN-host-originated connection to the router's own public IP (NAT hairpin/loopback) **did not appear in the LAN-side capture at all**, despite succeeding at the connection level and despite the software `show ip nat tcp` table not showing an entry for it either shortly after. Genuine externally-sourced traffic (the Cloudflare/scanner hits above) *did* appear in the software NAT table and WAN-side capture growth.

This is consistent with — but not proof of — the hypothesis (also raised in the original forum thread) that KeeneticOS's hardware NAT acceleration (`ppe hardware`, "hwnat") maintains a flow table that can diverge from the software `nf_conntrack`/`StaticNat` state, particularly around reconnection/reboot events. Further investigation with the native packet-capture feature (see §7) across an additional full reboot cycle would help confirm or rule this out definitively.

---

## 7. Tooling Notes (for reproducing this test)

- KeeneticOS's Telnet/SSH CLI (NDMS) does **not** expose a raw Linux shell, even after installing the `opkg` / `opkg-kmod-netfilter` components. `opkg chroot`, `opkg disk`, etc. are configuration toggles, not command execution.
- A **native, built-in packet-capture feature** exists and does not require Entware/opkg at all:
  ```
  monitor
  capture
  interface <ifname>
      output-directory <disk-label>:/
      filter "<bpf-expression>"
      direction in-out
      enable
  ```
  This writes real `.pcap` files (+ `.stats`) to any mounted storage, and can be run simultaneously on multiple interfaces (e.g., `PPPoE0` and `Home`) — directly satisfying the "simultaneous WAN/LAN tcpdump" requirement without any third-party package installation.
- Component installation (`components` mode, `install <name>`, `commit`) can trigger an **automatic router reboot** when kernel modules are involved (e.g., `opkg-kmod-netfilter`) — this caused two of the reboots observed during this investigation and should be expected/scheduled deliberately.

---

## 8. Conclusion

This investigation independently reproduces, with router-native evidence (system log + live NAT table + real external traffic + a controlled behavioral before/after test), the exact defect reported in Keenetic Forum topic #26592: **a MAC-based static TCP port-forwarding rule can remain bound to a stale/expired DHCP lease after a soft reboot, silently black-holing external and internal connections, until the rule is manually toggled off and back on.** The DHCP server itself, and the `show ip neighbour` table, are correct throughout — the discrepancy is isolated to the `Network::StaticNat` rule-resolution/caching logic not being refreshed on every boot cycle.

### Recommended fix directions for Keenetic engineering
1. `Network::StaticNat` should re-resolve (or at minimum re-validate) each MAC-based static NAT rule's target IP against the current DHCP lease table on every boot, not only when the rule is explicitly (re)configured.
2. Alternatively/additionally, subscribe `Network::StaticNat` to DHCP lease-change/ACK events so a lease renewal (not just a config change) triggers re-resolution.
3. Audit whether `ppe hardware` (hwnat) flow entries are invalidated in lock-step with software `StaticNat`/`nf_conntrack` state during boot and interface flush events.

### Practical workaround (confirmed effective)
```
no ip static tcp <interface> <port> <mac>
ip static tcp <interface> <port> <mac>
```
Run this after every reboot (or automate it, e.g., via a startup script/schedule) until Keenetic ships a fix.

---

## 9. Request for Follow-up

We would appreciate an acknowledgment and status update from Keenetic's engineering team regarding this report. We are happy to provide additional diagnostic files, raw packet captures (`.pcap`), or perform further tests on request. Since this defect causes silent, hard-to-diagnose connectivity failures on a commonly used configuration (MAC-based static port forwarding), we believe it warrants investigation and a fix in an upcoming firmware release.

---
*Report compiled from live CLI/Telnet sessions against the affected router, with the owner's knowledge and consent, on 18 July 2026. Identifying details redacted for public sharing — see note at top of document.*
