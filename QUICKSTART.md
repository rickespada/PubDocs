# CPMS Quick Start

Build one master image, seal it, and clone it into endpoints and controllers.
Each clone boots to a setup menu, configures itself once, and then never shows
the menu again.

- **Target platform:** Ubuntu Server 26.04 on VMware
- **Default account:** `perfadmin`
- **Roles:** `endpoint` (measures) or `controller` (orchestrates). A single
  appliance cannot be both.

---

## Terms and Abbreviations

| Term | Meaning |
|---|---|
| MTU | Maximum Transmission Unit — the largest frame the network path carries without fragmenting |
| NIC | Network Interface Card |
| VLAN | Virtual Local Area Network |
| NUMA | Non-Uniform Memory Access |
| SSH | Secure Shell |
| DHCP | Dynamic Host Configuration Protocol |
| NTP | Network Time Protocol |
| OVA | Open Virtualization Appliance |
| TLS | Transport Layer Security |
| TCP | Transmission Control Protocol |
| UDP | User Datagram Protocol |
| RTT | Round-Trip Time |
| CV | Coefficient of Variation |
| BDP | Bandwidth-Delay Product |
| MCP | Model Context Protocol |
| NDT7 | Network Diagnostic Tool version 7 |
| OWAMP | One-Way Active Measurement Protocol (RFC 4656) |
| TWAMP | Two-Way Active Measurement Protocol (RFC 5357) |
| OWD | One-Way Delay |
| tmpfs | Temporary filesystem stored in RAM |

---

## How It Works

Three states, one image. The file `/etc/cpms/.configured` controls which
state the appliance is in. Sealing removes it. A successful apply writes it.
The first-boot menu runs only while the file is absent.

```
  Master VM  ──cpms-seal──>  Sealed image  ──export──>  OVA / template
  provisioned                no hostname                never booted
  and tested                 no addressing              after sealing
                             no SSH host keys
                             no machine-id
                             no CPMS config
                                                              │
                                                            clone
                                                              │
                                                              v
  Live appliance  <──apply──  Setup menu on tty1  <──first boot──
  boots to login              (cpms-firstboot)
```

---

## 01 · Build the Master

Provision one VM, verify it works, then seal it. Every step in this section
runs on that single machine.

### Size the VM

The master provisions as an endpoint. An endpoint is the larger of the two
roles. A controller clone leaves the extra capacity unused.

| Resource | `endpoint` | `controller` | Reason |
|---|---|---|---|
| vCPU | 8 | 4 | Keep all CPUs inside one NUMA node. Disable CPU hot-add — it silently disables vNUMA. |
| RAM | 16 GB | 8 GB | The endpoint runs an 8 GB tmpfs for result staging. |
| Disk | 50 GB | 100 GB or more | The controller grows as it stores results. |
| vNIC | 1 (VMXNET3) | 1 (VMXNET3) | One NIC is the standard configuration. |
| VLAN | test | management | The controller must not send traffic on the path under test. |
| MTU | 9000 | 1500 | Or the maximum the measured path supports. Never exceed the path maximum. |

Three VMware settings affect measurement quality:

1. **CPU reservation** — Leave at 0 and the appliance competes for CPU with
   other guests. Measurement runs use 91–186% CPU. Competition from other
   guests appears as interval variance that looks like a network problem.
   Reserve CPU if the numbers must be reliable.
2. **Virtualized CPU performance counters** — Off by default. Without them,
   the `perf` hardware counters do not work inside the guest.
3. **CPU hot-add** — Leave disabled. It silently disables vNUMA.

One NIC is the standard configuration. Dual-NIC is fully supported for
operators who need management isolation. An unconfigured appliance assumes
one NIC.

### Get the Repo onto the VM

Two methods land the repo at `~/cpms` on the master. All steps below assume
that directory.

**Method A: Download a ZIP file**

This method does not require git or SSH keys on the VM. The repo is private,
so you need a GitHub session already authenticated in your browser.

1. On github.com: select **Code**, then select **Download ZIP**.
2. Transfer the ZIP file to the VM. Use `scp` or WinSCP. Copy it to
   `/tmp/CPMS-main.zip`.
3. On the VM, run:
   ```bash
   sudo apt install unzip
   cd /tmp && unzip CPMS-main.zip
   mv CPMS-main ~/cpms
   find ~/cpms -maxdepth 1 -name '*.sh' ! -perm -u+x -exec chmod +x {} \;
   cd ~/cpms
   ```
   **Note:** GitHub nests the repo inside a `<repo>-<branch>` folder (for
   example, `CPMS-main` for a ZIP of the `main` branch). The `mv` command
   removes that extra level. The ZIP does not preserve the executable bit, so
   the `find`/`chmod` step is required every time.
4. Verify the transfer. Compare checksums between your local copy and the VM:
   ```bash
   # On your local machine:
   md5sum cpms-provision.sh cpms-setup.sh sql/schema.sql catalog/catalog.json CLAUDE.md
   # On the VM:
   cd ~/cpms && md5sum cpms-provision.sh cpms-setup.sh sql/schema.sql catalog/catalog.json CLAUDE.md
   ```
5. Confirm the executable bit is set on every script:
   ```bash
   find ~/cpms -maxdepth 1 -name '*.sh' ! -perm -u+x
   ```
   Expect no output. No output means every script is executable.

**Why Method A is preferred:** A ZIP download leaves no `.git` directory
on the VM. The VM's `~/cpms` directory is an `scp`-only staging area by
design: the repo is private, and VMs must not hold git credentials that
could cache a pull or clone. A stale `.git` state on a VM (checked in
wrong commit, diverged by ad-hoc edits) is a known source of drift.
The ZIP method has none of that surface.

**Method B: Direct scp or WinSCP**

This method works when pushing individual file updates to an
already-provisioned VM. When copying from Windows, `scp` and WinSCP
silently remove the executable bit from every `.sh` file. Run the same
`find`/`chmod` sweep from Method A step 3 after every transfer.

### Install the Scripts

Install all scripts on the master regardless of role. The clone's role is set
later, at first boot. The master must carry every script so any clone can
become any role.

Run this from inside `~/cpms`:

```bash
for f in cpms-netrollback cpms-pathmtu cpms-quiesce cpms-toolset \
         cpms-setup cpms-firstboot cpms-seal cpms-provision \
         cpms-bench cpms-datastore cpms-orchestrate; do
    sudo install -m 0755 "${f}.sh" "/usr/local/bin/${f}"
done
sudo install -m 0755 cpms-ingest.py /usr/local/bin/cpms-ingest
sudo install -m 0755 cpms_mcp.py    /usr/local/bin/cpms_mcp.py
```

**Note:** When copying from Windows, the `.sh` files arrive without an
executable bit. The `install` commands above set the mode on the destination
directly and are not affected by that.

### Initialize

Run `cpms-toolset` first. Then run `cpms-provision`. Both steps are
idempotent: running them again skips what already exists. Budget 10–20 minutes
on a bare VM. Most of that time is `apt` package installation and the iperf3
3.21 source build. Run from the VM console or inside a `tmux` session. A
dropped SSH session mid-`apt` leaves a `dpkg` lock that must be cleared
manually.

```bash
sudo cpms-toolset install
tmux new -s prov 'sudo cpms-provision --role endpoint 2>&1 | tee /tmp/prov.log'
```

The master provisions as an endpoint even though you will clone it into
controllers too. That is deliberate. It builds the iperf3 binary now so every
clone inherits it. A controller clone removes the endpoint-specific services
when it provisions for its own role.

#### The Fourteen Stages

| # | Stage | `endpoint` | `controller` |
|---|---|---|---|
| 1 | Base packages | Measurement tool set | Core packages, app packages, firewall, and container runtime |
| 2 | Quiesce background activity | Full quiesce | Package update timers only — never the archive or scheduler |
| 3 | iperf3 source build | Yes — installs to `/opt/iperf3-3.21` | Skipped |
| 4 | Sysctl and cpms-perfprofile | Yes | Baseline sysctl only |
| 5 | tmpfs ramdisk | Yes — 8 GB at `/mnt/ramdisk` | Removes it if present |
| 6 | Measurement listeners | Enabled (iperf3, iperf2, nuttcp) | Disabled |
| 7 | cpms-perfenv fingerprint | Yes | Yes |
| 8 | First-boot setup service | Yes | Yes |
| 9 | Orchestration hookup | Installs `/etc/sudoers.d/cpms-endpoint`: `perfadmin` may run `cpms-bench`, `cpms-quiesce`, `cpms-perfenv`, and `cpms-perfprofile` as root without a password. Nothing else. | Creates `/etc/cpms/` and `/var/lib/cpms/incoming`. Installs `/etc/sudoers.d/cpms-controller` (MCP token reader rule). Disables `cpms-ndt7.service` if present (convergence from endpoint role). Runs `ensure_mcp_token`. Runs `ensure_prometheus_targets`. Calls `cpms-datastore init` via `bootstrap_datastore`. Installs and enables `cpms-mcp.service`. Runs `cpms-orchestrate keygen`. Installs `cpms-ndt7-fetch.timer`. |
| 10 | OWAMP/TWAMP source build | Enabled (ports 861 and 862) | Disabled |
| 11 | NDT7 source build | Enabled (ports 443 and 80) | Disabled |
| 12 | Observability (node_exporter) | Yes | Yes |
| 13 | Local storage-test disk | Formats and mounts a dedicated ext4 disk at `/dev/sdb` (added in the hypervisor). Soft-warns and continues if the disk has not been added yet. | Skipped |
| 14 | Validation | Measurement checks | Controller checks |

**Note about stage 9 (endpoint):** Without the sudoers rule, controller-driven
runs hang at a password prompt with no terminal to answer it. That looks
exactly like a network problem.

#### Check Five Things

1. **Stage 3** must end with the pinned build at `/opt/iperf3-3.21`. A
   download warning here is the one failure worth stopping for. The accuracy
   argument rests on the pinned version.
2. **Stage 6** must name your real interface, not a guessed one.
3. **Stage 10** builds OWAMP and TWAMP from GitHub source. Stage 10 requires
   external network access to `github.com`. It fails soft (a `[!]` warning,
   not `FAILED`) if `github.com` is not reachable. Re-run `--stage 10` once
   the network path is open.
4. **Stage 11** builds NDT7 from GitHub source. The build also requires Go
   1.25. It fails soft the same way as stage 10.
5. **Stage 14** is the verdict. Any `FAILED` line goes into every clone you
   make from this master.

```bash
grep -nE '\[!\]|\[x\]|FAILED|could not' /tmp/prov.log
/opt/iperf3-3.21/bin/iperf3 --version | head -1
ss -lnt | grep 5201
```

Expect: no `grep` output, `iperf 3.21`, and a `LISTEN` on the interface
address — not `0.0.0.0`.

#### Cache the Controller MCP Venv

The MCP server's Python virtual environment (`/opt/cpms-mcp`) normally builds
the first time a clone provisions as a controller. That step requires PyPI
access from whatever network that clone lives on. Build it now, while this
master already has confirmed internet access. Every future controller clone
then skips straight to starting `cpms-mcp.service`.

```bash
sudo cpms-provision --build-mcp-venv
```

**Note:** This is not the same as running stage 9 with `--role controller`.
That would also generate a bearer token and start `cpms-mcp.service` listening
on the master. `--build-mcp-venv` builds only the virtual environment and does
nothing else. It is safe to run on a master regardless of role.

#### Cache the Datastore's Docker Images (Air-Gapped Controllers)

The Postgres, Grafana, backend, and frontend images normally pull from a
registry the first time a clone provisions as a controller. If that controller
will be air-gapped, pull and build the images now:

```bash
sudo cpms-provision --stage-datastore-images
```

`cpms-seal` does not remove `/var/lib/docker`, so the images survive sealing
and cloning. A controller clone finds every image already present and starts
the archive without registry access. Role convergence stops and disables Docker
on an endpoint clone, so pre-staged images never leave a network bridge or
firewall rules on a measurement host.

If you have already deployed a controller and need to move it to an air-gapped
network later:

1. Run `sudo cpms-datastore pull` on the live controller.
2. Export with `docker save`.
3. Import on the air-gapped host with `docker load`.

#### Smoke Test Before You Seal

Confirm the toolchain works end to end. This is a tooling check, not a
benchmark. The master has no jumbo MTU configuration, so do not record this
number.

```bash
iperf3-cpms -c <peer-address> -t 10 -P 4 --get-server-output
```

### Seal the Master

Sealing strips identity from the image so clones do not inherit it. It removes
the hostname, all network configuration, the machine ID, SSH host keys, CPMS
configuration, results, and volatile state. Your own `authorized_keys` file
stays, so `perfadmin` can still reach a clone once it has an address.

1. Run the dry run first:
   ```bash
   sudo cpms-seal --dry-run
   ```
   The output **must** include a `would disable:` line for each
   `/etc/netplan/*.yaml` and a `would write:` line for
   `99-cpms-disable-network.cfg`. If those lines are missing, the installed
   `cpms-seal` is stale. Your clones will come up on DHCP.

2. **Take a VM snapshot now.** Sealing is irreversible. A snapshot taken
   before sealing is the only way back. The script requires you to type `SEAL`
   to confirm you have one.

3. Run the seal:
   ```bash
   sudo cpms-seal
   ```

4. **Do not boot the master again.** Booting a sealed master regenerates its
   machine ID and SSH host keys. Every clone taken afterward shares them.
   Export the OVA or convert to a template immediately from the powered-off
   state.

---

## 02 · Clone and First Boot

Clone at least three times: two endpoints and one controller. Two endpoints on
the same host make the same-host calibration run possible. Nothing downstream
is trustworthy until that calibration passes.

Each clone boots straight to the setup menu on its console. The setup menu
owns `tty1` from the moment it starts, so there is no shell at the physical
console. Use SSH to check anything.

**A clone may already show an address** in your hypervisor guest summary before
you touch the console. On this single-NIC appliance that is expected: a DHCP
fallback in the base image can hand the interface a temporary address before
setup runs. `perfadmin` can log in as soon as that address appears.

> **Change the default password at first login.** The `perfadmin` account ships
> with a default password. A freshly cloned appliance can be reachable over the
> network before you have secured it. Run `passwd` at first login.

---

## 03 · Configure

The menu stages every edit. It does not apply anything until you choose
**Apply**. Set role last (item 7). Set every other option first, then commit
to a role.

### Endpoint

1. Select **1 · Interface addresses**. Enter a static address on the test
   VLAN, with its gateway. (Item 0, NIC mode, is already set to `single`.
   Skip it.)
2. Select **3 · MTU**. Enter `9000`, or the maximum the measured path supports.
3. Select **6 · Hostname**. Enter a unique hostname for this clone.
4. Select **7 · Appliance role**. Select `endpoint`. Enter the controller
   address that may drive it.
5. Select **8 · Review**. Confirm the staged settings look correct.
6. Select **9 · Apply**.

### Controller

1. Select **1 · Interface addresses**. DHCP is the default and is acceptable
   here. The controller's NTP configuration uses `allow all`, which was
   written specifically to tolerate the management address changing under
   DHCP. Select `static` only if your management network does not provide
   stable, reserved leases.
2. Select **3 · MTU**. Enter `1500`. Jumbo MTU on the controller risks the
   measurement path it drives.
3. Select **6 · Hostname**. Enter a unique hostname for this clone.
4. Select **7 · Appliance role**. Select `controller`. The menu warns that the
   measurement listener will be disabled.
5. Select **8 · Review**. Confirm the staged settings look correct.
6. Select **9 · Apply**.

> **Applying network changes.** A network change arms a detached 60-second
> rollback timer before it applies. Confirm with Enter on the console, or run
> `sudo cpms-setup --confirm` from a second SSH session. Do nothing for 60
> seconds and the previous configuration restores automatically. That automatic
> restore is the correct result when a change cuts off your SSH session. Role,
> hostname, and NTP changes are not network changes and do not arm the timer.

After a role change, Apply asks: `Run it now? [Y/n]:` — default is yes. Both
roles are fast on a clone (approximately 35 seconds for a controller, 15 seconds
for an endpoint). The expensive part, the iperf3 source build, already ran once
on the master. If you answer `n`, run provisioning manually later:

```bash
sudo cpms-provision --role endpoint     # or --role controller
```

### Confirm the Role

On a controller:

```bash
ss -lnt | grep -E '5201|5001|5000'   # expect nothing
systemctl is-active cpms-iperf3       # expect inactive
systemctl is-active cpms-iperf2       # expect inactive
systemctl is-active cpms-nuttcp       # expect inactive
sudo cpms-quiesce on                  # expect a refusal naming the role
```

A controller that still answers on any of those ports is still a measurement
target. It would appear in a measurement mesh as an endpoint. `cpms-quiesce`
refuses to run on a controller because it would stop the archive and scheduler
that the controller exists to run.

### Reboot Each Clone Once

The second boot must go straight to a login prompt. If the setup menu appears
again, nothing was applied. The marker file is written only on a successful
apply, not on menu exit.

### Verify Clone Independence

Check every clone — including clones taken from other clones. Clones with
identical machine IDs fight over DHCP leases. Clones with identical SSH host
keys are a real security problem.

```bash
cat /etc/machine-id
```

```bash
ssh-keyscan -t ed25519 localhost 2>/dev/null
```

Compare the output across every clone. All values must differ.

---

## 03b · Stand Up the Archive (Controller)

Stage 9 of controller provisioning already runs `cpms-datastore init`
automatically via `bootstrap_datastore`. You normally do not need to run it
again.

Run this command only when stage 9's own attempt failed. That attempt requires
the Docker image registry to be reachable. If the registry was not reachable
when stage 9 ran — for example, on a controller that was air-gapped before its
images were pre-staged — run the command manually once the registry is
accessible:

```bash
sudo cpms-datastore init
```

This command generates credentials, starts the Postgres and Grafana containers,
and applies the database schema. It prints the Grafana URL and a generated
admin password. **That output is the only time the full password is shown.**
The password is stored in `/etc/cpms/datastore.env` (mode 0600, not in git).
`cpms-seal` removes that file, so every clone generates its own credentials.

To read the password again at any time:

```bash
sudo grep GRAFANA_ADMIN_PASSWORD /etc/cpms/datastore.env
```

Check container health, schema version, and result counts:

```bash
sudo cpms-datastore status
```

To load results into the archive:

```bash
cpms-ingest --dry-run /path/to/result.json    # parse only, no database write
sudo cpms-ingest /path/to/result.json         # load it
```

`cpms-ingest` reads credentials from `/etc/cpms/datastore.env` (mode 0600).
That is why the load command requires `sudo`. To use an external archive, set
`PGHOST`/`PGDATABASE`/`PGUSER`/`PGPASSWORD`, or pass `--dsn`. Those take
precedence over the local credentials file.

`cpms-ingest` refuses a result with no `cpms_env` block. See section 06 for
why the merge step is not optional.

> **Air-gapped controllers:** Pre-stage images on the master before sealing
> (`sudo cpms-provision --stage-datastore-images`, see "Build the Master"
> above). Every clone then starts the archive without registry access. If you
> need to catch up on a controller already deployed: run
> `sudo cpms-datastore pull`, then move images with `docker save` /
> `docker load`.

---

## 03c · Enroll the Endpoints (Controller)

This step lets the controller drive tests on the endpoints. The controller
never measures. It holds the SSH key, decides which measurement to run on
which endpoint, pushes the command to that endpoint, and files the result.
The traffic runs entirely on the endpoint.

The controller's SSH key (`/etc/cpms/id_cpms`, root-only, no passphrase) is
created by stage 9 of `cpms-provision --role controller`. No separate keygen
step is needed.

```bash
sudo cpms-orchestrate enroll <endpoint-1-address> <endpoint-2-address>
```

You can pass one address or several in one call. The command asks for the
`perfadmin` password once per new host. Already-enrolled hosts in the same
call are skipped and re-verified.

For each host, `enroll`:

- Writes the host to the registry only after a key login succeeds and returns
  the hostname. A registry row that does not answer is worse than no row — it
  keeps getting selected, and every failure looks like a network problem.
- Adds `restrict` to its own line in `authorized_keys` on the endpoint. That
  key can run commands. It cannot open a shell or create a tunnel. Your own
  keys in that file are not changed. Re-enrolling is idempotent.
- Marks a controller as `target-only`. A controller is a valid measurement
  target but must never be scheduled as a measurement source.

One host failing does not stop the rest of the batch. Each host is attempted
independently. The command's own exit status reflects whether all hosts
succeeded.

Confirm the fleet:

```bash
sudo cpms-orchestrate check
```

```
  [+] cpms-endpoint01        ready
  [+] cpms-endpoint02        ready
  [!] cpms-endpoint03        reachable, missing: cpms-bench
```

`ready` means the host is reachable and carries `cpms-bench`, `cpms-perfenv`,
and `iperf3`. Fix a `missing:` line before scheduling any test against that
host.

### Run a Test from the Controller

Run a dry run first. It prints the plan, takes no lock, and touches no host.

```bash
sudo cpms-orchestrate run --source cpms-endpoint01 --target <endpoint-2-address> --matrix quick --dry-run
```

Then run for real:

```bash
sudo cpms-orchestrate run --source cpms-endpoint01 --target <endpoint-2-address> --matrix baseline
```

The endpoint's output streams back to the console. When the test finishes,
results are fetched to `/var/lib/cpms/incoming/<timestamp>/` and ingested
automatically.

**`--source` must be an enrolled endpoint.** Anything else is refused.

**`storage` is the one matrix with no `--target`.** It measures a local disk
or an already-mounted path on the source endpoint, not a peer:

```bash
sudo cpms-orchestrate run --source cpms-endpoint01 --matrix storage \
     --storage-path /mnt/data --storage-profile file_general
```

`cpms-bench` does not mount anything for you. `--storage-path` must already
be a writable, mounted directory. See TOOLS.md for the other five workload
profiles.

### The Global Lock

`run` holds a single fleet-wide lock for the entire remote execution. Two
measurements running at the same time corrupt each other's numbers. Neither
produces an error — both report a plausible but lower value. A second `run`
started while one is in progress is refused, and the refusal names who holds
the lock.

The lock is stored in the archive with an expiry. A killed run cannot prevent
further runs permanently. To release a lock left by a dead run:

```bash
sudo cpms-orchestrate unlock
```

Release the lock only when the run is genuinely dead. Releasing it under a
live run makes both that run's numbers and the next run's numbers suspect.

### File a Result from a Manual Bench Run

`run` fetches and ingests automatically, but only for a measurement it started
itself. If someone SSH-es into an endpoint and runs `cpms-bench` by hand, that
result stays on the endpoint until you file it. Use `fetch`:

```bash
sudo cpms-orchestrate fetch --from cpms-endpoint01
```

This command pulls every `*.json` file directly in the endpoint's results
directory (`/mnt/ramdisk/results`). It ingests them, then moves the endpoint's
copies into `.fetched/` so a second `fetch` does not re-ingest the same files.
A failure at any step leaves the endpoint files untouched. Nothing is moved
until ingest succeeds. `--dry-run` and `--no-ingest` work the same way as they
do for `run`.

> **After sealing and cloning a controller**, the SSH key is gone from the
> image by design. A clone carrying the master's key could log in to every
> endpoint the master enrolled. Cloned endpoints also lose the controller's
> key. Re-run `keygen` and `enroll`. This is not a bug.

---

## 04 · Running Measurements

Measure the path before you trust a number from it. Silence the machine while
you measure.

```bash
cpms-pathmtu <peer-address>       # true path MTU, by binary search
cpms-quiesce check                # report what would interfere; changes nothing
cpms-quiesce baseline             # snapshot NIC counters
sudo cpms-quiesce run -- iperf3-cpms -c <peer-address> -t 60 -P 4 -O 10 -J \
     --get-server-output > /mnt/ramdisk/run.json
cpms-perfenv --merge /mnt/ramdisk/run.json --iface <interface-name>
```

`run` baselines counters, quiesces, runs the command as the invoking user,
restores everything, then reports the counter delta on stderr. The redirect
above captures clean JSON. Read the counter delta, not the lifetime totals.
Lifetime totals accumulate from boot.

Two arguments are critical:

- **`cpms-perfenv --merge` is not optional.** Without it, the file is a bare
  iperf3 document with no record of the kernel, NIC, MTU, ring sizes,
  congestion control, or clock state. A throughput number without those is not
  reproducible. Run `--merge` immediately after the measurement, before
  changing any profile. `cpms-perfenv` reads the live system, so a fingerprint
  taken later describes a different state.
- **`-O 10`** omits slow-start intervals. Those intervals sit far below the
  steady-state rate. Included, they dominate the variance. Every figure in the
  spec was taken with `-O 10`.

### cpms-perfenv — What It Captures

`cpms-perfenv` is an endpoint-only Python script. It has no dependencies
beyond the standard library and the host's own tools (`ethtool`, `chronyc`,
`lscpu`, `sysctl`). Run it bare to print one JSON document to stdout. Use
`--merge <file>` to embed that document into an existing result under the
top-level `cpms_env` key.

| Section | What it reads | Why it matters |
|---|---|---|
| `host` | Hostname, `/etc/os-release`, kernel, arch, `systemd-detect-virt`, CPU count and model, RAM, NUMA nodes | A kernel or hypervisor difference between two hosts can explain a throughput gap that looks like a network problem |
| `tools` | Version strings for the pinned iperf3 (`/opt/iperf3-3.21`), distro iperf3, iperf2, fio, tcpdump, nuttcp | A result taken with the wrong iperf3 build is not comparable to the pinned-build baseline |
| `tcp` | `net.ipv4.tcp_congestion_control`, available CC algorithms, `default_qdisc`, `rmem_max`/`wmem_max`, `tcp_rmem`/`tcp_wmem`, `tcp_mtu_probing`, `rp_filter`, and the active `cpms-perfprofile` (from `/run/cpms-profile`, or `null` if never set) | This is the tuning state a `cpms-perfprofile` switch changes |
| `nic` | Name, driver, speed, MTU, MAC, IPv4, `ethtool -k` offload flags, `ethtool -g` RX/TX ring sizes, for the interface named by `--iface` or `$CPMS_TEST_IFACE` | Ring size and offload state are exactly the "why is this run different" questions a bare throughput number cannot answer |
| `clock` | `chronyc tracking`'s leap status, stratum, system time offset, root delay/dispersion, and a derived `synced` boolean | OWD, OWAMP, and TWAMP figures are trustworthy only when `synced` is true |

`--merge` is destructive to the file, not the host. It reads the live system,
writes the result back with `cpms_env` attached, and does nothing else. Do not
call it twice with different live state. The second call overwrites the first
fingerprint.

### cpms-perfprofile — Switching the TCP Tuning Profile

`cpms-perfprofile` is an endpoint-only tool. A controller has nothing to tune
and skips this stage entirely. This tool provides a way to restore the original
sysctl state after a tuning change without rebooting.

```bash
cpms-perfprofile show                    # active profile and live sysctl values
cpms-perfprofile list                    # profile names and one-line descriptions
sudo cpms-perfprofile <profile>          # apply a profile (writes /run/cpms-profile)
sudo cpms-perfprofile baseline           # restore /etc/sysctl.d/90-cpms-baseline.conf
```

From the controller, without an interactive login to the endpoint:

```bash
sudo cpms-orchestrate perfprofile --source <endpoint> show
sudo cpms-orchestrate perfprofile --source <endpoint> lan-10g
```

`show` and `list` need no privilege on the endpoint and take no lock.
Applying a profile briefly holds the global lock (section 03c) — a profile
change affects a concurrent run's numbers — then runs `sudo -n
cpms-perfprofile <profile>` on the endpoint over SSH.

The Web UI's Run page (section 07b, below) offers the same read/switch as a
control next to any matrix a profile actually affects. That control is a thin
wrapper over the MCP server's `cpms_set_perfprofile` tool (section 07). That
tool itself calls `cpms-orchestrate perfprofile`.

| Profile | Congestion control | qdisc | `rmem_max`/`wmem_max` | Intended for |
|---|---|---|---|---|
| `lan-10g` | cubic | fq | 64 MiB | Local VLAN, low RTT — the shape the spec's regression floor was measured under |
| `wan-dx` | bbr | fq | 256 MiB | Direct Connect / high-BDP paths |
| `wan-cubic` | cubic | fq | 256 MiB | WAN control run, to compare against `wan-dx` on the same path |
| `psonar` | htcp | fq_codel | 256 MiB | Matches perfSONAR defaults for cross-tool comparability |
| `baseline` | *(restores sysctl default)* | — | — | Required before comparing a result against the spec's reference figures, which were taken under baseline |

**Only `reno` and `cubic` are confirmed available on this image.** `wan-dx`
(bbr) and `psonar` (htcp) load their kernel module on first use and fail
loudly if the module is not present. Neither has been verified against a real
high-BDP path.

**A profile is runtime state, not persisted configuration.** It lives in
`/run/cpms-profile` (a tmpfs path) and does not survive a reboot.
`cpms-perfenv`'s `tcp.profile` field records which profile was active when a
result was taken.

### Counter Interpretation

| Counter | Meaning | Verdict |
|---|---|---|
| `rx_errors` | Malformed or oversized frames | **Invalidates the run** |
| `rx_oob` | Receive ring could not absorb the offered rate | **Invalidates the run** |
| `rx_missed_errors` | Ring descriptors exhausted (other drivers) | **Invalidates the run**, but vmxnet3 never populates it |
| `rx_dropped` alone | No protocol handler (LLDP, STP, mDNS, IPv6 RA) | Harmless |

`rx_oob` is the counter that fires on this platform. `rx_missed_errors` stays
0 on vmxnet3 regardless of ring state. That is why the tooling reads `rx_oob`
instead.

**Do not respond to `rx_oob` by raising the ring buffers.** At the driver
defaults, a full 60-second line-rate run to a physical 10G target produced an
`rx_oob` delta of zero. A larger ring has nothing to recover. Above roughly
10 Gbps — a same-host VM pair can reach 30 Gbps or more — `rx_oob` records
that the offered rate exceeded what the receive path could absorb. That is a
fact about the test, not a fault. Raise the ring size only against a non-zero
`rx_oob` delta on a real path. Never raise it against a lifetime total.

### UDP Loss Tests

```bash
sudo cpms-quiesce run -- iperf3-cpms -c <peer-address> -u -b 2G -l 8972 --dont-fragment \
     -t 30 -J --get-server-output > /mnt/ramdisk/udp-2g.json
cpms-perfenv --merge /mnt/ramdisk/udp-2g.json --iface <interface-name>
```

- **`-l 8972`** is the correct jumbo payload size (9000 − 20 IP − 8 UDP).
  iperf3 warns that it exceeds the TCP MSS. That warning is irrelevant for a
  UDP test.
- **`--dont-fragment`** enforces the MTU assumption. Without it, a datagram
  that needs fragmentation is fragmented silently. The loss figure then
  measures something other than what you intended.
- **Sweep the rate.** A single UDP run is not a measurement. Sweep 1G, 2G, 5G,
  and 8G. Look for the rate at which loss starts.

> **Counter direction warning.** `cpms-quiesce` samples the machine it runs on
> — the sender for a forward test — which drops nothing. The receiver is where
> UDP loss happens. A UDP run reporting clean counters has said nothing about
> the receiving endpoint. Check that host separately with
> `ethtool -S <iface> | grep OOB` until the controller can collect both sides.

**Regression floor:** On the jumbo path, a healthy endpoint pair sustains
9.8 Gbps or more with a CV around 1%. Persistent per-stream imbalance at
9000 MTU points at vCPU scheduling or RSS queue distribution, not the fabric.

**Stream count decides stability, not just speed.** On a CPU-bound path (two
VMs on one host, no wire), a single stream measured higher than eight streams
— 37.9 Gbps vs 35.3 Gbps — but with CV 18.7% against CV 2.0%. A metric with
19% variance cannot detect a 10% regression regardless of how many times you
repeat it. Use `-P 4` or `-P 8` for any tracked measurement. Keep `-P 1` for
diagnosis. On a link-bound path both stream counts sit at line rate and the
difference disappears.

---

## 05 · When Something Looks Wrong

Every entry below occurred during real deployment.

| Symptom | Cause | Fix |
|---|---|---|
| Console dead at `Reached target graphical.target`. SSH and ping work. | An old first-boot unit carrying `Conflicts=getty@tty1.service`. On the second boot the unit is skipped by its condition, but `getty` was already dropped from the transaction. | Run `sudo systemctl start getty@tty1`. Then re-run provision stage 8 for the corrected unit. |
| Clone comes up with a DHCP address | Stale `cpms-seal`. A master never taken through `cpms-setup` keeps its install-time netplan. Older seal versions left those files in place. | Install the current `cpms-seal`. Confirm the dry run lists `would disable` for each `*.yaml`. Re-seal. |
| Clone configures both NICs; two default routes | An old build defaulted `CPMS_NIC_MODE` to `dual` (DEFECT-7). | Current builds default to `single`. Confirm with `sudo cpms-setup --gen-netplan` — expect one interface stanza and one default route. |
| Listener bound to `0.0.0.0:5201` or `:5001` | The configured test interface does not exist on this host. `cpms-iperf3` and `cpms-iperf2` fell back to every address. (`cpms-nuttcp` always binds every interface — no equivalent flag exists for nuttcp. That is expected behavior, not a symptom.) | Re-run `sudo cpms-provision --role endpoint`. It resolves the interface from applied configuration and warns on a mismatch. |
| Setup menu appears on a working appliance | `/etc/cpms/.configured` is absent. The appliance was provisioned but never had a configuration applied. | Run `sudo cpms-setup` and apply. Nothing suppresses the menu except a successful apply. |
| Locked out after a network change | Expected, and handled automatically. | Do nothing for 60 seconds. The detached rollback restores the previous configuration. Or run `sudo cpms-setup --rollback`. |
| `sudo: cannot execute ./cpms-*.sh: Permission denied` | Missing executable bit after a copy from Windows. | Use the installed `/usr/local/bin` names, or run `chmod +x *.sh`. |

---

## 06 · Command Reference

| Command | Does |
|---|---|
| `cpms-setup` | Console menu — role, NIC mode, addresses, routing, MTU, DNS, NTP, hostname |
| `cpms-setup --show` | Print the current configuration file |
| `cpms-setup --role <r>` | Set role non-interactively. No network change. No rollback timer. |
| `cpms-setup --confirm` | Confirm a network change from a second session, canceling rollback |
| `cpms-setup --rollback` | Restore the most recent netplan backup |
| `cpms-setup --reset` | Discard staged edits and re-read the live system |
| `cpms-setup --gen-netplan` | Print the netplan that would be generated |
| `cpms-provision --role <r>` | Run all fourteen stages for a role; idempotent |
| `cpms-provision --stage N` | Run one stage |
| `cpms-provision --list` | List the stages |
| `cpms-provision --dry-run` | Show what would happen |
| `cpms-provision --build-mcp-venv` | Build `/opt/cpms-mcp` only. No role resolution. No controller-only side effects. |
| `cpms-provision --stage-datastore-images` | Pull and build every datastore image. For an air-gapped controller, run this on the master before sealing. |
| `cpms-toolset list` | Tool groups and their install state |
| `cpms-toolset install` | Install every diagnostic tool group (the default) |
| `cpms-quiesce check` | Report what would interfere; changes nothing |
| `cpms-quiesce run -- <cmd>` | Baseline, quiesce, run, restore, and report counter deltas |
| `cpms-pathmtu <target>` | True path MTU by binary search, with Don't Fragment set on every probe |
| `cpms-perfprofile show` | Current profile and live sysctl values |
| `cpms-perfprofile list` | Profile names and one-line descriptions |
| `sudo cpms-perfprofile lan-10g` | cubic + fq, 64 MiB buffers — local VLAN, low RTT; the shape the spec's regression floor was measured under |
| `sudo cpms-perfprofile wan-dx` | bbr + fq, 256 MiB buffers — Direct Connect / high-BDP paths; **unverified on this image** — only reno and cubic are confirmed available |
| `sudo cpms-perfprofile wan-cubic` | cubic + fq, 256 MiB buffers — WAN control run, to compare against `wan-dx` on the same path |
| `sudo cpms-perfprofile psonar` | htcp + fq_codel, 256 MiB buffers — matches perfSONAR defaults for cross-tool comparability; **unverified on this image** |
| `sudo cpms-perfprofile baseline` | Restore `/etc/sysctl.d/90-cpms-baseline.conf` and clear `/run/cpms-profile` — required before comparing a result against the spec's reference figures |
| `cpms-perfenv --iface <interface-name>` | Environment fingerprint as JSON — `--merge <file>` embeds it in a result |
| `cpms-seal --dry-run` | Print the exact seal manifest; change nothing |
| `cpms-seal` | Strip identity and power off. No undo. |

**Endpoint only**

| Command | Does |
|---|---|
| `cpms-bench --list` | Every matrix and what each is for |
| `cpms-bench --target <t> --matrix baseline` | Run a matrix both directions, quiesced, fingerprinted |
| `cpms-bench --target <t> --matrix iso --capture` | Header-only pcap alongside a low-rate matrix |
| `cpms-bench --target <t> --matrix idle --port 8443` | TCP session-idle-timeout discovery ladder |
| `cpms-bench --target <t> --dry-run` | Show the plan; run nothing |

**Controller only**

| Command | Does |
|---|---|
| `cpms-datastore init` / `up` / `status` | Postgres and Grafana lifecycle |
| `cpms-ingest --dry-run <file>` | Parse a result; load nothing |
| `cpms-ingest <file>` | Load a result into the archive |
| `cpms-orchestrate keygen` | Create the controller SSH key. Already run by stage 9 of `cpms-provision --role controller`. Safe to re-run; no-ops if a key already exists. |
| `cpms-orchestrate enroll <host> [<host>...]` | Push the key, verify, and register. One host or several in one call. |
| `cpms-orchestrate hosts` | The registry — who is enrolled, and as what |
| `cpms-orchestrate check [host]` | Is the host reachable and does it carry its toolchain? |
| `cpms-orchestrate run --source <h> --target <t>` | Run a measurement on an endpoint, fetch the result, and ingest it |
| `cpms-orchestrate perfprofile --source <h> show` \| `list` | Read the endpoint's active `cpms-perfprofile` over SSH. No lock, no privilege needed on the endpoint. |
| `cpms-orchestrate perfprofile --source <h> <profile>` | Apply a profile on the endpoint over SSH. Holds the global lock briefly — a profile change affects a concurrent run's numbers. |
| `cpms-orchestrate fetch --from <h>` | Ingest a result already on an endpoint from a manual `cpms-bench` run |
| `cpms-orchestrate ndt7-fetch --from <h>` | Fetch browser NDT7 results from one endpoint's data directory and ingest them |
| `cpms-orchestrate ndt7-fetch --all` | Same fetch for every enrolled endpoint in one call |
| `cpms-orchestrate unlock` | Release a lock left by a dead run |

### Files Worth Knowing

| Path | Holds |
|---|---|
| `/etc/cpms/setup.conf` | Staged intent — what you have asked for |
| `/etc/default/cpms` | Applied state — what the machine actually is |
| `/etc/cpms/.configured` | Written on a successful apply; suppresses the first-boot menu |
| `/etc/cpms/backups/<ts>/` | Pre-apply netplan backups |
| `/etc/netplan/60-cpms.yaml` | The only active network configuration. All other netplan files are renamed `.cpms-disabled`. |
| `/mnt/ramdisk` | 8 GB tmpfs — captures and staging, so the VMDK never enters the measurement path |
| `/var/log/cpms-provision.log` | Full provisioning detail |
| `/etc/cpms/id_cpms` | Controller's SSH key, root-only. Removed by sealing. |
| `/etc/cpms/known_hosts` | Host keys learned at enrollment. A changed key fails the connection. |
| `/etc/sudoers.d/cpms-endpoint` | Endpoint: NOPASSWD on four measurement commands, nothing else |
| `/var/lib/cpms/incoming/<ts>/` | Controller: results fetched from endpoints |

---

## 07 · MCP — Connecting LLM Tools to CPMS (Controller)

Optional. An MCP server exposes the archive to any LLM (Large Language Model)
client. Ask questions in plain language, explain a verdict, or start a
`cpms_run_benchmark`. The MCP server is built and started as part of controller
provisioning (stage 9). No further action is needed on the controller unless a
client needs wiring. The transport rationale, deeper troubleshooting, and the
full tool reference live in this project's internal engineering
documentation, which is not part of this public guide. This section covers
the steps needed to connect a client.

If stage 9 has not run yet (a master built with only
`cpms-provision --build-mcp-venv`):

```bash
sudo cpms-provision --stage 9
```

### Get Your Auth Token

Stage 9 generates the token once into `/etc/cpms/mcp.env`. A later
`cpms-provision` run does not regenerate it. Set this token once per client:

```bash
sudo cat /etc/cpms/mcp.env
# CPMS_MCP_TOKEN=...
```

### LM Studio and Claude Desktop — Identical Configuration

Both apps need `npx` on the **client machine**, not the controller. Both must
use the network transport (`streamable-http`). Neither app can use the SSH
transport described below for Claude Code. Paste the same block into LM
Studio's `mcp.json` (its integrations
panel edits this file) and into `claude_desktop_config.json` (Windows:
`%APPDATA%\Claude\`):

```json
{
  "mcpServers": {
    "cpms": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "http://<controller-address>:8765/mcp",
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

Quit and relaunch the client after editing. Both apps read configuration only
at startup. Confirm the connection works by asking the client to call
`cpms_archive_status`. A schema-version table in the response means the whole
chain is working.

### Claude Code — SSH stdio, No Token Needed

Run this on the controller itself:

```bash
claude mcp add cpms -- sudo /opt/cpms-mcp/bin/python /usr/local/bin/cpms_mcp.py
```

Or wrap it in SSH to run from anywhere else:

```bash
ssh perfadmin@<controller-address> sudo /opt/cpms-mcp/bin/python /usr/local/bin/cpms_mcp.py
```

This requires the controller's SSH host key to be already trusted and
passwordless key login to be working. Confirm both with a plain `ssh
perfadmin@<controller-address>` login — it must succeed with no host-key
prompt and no password prompt before you wire the client.

If something is not connecting, start with `cpms_archive_status`. If it
responds, the wiring is correct. If it does not, re-check the token (network
transport) or the SSH path (stdio transport) independently before assuming
the server itself is at fault.

---

## 07b · The Web UI (Controller)

The Web UI was added 2026-09-02. Two Docker containers, already part of
`docker/compose.yaml`, manage it alongside Postgres and Grafana.
`cpms-datastore up`/`down`/`status` already control all of them. No separate
lifecycle commands are needed.

```bash
sudo cpms-provision --role controller --stage 1   # stages docker/backend, docker/frontend, catalog.json
sudo docker compose --env-file /etc/cpms/datastore.env \
    -f /usr/local/share/cpms/docker/compose.yaml build backend frontend
sudo cpms-datastore up
```

Reach the Web UI at `http://<controller-address>:8090`.

The Web UI has **six pages** — Hosts, Catalog, Run, History, Compare, and
Speed Test — plus a **Chat tab**. Each page is a view over the same
`cpms_mcp` tools described in section 07.

| Page | What it does |
|---|---|
| **Hosts** | Shows the enrolled host registry. Includes a direct link to each endpoint's NDT7 browser speed test (connects to that endpoint's port 443 directly from your browser). |
| **Catalog** | Lists scheduled tasks. Toggle a task on or off. |
| **Run** | Submit a one-off measurement from the browser. For matrices where TCP tuning affects the result (`quick`, `baseline`, `scaling`, `udp-ramp`, `nuttcp`), the page also shows a "TCP profile" control and an "Apply to" control. "Apply to" defaults to both endpoints when Source and Target are both enrolled endpoints, because buffer sizing affects both ends of a path. Change "Apply to" to target one endpoint instead. "Apply profile" and "Show current" call the same profile switch as the CLI (see "cpms-perfprofile — Switching the TCP Tuning Profile", section 04, above). |
| **History** | Shows past results with headline metrics. Select a Run ID to download the raw result JSON. |
| **Compare** | Compare two runs side by side. |
| **Speed Test** | Runs an NDT7 test directly from your browser to a selected endpoint. The page sets `protocol: 'ws'` with no explicit port; port 80 is the `ws://` protocol default. The test connects directly to the endpoint — not proxied through the Web UI's nginx. NDT7 derives its throughput and RTT numbers from the kernel's TCP_INFO on the measurement socket. Routing that socket through a proxy would create two separate TCP sessions and invalidate the numbers. |

The Chat tab is wired to an LLM (LM Studio or Ollama, OpenAI-compatible).
Every `cpms_mcp` tool is available as a function call, so answers come from
the real archive.

**To enable the Chat tab:**

1. Open `cpms-setup` and select item 7 (Appliance role).
2. Select the controller role.
3. Enter the LLM base URL at the prompt.
4. Select **Apply** (item 9). Setting the URL in item 7 only stages it.
   Apply is required.
5. Leave the URL blank to disable the Chat tab. The other six pages do not
   need it.

The scheduler runs inside the backend container automatically once a task is
enabled. No separate command is needed. Enable a task with
`cpms_set_task_enabled` or the Catalog page's own toggle. The change takes
effect within approximately 30 seconds.

---

## 08 · Firewall and Port Reference

This section lists every port CPMS listens on, with the source and destination
roles for each.

**Endpoint measurement ports** (section A) carry endpoint-to-endpoint
measurement traffic only. The controller never generates test traffic and never
initiates connections to those ports. Exception: ports 80 and 443 on an
endpoint also serve NDT7 browser speed tests — the browser connects directly
to the endpoint, with no controller involvement in the measurement.

Where a range is listed for OWAMP and TWAMP, that is the per-session UDP
test-packet range negotiated over the TCP control connection. The entire range
must be open, not just the control port.

### A: Endpoint Role — Inbound

| Port | Proto | Direction | Service | Notes |
|---|---|---|---|---|
| 5201 | tcp | Source: endpoint → Destination: endpoint | `iperf3-cpms` | `quick`/`baseline`/`scaling`/`udp-ramp` matrices |
| 5001 | tcp | Source: endpoint → Destination: endpoint | `iperf2` (TCP) | `owd` matrix (`cpms-iperf2-client`) |
| 5001 | udp | Source: endpoint → Destination: endpoint | `iperf2` (UDP) | `iso` matrix |
| 5000 | tcp | Source: endpoint → Destination: endpoint | `nuttcp` control | `nuttcp` matrix |
| 5101 | tcp | Source: endpoint → Destination: endpoint | `nuttcp` data | Dynamically negotiated over the control channel. Do not assume this port stays fixed across nuttcp versions. |
| 8443 | tcp | Source: endpoint → Destination: endpoint | TLS echo listener | `idle` matrix (port configurable via `--port`, default 8443) |
| 443 | tcp | Source: browser → Destination: endpoint | `ndt7-server` (TLS) | NDT7 browser speed test. The browser connects to this endpoint directly, not through the controller. Self-signed certificate; expect a one-time browser trust warning per endpoint. |
| 80 | tcp | Source: browser → Destination: endpoint | `ndt7-server` (cleartext) | NDT7 Speed Test page uses `ws://` on this port. The browser connects directly to the endpoint. Not proxied through nginx. |
| 3001 / 3002 / 3010 | tcp | (no CPMS component initiates connections to these ports) | `ndt7-server` legacy NDT5 | Bundled in the ndt7-server binary. No separate flag disables them. |
| 861 | tcp | Source: endpoint → Destination: endpoint | OWAMP-Control | `owamp` matrix — real one-way delay, not the `owd` matrix's iperf2 approximation |
| 8760–9960 | udp | Source: endpoint → Destination: endpoint | OWAMP test traffic | Per-session range from `owamp-server.conf`, negotiated over port 861 |
| 862 | tcp | Source: endpoint → Destination: endpoint | TWAMP-Control | `twamp` matrix |
| 18760–19960 | udp | Source: endpoint → Destination: endpoint | TWAMP test traffic | Per-session range from `twamp-server.conf`, negotiated over port 862 |

### B: Controller Role — Inbound

| Port | Proto | Direction | Service | Notes |
|---|---|---|---|---|
| 8765 | tcp | Source: MCP client → Destination: controller | `cpms_mcp.py` (MCP server) | Bound to all interfaces; token-authenticated (`/etc/cpms/mcp.env`) |
| 8090 | tcp | Source: browser → Destination: controller | Web UI (nginx/frontend) | Management VLAN, all interfaces — the operator's browser entry point |
| 3000 | tcp | Source: browser → Destination: controller | Grafana | Management VLAN, all interfaces |
| 5432 | tcp | Loopback only within the controller | Postgres | `127.0.0.1:5432` in `docker/compose.yaml`. No network firewall rule needed. |
| 8081 | tcp | Loopback only within the controller | Web UI backend (FastAPI) | `127.0.0.1:8081`. nginx proxies to it. No network firewall rule needed. |
| 123 | udp | Source: endpoint → Destination: controller | `chrony` (NTP server) | Bound to all interfaces (`allow all` in `/etc/chrony/conf.d/99-cpms.conf`, written and `chrony` restarted on every apply on a controller). Endpoints sync from the controller. Network segmentation is the trust boundary. |

### C: Both Roles — Inbound

| Port | Proto | Direction | Service | Notes |
|---|---|---|---|---|
| 22 | tcp | Source: controller → Destination: endpoint (orchestration); also Source: operator workstation → Destination: controller or endpoint (admin access) | SSH | Controller uses key at `/etc/cpms/id_cpms` for orchestration. Operator workstation uses its own key for admin access. |
| 9100 | tcp | Source: controller (Prometheus) → Destination: endpoint or controller | `node_exporter` | CPU, RAM, disk, and network metrics. Bound to all interfaces on the packaged default. Scraped by the controller's Prometheus container. |

Prometheus and `postgres_exporter` (both controller-only) publish no host
port. They are internal to the compose network and reach each other by service
name only.

**Not a fixed CPMS port:** The Chat tab's LLM endpoint (`CPMS_LLM_URL`, set
in `cpms-setup` item 7) is whatever address the operator's LM Studio or
Ollama host actually listens on (commonly port 1234 for LM Studio's
OpenAI-compatible API). This is outbound from the controller only. CPMS does
not open it.


