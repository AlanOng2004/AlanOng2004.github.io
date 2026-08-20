---
title: Phase 1 - Storage Architecture & Media Automation
description: Building the Docker foundation, isolated analytical storage, and atomic hardlinks for a homelab data platform.
date: 2026-08-20 23:26:00 +0800
categories: [Homelab, Data Engineering]
img_path: /assets/img/posts/
tags: [Docker, Linux, Infrastructure, Jellyfin, Data Engineering]
---

# **1. Building the Foundation**

I am building a dual-purpose, self-hosted homelab on my Linux mini PC. The system is split into two parts: an operational, fully automated media server—the "Netflix" experience—and an analytical, production-grade Modern Data Stack (MDS).

Before writing any extraction pipelines or transformation models, the underlying storage architecture needs to be structurally sound. In a naive setup, downloaded files are physically copied from one directory to another. This wastes disk space and creates unnecessary disk I/O. Phase 1 focuses on designing a unified directory structure that avoids these limitations while keeping the analytical stack isolated from the media stack.

## **1.1. The Motivation: Atomic Hardlinks**

**Goal:** Design a storage mapping that allows the operating system to make completed downloads available to the media library without duplicating their data.

**The problem:** Many tutorials map `/data/torrents` to a `/downloads` Docker volume and `/data/media` to a separate `/media` volume. Even when both host directories live on the same disk, those separate mounts appear as different mount points inside the container. Applications such as Radarr and Sonarr can no longer create a hardlink between them, so importing a file falls back to a full copy.

**The solution:** Mount one unified parent directory—`/data`—at the same path inside every application that participates in the import workflow. Both the torrent and media paths then remain visible through one mount, allowing Radarr and Sonarr to create hardlinks.

## **1.2. The Theory: Inodes and Filesystems**

**Goal:** Understand the Linux file-management mechanics that make hardlinking possible.

**Key concepts:**

* **Inodes:** On a Linux filesystem, an inode stores a file's metadata and references its data blocks. A filename is a directory entry that points to that inode.
* **Hardlinks:** A hardlink creates another directory entry that points to the same inode. The file data is not duplicated; the inode's link count is incremented instead. The data remains on disk until the final hardlink is removed.
* **Cross-device constraints:** Hardlinks cannot span filesystems because inode numbers only have meaning within their own filesystem. Separate container mounts can also hide the shared filesystem relationship from an application, even when their host paths reside on the same drive.

---

# **2. Technical Interview Application**

This setup translates directly into useful Systems and Data Engineering interview concepts.

**Q: What is the difference between a hardlink and a symbolic link at the OS level?**
> **A:** A hardlink is another directory entry that points to the same inode. Deleting one name does not remove the data while another hardlink still exists. A symbolic link is a separate file, with its own inode, that stores a path to another file. If the target is removed, the symlink becomes dangling. Symlinks can cross filesystem boundaries; hardlinks cannot.

**Q: A containerized data pipeline is moving 100 GB files between two directories on the same NVMe SSD, but each move performs a full copy. What is a likely architectural flaw?**
> **A:** The directories may have been exposed to the container through separate bind mounts. The application's mount namespace therefore prevents it from treating the source and destination as locations on the same filesystem. Exposing their shared parent as one mount allows an atomic `rename()` when moving a pathname, or a hardlink when both names must remain available.

---

# **3. Infrastructure & Execution**

## **3.1. Unified Directory Structure**

**Goal:** Create the host directories with permissions that support the containers and background services.

I first created the project directory and the unified media tree:

```bash
mkdir -p ~/homelab
cd ~/homelab

sudo mkdir -p \
  /data/torrents/movies \
  /data/torrents/tv \
  /data/media/movies \
  /data/media/tv
```

I then assigned the tree to the host's primary user and applied directory permissions that allow services to traverse and read it:

```bash
sudo chown -R 1000:1000 /data
sudo chmod -R 755 /data
```

The resulting layout is:

```text
/data
├── torrents/
│   ├── movies/
│   └── tv/
└── media/
    ├── movies/
    └── tv/
```

The important detail is not the directory names themselves. It is that `/data/torrents` and `/data/media` live on the same filesystem and are exposed to the relevant containers through the same `/data:/data` mount.

## **3.2. Docker Compose Configuration**

**Goal:** Write the declarative infrastructure needed to launch the media automation services and the isolated Modern Data Stack.

Three Compose fields do most of the work:

* **`image`:** Selects the pre-built application image pulled from a container registry.
* **`environment`:** Defines runtime configuration. For LinuxServer.io images, `PUID=1000` and `PGID=1000` make application-created files belong to the host's primary user instead of `root`, avoiding locked-file permission problems.
* **`volumes`:** Maps host storage into each container. The media applications share `/data:/data` so their internal paths agree and hardlinks can succeed. PostgreSQL and MinIO instead use `/opt/appdata/...`, deliberately isolating analytical storage formats from raw media files.

The complete `compose.yaml` is:

```yaml
services:
  # ==========================================
  # MEDIA AUTOMATION STACK
  # ==========================================

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent
    container_name: qbittorrent
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Asia/Singapore
      WEBUI_PORT: 8080
    volumes:
      - /opt/appdata/qbittorrent:/config
      - /data:/data # Unified mount for atomic hardlinks
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr
    container_name: radarr
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Asia/Singapore
    volumes:
      - /opt/appdata/radarr:/config
      - /data:/data # Identical path keeps imports on one mount
    ports:
      - "7878:7878"
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr
    container_name: sonarr
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Asia/Singapore
    volumes:
      - /opt/appdata/sonarr:/config
      - /data:/data
    ports:
      - "8989:8989"
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr
    container_name: prowlarr
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Asia/Singapore
    volumes:
      - /opt/appdata/prowlarr:/config
    ports:
      - "9696:9696"
    restart: unless-stopped

  jellyseerr:
    image: lscr.io/linuxserver/jellyseerr
    container_name: jellyseerr
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Asia/Singapore
    volumes:
      - /opt/appdata/jellyseerr:/config
    ports:
      - "5055:5055"
    restart: unless-stopped

  # ==========================================
  # DATA ENGINEERING STACK (MDS)
  # ==========================================

  postgres:
    image: postgres:15
    container_name: postgres
    environment:
      POSTGRES_USER: alan
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
      POSTGRES_DB: analytics
    volumes:
      # Isolated structural storage for relational data
      - /opt/appdata/postgres:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped

  minio:
    image: minio/minio
    container_name: minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: alan
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:?Set MINIO_ROOT_PASSWORD in .env}
    volumes:
      # Isolated object storage for the S3-compatible data lake
      - /opt/appdata/minio:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    restart: unless-stopped
```

I stored the two secrets in a local `.env` file beside `compose.yaml`:

```dotenv
POSTGRES_PASSWORD=<REDACTED_DB_PASSWORD>
MINIO_ROOT_PASSWORD=<REDACTED_MINIO_PASSWORD>
```

The `.env` file must be excluded from version control. I also omitted the legacy top-level `version: "3.8"` field because current Docker Compose uses the Compose Specification and no longer requires it.

## **3.3. Integrating the Existing Jellyfin Server**

**Goal:** Give my existing, Tailscale-exposed Jellyfin server access to `/data/media` without allowing the presentation layer to alter the source library.

Jellyfin is the presentation layer of the media stack—the closest equivalent to a BI tool serving an analytical dataset. It only needs to read the curated media library. If Jellyfin is running in Docker, I can add this bind mount to its Compose service:

```yaml
volumes:
  - /data/media:/data/media:ro
```

The `:ro` flag makes the mount read-only inside the container. Jellyfin can scan and stream the files, but it cannot rename, modify, or delete them.

For a native systemd installation, the same boundary can be enforced through filesystem permissions or a systemd read-only bind path. The principle is identical: the service account receives read and traverse access to `/data/media`, but no write access. Jellyfin's own configuration, cache, and metadata directories remain writable elsewhere.

## **3.4. Stack Deployment**

**Goal:** Start the containers in the background and verify that they are running.

From the project root, I first asked Compose to validate and render the configuration, then launched the stack:

```bash
cd ~/homelab
docker compose config
docker compose up -d
```

The `-d` flag runs the containers in detached mode, returning control of the terminal while the services continue starting in the background.

I then checked their state and published ports:

```bash
docker compose ps
docker ps
```

At this stage, a container being listed as running only proves that its main process has started. I still verified each web interface and checked any service logs that reported repeated restarts:

```bash
docker compose logs --tail=100 <service-name>
```

## **3.5. Verifying Atomic Hardlinks: The Inode Test**

**Goal:** Prove that the unified volume mapping is working and that Radarr is not duplicating file data during import.

I triggered a test download through Radarr, which sent the torrent to qBittorrent. The completed file appeared at:

```text
/data/torrents/movies/TestMovie (2026)/TestMovie.mkv
```

Radarr then imported it into:

```text
/data/media/movies/TestMovie (2026)/TestMovie.mkv
```

I used `stat` on both paths to compare their device identifiers, inode numbers, and link counts:

```console
alan@alan-mini:~$ stat "/data/torrents/movies/TestMovie (2026)/TestMovie.mkv" | grep -E 'Device|Inode'
Device: 10302h/66306d   Inode: 14450912    Links: 2

alan@alan-mini:~$ stat "/data/media/movies/TestMovie (2026)/TestMovie.mkv" | grep -E 'Device|Inode'
Device: 10302h/66306d   Inode: 14450912    Links: 2
```

Both paths report the same device and the exact same inode—`14450912`. The link count is `2`, confirming that two filenames point to one underlying file. The physical media data is therefore stored on the SSD only once.

If the inode numbers differed, Radarr would have created an independent copy. If the device identifiers differed, the paths would not be eligible for hardlinking in the first place.

---

# **4. Conclusions**

Phase 1 was about designing the storage foundation for a high-efficiency homelab. By understanding Linux inodes and Docker mount namespaces, I could structure the media workflow around atomic hardlinks. This eliminates duplicate file data and avoids a large amount of unnecessary disk I/O. At the same time, the operational media files and the analytical MDS data remain separated into distinct storage locations, reducing the risk of accidental cross-contamination.

Overall, this was a pretty fun experience. Much like the work I documented in my HPL benchmarking write-up, going from low-level theory to a working configuration had me asking, "Why is this so annoying?" every five minutes. I still do not fully understand all the kernel internals behind mount namespaces and how container paths resolve to host inodes, but I am looking forward to learning more as I explore the rest of the platform.

With Phase 1 complete, the infrastructure is live. Next is **Phase 2: Data Ingestion**. I will write custom Python extraction scripts to pull JSON payloads from the Radarr and Sonarr APIs, then land that raw data directly in the new MinIO S3 bucket.
