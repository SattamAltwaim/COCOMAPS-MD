# COCOMAPS-MD

Analyze protein-protein interactions from MD trajectory PDB files. Runs CoCoMaps per-frame, aggregates results, detects conserved interface patches, and visualizes everything through a web app or command-line tool.

---

## Web App — Server Deployment

Host the web interface for your lab or department. Users open a browser, upload a PDB, and get results. No software installation required on their end.

### What you need on the server

- A Linux x86_64 machine (physical or VM)
- Docker Engine and Docker Compose installed ([install guide](https://docs.docker.com/engine/install/))

That's it. Docker pulls all dependencies automatically — no Python, no Node.js, no manual package installs.

### Step 1: Create a project folder

```bash
mkdir cocomaps-md && cd cocomaps-md
```

### Step 2: Create `docker-compose.yml`

Create a file called `docker-compose.yml` with this content:

```yaml
services:
  backend:
    image: sattamaltwaim/cocomaps-md-backend
    platform: linux/amd64
    # Not published to the host — the frontend reaches it over the internal
    # Compose network and proxies /api to it. Avoids host port-5001 conflicts.
    expose:
      - "5001"
    volumes:
      - systems-data:/app/systems
    environment:
      - COCOMAPS_DEPS_DIR=/app/deps
      - PYTHONUNBUFFERED=1
    restart: unless-stopped

  frontend:
    image: sattamaltwaim/cocomaps-md-frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  systems-data:
```

### Step 3: Start the app

```bash
docker compose up -d
```

Docker will download the images from Docker Hub (~1.5 GB total, one-time) and start both services. This takes a few minutes the first time.

### Step 4: Verify

```bash
docker compose ps
```

You should see both `backend` and `frontend` running.

The frontend serves the app at the `/BioTools/COCOMAPS-MD/` path. If your server is at `biotools.example.edu`, users visit:

```
https://biotools.example.edu/BioTools/COCOMAPS-MD/
```

Point your existing reverse proxy (nginx, Apache, Caddy, etc.) at the frontend container on whichever port you chose.

### Updating to a newer version

```bash
docker compose pull
docker compose up -d
```

### Stopping the app

```bash
docker compose down
```

Analyzed data is stored in a Docker volume (`systems-data`) and persists across restarts and updates. To back it up:

```bash
docker compose cp backend:/app/systems ./systems-backup
```

### Changing ports

Only the frontend is published to the host. `"80:80"` is the host-side port — change the left number to any available port on your server:

```yaml
ports:
  - "8080:80"    # frontend on host port 8080
```

The backend isn't published to the host (the frontend proxies `/api` to it over the internal Compose network), so there's no backend port to change.

### Job privacy

Job IDs are stored in each user's browser via `localStorage`. The Jobs page only shows jobs whose IDs are present in that list, so each user sees only their own submissions. Analysis URLs remain publicly shareable — anyone with a direct link can view the results.

---

## CLI — For Local Use

Run the same analysis pipeline on your own machine from the terminal, with no 50-frame limit. Works on macOS, Linux, and Windows. The analysis runs inside Docker, which bundles the native dependencies (reduce, hbplus, naccess) and the chart renderers — you don't install anything else. Start it either with the small `coco-md` launcher (recommended: it builds the Docker command for you) or by typing the `docker run` command yourself.

### What you need

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (free for academic use). On Linux, an existing Docker Engine installation also works.
- For the launcher: Python 3. macOS and most Linux distributions already include it; on Windows install it from [python.org](https://www.python.org/downloads/) and tick "Add python.exe to PATH" in the installer.

### Step 1: Install the `coco-md` launcher

The launcher is a small Python script. It mounts the folders it needs, adds `--platform linux/amd64`, and passes every option through to the CLI, so relative and absolute paths both work.

**macOS / Linux** (you will be asked for your password):

```bash
sudo curl -fsSL https://raw.githubusercontent.com/sattamaltwaim/COCOMAPS-MD/master/coco-md -o /usr/local/bin/coco-md
sudo chmod +x /usr/local/bin/coco-md
coco-md --update
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\bin"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/sattamaltwaim/COCOMAPS-MD/master/coco-md" -OutFile "$env:USERPROFILE\bin\coco-md.py"
[Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";$env:USERPROFILE\bin", "User")
```

Close and reopen PowerShell, then download the analysis image:

```powershell
coco-md.py --update
```

On Windows the command is `coco-md.py` (with the extension) wherever the examples below show `coco-md`. If PowerShell does not recognise it, run `python coco-md.py` instead.

The `--update` step downloads the analysis image (about 1.2 GB, once). If you skip it, the first analysis downloads it instead.

### Step 2: Run

Make sure Docker is running, then open a terminal in the folder that contains your PDB file (or pass the full path to the file). Replace `trajectory.pdb` with your filename and `A B` with the two chain IDs to analyse. Keep quotation marks around paths that contain spaces.

```bash
coco-md trajectory.pdb -c A B -o results
```

**File anywhere on your machine:**

```bash
coco-md /Users/me/data/dimer.pdb -c A B -o /Users/me/data/dimer_results
```

**Guided mode** — run with no arguments to be asked for the file and settings step by step:

```bash
coco-md
```

The CLI displays the detected frames, chains, and chosen settings. When asked whether to customise, press Enter to keep the defaults, or type `y` to edit them.

Results are saved in the folder given with `-o`. Use a new output folder name for each run: an existing folder is not cleared, and leftover files can be picked up by the next analysis. If `-o` or the PDB path is left out, the CLI asks for it; the default output folder is `systems/<pdb-name>` inside your current folder.

### Conservation thresholds and frame range

CR is the residue-pair conservation threshold and CA the interaction-type conservation threshold (both default to 50%). They filter the conservation matrix, interaction heatmap, interaction trends, and distance distributions. Interface-patch charts and the Mol* snapshot always use pairs conserved in at least 70% of frames, and the interface area chart is not filtered. Frame-range options change which frames are analysed; leave out `-s`, `-e`, and `-n` to process the full trajectory.

```bash
# Set CR to 50% and CA to 40%
coco-md trajectory.pdb -c A B -o results_cr50_ca40 --pair-conservation 50 --interaction-conservation 40

# Analyse frames 1 through 50, inclusive
coco-md trajectory.pdb -c A B -o results_frames_1_50 -s 1 -e 50

# Analyse every tenth frame
coco-md trajectory.pdb -c A B -o results_every_10 -n 10
```

### Updating

```bash
coco-md --update
```

The launcher reminds you after a run when your local image is more than 30 days old.

### What you get

```
results/
├── trajectory.pdb              # Original input copy
├── viewer.pdb                  # Cleaned first processed frame
├── _metadata.json              # Frame count, chains, job ID
├── _interactions.csv           # Residue-pair interactions
├── _atom_pairs.csv             # Atom-pair data
├── _area.csv                   # Surface-area data
├── _trends.csv                 # Interaction trends
├── _water_mediated.csv         # Water-mediated contacts (when present)
├── _metal_mediated.csv         # Metal-mediated contacts (when present)
├── _interface_patches.json     # Interface-patch data
└── charts/
    ├── conservation_matrix.png
    ├── interaction_heatmap.png
    ├── interaction_trends.png
    ├── interface_area.png
    ├── interface_patch_3d.png  # Mol* snapshot, largest interface patch highlighted
    ├── distance_distribution/
    │   └── dist_*.png
    └── interface_patches/
        └── *.png
```

Chart files depend on the enabled charts and the interaction data available. Temporary `frame_*` directories are removed after aggregation and viewer preparation, before chart export.

### CLI options

Show the full help with `coco-md --help`. Options go after the PDB path; leave them unchanged for a standard run.

```
coco-md [pdb_file] [OPTIONS]

  -c A B         Chain IDs to analyse (default: first two chain IDs in the file, alphabetically)
  -o DIR         Output directory (if omitted the CLI asks; default: systems/<pdb_name>)
  -r             Add hydrogens with Reduce before analysis (default: off)
  -i CUTOFF      Interface cutoff in Å (default: 5.0)
  -w CUTOFF      Water bridge cutoff in Å (default: same as -i)
  -s N           Start frame, 1-indexed (default: first)
  -e N           End frame, 1-indexed inclusive (default: last)
  -n N           Frame step — analyse every Nth frame (default: 1)
  --pair-conservation N
                 CR threshold % for charts (0–100, default: 50; alias: -t)
  --interaction-conservation N
                 CA threshold % for charts (0–100, default: 50)
  -u UNIT        Time axis label for charts (default: Frame)
  -p FILE        JSON file with CoCoMaps parameter overrides
  -C             Enter interactive parameter customisation before running
  --update       Launcher only: download the latest analysis image
  -h, --help     Show command-line help
```

### Without the launcher

Download the image, then run it with your PDB folder mounted at `/data`. These are the commands the launcher runs for you. Keep `--platform linux/amd64` in every command, including on Apple Silicon Macs, and keep `-it` so that the prompts work.

```bash
docker pull --platform linux/amd64 sattamaltwaim/cocomaps-md-cli:latest
```

**macOS / Linux:**

```bash
cd "/path/to/pdb-folder"
docker run --rm -it --platform linux/amd64 -v "$PWD:/data" sattamaltwaim/cocomaps-md-cli:latest /data/trajectory.pdb -c A B -o /data/results
```

**Windows (PowerShell, not Command Prompt):**

```powershell
Set-Location "C:\path\to\pdb-folder"
docker run --rm -it --platform linux/amd64 -v "${PWD}:/data" sattamaltwaim/cocomaps-md-cli:latest /data/trajectory.pdb -c A B -o /data/results
```

Options go after the PDB path exactly as with the launcher, for example `--pair-conservation 50 --interaction-conservation 40`. The `/data` prefix refers to your mounted folder inside Docker; for a filename containing spaces, quote the complete input path, such as `"/data/my trajectory.pdb"`. A `-p` parameter file must also be inside the mounted folder (`-p /data/params.json`). To update, repeat the `docker pull` command.

### Troubleshooting

- **Zero BSA or Interface Area values** — these need the bundled NACCESS executable. The CLI checks for its surface-area summary after the frames are analysed and stops if it is missing; no CSV tables or charts are written and the temporary `frame_*` folders remain. Run `coco-md --update` (or the `docker pull` command) and rerun in a new output folder. If it still fails, keep the terminal error message when reporting the problem.
- **`coco-md` command not found** — the launcher is not installed or not on your PATH. Reinstall it with the commands above (on Windows run `coco-md.py` or `python coco-md.py`), or use the `docker run` command instead.
- **Cannot connect to Docker** — open Docker Desktop and wait until it is running, then retry.
- **PDB file not found** — check that your terminal is in the folder containing the PDB, or pass the full path. With `docker run`, the filename after `/data/` must match exactly.
- **Chain not found** — use the chain IDs printed by the CLI; they are case-sensitive.
- **Mol\* image missing** — check the terminal for a snapshot warning. A job with no detected interface patch produces no snapshot. Other causes are an older image without the Mol\* renderer (run `coco-md --update`) or the snapshot exceeding its 120-second limit.
- **Prompts do not appear, or the run stops with `EOFError`** — when running Docker directly, the customise prompt, `-C`, and the prompts for a missing `-o` or PDB path need the `-it` flags. The launcher adds them automatically when run from a terminal.
- **Proximal contacts** are excluded from the charts by default; use `-C` to include them.
- **Old version still running** — neither the launcher nor `docker run` refreshes an image you already have. Run `coco-md --update` or the `docker pull` command.
- **Files owned by root (Linux)** — the container writes as root. Use `sudo chown -R $USER results` to take ownership.

---

## Docker Hub Images

| Image | Size | Purpose |
|---|---|---|
| [`sattamaltwaim/cocomaps-md-backend`](https://hub.docker.com/r/sattamaltwaim/cocomaps-md-backend) | ~1.2 GB | Analysis engine + REST API |
| [`sattamaltwaim/cocomaps-md-frontend`](https://hub.docker.com/r/sattamaltwaim/cocomaps-md-frontend) | ~50 MB | Web interface (nginx + Vue 3) |
| [`sattamaltwaim/cocomaps-md-cli`](https://hub.docker.com/r/sattamaltwaim/cocomaps-md-cli) | ~1.2 GB | Command-line tool |

All images are `linux/amd64`. The `coco-md` launcher and the compose file above handle the platform flag automatically — Apple Silicon Macs run them via Rosetta emulation with no extra steps.
