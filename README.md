# Lab: Configure a Static IPv4 Address with `nmcli`

**Series:** linux-ops-mastery — RHCSA Networking
**Subjects covered:** NetworkManager connection profiles, `nmcli con show`, `nmcli con mod` for `ipv4.method`, `ipv4.addresses`, `ipv4.gateway`, `ipv4.dns`, `nmcli con up`, `ip addr`, `ip route`, avoiding SSH lockout during changes
**Career arcs covered:** RHCSA (EX200 network reconfiguration tasks), RHCE (Ansible `community.general.nmcli`), SRE (incident triage on misconfigured NICs), DevOps (immutable infra vs ad-hoc fixes), AI/MLOps (GPU nodes with fixed management IPs)
**Prerequisite:** SSH or console access to a RHEL 9 VM; basic comfort with `sudo` and shell variables
**Time Estimate:** 35 to 50 minutes
**Difficulty arc:** Task 1 foundation · 2–3 nmcli static IPv4 core split · 4–5 verify with `ip` · 6 exam-realistic capstone

---

## Objective

Move from "the VM got an address from DHCP and I hope it sticks" to **deliberate IPv4 configuration** you can reproduce on paper. By the end of this lab you can identify the active NetworkManager connection, convert it to a manual IPv4 profile with address, prefix, gateway, and DNS, apply the change, and prove it with the `ip` suite — the same verification loop an RHCSA grader expects.

The capstone is the RHCSA-realistic prompt: *"Configure the primary interface with static IPv4 `192.168.50.100/24`, gateway `192.168.50.1`, and DNS `8.8.8.8`; bring the connection up and show address and default route."*

> **Lab safety note:** Changing IPv4 can **drop your SSH session** if the new subnet does not match your network. Practice on a **local VM console** or a disposable lab network first. This lab uses example addresses in the `192.168.50.0/24` range — **substitute addresses your instructor or cloud panel assigns** before you apply `nmcli con up`.

---

## Concept: NetworkManager Owns Profiles; the Kernel Owns Addresses

On modern RHEL, **NetworkManager** is the control plane. It reads and writes connection **profiles** (XML under `/etc/NetworkManager/system-connections/`). When you `nmcli con up`, NM pushes the chosen settings into the kernel through netlink — and the kernel exposes the result through `ip addr` and `ip route`.

```
   ┌──────────────────────────────────────────────────────────────┐
   │  You (nmcli)                                                  │
   │       │                                                       │
   │       ▼                                                       │
   │  NetworkManager  ──►  connection profile (UUID, NAME, IPv4)  │
   │       │                                                       │
   │       ▼                                                       │
   │  kernel networking  ──►  addresses, routes, DNS resolver glue │
   │       │                                                       │
   │       ▼                                                       │
   │  `ip addr` / `ip route`  ◄── what you verify on exam day      │
   └──────────────────────────────────────────────────────────────┘
```

**Mental model:** `nmcli` edits **intent**; `ip` shows **reality**. If they disagree, NM did not apply, the wrong profile is active, or a lower tool bypassed NM.

> **Why this matters:** RHCSA does not grade "I edited a file somewhere." It grades **working network state**. The repeatable spine is `nmcli con mod …` → `nmcli con up …` → `ip addr` / `ip route`. That spine survives reboots because the profile is persistent.

---

## 📜 Why Static IP Configuration Exists — The Story

Long before clouds auto-assigned private addresses, every UNIX machine on a LAN needed an operator-chosen IPv4 address, subnet mask, and default gateway typed into a configuration file — or burned into a bootp response from a central server. As networks grew, **BOOTP and then DHCP** automated address assignment so laptops could roam between buildings without renumbering.

But DHCP never replaced static configuration — it **complemented** it. Routers, firewalls, storage arrays, and database primaries still use **fixed addresses** so DNS records, firewall rules, and monitoring targets stay stable across reboots. Data centers standardized on **prefix notation** (`192.168.10.5/24`) instead of separate "subnet mask" fields because the slash carries the same information in one token.

Linux historically fragmented: `ifconfig` and `/etc/sysconfig/network-scripts/` on RHEL, `/etc/network/interfaces` on Debian, `netplan` on Ubuntu. RHEL 7 through 9 converged on **NetworkManager** as the single supported user interface for desktop and server installs. The `nmcli` command exists so automation and remote shells can do what GUI tools did — without X11.

Today, even "cloud" VMs often need a **static private IP** on a secondary NIC for storage traffic, or a **predictable management address** for out-of-band recovery. Kubernetes nodes, GPU training hosts, and bastions are typical static-IP workloads. DHCP remains the default for laptops; static remains the default for infrastructure identities.

> **The point of the story:** Static IPv4 is not legacy trivia — it is how infrastructure **declares identity**. NetworkManager is how RHEL **declares intent** in a supportable, exam-aligned way.

---

## 👪 The NetworkManager IPv4 Family — Who Lives There

### Core `nmcli` verbs for connections

| Verb | What it does | Typical pairing |
|---|---|---|
| `nmcli con show` | Lists profiles and UUIDs | First step before any `con mod` |
| `nmcli con show NAME` | Dumps one profile in key/value form | Confirm `ipv4.method` before/after |
| `nmcli con mod NAME …` | Writes persistent profile properties | Always followed by `con up` |
| `nmcli con up NAME` | Activates profile; applies changes | The "make it real" step |
| `nmcli dev status` | Shows device ↔ connection mapping | Find which `con` is on `eth0` |

### Common `ipv4.*` properties you will set in this lab

| Property | Values you will use | Meaning |
|---|---|---|
| `ipv4.method` | `auto` (DHCP) or `manual` | How IPv4 is obtained |
| `ipv4.addresses` | `A.B.C.D/NN` | Static address + prefix length |
| `ipv4.gateway` | `A.B.C.D` | Default route via this IPv4 |
| `ipv4.dns` | space-separated list | Resolver addresses pushed to stub resolver stack |

### Verification commands (`iproute2`)

| Command | What it proves |
|---|---|
| `ip addr show DEV` | Addresses actually assigned to interface |
| `ip route show` | Kernel routing table including `default via` |
| `ip -4 route show default` | Narrow view of default gateway |

> **The point of the family tree:** Editing addresses without **activating** the profile is the most common "I changed it but nothing happened" mistake. `con mod` writes; `con up` applies.

---

## 🔬 The Anatomy of `nmcli con mod` — In One Diagram

```
$ nmcli con mod "Wired connection 1" ipv4.method manual \
      ipv4.addresses 192.168.50.100/24 ipv4.gateway 192.168.50.1 ipv4.dns "8.8.8.8"
  │      │   │                           │                │                 │
  │      │   │                           │                │                 └─ Resolver list (quoted if multiple)
  │      │   │                           │                └─ Default gateway IPv4
  │      │   │                           └─ Address + CIDR prefix (one assignment)
  │      │   └─ Property key namespace for IPv4 settings
  │      └─ Subcommand: modify an existing connection profile by name or UUID
  └─ NetworkManager CLI entry point

Follow-up (not optional):
$ nmcli con up "Wired connection 1"
  └─ Re-activates the profile so the kernel receives the new IPv4, routes, and DNS hints.

Sanity check afterwards:
$ ip -4 addr show eth0
$ ip -4 route show default
  └─ These read kernel state — if output is wrong, the profile did not apply to the device you think it did.
```

> **Reading rule:** Treat `con mod` + `con up` as a **single transaction**. Mod without up is an unfinished transaction.

---

## 📚 Static IPv4 with NetworkManager — Reference Table

| Task | Command pattern | Notes |
|---|---|---|
| List connections | `nmcli con show` | Note exact NAME spelling |
| Show active connection on a device | `nmcli -t -f DEVICE,NAME dev status` | Maps `eth0` → profile |
| Set static IPv4 | `nmcli con mod NAME ipv4.method manual ipv4.addresses IP/PREFIX …` | Include prefix |
| Set gateway | `nmcli con mod NAME ipv4.gateway GW` | Creates default route when profile is up |
| Set DNS | `nmcli con mod NAME ipv4.dns "8.8.8.8 1.1.1.1"` | Space-separated inside quotes |
| Apply profile | `nmcli con up NAME` | Required after edits |
| Revert to DHCP | `nmcli con mod NAME ipv4.method auto` then `nmcli con up NAME` | Lab cleanup pattern |
| Verify address | `ip -4 addr show DEV` | Replace DEV with your NIC |
| Verify default route | `ip -4 route show default` | Should show `via` gateway |

> **Rule one of static IPv4:** Always carry **prefix length** with the address (`/24`). Forgetting the prefix produces ambiguous or rejected profiles.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | EX200 frequently includes "configure this interface with this address/gateway/DNS using nmcli" — speed and zero typos matter. |
| **RHCE candidate** | Ansible `community.general.nmcli` wraps the same property keys; understanding `ipv4.method manual` maps directly to playbooks. |
| **SRE / Platform** | Mis-set gateways are a top-ten outage cause; `ip route` is your 10-second triage command. |
| **DevOps** | Immutable images still need bootstrap networking; cloud-init often generates an NM profile underneath. |
| **AI / MLOps** | Training nodes often pin a management IP separate from high-speed RDMA interfaces — same NM skills, more NICs. |

---

## 🔧 The 6 Tasks

> Six phases that build the **inspect → modify → activate → verify** habit without skipping the apply step.

---

### Task 1 — Discover the active connection and baseline IPv4 state

**Purpose:** Capture the exact NetworkManager profile name and current IPv4 configuration before you change anything.

```bash
sudo -i
nmcli dev status
nmcli con show --active
PRIMARY=$(nmcli -t -f NAME con show --active | head -n 1)
echo "PRIMARY_CONNECTION=${PRIMARY}"
nmcli -f ipv4 con show "$PRIMARY"
ip -4 addr
ip -4 route
```

**Human-Readable Breakdown:** Elevate to root for consistent NM permission behavior, list devices and active profiles, stash the first active connection name in `PRIMARY`, print the current IPv4 method and addresses from NM's perspective, then show kernel addresses and routes.

**Reading it left to right:** `nmcli dev status` correlates interfaces with profiles. `con show --active` confirms what NM thinks is running. `nmcli -f ipv4 con show` filters to IPv4 fields only. `ip -4` limits output to IPv4 lines.

**The story:** You cannot configure what you cannot name. The exam loves slightly ugly default profile names — copy them exactly.

**Expected output:**

```text
DEVICE  TYPE      STATE      CONNECTION
eth0    ethernet  connected  Wired connection 1
...
PRIMARY_CONNECTION=Wired connection 1
ipv4.method:                            auto
ipv4.addresses:                        192.168.122.45/24
ipv4.gateway:                           192.168.122.1
...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.122.45/24 brd 192.168.122.255 scope global dynamic noprefixroute eth0
default via 192.168.122.1 dev eth0 proto dhcp src 192.168.122.45 metric 100
```

**Switches**

| Token | Meaning |
|---|---|
| `sudo -i` | Root shell — avoids per-command password prompts |
| `nmcli -t` | Tab-separated "terse" output for scripting |
| `head -n 1` | Grab first active profile when multiple exist |
| `ip -4` | Show IPv4 objects only |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `PRIMARY` is empty | No active connection — run `nmcli con up` on the wired profile first |
| Two active profiles | Pick the one bound to `eth0` via `nmcli dev status` |
| `ip` shows no `inet` | Link down — check virtual cable / `nmcli dev connect eth0` |

---

### Task 2 — Prepare variables and set manual IPv4 with `nmcli con mod`

**Purpose:** Convert the active profile to **manual** IPv4 with address, gateway, and DNS — without activating yet (we practice the mod step in isolation).

```bash
DEV=eth0
CON=$(nmcli -t -f DEVICE,NAME dev status | awk -F: -v d="$DEV" '$1==d{print $2; exit}')
echo "CONNECTION=$CON DEVICE=$DEV"
nmcli con mod "$CON" ipv4.method manual \
  ipv4.addresses 192.168.50.100/24 \
  ipv4.gateway 192.168.50.1 \
  ipv4.dns "8.8.8.8"
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns con show "$CON"
```

**Human-Readable Breakdown:** Map `eth0` to its bound connection name automatically, assign a static `/24` address, default gateway, and one DNS resolver, then re-read the IPv4 fields from the profile to confirm the write.

**Reading it left to right:** `awk` selects the connection name whose device column matches `eth0`. `ipv4.method manual` disables DHCP for IPv4 on this profile. `ipv4.addresses` uses CIDR notation. `ipv4.dns` accepts a quoted list.

**The story:** Splitting **modify** and **activate** is a teaching step — on the exam you will usually run them back-to-back, but separating them helps you see that the profile can be correct while the kernel is still old.

**Expected output:**

```text
CONNECTION=Wired connection 1 DEVICE=eth0
ipv4.method:                            manual
ipv4.addresses:                        192.168.50.100/24
ipv4.gateway:                           192.168.50.1
ipv4.dns:                               8.8.8.8
```

**Switches**

| Token | Meaning |
|---|---|
| `ipv4.method manual` | Static IPv4 — NM will not DHCP this profile |
| `ipv4.addresses …/24` | Address plus prefix length (bits) |
| `ipv4.gateway` | Installs default IPv4 route when profile is active |
| `ipv4.dns` | Upstream resolvers associated with this connection |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Error: invalid <setting>.<property> '…'.` | Typo in property name — compare to `nmcli con show` output |
| `gateway … is not in the subnet` | Gateway must live inside the connected prefix for typical LANs |
| Wrong `CON` selected | Print `nmcli dev status` and set `DEV=` to your real NIC name |

---

### Task 3 — Activate the profile and confirm with `ip addr` and `ip route`

**Purpose:** Apply the profile and prove the kernel picked up address and default route.

```bash
nmcli con up "$CON"
ip -4 addr show "$DEV"
ip -4 route show
ip -4 route show default
```

**Human-Readable Breakdown:** Re-activate the modified profile, then read back the IPv4 address on the device and the full IPv4 routing table, with an extra narrowed view of the default route.

**Reading it left to right:** `con up` triggers the activation transaction. `ip addr show DEV` scopes to one NIC. `ip route show default` filters to the `default` entry only.

**The story:** This is the **moment of truth** — if your lab network truly is `192.168.50.0/24`, you should see the new `inet` line. If your lab is still on `192.168.122.0/24`, these addresses were for pedagogy only — substitute instructor-provided values before running Task 3 on a live SSH session.

**Expected output:**

```text
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.50.100/24 brd 192.168.50.255 scope global noprefixroute eth0
default via 192.168.50.1 dev eth0 proto static metric 100
192.168.50.0/24 dev eth0 proto kernel scope link src 192.168.50.100 metric 100
default via 192.168.50.1 dev eth0 proto static metric 100
```

**Switches**

| Token | Meaning |
|---|---|
| `nmcli con up` | Apply profile now (not only at next boot) |
| `ip -4 addr show` | Interface-scoped IPv4 addresses |
| `ip -4 route show` | IPv4 forwarding table |
| `proto static` | Route came from NM static config (not DHCP) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Error: Connection activation failed` | Journal: `journalctl -u NetworkManager -n 50 --no-pager` |
| Address unchanged after `con up` | Wrong profile — re-derive `CON` from `nmcli dev status` |
| No default route | Missing `ipv4.gateway` or conflict with second NIC |

---

### Task 4 — Persistence check: reboot semantics and on-disk profile

**Purpose:** Demonstrate that NetworkManager profiles survive reboot — without forcing a reboot in a short lab — by inspecting the stored connection file.

```bash
ls -1 /etc/NetworkManager/system-connections/
grep -E '^(address|gateway|dns|method)=' "/etc/NetworkManager/system-connections/${CON}.nmconnection" 2>/dev/null || \
  sudo grep -E '^(address|gateway|dns|method)=' /etc/NetworkManager/system-connections/*.nmconnection | head
nmcli con show "$CON" | grep -E '^connection\.(id|uuid|autoconnect)'
```

**Human-Readable Breakdown:** List system connection files, locate IPv4-related keys in the backing `.nmconnection` file (name may vary slightly by NM version), and show autoconnect metadata from `nmcli`.

**Reading it left to right:** `system-connections` is the on-disk source of truth for persistent profiles. `connection.autoconnect yes` means NM will bring this profile up when the link appears.

**The story:** RHCSA rewards **reboot-safe** configuration. If your static IPv4 only exists as a one-off `ip addr add`, you will lose points — NM profiles are the supported persistence mechanism.

**Expected output:**

```text
Wired connection 1.nmconnection
...
method=manual
address1=192.168.50.100/24,192.168.50.1
dns=8.8.8.8;
...
connection.id:                          Wired connection 1
connection.uuid:                       6b697373-6964-0000-0000-000000000001
connection.autoconnect:                yes
```

**Switches**

| Token | Meaning |
|---|---|
| `*.nmconnection` | Keyfile format used by modern NM |
| `connection.autoconnect` | Whether NM auto-activates on carrier |
| `grep -E` | Extended regex filter |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| File not named exactly like `CON` | Use `ls` and open the file matching your UUID from `nmcli con show` |
| Permission denied | Run `sudo grep` against files owned by root |
| `method=auto` still | `con mod` targeted wrong profile — re-check `CON` |

---

### Task 5 — Edge case: secondary address vs replacement (nmcli `+ipv4.addresses`)

**Purpose:** Learn how to **add** an additional IPv4 alias without wiping the first address — a common follow-up in multi-homed designs.

```bash
nmcli con mod "$CON" +ipv4.addresses 192.168.50.200/24
nmcli con up "$CON"
ip -4 addr show "$DEV"
```

**Human-Readable Breakdown:** The leading `+` on `+ipv4.addresses` appends a second address to the profile instead of replacing the list, re-activate, and show both `inet` lines on the interface.

**Reading it left to right:** Without `+`, `ipv4.addresses` replaces the entire list. With `+`, NM merges. `con up` reapplies the interface configuration.

**The story:** Production nodes often carry a **service IP** plus a **management IP** on the same NIC. The exam may not ask for aliases, but interviewers love the distinction between replace and merge.

**Expected output:**

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.50.100/24 brd 192.168.50.255 scope global noprefixroute eth0
    inet 192.168.50.200/24 brd 192.168.50.255 scope global secondary noprefixroute eth0
```

**Switches**

| Token | Meaning |
|---|---|
| `+ipv4.addresses` | Append an address block to existing list |
| `secondary` keyword in `ip` output | Additional address on same interface |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| First address disappeared | You used `ipv4.addresses` without `+` — restore from Task 2 values |
| Duplicate /32 errors | Ensure prefixes do not contradict connected routes |

---

### Task 6 — Capstone: full static IPv4 answer, verify, and safe cleanup

**Task statement:** *"Configure the primary Ethernet connection with static IPv4 `192.168.50.100/24`, gateway `192.168.50.1`, and DNS `8.8.8.8`. Activate the profile. Show IPv4 address and default route. Return the host to DHCP when finished."*

**Purpose:** Execute the full exam-style sequence on one spine, verify with `ip`, then revert to DHCP to avoid bricking future lab sessions.

```bash
sudo -i
DEV=eth0
CON=$(nmcli -t -f DEVICE,NAME dev status | awk -F: -v d="$DEV" '$1==d{print $2; exit}')
nmcli con mod "$CON" ipv4.method manual \
  ipv4.addresses 192.168.50.100/24 \
  ipv4.gateway 192.168.50.1 \
  ipv4.dns "8.8.8.8"
nmcli con up "$CON"
ip -4 addr show "$DEV"
ip -4 route show default
getent hosts redhat.com | head -n 2
```

**Human-Readable Breakdown:** Resolve the device-bound connection, write static IPv4 fields, activate, show address and default route, and perform a lightweight DNS resolution sanity check via libc (`getent hosts`).

**Layer stack you verified:**

```text
Application (getent) → stub resolver → NSS → unbound/systemd-resolved ← hints from NM
Kernel FIB (`ip route`) ← default via gateway from NM profile
Kernel addresses (`ip addr`) ← `ipv4.addresses` from NM profile
```

**The story:** This is the **four-minute exam answer**: name the profile, `con mod`, `con up`, `ip` twice. DNS in NM populates downstream resolver stacks; a quick `getent` proves usefulness beyond ping.

**Expected verification output:**

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.50.100/24 brd 192.168.50.255 scope global noprefixroute eth0
default via 192.168.50.1 dev eth0 proto static metric 100
2600:1406:bc00:53::b81e:94ce  redhat.com
2600:1406:3a00:21::173e:2e66  redhat.com
```

**Cleanup**

```bash
nmcli con mod "$CON" ipv4.method auto
nmcli con mod "$CON" ipv4.addresses "" ipv4.gateway "" ipv4.dns ""
nmcli con up "$CON"
ip -4 addr show "$DEV"
ip -4 route show default
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Locked out of SSH | Use hypervisor console; fix gateway/address to match real LAN |
| DNS still broken after static | Add `ipv4.ignore-auto-dns yes` when DHCP DNS conflicts |
| `con up` hangs | Carrier down — attach link or `nmcli dev connect "$DEV"` |

---

## 🔍 Static IPv4 Decision Guide

```
Need to set IPv4 on RHEL 9?
  │
  ├── "Temporary test — survives reboot not required"
  │       └── ✅ `ip addr add …` (not exam-primary) — remember to delete
  │
  ├── "Exam / production — must survive reboot"
  │       └── ✅ `nmcli con mod` + `nmcli con up`
  │
  ├── "I have multiple NICs"
  │       └── ✅ Run mapping step per `DEV=` — never assume `eth0`
  │
  ├── "I need an alias IP on same NIC"
  │       └── ✅ `nmcli con mod CON +ipv4.addresses NEW/PREFIX`
  │
  └── "I need to undo everything quickly"
          └── ✅ `ipv4.method auto` + clear static fields + `con up`
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Discover active NM connection and baseline `ip` state
- [ ] 02 `nmcli con mod` manual address, gateway, and DNS
- [ ] 03 `nmcli con up` and verify with `ip addr` / `ip route`
- [ ] 04 Inspect on-disk `.nmconnection` persistence metadata
- [ ] 05 Append secondary IPv4 with `+ipv4.addresses`
- [ ] 06 Capstone static IPv4 + `getent` verify + DHCP cleanup

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `con mod` without `con up` | `ip` still shows old DHCP address | Always activate after modify |
| Forgot `/prefix` | NM rejects or mis-interprets address | Use full CIDR (`/24`, `/16`, …) |
| Wrong profile name | Changes apply to Wi-Fi, not Ethernet | Derive `CON` from `nmcli dev status` |
| Static IP on wrong subnet for your SSH path | Immediate disconnect | Use console or instructor addresses |
| Wiped alias list without `+` | Secondary IPs vanish | Use `+ipv4.addresses` to append |
| Mixed `ip` and `ifconfig` mental models | Confusion about netmask | Prefer `ip` only on RHEL 9 |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize the property keys: `ipv4.method manual`, `ipv4.addresses`, `ipv4.gateway`, `ipv4.dns`. Verbatim spelling beats guessing on exam day.

**RHCE candidate**
- Explain how your Ansible role maps `ipv4.dns` to a list variable and guards against empty gateway strings.

**SRE / Platform interview**
- Walk through "I can ping the gateway but not the internet" — distinguish **missing default route** (`ip route`) from **DNS failure** (`getent` vs `ping 8.8.8.8`).

**DevOps**
- Discuss why golden images still embed NM profiles instead of raw `ip` commands in systemd units.

**AI / MLOps**
- Multi-homed GPU nodes: separate **cluster traffic** NIC profile from **out-of-band** profile — duplicate this lab per device.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 32 — Check Network Connectivity | Validates the static IP with `ping` and path tracing |
| Lab 33 — Display IP and Routing Info | Deepens `ip addr` / `ip route` reading skills |
| Lab 34 — Inspecting Listening Sockets | Moves from host IP to listening services |
| Lab 35 — Text-Based Network Config `nmtui` | GUI-like alternative to `nmcli` |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
