# Talking to a mini-PC's embedded controller: fan and power-mode control on EVO-X2/Strix Halo

Field notes for people running AMD Strix Halo mini-PCs on Linux — the class of
box (GMKtec EVO-X2, Bosgame M5, FEVM FA-EX9 — all the same Sixunited AXB35-02
board) whose firmware exposes no standard fan control at all: no hwmon PWM, no
fan tachometer, no ACPI fan object. The only lever is the Embedded Controller
(EC), and the only documentation is what you can measure yourself.

Everything below is adapted from
[KyaniteLabs/evo-x2-ec](https://github.com/KyaniteLabs/evo-x2-ec) (MIT) — that
repo is the canonical source. It holds the register map, the validation
scripts, and the deploy units this writeup restates. Provenance lines like
`Provenance: evo-x2-ec/README.md` point at files in that repo.

Safety first · The problem · Write access and the silent-discard trap ·
The register map · Two laws of EC writes · Validating a register without
vendor docs · The re-assertion deployment pattern · What the tuning measured ·
Sending findings upstream · The honest gap · Reproduce this · Credits and
license

---

## Safety first

The block quote is from the canonical repo's own safety notes, and it is the
right place to start:

> You are poking an Embedded Controller that also runs the thermal
> protection. Registers were characterized empirically; there is no vendor
> documentation. Use at your own risk.
>
> Register semantics beyond the table above were **not** characterized; writing
> unlisted registers is unwritten territory.

`Provenance: evo-x2-ec/README.md · "Safety notes"`

Three practical consequences, all stated plainly:

- Writes go to a microcontroller that also enforces thermal protection. A bad
  write can affect cooling behavior. The failure mode actually observed on
  one machine was benign — control silently reverting to the firmware's auto
  fan curve — but that is one machine's experience, not a safety guarantee.
- Only the registers in the map below were characterized. Do not write
  anything else. Reading is low-risk; every write is a decision.
- The deployed fan daemon was chosen partly for its fail-safe design: on any
  error or stop it hands the fans back to the firmware auto curve, and the
  worst realistic outcome is a fan stuck at 100% (loud, but safe) until
  reboot. Any setup you build should have an equally defined hand-back path.

`Provenance: evo-x2-ec/README.md · "Safety notes", "Fan daemon deploy recipe"`

## The problem

The machine is a GMKtec EVO-X2: AMD Ryzen AI Max+ 395 ("Strix Halo") on a
Sixunited AXB35-02 mainboard. The firmware's auto fan curve is tuned for
quiet and lets the package ride the thermal edge under sustained load — it
stops ramping past 90 °C entirely (measured below). Linux sees no fan
controls to fix this with, because the vendor never exposed any. Direct EC
access over the kernel `ec_sys` interface is the workaround, and it works.

| Fact | Value |
|---|---|
| Machine | GMKtec EVO-X2 mini-PC |
| APU | AMD Ryzen AI Max+ 395 (Strix Halo) |
| Mainboard | Sixunited AXB35-02 (shared with Bosgame M5, FEVM FA-EX9) |
| EC | ITE IT5570 on the M5; EVO-X2 exposes the same register layout (chip model on the EVO-X2 unit not independently verified) |
| Linux fan control exposed by firmware | none |

`Provenance: evo-x2-ec/README.md · "Hardware context"`

The register map itself is a two-party result. The fan duty and tachometer
offsets were reverse-engineered by
[nathanmarlor/strix-halo-fan-control](https://github.com/nathanmarlor/strix-halo-fan-control)
from the ACPI DSDT of the Bosgame M5 — the same board — and that map was then
confirmed on a second chassis, the EVO-X2, in August 2026. The APU power-mode
register (`0x31`) and the read-side telemetry registers were
reverse-engineered directly on the EVO-X2 in May 2026 by Simon Gonzalez de
Cruz, assisted by GLM-5.3.

`Provenance: evo-x2-ec/README.md · "Honest framing", "Credits"`

## Write access and the silent-discard trap

All reads and writes go through the `ec_sys` module's debugfs file:

```
/sys/kernel/debug/ec/ec0/io
```

The trap that costs the most time: by default `ec_sys` loads **read-only**,
and in that state the file still *looks* writable — writes to it are
**silently discarded**. A `dd` against it exits `0` with no error. You will
believe your write landed; the EC never saw it. This exact accident happened
twice on the source machine, once against the power-mode register and once
against the fan duty registers.

The fix is to make write support the module default and load the module at
boot. Two one-line config files:

`/etc/modprobe.d/ec_sys-write.conf`
```
options ec_sys write_support=1
```

`/etc/modules-load.d/ec_sys.conf`
```
ec_sys
```

Then verify:

```
cat /sys/module/ec_sys/parameters/write_support   # want: Y
```

`Provenance: evo-x2-ec/README.md · "EC access path and the ec_sys write-support trap"` ·
`deploy/ec_sys-write.conf`, `deploy/ec_sys.conf`

One corollary worth memorizing: a write that succeeds *and sticks* (see the
one-shot revert law below) is the only real proof the trap is not biting you.

## The register map

This is the whole usable surface: one power-mode register, two fan duty
registers, three tachometer pairs, one temperature byte, and six fan
mode/level bytes nobody has decoded. All values are hexadecimal.

| Register | Value / encoding | Effect | Known since |
|---|---|---|---|
| `0x21` | byte | fan1 mode — semantics not characterized (firmware-internal fan state) | May 2026 |
| `0x22` | byte | fan1 level — semantics not characterized | May 2026 |
| `0x23` | byte | fan2 mode — not characterized | May 2026 |
| `0x24` | byte | fan2 level — not characterized | May 2026 |
| `0x25` | byte | fan3 mode — not characterized | May 2026 |
| `0x26` | byte | fan3 level — not characterized | May 2026 |
| `0x28`/`0x29` | big-endian pair: `(b[0x28]<<8)\|b[0x29]` | fan3 tachometer (RPM). `8000` (`0x1F40`) is a "no fan installed" sentinel — treat as 0 | May 2026 |
| `0x31` | `0x00` balanced · `0x01` performance · `0x02` quiet | APU power mode ("P-MODE") — the firmware's own power-profile setting | May 2026 |
| `0x33` | write `0x80 \| duty` (duty 0–100); write `0x00` | fan1 duty: manual fan speed; `0x00` returns the fan to firmware auto | Aug 2026 (predicted by the M5 DSDT work) |
| `0x34` | same encoding as `0x33` | fan2 duty | Aug 2026 (predicted by the M5 DSDT work) |
| `0x35`/`0x36` | big-endian pair — **the lower offset holds the high byte** | fan1 tachometer (RPM) | May 2026 |
| `0x37`/`0x38` | big-endian pair, same packing | fan2 tachometer (RPM) | May 2026 |
| `0x70` | byte, °C | package temperature | May 2026 |

`Provenance: evo-x2-ec/README.md · "EC register map"` · read-side offsets cross-checked against
`scripts/gpu-host-ec-readonly.py` (`REGS`) and `scripts/gpu-host-pmode.py` (`POWER_REGISTER = 0x31`)

Two details that will bite anyone writing code against this table:

**Tach tearing.** The EC updates each tachometer's two bytes non-atomically,
so a single read can tear — one stale byte plus one fresh byte is garbage.
Read the full EC image twice about 20 ms apart and accept a fan's RPM only if
both reads agree and are plausible; otherwise hold the last-known-good value.

`Provenance: evo-x2-ec/README.md · "Tach tearing"`

**The third tachometer.** `0x28`/`0x29` is a third fan channel, distinct from
the duty/tach pairs at `0x33`–`0x38` — evidence the board supports (or at
least reserves telemetry for) a third fan. The EVO-X2 unit measured reads the
`8000` sentinel, so no physical third fan could be confirmed. This register
was not documented in the upstream fan-control repo when found, and was
offered back to that project (see "Sending findings upstream").

`Provenance: evo-x2-ec/README.md · "Fan3: a third tachometer channel"`

## Two laws of EC writes

Two firmware and kernel behaviors explain nearly every "my EC write didn't
work" report on these boxes. Neither produces an error message.

### The one-shot revert law

A single write to a fan duty register (`0x33`/`0x34`) does not stick. The
firmware observes manual overrides and reverts them within single-digit
seconds. The only way to hold a manual duty is continuous re-assertion — a
userspace loop that rewrites the register faster than the firmware reverts
it. Every working EC fan daemon on this platform is a polling daemon for this
reason; the deployed one rewrites duty every `2` s and holds indefinitely.

The same policing class applies to the power-mode register: a one-shot
`0x31` write is not guaranteed to survive other agents (vendor tooling, the
front-panel button, firmware policy), so the deployed setup re-asserts it
every `30` s.

The corollary is a test protocol: "I wrote the register and it changed"
proves nothing. Write it, sleep 5–10 s, read it back. If it reverted, either
the firmware policed it (this law) or your write was silently discarded (the
`ec_sys` trap above). The two failure modes look identical from userspace —
check `write_support` first.

`Provenance: evo-x2-ec/README.md · "The one-shot revert law"`

### The module-cycling hazard

Two independent EC writers ran on the source machine: the P-MODE timer (every
`30` s) and the fan daemon (every `2` s). The original P-MODE script was a
careful single-user citizen: it enabled `ec_sys` write support only around
its own write, then reloaded the module read-only afterwards:

```python
def restore_readonly_ec_sys() -> None:
    run(["modprobe", "-r", "ec_sys"], check=False)
    run(["modprobe", "ec_sys"], check=False)   # without write_support!
```

Once a second EC writer arrived, that politeness became a hazard. Every
cycle, `modprobe -r` transiently removes `/sys/kernel/debug/ec/ec0/io` out
from under the fan daemon, and the reload comes back **without**
`write_support` — from that moment the daemon's writes are silently discarded
while the daemon looks perfectly healthy. Net effect: fan control silently
degrades to the firmware's auto curve, with no error anywhere.

The mitigation, as deployed: once
`options ec_sys write_support=1` is the module default in
`/etc/modprobe.d/`, *any* reload — by anyone — comes up write-capable, so the
downgrade leg of the accident cannot happen. Architecturally, prefer a single
EC writer per machine, or have one daemon own both power mode and fan duty.

`Provenance: evo-x2-ec/README.md · "The module-cycling hazard (multiple EC writers)"`

## Validating a register without vendor docs

The `0x31` power-mode register is the worked example of the method this
writeup most wants to pass on: nobody had documentation for it, and its
existence and effect were proven anyway, in May 2026, with a bounded workload
plus a telemetry harness. The pattern generalizes to any undocumented control
register whose effect should show up in power, clock, or thermal telemetry.

The idea: you do not need to observe the control itself (a physical button
press, a vendor app). You need to observe its *consequences*, while holding
everything else still. So:

1. **Run a fixed, bounded workload** so the machine has a repeatable load
   level. The source harness used `openssl speed -multi <threads> sha256`
   wrapped in `timeout` — free, CPU-bound, no install. Passive mode (no
   synthetic load) also works while a real workload is already running.
2. **Log telemetry at a fixed cadence** from independent instruments, so no
   single source has to be trusted: `turbostat` for package watts, average
   and busy MHz, and busy percent; `sensors -j` for `k10temp` `Tctl`, `amdgpu`
   edge temperature and PPT watts, NVMe and ACPI temperatures;
   `rocm-smi --showtemp --showpower --showclocks --json` for GPU power and
   `sclk`; cpufreq sysfs for governor, min/max/current frequency, EPP, and
   boost; plus the EC's own registers read back each sample.
3. **Cycle the candidate control at phase boundaries** — the harness prints
   "click P-MODE once now" every 25 s and you press the physical button (or
   write the candidate register). Later phases are compared per phase: if the
   register is real, package watts, clocks, and temperatures move in step
   with the mode change and stay moved.
4. **Bound the risk.** The harness has a thermal guard: if `Tctl` reaches a
   ceiling (default `82.0` °C) it kills the workload and stops, exiting
   non-zero. No phase runs unbounded — default run is `75` s.
5. **Record everything as data.** Every sample lands in a JSONL file; a
   per-phase summary (avg/max of watts, clocks, temperatures) is what the
   verdict is read off of. The claim "this register does something" is then
   backed by numbers anyone can re-derive from the log.

In the May 2026 session this is exactly how `0x31` was validated: bounded
workload, modes cycled, package watts / GPU watts / clocks / temperatures
logged per mode. The harness is
`scripts/gpu-host-pmode-click-scan.py`; the read/write tool is
`scripts/gpu-host-pmode.py`; the read-side register dump is
`scripts/gpu-host-ec-readonly.py`.

`Provenance: evo-x2-ec/README.md · "P-MODE mechanism and enforcement"` ·
`scripts/gpu-host-pmode-click-scan.py` (defaults: `--duration 75`, `--phase-seconds 25`,
`--max-tctl 82.0`, passive `--threads 0`; guard exit code `2`)

What counts as a validated register, by this standard: writing it changes
something the independent telemetry can see, the change survives the 5–10 s
read-back test (or is policed, and the policing is the confirmation), and
the result reproduces across at least two mode positions. What does not
count: the register's byte changed and nothing measurable followed.

## The re-assertion deployment pattern

Because of the one-shot revert law, both working controls on this machine are
deployment patterns, not scripts: a thing that writes, sleeps less than the
firmware's patience, and writes again — forever.

**Power mode: systemd oneshot + timer.** A oneshot service writes
`0x31 = 0x01` (performance) and reads the status back; a timer fires it every
`30` s (`OnBootSec=15s`, `OnUnitActiveSec=30s`, `AccuracySec=5s`,
`Persistent=true`). Performance mode has held continuously on the source
machine since May 2026 under this pattern.

```ini
# gpu-host-pmode-performance.timer (as deployed)
[Timer]
OnBootSec=15s
OnUnitActiveSec=30s
AccuracySec=5s
Persistent=true
Unit=gpu-host-pmode-performance.service
```

`Provenance: evo-x2-ec/README.md · "P-MODE mechanism and enforcement"` · `deploy/gpu-host-pmode-performance.timer`,
`deploy/gpu-host-pmode-performance.service`

**Fan duty: a polling daemon.** The deployed daemon is
[nathanmarlor/strix-halo-fan-control](https://github.com/nathanmarlor/strix-halo-fan-control)
(MIT) — a single-file Python daemon, `strix-halo-fand`, that reads the
hottest of `amdgpu`/`k10temp`, maps it through a curve, and writes
`0x80|duty` to `0x33`/`0x34` every `2` s. It was chosen for its fail-safe
design: on any error or `SIGTERM` it writes `0x00` — handing both fans back
to the firmware auto curve — and a minimum-duty floor keeps airflow up even
if something wedges. Its systemd unit also re-loads `ec_sys` with
`write_support=1` on start, as a belt-and-braces measure.

The full deploy recipe, in order:

1. Install the two module config files from the trap section above.
2. Install the daemon from the upstream repo (it ships an install script and
   a systemd unit).
3. Replace the daemon's default `CURVE` with a tuned one (next section).
4. `systemctl enable --now strix-halo-fand`.

`Provenance: evo-x2-ec/README.md · "Fan daemon deploy recipe and tuned curve"` · `deploy/strix-halo-fand.service`

## What the tuning measured

All numbers below were measured on one EVO-X2 unit, August 2026, and they are
the reason the effort is worth it: the stock curve gives up exactly where the
machine needs help.

The stock firmware auto curve saturates — it stops ramping as the package
climbs past 90 °C — leaving roughly 40% of the louder fan's headroom unused
at exactly the temperatures where it is needed.

| Stock firmware auto curve | Value |
|---|---|
| Saturation point | ~60% duty / ~3260 RPM at 90 °C and above |

At full manual duty the two fans are asymmetric, which matters when tuning:
a single shared duty value is effectively fan1-limited at the top end.

| Fan | RPM at 100% duty |
|---|---|
| fan1 | ~3480 |
| fan2 | 4539 |

`Provenance: evo-x2-ec/README.md · "Measurements: stock curve and fan maxima"`

The tuned curve ("quiet-idle tune", 2026-08-15) as installed in the daemon:

```python
CURVE = [(45, 30), (58, 38), (68, 60), (78, 85), (82, 95), (999, 100)]
FLOOR = 30        # minimum duty %, never starve airflow
INTERVAL = 2.0    # seconds between re-assertions (must beat the revert law)
```

The shape in one line: quiet idle, early ramp, full duty by 82 °C.

| Package temp | Duty |
|---|---|
| below 45 °C | 30% |
| 45–58 °C | 38% |
| 58–68 °C | 60% |
| 68–78 °C | 85% |
| 78–82 °C | 95% |
| 82 °C and above | 100% |

Measured peak package temperature under a standardized ~105 s GPU load —
first pass, n=1:

| Setup | Peak temp |
|---|---|
| Stock firmware auto curve | 97.5–97.8 °C |
| Daemon + tuned curve | 96.5 °C |

The honest read of that first pass: a modest ~1–1.3 °C delta; the practical
wins were the *shape* (defined quiet idle, earlier ramping, and 100% duty
engagement above 82 °C that the stock curve never reaches). A later n=3
re-run with the daemon continuously running measured peaks of 93 / 92 / 94 °C
against the same stock band — a −3.5 to −5.8 °C delta, larger than the first
pass. Labeled caveat from the source: the stock baseline was **not re-run**
for the n=3 pass (daemon-untouched rule); the stock figures come from the
2026-08-15 measurement ledger. Both passes stand with their labels.

`Provenance: evo-x2-ec/README.md · "Fan daemon deploy recipe and tuned curve" (incl. 2026-08-16 update)`

## Sending findings upstream

The third gift of this writeup is the contribution path, because a validated
register is worth more in the project people actually install than in a
personal notes file. On 2026-08-15 a docs-only pull request was opened
against the upstream daemon:
[nathanmarlor/strix-halo-fan-control#1](https://github.com/nathanmarlor/strix-halo-fan-control/pull/1).
It carried exactly three things, which is a good template for any EC finding:

1. **Validation data** — confirmation that the upstream register map
   (reverse-engineered on the Bosgame M5) checks out byte-for-byte on a
   second chassis, the EVO-X2, plus measured numbers the project could fold
   into its README.
2. **A new register** — the third tachometer at `0x28`/`0x29` with its
   `8000` no-fan sentinel, offered with the honest limit attached (sentinel
   reads on this unit, so no physical third fan confirmed).
3. **A failure-mode warning** — the `ec_sys` module-cycling hazard, written
   as a suggested README warning any user could act on.

Docs-only was a deliberate scope: no code changes, no maintenance burden
transferred, and the upstream author free to reword or reject. If you find a
register on sibling hardware, that is the shape of the offer: what you
confirmed, what is new, what will bite people, each with its uncertainty
label intact.

`Provenance: evo-x2-ec/PR-TO-NATHANMARLOR.md · status header ("SENT", opened 2026-08-15)`

## The honest gap

What is *not* known, and what carries risk. This section is a condition of
trusting anything above.

- **The fan mode/level bytes (`0x21`–`0x26`) are undecoded.** They are read
  and logged; nobody knows their semantics. They are in the map so the next
  person does not have to re-find them.
- **The EC chip on the EVO-X2 was not independently verified.** The ITE
  IT5570 identification comes from the Bosgame M5; the EVO-X2 is only
  confirmed to expose the same register layout.
- **One unit.** Every measurement above is from a single EVO-X2. The thermal
  numbers are single-sample or n=3 on one machine, and the n=3 stock baseline
  was not re-run (stated above, kept labeled).
- **The revert timing is characterized only as "single-digit seconds."** No
  exact bound was measured; interval choices (`2` s, `30` s) have large
  margin, but the actual firmware timer is unknown.
- **The WMI/FCMI path was never attempted on this box.** Two sibling
  projects drive the same EC through proper ACPI/WMI methods —
  [MintyMods/ip3-power-switch](https://github.com/MintyMods/ip3-power-switch)
  (MIT) and
  [pettijohn/corsair-ai-workstation-performance-level-linux](https://github.com/pettijohn/corsair-ai-workstation-performance-level-linux)
  (GPL-3.0, cited only; no code included here). That is likely the better
  long-term route — a kernel WMI driver instead of debugfs poking — and it is
  untested on the EVO-X2.
- **What risks a brick:** writing registers outside the map is unwritten
  territory (the source's own words above); the EC also runs thermal
  protection, so a bad write can affect cooling. The observed failure modes
  on this machine were all benign-and-silent (writes discarded or reverted),
  but nothing here is a safety certification — one unit, no vendor docs, use
  at your own risk.

`Provenance: evo-x2-ec/README.md · "Safety notes", "The WMI/FCMI alternative", register table`

## Reproduce this

The canonical repo holds everything: scripts, deploy units, and the register
map this writeup restates.

- Canonical repo: [KyaniteLabs/evo-x2-ec](https://github.com/KyaniteLabs/evo-x2-ec) (MIT)
  - `scripts/gpu-host-pmode.py` — read/set EC `0x31` (`status` / `set <balanced|performance|quiet>`; `set` requires root)
  - `scripts/gpu-host-ec-readonly.py` — dump the known read-side registers as JSON
  - `scripts/gpu-host-pmode-click-scan.py` — the bounded-workload + telemetry validation harness
  - `deploy/` — module configs and the P-MODE oneshot + timer units
- Fan daemon: [nathanmarlor/strix-halo-fan-control](https://github.com/nathanmarlor/strix-halo-fan-control) (MIT)

Minimum viable read-only first step, no writes involved:

```
sudo modprobe ec_sys
sudo python3 gpu-host-ec-readonly.py
```

Then, only if the machine is yours and the safety section above has been
read: verify write support (`cat /sys/module/ec_sys/parameters/write_support`
→ `Y`), and prefer installing the upstream daemon over hand-rolling writes.

## Credits and license

- **Simon Gonzalez de Cruz (May 2026), assisted by GLM-5.3** — original
  reverse-engineering of the EVO-X2 P-MODE register (`0x31`), the read-side
  register map, the power/thermal validation harness, and the 30 s P-MODE
  enforcement timer. The `scripts/` directory of the canonical repo is his
  work. August 2026 validation/extension session: register-map confirmation,
  revert law, `ec_sys` traps, measurements, daemon deployment.
- **[nathanmarlor/strix-halo-fan-control](https://github.com/nathanmarlor/strix-halo-fan-control)**
  (MIT) — the fan daemon deployed and validated; its duty/tach offsets were
  reverse-engineered from the Bosgame M5's ACPI DSDT and confirmed here.
- **[MintyMods/ip3-power-switch](https://github.com/MintyMods/ip3-power-switch)**
  (MIT) and
  **[pettijohn/corsair-ai-workstation-performance-level-linux](https://github.com/pettijohn/corsair-ai-workstation-performance-level-linux)**
  (GPL-3.0) — prior art for the WMI route to the same EC, cited only.

This writeup is MIT-licensed (see [LICENSE](LICENSE), Copyright (c) 2026
Kyanite Labs). The canonical repository
[KyaniteLabs/evo-x2-ec](https://github.com/KyaniteLabs/evo-x2-ec) is MIT
(Copyright (c) 2026 Simon Gonzalez de Cruz); the upstream daemon retains its
own MIT license. When this document and the canonical repo disagree, the
canonical repo wins — check there first.
