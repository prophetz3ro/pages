---
layout: page
title: VM networking
permalink: /vm_networking/
---
# Network Bridge — Essentials

A bridge is a **Layer 2 virtual switch**. It connects multiple interfaces to the same Ethernet network.

                    br0
              virtual switch
          ┌────────┼────────┐
        vnet0    vnet1    enp3s0
          │         │         │
         VM1       VM2    Physical LAN

## Bridge ports

- `vnet0` — connects VM1 to the bridge.
- `vnet1` — connects VM2 to the bridge.
- `enp3s0` — connects the bridge to the physical LAN.
- `br0` — represents the bridge and the host's connection to it.

    VM eth0 ←→ vnet0 ←→ br0 ←→ enp3s0 ←→ router

Physical analogy:

    PC NIC ←→ cable ←→ switch port ←→ physical switch
    VM eth0 ←→ virtual link ←→ vnet0 ←→ Linux bridge

## IP addresses

When the physical NIC joins the bridge, the host's IP is normally moved to `br0`:

    br0:     192.168.55.20   # Host IP
    enp3s0:  no IP           # Physical bridge port
    vnet0:   no IP           # VM bridge port
    VM eth0: 192.168.55.30   # VM IP

The host and VM appear as separate devices on the same LAN.

## Forwarding

The bridge learns which MAC addresses are reachable through which ports:

    VM MAC       → vnet0
    Router MAC   → enp3s0

It forwards Ethernet frames based on their **destination MAC address**, not their IP address.

## Firewall

Bridged VM traffic does not normally enter the host's TCP/IP stack:

    VM → vnet0 → br0 → enp3s0

Therefore, host-only firewall paths generally do not apply:

- `INPUT` — traffic addressed to the host.
- `OUTPUT` — traffic created by the host.

A bridge-aware firewall, such as the nftables `bridge` family, can inspect and block frames while they pass through the bridge.

It can filter using:

- MAC addresses
- ARP information
- IP addresses
- TCP/UDP ports

> A bridge connects interfaces into one Ethernet network and forwards frames like a switch. In bridge mode, the VM appears as an independent device on the physical LAN.

# NAT and VM NAT Mode — Essentials

## NAT in general

NAT (**Network Address Translation**) translates addresses as packets move between networks.

Typical home network:

    PC: 192.168.1.10
            │
            ▼
    Router performs NAT
    Public IP: 203.0.113.5
            │
            ▼
         Internet

Example connection:

    Before NAT:
    192.168.1.10:50000 → 8.8.8.8:443

    After NAT:
    203.0.113.5:61000 → 8.8.8.8:443

The router records the translation:

    203.0.113.5:61000 ↔ 192.168.1.10:50000

Replies are translated back and delivered to the original device.

NAT using both addresses and ports is technically called **PAT/NAPT**, but it is commonly called NAT.

Unsolicited incoming traffic normally has no matching translation. **Port forwarding** creates a permanent mapping for selected incoming traffic.

## VM NAT mode

In VM NAT mode, the host acts as a router and performs NAT for the VM:

    VM eth0: 192.168.122.x
               │
               ▼
             vnet7
               │
               ▼
    virbr0: 192.168.122.1
               │
               ▼
      Host routing/firewall/NAT
               │
               ▼
       wlo1: 192.168.55.x
               │
               ▼
         Physical network

Components:

- VM `eth0` — virtual NIC inside the VM.
- `vnet7` — bridge port connecting the VM.
- `virbr0` — virtual switch and host gateway for the VM network.
- `wlo1` — host's physical Wi-Fi interface.

The VM normally uses `192.168.122.1` as its default gateway.

## Bridge and routing roles

`virbr0` connects devices inside the same VM network:

    VM1 ─┐
         ├── virbr0 ── Host gateway
    VM2 ─┘

The host routes between different networks:

    192.168.122.0/24
             │
         Host router
             │
    192.168.55.0/24

`wlo1` is not a port of `virbr0`. Traffic between them passes through the host's routing stack.

## Firewall path

VM traffic passing through the host uses the `FORWARD` chain:

    VM → virbr0 → FORWARD → NAT → wlo1

Main firewall paths:

- `INPUT` — traffic addressed to the host.
- `OUTPUT` — traffic created by the host.
- `FORWARD` — traffic routed through the host.

UFW `route` rules are placed in the `ufw-user-forward` chain.

Other software, such as libvirt or Docker, may install earlier rules. If an earlier rule accepts a packet, UFW rules placed later are not reached.

## NAT mode versus bridge mode

    NAT:
    VM → vnet7 → virbr0 → host routing/NAT → wlo1

    Bridge:
    VM → vnet7 → br0 → physical NIC

In NAT mode:

- The VM uses a separate private subnet.
- The host acts as a router.
- The host rewrites the VM's source address.
- The physical network normally sees traffic as coming from the host.
- Incoming connections to the VM generally require port forwarding.

In bridge mode:

- The VM joins the physical LAN directly.
- The VM receives its own LAN address.
- No host NAT is needed.
- The physical network sees the VM as a separate device.

> A bridge connects devices within one Layer 2 network. A router connects different IP networks. NAT additionally rewrites addressing information between those networks.

# VM NAT Firewall Isolation

## Network topology

    Main network: 192.168.55.0/24
    - ISP router
    - Personal devices

              │
          Lab router/NAT
              │

    Lab network: 192.168.50.0/24
    - Ubuntu lab host

              │
          KVM/libvirt NAT
              │

    VM network: 192.168.122.0/24
    - Kali/Windows VMs

VM traffic path:

    VM eth0
       │
     vnet7
       │
    virbr0: 192.168.122.1
       │
    Host routing + NAT
       │
     wlo1
       │
    Lab network / Internet

## Interfaces

- VM `eth0` — network interface inside the VM.
- `vnet7` — VM-side port attached to the virtual network.
- `virbr0` — virtual switch and VM gateway.
- `wlo1` — host Wi-Fi interface.

`wlo1` is not a port of `virbr0`. The host routes traffic between them.

## Firewall paths

- `INPUT` — traffic addressed to the host.
- `OUTPUT` — traffic generated by the host.
- `FORWARD` — traffic routed through the host.

VM traffic going to another network uses:

    VM → virbr0 → FORWARD → NAT → wlo1

VM traffic addressed directly to the host uses `INPUT`.

## Initial UFW rules

    192.168.55.0/24
        DENY OUT
        From: Anywhere

    192.168.50.0/24 on wlo1
        DENY FWD
        From: 192.168.122.0/24 on virbr0

    192.168.55.0/24 on wlo1
        DENY FWD
        From: 192.168.122.0/24 on virbr0

Meaning:

- The lab host cannot initiate connections to the main network.
- UFW was configured to block VM traffic to both physical networks.

## Why the UFW forwarding rules did not work

The `FORWARD` chain was ordered approximately as follows:

    FORWARD
      ├── DOCKER chains
      ├── LIBVIRT chains
      └── UFW chains

Libvirt accepted VM traffic before it reached `ufw-user-forward`.

The zero counters confirmed this:

    ufw-user-forward:
    0 packets matched

A chain can contain rules that jump to other chains:

    FORWARD → LIBVIRT_FWO
            → ufw-before-forward
            → ufw-user-forward

If an earlier rule returns `ACCEPT` or `DROP`, processing stops.

## Temporary solution

We inserted narrow rules at the beginning of `FORWARD`:

    sudo iptables -I FORWARD 1 \
      -i virbr0 -o wlo1 \
      -s 192.168.122.0/24 \
      -d 192.168.55.0/24 \
      -j DROP

    sudo iptables -I FORWARD 1 \
      -i virbr0 -o wlo1 \
      -s 192.168.122.0/24 \
      -d 192.168.50.0/24 \
      -j DROP

Meaning:

- `-I FORWARD 1` — insert at the top of `FORWARD`.
- `-i virbr0` — packet arrived from the VM network.
- `-o wlo1` — packet would leave through Wi-Fi.
- `-s` — VM source subnet.
- `-d` — blocked destination subnet.
- `-j DROP` — silently discard the packet.

## Effective result

    VM → 192.168.50.0/24     BLOCKED
    VM → 192.168.55.0/24     BLOCKED
    VM → Internet            ALLOWED
    VM → host                POSSIBLE

The increasing DROP counters confirmed that the rules worked:

    sudo iptables -L FORWARD -n -v --line-numbers

The VM can still reach the host because this traffic uses `INPUT`, not `FORWARD`. This also allows access to necessary libvirt services such as DHCP and DNS.

## Useful inspection commands

List the forwarding chain:

    sudo iptables -L FORWARD -n -v --line-numbers

List a specific custom chain:

    sudo iptables -L LIBVIRT_FWO -n -v --line-numbers
    sudo iptables -L ufw-user-forward -n -v --line-numbers

Show all filter rules in exact order:

    sudo iptables-save -t filter

Show NAT rules:

    sudo iptables -t nat -L -n -v --line-numbers

Show the nftables backend:

    sudo nft list ruleset

Check the route to a destination:

    ip route get 192.168.55.1

## Persistence

The direct iptables rules may disappear after:

- reboot;
- UFW reload;
- libvirt restart;
- firewall reconfiguration.

Saving the complete ruleset with `iptables-persistent` is not ideal because UFW, Docker and libvirt dynamically manage their own rules.

A better permanent solution is a dedicated nftables forwarding chain evaluated before libvirt:

    table inet vm_isolation {
        chain forward {
            type filter hook forward priority -10; policy accept;

            iifname "virbr0" oifname "wlo1" \
            ip saddr 192.168.122.0/24 \
            ip daddr { 192.168.50.0/24, 192.168.55.0/24 } \
            counter drop
        }
    }

Before configuring persistence, inspect the current environment:

    iptables --version
    sudo systemctl status nftables
    sudo nft list ruleset
    sudo sed -n '1,200p' /etc/nftables.conf

# Linux Firewall and nftables Notes

## Main components

- **Netfilter** — packet-processing framework inside the Linux kernel.
- **nftables** — modern firewall system built on Netfilter.
- **`nft`** — command used to configure and inspect nftables.
- **UFW, Docker and libvirt** — tools that can create and modify firewall rules.

## Structure

```text
Ruleset
└── Tables
    └── Chains
        └── Rules
```

- **Table** — container used to organize chains.
- **Chain** — ordered list of rules.
- **Rule** — matches packets and performs an action.
- Only a **base chain** connects directly to a Netfilter hook.
- A rule can `jump` to another chain for additional processing.

## Main hooks

- `input` — traffic addressed to the Linux host.
- `output` — traffic generated by the Linux host.
- `forward` — traffic routed through the host, such as VM or container traffic.

For a NAT-connected VM:

```text
VM → Linux host → Internet = forward
```

## Rule processing

Rules are checked from top to bottom.

Common actions:

- `accept` — permit the packet.
- `drop` — silently discard the packet.
- `reject` — discard it and send an error response.
- `jump` — enter another chain and return if no final verdict is reached.
- `return` — return to the calling chain.

If a packet reaches the end of a base chain, its policy applies:

```nft
policy drop;
```

## Counters

Counters are optional:

```nft
tcp dport 22 counter accept
```

Example output:

```text
counter packets 15 bytes 1260
```

A counter beside `jump` means the packet reached that rule. It does not mean the target chain accepted the packet.

Inspect the complete ruleset:

```bash
sudo nft list ruleset
```

Include rule handles:

```bash
sudo nft -a list ruleset
```

## Multiple firewall managers

UFW, Docker and libvirt can all manipulate firewall chains. Their rules may interact because ordering matters.

Example:

```nft
jump LIBVIRT_FWI
jump ufw-before-forward
```

If the libvirt chain gives a terminal verdict first, the later UFW rule may never see the packet.

Possible problems:

- One tool accepts traffic before another tool can block it.
- A service reload removes or reorders manually inserted rules.
- Docker, libvirt and UFW create overlapping filtering or NAT rules.

Avoid manually placing permanent rules inside chains owned by another application.

## Persistent VM isolation

For important VM isolation, use a separate table and an earlier base-chain priority:

```nft
table inet vm_guard {
    chain forward {
        type filter hook forward priority -10; policy accept;

        iifname "virbr0" oifname "wlo1" \
            ip saddr 192.168.122.0/24 \
            ip daddr { 192.168.50.0/24, 192.168.55.0/24 } \
            counter drop
    }
}
```

Meaning:

- `inet` supports both IPv4 and IPv6 rules, although this rule matches IPv4.
- `hook forward` processes routed VM traffic.
- `priority -10` runs before ordinary priority `0` filtering chains.
- Traffic from the VM network to either protected LAN is dropped.
- Nonmatching traffic continues through other base chains, where UFW, Docker or libvirt can still block it.

An earlier `drop` is final; a later tool cannot override it with `accept`.

## Persistence and verification

Store the table in `/etc/nftables.conf` and enable the service:

```bash
sudo systemctl enable --now nftables
```

Verify the table:

```bash
sudo nft list table inet vm_guard
sudo systemctl status nftables
```

After rebooting or reloading UFW, Docker or libvirt, verify the table again and test connectivity from the VM.

Do not use:

```nft
flush ruleset
```

It removes the entire ruleset, including rules managed by other tools.

## Core model to remember

table    → organizes chains
chain    → contains ordered rules
rule     → match plus action
hook     → where packets enter a base chain
verdict  → what happens to a packet
priority → order of base chains on the same hook
 
# nftables Guard Setup

## VM Guard

Create the rules file:

```bash
sudo vim /etc/nftables.d/vm-guard.nft
```

```nft
table inet vm_guard {
    chain forward {
        type filter hook forward priority -10; policy accept;

        iifname "virbr0" oifname "wlo1" \
            ip saddr 192.168.122.0/24 \
            ip daddr { 192.168.50.0/24, 192.168.55.0/24 } \
            counter drop
    }
}
```

Create the service:

```bash
sudo vim /etc/systemd/system/vm-guard.service
```

```ini
[Unit]
Description=VM network isolation rules
After=ufw.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=-/usr/sbin/nft delete table inet vm_guard
ExecStart=/usr/sbin/nft -f /etc/nftables.d/vm-guard.nft
ExecReload=-/usr/sbin/nft delete table inet vm_guard
ExecReload=/usr/sbin/nft -f /etc/nftables.d/vm-guard.nft
ExecStop=-/usr/sbin/nft delete table inet vm_guard

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now vm-guard.service
```

## Lab Guard

Create the rules file:

```bash
sudo vim /etc/nftables.d/lab-guard.nft
```

```nft
table inet lab_guard {
    chain output {
        type filter hook output priority -10; policy accept;

        ip daddr 192.168.55.0/24 counter drop
    }
}
```

Create the service:

```bash
sudo vim /etc/systemd/system/lab-guard.service
```

```ini
[Unit]
Description=Host lab network isolation rules
After=ufw.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=-/usr/sbin/nft delete table inet lab_guard
ExecStart=/usr/sbin/nft -f /etc/nftables.d/lab-guard.nft
ExecReload=-/usr/sbin/nft delete table inet lab_guard
ExecReload=/usr/sbin/nft -f /etc/nftables.d/lab-guard.nft
ExecStop=-/usr/sbin/nft delete table inet lab_guard

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now lab-guard.service
```
# Linux Netfilter and nftables

## Components

- **Netfilter** — packet-processing framework inside the Linux kernel.
- **nftables** — modern firewall system built on Netfilter.
- **`nft`** — command used to configure and inspect nftables.
- **UFW, Docker and libvirt** — tools that can create their own firewall rules.

## Structure

```text
Ruleset
└── Table
    └── Chain
        └── Rule
```

- **Table** — organizes chains.
- **Chain** — contains ordered rules.
- **Rule** — matches traffic and performs an action.
- **Base chain** — directly attached to a Netfilter hook.
- **Regular chain** — processed only when reached using `jump`.

Tables organize rules but do not determine packet-processing order.

## Essential hooks

```text
input    → traffic addressed to the Linux host
output   → traffic generated by the Linux host
forward  → traffic routed through the host
```

For a NAT-connected VM:

```text
VM → Ubuntu → Internet = forward
Ubuntu → Internet      = output
Network → Ubuntu       = input
```

## Rule processing

Rules inside one chain are evaluated from top to bottom:

```nft
ip saddr 192.168.122.0/24 \
    ip daddr 192.168.50.0/24 \
    counter drop
```

Meaning:

```text
source matches
AND destination matches
→ increment counter
→ drop packet
```

Common verdicts:

- `accept` — permit the packet.
- `drop` — silently discard the packet.
- `reject` — discard it and return an error.
- `jump` — process another chain and potentially return.
- `return` — return to the calling chain.

## Base chains and regular chains

A base chain is attached directly to a hook:

```nft
chain forward {
    type filter hook forward priority -10;
}
```

A regular chain does not have a hook:

```nft
chain custom_rules {
    ...
}
```

It must be called from another chain:

```nft
jump custom_rules
```

## Accept and drop behaviour

### Inside one base chain

If a jumped regular chain returns `accept`, processing of the current base chain ends.

```text
FORWARD base chain
├── jump LIBVIRT_FWI → accept
└── jump UFW         → never reached
```

This was the problem with the previous UFW forwarding rules:

```nft
jump LIBVIRT_FWI
jump LIBVIRT_FWO
jump ufw-before-forward
```

Libvirt accepted the packet before the UFW jump was reached.

### Between separate base chains

An `accept` in one base chain does not prevent later base chains on the same hook from processing the packet.

```text
Base chain priority -10 → accept
Base chain priority 0   → drop

Final result: drop
```

A `drop` verdict is final:

```text
Base chain priority -10 → drop
Base chain priority 0   → never reached
```

Simplified model for separate base chains:

```text
accept = “this chain does not block the packet”
drop   = “block the packet completely”
```

Therefore, if any relevant base chain reached by the packet returns `drop`, the packet is dropped.

## Chain priority

Priority orders base chains attached to the same hook.

Lower numbers run first:

```text
-300 → -100 → -10 → 0 → 10 → 100
```

Example:

```nft
priority -10
```

runs before:

```nft
priority filter
```

because:

```text
filter = 0
filter - 10 = -10
```

Your processing order is:

```text
vm_guard forward chain: priority -10
shared FORWARD chain:   priority 0
```

Priority still matters because:

- An earlier `drop` prevents later processing.
- Earlier rules can modify packets.
- Logging and counters depend on which chains are reached.
- Chains with the same priority have no reliable relative order.

## Why `vm_guard` works

```nft
table inet vm_guard {
    chain forward {
        type filter hook forward priority -10; policy accept;

        iifname "virbr0" oifname "wlo1" \
            ip saddr 192.168.122.0/24 \
            ip daddr { 192.168.50.0/24, 192.168.55.0/24 } \
            counter drop
    }
}
```

Processing:

```text
VM packet
→ vm_guard at priority -10
    → protected network: drop
    → other destination: continue
→ shared FORWARD chain at priority 0
→ Docker, libvirt and UFW rules
```

Libvirt cannot override the earlier `drop`.

If the VM accesses the internet, the rule does not match. The `accept` policy lets the packet continue to the normal libvirt forwarding and NAT rules.

## Default policy

The policy applies when a packet reaches the end of a base chain:

```nft
policy drop;
```

Your guard chains use:

```nft
policy accept;
```

They block only explicitly matched traffic and allow other firewall systems to continue processing everything else.

Using `policy drop` in `vm_guard` would block all unmatched forwarded traffic, including the VM's internet access and potentially Docker traffic.

## Connection tracking

Netfilter tracks connections using conntrack:

```nft
ct state established,related accept
```

Important states:

- `new` — starting a connection.
- `established` — belongs to an existing connection.
- `related` — associated with another connection.
- `invalid` — cannot be associated with a valid connection.

Connection tracking permits response packets without requiring separate rules for every temporary client port.

## Address families

- `ip` — IPv4
- `ip6` — IPv6
- `inet` — supports IPv4 and IPv6
- `bridge` — bridge-layer traffic
- `arp` — ARP traffic

An `inet` table can contain both IPv4 and IPv6 rules, but:

```nft
ip daddr 192.168.50.0/24
```

matches only IPv4.

## Counters

Counters are optional:

```nft
counter drop
```

Example:

```text
counter packets 25 bytes 2100
```

A counter beside `jump` means the packet reached the jump. It does not mean the target chain accepted the packet.

## Essential commands

Display the complete ruleset:

```bash
sudo nft list ruleset
```

Include rule handles:

```bash
sudo nft -a list ruleset
```

List tables:

```bash
sudo nft list tables
```

Inspect a table:

```bash
sudo nft list table inet vm_guard
sudo nft list table inet lab_guard
```

Inspect a chain:

```bash
sudo nft -a list chain ip filter FORWARD
```

Load rules from a file:

```bash
sudo nft -f /path/to/rules.nft
```

Validate a file without applying it:

```bash
sudo nft -c -f /path/to/rules.nft
```

Delete a rule using its handle:

```bash
sudo nft delete rule ip filter FORWARD handle 220
```

Delete one table:

```bash
sudo nft delete table inet vm_guard
```

Monitor changes to the active ruleset:

```bash
sudo nft monitor
```

Inspect tracked connections:

```bash
sudo conntrack -L
```

## Dangerous command

Do not casually run:

```bash
sudo nft flush ruleset
```

It removes the complete active ruleset, potentially including rules created by UFW, Docker and libvirt.

Avoid manually inserting permanent rules into chains owned by other applications because those chains may be reordered or recreated.

## Analysis checklist

When investigating firewall behaviour:

```text
1. Determine the packet path: input, output or forward.
2. Find every base chain attached to that hook.
3. Compare their priorities: lower numbers run first.
4. Read rules inside each chain from top to bottom.
5. Follow jump rules into regular chains.
6. Check whether accept ended the current base chain.
7. Remember that later separate base chains can still drop.
8. Treat drop as final.
9. Check counters to confirm which rules were reached.
```

## Core model

```text
hook     → packet-processing stage
priority → order between base chains on the same hook
table    → organizational container
chain    → ordered collection of rules
rule     → match conditions plus action
jump     → enter a regular chain
accept   → stop the current base chain
drop     → final rejection
counter  → evidence that a rule was matched
```