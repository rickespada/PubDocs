# CPMS Quick Start

> Mirrored from the private `CPMS` source repo's own `QUICKSTART.md`
> (linked from the appliance's Web UI) - manually synced, not automated,
> so it can lag the source after an edit there. One deliberate difference
> from the source: the exact default-credential pattern for an
> unconfigured clone is redacted here and only ever shown live, post-login,
> on the appliance itself (its login banner) - not published publicly.

Build one golden image, seal it, and clone it into as many measurement
endpoints and controllers as you need. Every clone boots to a console menu,
configures itself once, and never again.

CPMS-SPEC.md, the authoritative design document, lives in the private
source repo alongside the code - this is the operational path through it.
Where the two disagree, the spec wins and this file is wrong.

- **Target:** Ubuntu Server 26.04 on VMware
- **Default account:** `perfadmin`
- **Roles:** `endpoint` (measures) or `controller` (orchestrates) — never both

---

## The shape of it

Three states, one image. The file `/etc/cpms/.configured` is the hinge:
sealing removes it, a successful apply writes it, and the first-boot menu
runs only while it is absent (§4.7).

```
  Master VM  ──cpms-seal──>  Sealed image  ──export──>  OVA / template
  provisioned                no hostname                never booted
  and tested                 no addressing              after sealing
                             no host keys
                             no machine-id
                                                              │
                                                            clone
                                                              │
                                                              v
  Live appliance  <──apply──  Setup menu on tty1  <──first boot──
  boots to login              (cpms-firstboot)
```

---

## Build the Master

Provision one VM, prove it works, seal it. Everything below happens on
that single machine before it is ever cloned.

### Size the VM

One image serves both roles, but they want different hardware. Size the
master as an **endpoint** — it is the heavier of the two, and a controller
clone simply leaves capacity unused (spec §2.2).

| Resource | `endpoint` | `controller` | Why |
|---|---|---|---|
| vCPU | 8 | 4 | Keep inside one NUMA node. Disable CPU hot-add — it silently disables vNUMA. |
| RAM | 16 GB | 8 GB | An 8 GB tmpfs lives on the endpoint. |
| Disk | 50 GB | 100 GB+ | Controller grows with result retention. |
| vNIC | 1 (VMXNET3) | 1 (VMXNET3) | Single NIC is the shipped topology. |
| VLAN | test | management | A controller must not generate traffic on the path under test. |
| MTU | 9000 | 1500 | Or the measured path maximum — never above it. |

Three vSphere settings the table doesn't cover (spec §2.3):

- **CPU reservation** — leave at 0 and the appliance competes for CPU with
  every other guest; measurement runs hit 91–186% CPU, so contention shows
  up as interval variance indistinguishable from a network fault. Reserve
  if the numbers matter.
- **Virtualized CPU performance counters** — off by default; without them
  `perf` hardware counters don't work in the guest.
- **CPU hot-add** — leave disabled; it silently disables vNUMA.

CPU governor and SMT are ESXi-host settings, not guest-controllable — don't
look for them in `cpms-provision`.

One NIC is the shipped default; dual-NIC is fully supported for operators
who need management isolation, just not what an unconfigured appliance
assumes (spec §2.1).

### Scripts

Copy the repo to the **master** VM, then install everything on it — all of
it, regardless of role. A clone's role is chosen later, at first boot, and
can change again after that, so the master needs every script no matter
which role a given clone ends up as (spec §3 has the full reasoning).

```bash
for f in cpms-netrollback cpms-pathmtu cpms-quiesce cpms-toolset \
         cpms-setup cpms-firstboot cpms-seal cpms-provision \
         cpms-bench cpms-datastore cpms-orchestrate; do
    sudo install -m 0755 "${f}.sh" "/usr/local/bin/${f}"
done
sudo install -m 0755 cpms-ingest.py /usr/local/bin/cpms-ingest
sudo install -m 0755 cpms_mcp.py    /usr/local/bin/cpms_mcp.py
```

> **If you copied from Windows**, the `.sh` files arrive without an
> executable bit. The `install` commands above are unaffected — they set
> the mode on the destination.

*Planned improvement:* package this as a single tarball instead of copying
the repo file-by-file. Not built yet.

### Initialize

Toolset first, then provision — that's the dependency order. Both are
idempotent: re-running skips what already exists. Budget 10–20 minutes on a
bare VM, most of it apt and the source build of iperf3 3.21. Run it from
the VM console or inside tmux — a dropped SSH session mid-apt leaves a
dpkg lock to clean up.

```bash
sudo cpms-toolset install safe
tmux new -s prov 'sudo cpms-provision --role endpoint 2>&1 | tee /tmp/prov.log'
```

The master provisions as `endpoint` even though it will be cloned into
controllers too — deliberate, not permanent. It front-loads the iperf3
source build so every clone inherits the binary, and a controller clone
converges away the rest once configured (see §05 below; spec §3 has the
full reasoning).

#### The ten stages

| # | Stage | `endpoint` | `controller` |
|---|---|---|---|
| 1 | Base packages | measurement set | core, app, firewall + container runtime |
| 2 | Quiesce background activity | full | package timers only, never its own services |
| 3 | iperf3 source build | yes | skipped |
| 4 | Sysctl + perfprofile | yes | baseline sysctl only |
| 5 | tmpfs ramdisk | yes | removed if present |
| 6 | iperf3 listener | enabled | disabled |
| 7 | perfenv fingerprint | yes | yes |
| 8 | First-boot service | yes | yes |
| 9 | Orchestration hookup | scoped NOPASSWD sudo rule | key directory |
| 10 | Validation | measurement checks | controller checks |

Stage 9 is what lets a controller drive an endpoint later. On an endpoint
it installs `/etc/sudoers.d/cpms-endpoint` so `perfadmin` can run
`cpms-bench`, `cpms-quiesce` and `perfenv` as root without a password —
nothing else. Skip it and controller-driven runs hang on a password
prompt with no terminal to answer it, which looks exactly like a network
fault.

#### Watch three things

- **Stage 3** must end with the pinned build at `/opt/iperf3-3.21`. A
  download warning here is the one failure worth stopping for — the
  provenance argument (spec §1) rests on the pinned version.
- **Stage 6** should name your real interface, not a guessed one.
- **Stage 10** is the verdict. Any `FAILED` line goes into every clone.

```bash
grep -nE '\[!\]|\[x\]|FAILED|could not' /tmp/prov.log
/opt/iperf3-3.21/bin/iperf3 --version | head -1
ss -lnt | grep 5201
```

Expect: no grep output, `iperf 3.21`, and a `LISTEN` on the interface
address — **not** `0.0.0.0`.

#### Smoke test before you commit

Confirm the toolchain runs end to end. This is a tooling check, not a
benchmark — an unconfigured master has no jumbo config, so the number is
not one to keep.

```bash
iperf3-cpms -c <peer-ip> -t 10 -P 4 --get-server-output
```

### Seal the master

Sealing strips identity so clones don't inherit it: hostname, all network
config, machine-id, SSH host keys, CPMS config, results and volatile state
(spec §4.7). Your own `authorized_keys` stays, so `perfadmin` can still
reach a clone once it has an address.

```bash
sudo cpms-seal --dry-run
```

The plan **must** include a `would disable:` line for each
`/etc/netplan/*.yaml` and a `would write:` line for
`99-cpms-disable-network.cfg`. If those are missing, the installed
`cpms-seal` is stale and your clones will come up on DHCP.

> **Snapshot the VM now.** There is no undo. A snapshot taken before sealing
> is the only way back, and the script asks you to type `SEAL` to confirm you
> have one.

```bash
sudo cpms-seal
```

> **Do not boot the master again.** Booting a sealed master regenerates its
> machine-id and host keys — every clone taken afterwards would share them.
> Export the OVA or convert to template straight from the powered-off state.

---

## Clone and First Boot

Clone at least **three**: two endpoints and one controller. Two endpoints on
the same host are what make the same-host calibration run possible (spec
§9, §11 step 4), and nothing downstream is trustworthy until that passes.

Each clone boots straight to the setup menu on its console — that program
owns tty1 the moment it starts, so there is no shell at the physical
console to check anything from. Check over SSH instead.

**A clone may already show an address**, visible in your hypervisor's
guest summary (vCenter, etc.) before you ever touch the console. On this
single-NIC appliance that's expected, not a sign the seal failed: a DHCP
fallback baked into the base image (independent of both netplan and
cloud-init) can hand the interface a transient address before setup runs
(spec §4.7). `perfadmin` can log in as soon as that address appears.

> **Change the default password once the system is configured.** A freshly
> cloned, unconfigured appliance ships with a known default credential for
> `perfadmin`, reachable as soon as its address appears — before an
> operator has secured anything. Run `passwd` at first login.

---

## 05 · Configure

The menu stages every edit and touches nothing until you choose **Apply**.
Role is item 7, asked last — set everything else first, then commit to a
role once the rest of the picture is staged (§10.8).

### Endpoint

1. **1 · Interface addresses** — static, on the test VLAN, with its gateway
   (item 0, NIC mode, is already `single`; skip it)
2. **3 · MTU** — 9000, or the measured path maximum
3. **6 · Hostname** — unique per clone
4. **7 · Appliance role** — `endpoint`, then the controller address that may
   drive it
5. **8 · Review**, then **9 · Apply**

### Controller

1. **1 · Interface addresses** — static, on the management VLAN
2. **3 · MTU** — 1500; jumbo here only risks the path it drives endpoints over
3. **6 · Hostname** — unique per clone
4. **7 · Appliance role** — `controller`; it warns that the listener gets
   disabled
5. **8 · Review**, then **9 · Apply**

> **Applying network changes.** A network change arms a detached 60-second
> rollback before it applies (§4.2). Confirm with Enter on the console, or
> `sudo cpms-setup --confirm` from a new SSH session. Say nothing and the
> previous config returns on its own — which is exactly what you want if the
> change cuts you off. Role, hostname and NTP changes are **not** network
> changes and never arm the timer.

After a role change, Apply asks whether to converge packages and services
right away — `Run it now? [Y/n]:`, default yes. Both roles are fast on a
clone (35s for controller, 15s for endpoint in real timing), since the
one expensive part — the iperf3 source build — already happened once on
the master. Answer `n` and run it yourself later:

```bash
sudo cpms-provision --role endpoint     # or --role controller
```

### Confirm the role took

On a controller:

```bash
ss -lnt | grep -E '5201|5001|5000' # expect nothing
systemctl is-active cpms-iperf3    # expect inactive
systemctl is-active cpms-iperf2    # expect inactive
systemctl is-active cpms-nuttcp    # expect inactive
sudo cpms-quiesce on                # expect a refusal naming the role
```

A controller that still answers on any of these ports is still a
measurement target and would appear in a mesh as an endpoint. `cpms-quiesce`
refuses to run there because the services it stops are the archive and
scheduler that machine exists to run (§2.2).

### Then reboot each clone once

The second boot must go straight to a login prompt. If the setup menu
appears again, nothing was applied — the marker is written on a successful
apply, not on menu exit.

### Verify the clone is independent

Every clone, not just of the master but of **each other** — identical
machine-ids mean clones fight over DHCP leases, and identical host keys are
a real security problem, not just an aesthetic one:

```bash
cat /etc/machine-id
```

```bash
ssh-keyscan -t ed25519 localhost 2>/dev/null
```

Compare across every clone you took from this master. All must differ from
one another.

---

## 05b · Stand up the archive (controller)

The controller carries its own Postgres and Grafana — no external stack
required (§2.2). One command generates credentials, starts both containers
and applies the schema:

```bash
sudo cpms-datastore init
```

It prints the Grafana URL and a generated admin password. **That file is the
only copy** — `/etc/cpms/datastore.env`, mode 0600, not in git, and removed
by `cpms-seal` so every clone generates its own.

```bash
sudo cpms-datastore status
```

Shows container health, schema version and row counts by verdict.

Then load results. `--dry-run` parses without touching the database, which
is the safe way to check a file before committing it:

```bash
cpms-ingest --dry-run /path/to/result.json    # parse only, no database
sudo cpms-ingest /path/to/result.json         # load it
```

`cpms-ingest` reads the datastore credentials itself from
`/etc/cpms/datastore.env`, which is mode 0600 — hence `sudo` for the load,
and no `PGPASSWORD` in your shell history. An external archive still works
the usual way: set `PGHOST`/`PGDATABASE`/`PGUSER`/`PGPASSWORD`, or pass
`--dsn`, and those take precedence.

`cpms-ingest` **refuses a result with no `cpms_env`** — see §06 for why the
merge step is not optional.

> Going air-gapped? Run `sudo cpms-datastore pull` first, then move the
> images with `docker save` / `docker load`. There is no registry access
> once the controller is isolated.

---

## 05c · Enrol the endpoints (controller)

This is what lets the controller drive tests. **The controller still never
measures** — it holds the key, decides what runs where, pushes the command
to an endpoint, and files the result. The traffic is generated entirely on
the endpoint (§2.2, §10.10).

Do this once per endpoint, from the controller.

```bash
sudo cpms-orchestrate keygen
```

Creates `/etc/cpms/id_cpms`, root-only, no passphrase — the controller runs
unattended, and a passphrase stored beside the key protects nothing. Run it
once. Regenerating orphans every endpoint already enrolled.

```bash
sudo cpms-orchestrate enroll 192.168.65.146
```

`ssh-copy-id` asks for the `perfadmin` password **once**. That is the only
time a password is needed. Enrol then does three things worth knowing:

- **Verifies before recording.** The host is only written to the registry
  after a key login actually succeeds and returns its hostname. A row that
  does not answer is worse than no row — it keeps getting picked, and every
  failure looks like a network problem.
- **Applies `restrict`** to its own line in `authorized_keys`, so the key
  runs commands and cannot open a shell or tunnel through the endpoint. Your
  own keys in that file are untouched; re-enrolling is idempotent.
- **Enrols a controller as `target-only`.** A controller is a legitimate
  thing to measure *against* and must never be scheduled as a source.

Repeat for each endpoint, then confirm the fleet:

```bash
sudo cpms-orchestrate check
```

```
  [+] cpms-endpoint01        ready
  [+] cpms-endpoint02        ready
  [!] cpms-endpoint03        reachable, missing: cpms-bench
```

`ready` means reachable **and** carrying `cpms-bench`, `perfenv` and
`iperf3`. Fix a `missing:` line before scheduling anything against it.

### Running a test from the controller

Dry run first — it prints the plan, takes no lock and touches no host:

```bash
sudo cpms-orchestrate run --source cpms-endpoint01 --target 192.168.65.252 --matrix quick --dry-run
```

Then for real:

```bash
sudo cpms-orchestrate run --source cpms-endpoint01 --target 192.168.65.252 --matrix baseline
```

The endpoint's output streams back live. When it finishes, results are
fetched to `/var/lib/cpms/incoming/<timestamp>/` and ingested automatically.

**`--source` must be an enrolled endpoint.** Anything else is refused rather
than quietly run on the controller.

### The global lock

`run` holds a single fleet-wide lock for the whole remote execution. A
measurement saturates a link; two at once corrupt each other's numbers with
no error anywhere — both simply report a plausible lower figure. A second
`run` started while one is in flight is refused and tells you who holds it.

The lock lives in the archive with an expiry, so a killed run cannot wedge
the appliance permanently. If one dies badly:

```bash
sudo cpms-orchestrate unlock
```

Only when the run is genuinely dead. Releasing it under a live run makes
both that run's numbers and the next one's suspect.

> **After sealing and cloning a controller**, its key is gone from the image
> by design — a clone carrying the master's key could log into every
> endpoint the master ever enrolled. Cloned *endpoints* likewise lose the
> controller's key. Re-run `keygen` and `enroll`. This is not a bug.

---

## 06 · Running measurements

Measure the path before you trust a number from it, and silence the machine
while you do.

```bash
cpms-pathmtu 192.168.65.65     # true path MTU, by binary search
cpms-quiesce check             # what would interfere; changes nothing
cpms-quiesce baseline          # snapshot NIC counters
sudo cpms-quiesce run -- iperf3-cpms -c <peer> -t 60 -P 4 -O 10 -J \
     --get-server-output > /mnt/ramdisk/run.json
perfenv --merge /mnt/ramdisk/run.json --iface ens33
```

`run` baselines counters, quiesces, runs the command as the invoking user,
restores everything, then reports the counter delta on **stderr** — so the
redirect above captures clean JSON. Read the delta, not the lifetime
totals; those accumulate from boot (§4.6).

Two arguments carry more weight than they look:

- **`perfenv --merge` is not optional.** Without it the file is a bare
  iperf3 document with no record of the kernel, NIC, MTU, ring sizes,
  congestion control or clock state it was taken under — and a throughput
  number without those is not reproducible. Merge immediately, before
  changing any profile: `perfenv` reads the *live* system, so a fingerprint
  taken later describes a different machine.
- **`-O 10`** omits slow-start, whose intervals sit far off the steady-state
  rate. Included, they dominate the variance. Every figure in spec §5 was
  taken with it, so a run without it is not comparable to them.

| Counter | Meaning | Verdict |
|---|---|---|
| `rx_errors` | Malformed / oversized frames | **Invalidates the run** |
| `rx_oob` | Receive ring could not absorb the offered rate | **Invalidates the run** |
| `rx_missed_errors` | Ring descriptors exhausted — on other drivers | **Invalidates the run**, but vmxnet3 never populates it |
| `rx_dropped` alone | No protocol handler: LLDP, STP, mDNS, IPv6 RA | Harmless |

`rx_oob` is the one that fires on this platform; `rx_missed_errors` stays 0
on vmxnet3 no matter how starved the ring is, which is why the tooling reads
OOB instead (§4.6).

**Do not respond to `rx_oob` by raising the ring buffers.** Spec §2.3
rejected `ethtool -G` on measurement: at the driver defaults, a full 60 s
line-rate run to the physical 10G target produced an OOB delta of **zero**,
so a larger ring has nothing to recover — and the recommendation was
withdrawn upstream for kernels ≥ 6.8, while this appliance runs 7.0. Above
roughly 10 Gbps — a same-host VM pair will reach 30+ — OOB simply means the
offered rate exceeded what the receive path can absorb. That is a fact about
the test, not a fault. Raise rings only against a non-zero OOB delta on a
*real* path, never against a lifetime total.

### UDP loss tests

```bash
sudo cpms-quiesce run -- iperf3-cpms -c <peer> -u -b 2G -l 8972 --dont-fragment \
     -t 30 -J --get-server-output > /mnt/ramdisk/udp-2g.json
perfenv --merge /mnt/ramdisk/udp-2g.json --iface ens33
```

- **`-l 8972`** is the correct jumbo payload (9000 − 20 IP − 8 UDP). iperf3
  warns that it exceeds the *TCP* MSS; that comparison is irrelevant for UDP
  and the warning can be ignored.
- **`--dont-fragment`** makes the MTU assumption enforced rather than
  assumed. Without it a datagram needing fragmentation is fragmented
  silently and the loss figure measures something else.
- **Ramp the rate.** A single UDP run is not a measurement: two runs at
  5 Gbps on the same path measured 3.6% and 4.9% loss. Sweep
  1G/2G/5G/8G and look for the knee.

> **The counter delta is describing the wrong host.** `cpms-quiesce` samples
> the machine it runs on — the *sender* for a forward test — which drops
> nothing. The receiver is where UDP loss happens. A UDP run reporting
> `clean` counters has said nothing about the receiving end; check that host
> separately with `ethtool -S <iface> | grep OOB` until the controller can
> collect both.

**Regression floor (§5):** on the jumbo path a healthy endpoint pair
sustains **≥ 9.8 Gbps** with a coefficient of variation around **1%**.
Persistent per-stream imbalance at 9000 MTU points at vCPU scheduling or RSS
queue distribution, not the fabric.

**Stream count decides stability, not just speed.** On a CPU-bound path
(two VMs on one host, no wire) single-stream measured *higher* than eight
— 37.9 vs 35.3 Gbps — but at **CV 18.7%** against **2.0%**. A metric that
swings 19% cannot detect a 10% regression however often you repeat it. Use
`-P 4`/`-P 8` for anything tracked over time; keep `-P 1` for diagnosis. On
a link-bound path both sit at line rate and the effect disappears.

---

## 07 · When something looks wrong

Every entry below was hit during real deployment, not imagined.

| Symptom | Cause | Fix |
|---|---|---|
| Console dead at `Reached target graphical.target`; SSH and ping fine | An old first-boot unit carrying `Conflicts=getty@tty1.service`. On the second boot the unit is skipped by its condition, but getty was already dropped from the transaction. | `sudo systemctl start getty@tty1`, then re-run provision stage 8 for the corrected unit. |
| Clone comes up with a DHCP address | Stale `cpms-seal`. A master never taken through `cpms-setup` keeps its install-time netplan, which older seals left in place. | Install the current `cpms-seal`, confirm the dry run lists *would disable* for each `*.yaml`, re-seal. |
| Clone configures both NICs; two default routes | An old build defaulted `CPMS_NIC_MODE` to `dual` (DEFECT-7). | Current builds default to `single`. Confirm with `sudo cpms-setup --gen-netplan` — one interface stanza, one default route. |
| Listener bound to `0.0.0.0:5201` or `:5001` | The configured test interface does not exist on this host, so `cpms-iperf3`/`cpms-iperf2` fell back to every address. `cpms-nuttcp` always binds every interface (no equivalent flag exists for it) - that one is expected, not a symptom. | Re-run `sudo cpms-provision --role endpoint`; it resolves the interface from applied config and warns on a mismatch. |
| Setup menu appears on a working appliance | `/etc/cpms/.configured` is absent — the box was provisioned but never had a config applied. | Run `sudo cpms-setup` and apply. Nothing suppresses the menu except a real apply. |
| Locked out after a network change | Expected, and handled. | Do nothing for 60 seconds — the detached rollback restores the previous config. Or `sudo cpms-setup --rollback`. |
| `sudo: cannot execute ./cpms-*.sh: Permission denied` | Missing executable bit after a copy from Windows. | Use the installed `/usr/local/bin` names, or `chmod +x *.sh`. |

---

## 08 · Command reference

| Command | Does |
|---|---|
| `cpms-setup` | Console menu — role, NIC mode, addresses, routing, MTU, DNS, NTP, hostname |
| `cpms-setup --show` | Print the current config file |
| `cpms-setup --role <r>` | Set role non-interactively; no network change, no rollback timer |
| `cpms-setup --confirm` | Confirm a network change from a second session, cancelling rollback |
| `cpms-setup --rollback` | Restore the most recent netplan backup |
| `cpms-setup --reset` | Discard staged edits, re-read the live system |
| `cpms-setup --gen-netplan` | Print the netplan that would be generated |
| `cpms-provision --role <r>` | Run all ten stages for a role; idempotent |
| `cpms-provision --stage N` | Run one stage |
| `cpms-provision --list` | List the stages |
| `cpms-provision --dry-run` | Show what would happen |
| `cpms-toolset list` | Tool groups and their install state |
| `cpms-toolset install` | Install every diagnostic tool group (the default) |
| `cpms-quiesce check` | Report what would interfere; changes nothing |
| `cpms-quiesce run -- <cmd>` | Baseline, quiesce, run, restore, report counter deltas |
| `cpms-pathmtu <target>` | True path MTU by binary search, DF set on every probe |
| `perfprofile show` | Current congestion control and buffer profile |
| `perfenv --iface <if>` | Environment fingerprint as JSON — embed it in every result |
| `cpms-seal --dry-run` | Print the exact seal manifest, change nothing |
| `cpms-seal` | Strip identity and power off. No undo. |

**Endpoint only**

| Command | Does |
|---|---|
| `cpms-bench --list` | Every matrix and what each is for |
| `cpms-bench --target <t> --matrix baseline` | Run a matrix both directions, quiesced, fingerprinted |
| `cpms-bench --target <t> --matrix iso --capture` | Header-only pcap alongside a low-rate matrix (spec 7 P4) |
| `cpms-bench --target <t> --matrix idle --port 443` | TCP session-idle-timeout discovery ladder (spec 7 P4) |
| `cpms-bench --target <t> --dry-run` | Show the plan, run nothing |

**Controller only**

| Command | Does |
|---|---|
| `cpms-datastore init` / `up` / `status` | Postgres + Grafana lifecycle |
| `cpms-ingest --dry-run <file>` | Parse a result, load nothing |
| `cpms-ingest <file>` | Load a result into the archive |
| `cpms-orchestrate keygen` | Create the controller's SSH key. Once. |
| `cpms-orchestrate enroll <host>` | Push the key, verify, register |
| `cpms-orchestrate hosts` | The registry — who is enrolled, and as what |
| `cpms-orchestrate check [host]` | Reachable, and carrying its toolchain? |
| `cpms-orchestrate run --source <h> --target <t>` | Run on an endpoint, fetch, ingest |
| `cpms-orchestrate unlock` | Release a lock left by a dead run |

### Files worth knowing

| Path | Holds |
|---|---|
| `/etc/cpms/setup.conf` | Staged intent — what you have asked for |
| `/etc/default/cpms` | Applied state — what the machine actually is |
| `/etc/cpms/.configured` | Written on a successful apply; suppresses the first-boot menu |
| `/etc/cpms/backups/<ts>/` | Pre-apply netplan backups |
| `/etc/netplan/60-cpms.yaml` | The only live network config; others renamed `.cpms-disabled` |
| `/mnt/ramdisk` | 8 GB tmpfs — captures and staging, so the VMDK never enters the measurement path |
| `/var/log/cpms-provision.log` | Full provisioning detail |
| `/etc/cpms/id_cpms` | Controller's SSH key, root-only. Removed by sealing. |
| `/etc/cpms/known_hosts` | Host keys learned at enrolment; a changed key fails the connection |
| `/etc/sudoers.d/cpms-endpoint` | Endpoint: NOPASSWD on three measurement commands, nothing else |
| `/var/lib/cpms/incoming/<ts>/` | Controller: results fetched from endpoints |

---

## 09 · Ask the archive in plain language (controller)

Optional. An MCP server that lets an LLM client query the archive, explain a
verdict, and start a run. **No Node, no `npx` running on the controller
host directly** — this is Python in a venv. A client that needs `npx` to
reach it (below) runs that on the client machine, not here. (The Web UI's
frontend, §09b, does use Node — but only inside an ephemeral Docker build
stage that never touches the controller's own OS; §1's "minimal host
package set" intent survives, it just no longer means "no Node exists
anywhere in this project.")

```bash
sudo apt-get install -y python3-venv
sudo python3 -m venv /opt/cpms-mcp
sudo /opt/cpms-mcp/bin/pip install mcp psycopg2-binary
sudo install -m 0755 cpms_mcp.py /usr/local/bin/cpms_mcp.py
sudo cpms-provision --stage 9
```

That last line matters. The server reads `/etc/cpms/datastore.env`, which is
mode 0600, and an MCP client launches it over a **stdio pipe with no TTY** —
a sudo password prompt there cannot be answered and the client reports it as
the server crashing. Stage 9 installs `/etc/sudoers.d/cpms-controller`
granting `perfadmin` that one exact command line. The alternative was putting
the archive password into the client's config file, which moves a
root-equivalent credential onto a laptop in plaintext.

Prove it before wiring any client — this exercises the import, the venv, the
credentials read and the SQL in one line:

```bash
sudo /opt/cpms-mcp/bin/python -c "import sys; sys.path.insert(0,'/usr/local/bin'); import cpms_mcp; print(cpms_mcp._query('SELECT count(*) AS runs FROM cpms_run'))"
```

Expect a row count. Everything except the transport is then confirmed.

### Connecting a client

The command is always the same; only the wrapper changes with where the
client runs.

| Client runs | Launch command |
|---|---|
| On the controller | `sudo /opt/cpms-mcp/bin/python /usr/local/bin/cpms_mcp.py` |
| Anywhere else | `ssh perfadmin@<controller> sudo /opt/cpms-mcp/bin/python /usr/local/bin/cpms_mcp.py` |

**Claude Code:**

```bash
claude mcp add cpms -- sudo /opt/cpms-mcp/bin/python /usr/local/bin/cpms_mcp.py
```

**LM Studio** (and anything else using the Claude Desktop config shape) — add
to its `mcp.json`, which LM Studio edits from its integrations panel:

```json
{
  "mcpServers": {
    "cpms": {
      "command": "ssh",
      "args": [
        "perfadmin@192.168.65.250",
        "sudo", "/opt/cpms-mcp/bin/python", "/usr/local/bin/cpms_mcp.py"
      ]
    }
  }
}
```

The MCP pipe has no TTY, so **anything ssh wants to ask is fatal** — a
password prompt, or the first-connection host-key question. Both make the
server appear to crash instantly with no error, because ssh dies before
Python ever starts. Two things must be true on the client machine first.

**1. The controller's host key must already be trusted.** Connect once by
hand and answer `yes`:

```
ssh perfadmin@<controller> true
```

**2. Key-based login must work without a password.** On Windows there is no
`ssh-copy-id` — OpenSSH for Windows does not ship it — so append the key
yourself:

```
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh perfadmin@<controller> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

(that `type` line is PowerShell; from Git Bash use `cat ~/.ssh/id_ed25519.pub | ssh ...`)

Then verify it is truly non-interactive — this must print `ok` with no
prompt of any kind:

```
ssh -o BatchMode=yes perfadmin@<controller> echo ok
```

`BatchMode=yes` makes ssh fail instead of prompting, which is exactly the
condition the MCP pipe runs under. If that command works, the transport
will too.

### If the client says the server disconnected immediately

A start-then-close inside a few hundred milliseconds, with no stderr in the
log, means the failure happened before Python ran. Test the pipe end to end
by hand — this sends a real MCP handshake:

```
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' | ssh -o BatchMode=yes perfadmin@<controller> sudo /opt/cpms-mcp/bin/python /usr/local/bin/cpms_mcp.py
```

A JSON reply naming `cpms_mcp` means the whole chain works and the problem
is in the client's config. Anything else — a password prompt, `Host key
verification failed`, `sudo: a terminal is required` — names the step to fix
and none of them are the server.

### Claude Desktop on Windows: use the network transport, not ssh

Claude Desktop spawns MCP servers from its bundled Node with no console
attached (Electron's `windowsHide`). Win32-OpenSSH's `ssh.exe` crashes in
that environment — exit 255, zero output, in well under 100ms, before it
opens a connection. This is not fixable from the client config; every
variant of the `ssh` launch above will hit it. Use the `streamable-http`
transport instead, which is exactly what the other Node-based MCP servers
in a Claude Desktop config already use.

Stage 9 (`sudo cpms-provision --stage 9`) starts `cpms-mcp.service`
automatically once the venv exists, listening on `:8765`, with a bearer
token generated into `/etc/cpms/mcp.env`:

```bash
sudo cat /etc/cpms/mcp.env
# CPMS_MCP_TOKEN=...
```

Add to `claude_desktop_config.json` (Windows: `%APPDATA%\Claude\`):

```json
{
  "mcpServers": {
    "cpms": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "http://<controller>:8765/mcp",
        "--allow-http",
        "--header",
        "Authorization:${AUTH_HEADER}"
      ],
      "env": {
        "AUTH_HEADER": "Bearer <token from mcp.env>"
      }
    }
  }
}
```

Fully quit and relaunch Claude Desktop after editing the config — it only
reads it at startup. Confirm it worked by asking Claude to call
`cpms_archive_status`; a schema-version table back means the whole chain
is live.

The token is generated once and never regenerated by a re-provision, so
editing the config again is not needed after every `cpms-provision` run.
Without `CPMS_MCP_TOKEN` set, the server refuses to bind to anything but
loopback — `cpms_run_benchmark` is not read-only, and an unauthenticated
listener on the management VLAN is a worse trust boundary than the SSH
pubkey auth the stdio path relies on.

### The tools

Nine read, two write. Grew from nine total (eight read, one write) building
the Web UI (§09b) — `cpms_list_catalog`/`cpms_set_task_enabled` were added
2026-09-02 so the catalog/scheduler had the same tool-surface access as
everything else.

| Tool | |
|---|---|
| `cpms_archive_status` | Schema version, run counts, data-quality problems |
| `cpms_list_runs` | Filterable, paginated; each row carries its conditions |
| `cpms_get_run` | One run in full, with its fingerprint |
| `cpms_compare_runs` | Side by side **and whether they are comparable at all** |
| `cpms_explain_verdict` | Walks §10.6 and shows which condition matched |
| `cpms_check_regression` | Against the §5 floor, reporting MTU alongside |
| `cpms_list_hosts` | Who may be a measurement source |
| `cpms_list_catalog` | The §10.4 catalog's tasks, joined against live schedule state |
| `cpms_plan_benchmark` | Plain-language goal → the right matrix, with reasoning |
| `cpms_run_benchmark` | **Not read-only.** Takes the lock, saturates a link. |
| `cpms_set_task_enabled` | **Not read-only.** Enables/pauses a scheduled task — never starts a benchmark itself |

Start with `cpms_archive_status`. If it answers, the wiring is right.

`cpms_compare_runs` is the one worth having. An LLM handed raw SQL rows will
happily attribute a throughput difference to the network when the two runs
had different MTU, congestion control, kernel, tool version or NIC ring
sizes. It refuses to let that pass silently, and flags runs whose CV puts
the mean beyond trust (§9) or whose NIC counters invalidate them (§4.6).

Leave `cpms_run_benchmark` until the read tools work.

---

## 09b · The Web UI (controller)

Added 2026-09-02, verified live against the real controller, both
endpoints, and a real LLM — see CPMS-SPEC.md §10.7 (private source repo). Two
containers, already part of `docker/compose.yaml`, so `cpms-datastore
up`/`down`/`status` already manage them alongside Postgres/Grafana — no
separate lifecycle commands to learn.

```bash
sudo cpms-provision --role controller --stage 1   # stages docker/backend, docker/frontend, catalog.json
sudo docker compose --env-file /etc/cpms/datastore.env \
    -f /usr/local/share/cpms/docker/compose.yaml build backend frontend
sudo cpms-datastore up
```

Reach it at `http://<controller-address>:3001`. Five pages — Hosts,
Catalog, Run, History, Compare — each a thin view over the same
`cpms_mcp` tools §09 already described, plus a **Chat** tab wired to an
LLM (LM Studio or Ollama, OpenAI-compatible) with every one of those tools
available as a function call, so answers are grounded in the real archive
rather than guessed.

**To enable the chat tab**, set the LLM's address via `cpms-setup` —
menu item 7 (Appliance role), controller role, the new "LLM base URL"
prompt — then **apply** (menu item 9; setting it in item 7 alone only
stages it). Blank leaves the chat tab disabled; the other four pages don't
need it.

The scheduler runs inside the backend container automatically once a
task is enabled — no separate command. `cpms_set_task_enabled` (§09, or
the Catalog page's own toggle) is the only way to turn one on; it takes
effect within about 30 seconds, not immediately.

---

## 10 · Firewall and port reference

Every port CPMS listens on, by role. Endpoint measurement ports are
**endpoint ↔ endpoint** traffic only — the controller never generates
test traffic itself (spec §2.2) and never needs inbound access to any of
them. Where a range is listed for OWAMP/TWAMP, that is the ephemeral
per-session UDP test-packet range negotiated over the TCP control
connection, not a single fixed port — the whole range needs to be open,
not just the control port.

**Endpoint role**

| Port | Proto | Service | Notes |
|---|---|---|---|
| 5201 | tcp | `iperf3-cpms` | `quick`/`baseline`/`scaling`/`udp-ramp` matrices |
| 5001 | tcp | `iperf2` (TCP) | `owd` matrix (`cpms-iperf2-client`) |
| 5001 | udp | `iperf2` (UDP) | `iso` matrix |
| 5000 | tcp | `nuttcp` control | `nuttcp` matrix |
| ~5101 | tcp | `nuttcp` data | Dynamically negotiated over the control channel — confirmed on the real binary, not `-p`'s documented default; don't assume this stays fixed across nuttcp versions |
| 443 | tcp | TLS echo listener | `idle` matrix (configurable via `--port`, default 443) |
| 861 | tcp | OWAMP-Control | Experimental (CLAUDE.md, 2026-09-02) — not installed by `cpms-provision` yet, built from source and verified live cross-host |
| 8760–9960 | udp | OWAMP test traffic | Per-session range from the sample `owamp-server.conf`, negotiated over 861/tcp |
| 862 | tcp | TWAMP-Control | Same experimental status as OWAMP above — confirmed live, `twampd` |
| 18760–19960 | udp | TWAMP test traffic | Per-session range from the sample `twamp-server.conf`, negotiated over 862/tcp |

**Controller role**

| Port | Proto | Service | Notes |
|---|---|---|---|
| 8765 | tcp | `cpms_mcp.py` (MCP server) | Bound to all interfaces; token-authenticated (`/etc/cpms/mcp.env`) |
| 3001 | tcp | Web UI (nginx/frontend) | Management VLAN, all interfaces — the operator's browser entry point |
| 3000 | tcp | Grafana | Management VLAN, all interfaces |
| 5432 | tcp | Postgres | **Loopback only** (`127.0.0.1:5432` in `docker/compose.yaml`) — never needs a firewall rule |
| 8081 | tcp | Web UI backend (FastAPI) | **Loopback only** (`127.0.0.1:8081`) — nginx proxies to it, no external rule needed |
| 123 | udp | `chrony` (NTP server mode) | Bound to all interfaces (`allow all` in `/etc/chrony/conf.d/99-cpms.conf`, written and `chrony` restarted whenever settings are applied on a controller) — network segmentation is the boundary, same trust model as the MCP server above. Endpoints sync from the controller (CPMS-SPEC.md's OWAMP/TWAMP section: same-VLAN sync measured tighter than the public pool) |

**Both roles**

| Port | Proto | Service | Notes |
|---|---|---|---|
| 22 | tcp | SSH | Controller→endpoint orchestration (`cpms-orchestrate`, key at `/etc/cpms/id_cpms`); also how this repo's changes reach any VM (§ "Deploying a change to the VMs" in CLAUDE.md) |

**Not a fixed CPMS port**: the chat tab's LLM endpoint (`CPMS_LLM_URL`,
`cpms-setup` menu item 7) is whatever the operator's LM Studio/Ollama
host actually listens on (commonly 1234 for LM Studio's OpenAI-compatible
API) — outbound from the controller only, nothing CPMS itself opens.

---

## Not built yet

- **A full endpoint-to-endpoint mesh.** The scheduler resolves a catalog
  task's `group` to exactly one source→target pair (the group's first two
  hosts) — right for the two-endpoint case this appliance ships with, not
  yet a general N-host mesh.
- **Live streaming status for a real (non-dry-run) Run-page submission.**
  `cpms_run_benchmark` is one blocking call, not itself a stream; check
  History if a submission outlasts the page's own request.
- **Per-run interval charts in the Web UI.** History's chart is a
  throughput/CV trend *across* runs, not the per-second-*within*-one-run
  view Grafana's "Run Detail" dashboard already has — no `cpms_mcp` tool
  exposes `cpms_interval` rows yet.

---

Anything touching netplan, systemd, sysctl or a real NIC can only be verified
on a live VM. Syntax-checked is not verified.
