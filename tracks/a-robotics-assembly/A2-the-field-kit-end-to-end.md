# A2 — The Field Kit, End to End

**Track:** Robotics Assembly  
**Source spec:** [SuperInstance/the-technician `01-FIELD-KIT-ARCHITECTURE.md`](https://github.com/SuperInstance/the-technician/blob/main/01-FIELD-KIT-ARCHITECTURE.md)  
**Training Port phases served:** Phase 0 (Pre-Observer), Phase 1 (Observer), Phase 2 (Assistant); reference mode in Phase 4 (Independent Technician)

## Why this module exists

The Jetson Field Kit (JFK) is the physical object a trainee will eventually carry onto a vessel. Before that happens, they need to understand its parts, its power paths, and its boot sequence well enough to debug it in a noisy engine room. This module walks the kit from the Pelican case to the first Telegram message.

## The real source material

The specification lives in `01-FIELD-KIT-ARCHITECTURE.md` in the [the-technician](https://github.com/SuperInstance/the-technician) repo. It is a buildable white paper: it names exact hardware, gives file paths, shows systemd units, and lists expected timings. The paper is itself the Field Kit's manual.

## What to study

### 1. The enclosure and power paths

The kit is built around a Pelican 1450 case with a custom laser-cut HDPE insert. Power enters through one of two paths:

- **Primary:** 12V DC marine/RV input via a 5.5mm x 2.1mm barrel jack, center positive. A wide-input (9-36V) DC-DC converter supplies a stable 12V/5A to the Jetson's official power input header.
- **Secondary:** 802.3bt PoE++ via an external 95W injector and a PoE HAT. This enables single-cable deployment for both power and data.

Study the cooling note: the Pelican insert has passive ventilation channels, the Jetson's native heatsink/fan is retained, and the case is *never operated fully sealed during active compute*. That sentence is a safety rule, not a suggestion.

### 2. Core compute and storage

The compute module is the NVIDIA Jetson Super Orin Nano Developer Kit (8GB shared GPU/System RAM). Storage is a Crucial P3 Plus 2TB NVMe M.2 2280 SSD. The design rationale is explicit: 2TB gives room for models, image/video logs, and local git repositories; NVMe gives rapid system cloning and model loading.

### 3. The boot sequence

The kit is designed for zero-touch provisioning. The sequence is:

1. Technician inserts the NVMe SSD into the M.2 slot.
2. Technician inserts a pre-flashed 32GB SD card.
3. Technician connects 12V power.
4. `firstboot.service` runs `/opt/provision/check_nvme.sh`.
5. If the NVMe is blank, `/opt/provision/clone_to_nvme.sh` runs:
   - Creates a GPT partition table.
   - Makes a 512MB ESP and an ext4 root partition.
   - Formats both partitions.
   - `rsync`s the running root filesystem to the NVMe.
   - Installs the bootloader to the NVMe ESP.
   - Updates `/mnt/etc/fstab` to use the NVMe UUID.
   - Disables `firstboot.service` on the target.
   - Reboots.
6. On the next boot, the system comes up from NVMe and runs `/opt/captain/scripts/startup.sh`.

The script explicitly writes `CLONE_COMPLETE` to `/tmp/clone_status` before rebooting. That file is a diagnostic artifact.

### 4. Network announcement and the three-tier connectivity ladder

After boot, the `startup.sh` script:

1. Starts `avahi-daemon` for `jetson.local` mDNS.
2. Launches the Starlette web server on port 8080.
3. Starts the Telegram/Discord bot polling process.
4. Sends a system status message to the pre-configured channel.

The paper documents three connectivity tiers:

- **Tier 1:** Boat WiFi or Starlink. High bandwidth. Failover triggers when ping latency to 8.8.8.8 exceeds 2000ms for 30 seconds.
- **Tier 2:** USB phone tethering. Limited to ~5Mbps; video streaming is disabled.
- **Tier 3:** Air-gapped local. After 2 minutes with no Tier 1 or 2, the kit notifies the user, falls back to Whisper.cpp and Piper TTS, keeps the LLM running via llama.cpp, and performs git operations locally with sync deferred.

The air-gapped mode is not a degraded novelty. It is the design center. The Manifesto line the spec quotes: "on a 120-foot longliner in the Bering Sea in January, the cloud is a fantasy."

### 5. File system layout

All application code, models, and data live under `/opt/captain/`:

```
/opt/captain/
├── models/      # AI models, versioned
├── repos/       # Git repositories for each project
├── data/        # SQLite DB, logs, uploads
├── web/         # Starlette web app
├── scripts/     # System management
└── config/      # Bot credentials and system config
```

The repo layout is intentional: every vessel install gets its own git repo under `/opt/captain/repos/`, and the course content itself will eventually sync there as just another repo.

## Comprehension questions

1. The kit boots from SD but never clones to NVMe. Name the exact script and status file you would check first, and explain why the boot sequence is designed to disable `firstboot.service` on the target after a successful clone.

2. A technician powers the kit from a boat's 12V DC bus. Draw the power path from the barrel jack to the Jetson module, naming the two components in between and the voltage/current spec each must handle.

3. The kit enters Tier 3 (air-gapped) after 2 minutes without connectivity. Which services keep running, which services stop, and why is the local LLM via llama.cpp considered air-gap-safe while cloud Whisper API is not?

4. The Starlette web UI is reachable at `http://jetson.local:8080`. Why is mDNS (`avahi-daemon`) the right choice for a field kit, and what would break if the kit relied on a hardcoded IP address instead?
