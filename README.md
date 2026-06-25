# COCOMAPS-MD

Analyze protein-protein interactions from MD trajectory PDB files. Runs CoCoMaps per-frame, aggregates results, detects conserved interaction islands, and visualizes everything through a web app or command-line tool.

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

Run the same analysis pipeline on your own machine from the terminal. Works on macOS, Linux, and Windows. Docker handles all the native dependencies (reduce, hbplus, naccess) internally — you don't install anything else.

### What you need

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (free for academic use)

### Step 1: Install the `coco-md` command

Download the wrapper script and put it on your PATH. This is a small Python file that translates your local file paths into Docker volume mounts so you can point at any PDB on your machine. It also pulls the Docker image automatically on first use — no separate `docker pull` needed.

**macOS / Linux:**

```bash
sudo curl -fsSL https://raw.githubusercontent.com/sattamaltwaim/COCOMAPS-MD/master/coco-md -o /usr/local/bin/coco-md && sudo chmod +x /usr/local/bin/coco-md
```

**Windows (PowerShell as Administrator):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\bin"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/sattamaltwaim/COCOMAPS-MD/master/coco-md" -OutFile "$env:USERPROFILE\bin\coco-md.py"
```

Then add the folder to your PATH (run this in the same PowerShell window):

```powershell
[Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";$env:USERPROFILE\bin", "User")
```

Close and reopen PowerShell for the change to take effect. (`$env:USERPROFILE` expands automatically — no need to replace it.)

### Step 2: Run

Make sure Docker Desktop is running, then open a terminal and navigate to the folder that contains your PDB file:

```bash
cd /path/to/my/pdb/files
```

**Interactive mode** — walks you through everything step by step:

```bash
coco-md
```

**Analyze a file in your current folder:**

```bash
coco-md my_protein.pdb
```

**File anywhere on your machine (absolute path):**

```bash
coco-md /Users/me/data/dimer.pdb
```

**Specify chains, output location, and frame range:**

```bash
coco-md trajectory.pdb -c A B -o results/ -s 1 -e 50
```

Results (CSVs, charts, summary tables) are written to your machine in the directory you specify with `-o`, or `systems/<pdb_name>/` in your current folder by default.

### CLI options

```
coco-md [pdb_file] [OPTIONS]

  -c A B         Chain IDs to analyze (default: auto-detect from PDB)
  -o DIR         Output directory (default: systems/<pdb_name>)
  -r             Add hydrogens with reduce before analysis
  -i CUTOFF      Interface selection cutoff in Å (default: 5.0)
  -w CUTOFF      Water bridge cutoff in Å (default: same as -i)
  -s N           Start frame, 1-indexed (default: first)
  -e N           End frame, 1-indexed inclusive (default: last)
  -n N           Frame step — analyze every Nth frame (default: 1)
  -t N           Conservation threshold % for charts (0-100, default: 50)
  -u UNIT        Time axis label for charts (fs, ps, ns; default: Frame)
  -p FILE        JSON file with CoCoMaps parameter overrides
  -C             Enter interactive parameter customization before running
```

### Updating

```bash
coco-md --update
```

### Without the wrapper script

If you prefer not to install the wrapper, you can run the Docker image directly. Mount your current directory so the container can see your files:

```bash
docker run --rm -it --platform linux/amd64 \
  -v "$(pwd)":/data/cwd -w /data/cwd \
  sattamaltwaim/cocomaps-md-cli my_protein.pdb -o results/
```

---

## Docker Hub Images

| Image | Size | Purpose |
|---|---|---|
| [`sattamaltwaim/cocomaps-md-backend`](https://hub.docker.com/r/sattamaltwaim/cocomaps-md-backend) | ~1.2 GB | Analysis engine + REST API |
| [`sattamaltwaim/cocomaps-md-frontend`](https://hub.docker.com/r/sattamaltwaim/cocomaps-md-frontend) | ~50 MB | Web interface (nginx + Vue 3) |
| [`sattamaltwaim/cocomaps-md-cli`](https://hub.docker.com/r/sattamaltwaim/cocomaps-md-cli) | ~1.2 GB | Command-line tool |

All images are `linux/amd64`. The `coco-md` wrapper and the compose files above handle the platform flag automatically — Apple Silicon Macs run them via Rosetta emulation with no extra steps.
