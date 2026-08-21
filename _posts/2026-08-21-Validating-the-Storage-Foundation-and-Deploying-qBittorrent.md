---
title: Phase 1, Day 2 - Validating the Storage Foundation and Deploying qBittorrent
description: Testing the real filesystem, correcting a path mistake, securing Docker port exposure, and deploying the first media automation container.
date: 2026-08-21 22:48:44 +0800
categories: [Homelab, Data Engineering]
img_path: /assets/img/posts/
tags: [Docker, Linux, Infrastructure, qBittorrent, Tailscale, Data Engineering]
---

# **1. Moving from Architecture to Evidence**

Yesterday, I designed the storage architecture for my media automation and data engineering homelab. The central idea was to expose one shared `/data` parent directory to qBittorrent, Radarr, and Sonarr so that completed downloads could be imported with hardlinks instead of full file copies.

Today, I moved from design to validation. The goal was not to launch every service at once. It was to prove the assumptions underneath the design, correct any mismatch between the plan and the real machine, and deploy one container with a known-good storage and network boundary.

This distinction matters in infrastructure work: a configuration file can be syntactically valid while still describing the wrong host paths, permissions, or exposure model.

---

# **2. Finding an Absolute-versus-Relative Path Mistake**

## **2.1. The Problem**

I initially created and tested this relative directory:

```text
./data
```

Because my working directory was `/home/alan`, it resolved to:

```text
/home/alan/data
```

However, the bind mount in Docker Compose used an absolute host path:

```yaml
volumes:
  - /data:/data
```

The host path on the left was `/data`, not `/home/alan/data`. These were two different directory trees, even though both happened to live on the same filesystem.

## **2.2. Inspecting the Real Storage**

I resolved the paths and inspected the existing root-level directory:

```bash
readlink -f ./data
ls -ld /data
```

This revealed that `/data` already contained my Jellyfin library:

```text
/data
├── Anime/
├── Movies/
├── Music/
└── TV/
```

Instead of replacing that working layout, I preserved it and added the download side of the workflow:

```text
/data
├── torrents/
│   ├── incomplete/
│   ├── movies/
│   └── tv/
├── Anime/
├── Movies/
├── Music/
└── TV/
```

The directory names do not determine whether hardlinking works. The important conditions are that the source and destination reside on the same filesystem and remain visible through one shared container mount.

I kept the Compose project separately at `/home/alan/homelab` and removed the empty duplicate tree under `/home/alan/data`. This gives configuration and runtime data distinct lifecycles:

```text
/home/alan/homelab/  → Compose configuration and environment file
/data/                → media and torrent payloads
/opt/appdata/         → persistent application configuration and state
```

## **2.3. The Theory: Absolute and Relative Paths**

An **absolute path** begins at the filesystem root and has the same meaning regardless of the current working directory. A **relative path** is resolved from the process's current directory.

This difference is particularly important in infrastructure configuration. A valid but incorrect bind-mount source may not cause a container to crash. The container can start successfully and simply see the wrong or empty directory, producing a semantic configuration failure rather than a syntax error.

---

# **3. Proving the Filesystem and Hardlink Assumptions**

## **3.1. Filesystem Check**

I verified the mount and filesystem type with:

```bash
findmnt -T /data
df -T /data
```

The media library and torrent directories both reside on the same `ext4` filesystem. This satisfies the first hardlink requirement: a hardlink cannot cross a filesystem boundary because inode numbers are local to a filesystem.

## **3.2. Hardlink Test on the Real Paths**

I created an empty file in the actual movie download directory and linked it into the existing movie library:

```bash
touch /data/torrents/movies/hardlink-test
ln /data/torrents/movies/hardlink-test /data/Movies/hardlink-test

ls -li \
  /data/torrents/movies/hardlink-test \
  /data/Movies/hardlink-test
```

Both names displayed the same inode number and a link count of `2`. This proved that the two directory entries referenced the same underlying data rather than two copied files.

I then removed both test names. Removing one hardlink does not immediately remove the underlying data; the filesystem reclaims the data only when the inode's final link is removed and no process still has it open.

## **3.3. Jellyfin Read Boundary**

Radarr and Sonarr will eventually need write access to their managed libraries. Jellyfin only needs to scan and stream them. I tested the existing library using Jellyfin's real service account:

```bash
sudo -u jellyfin ls /data/Movies /data/TV /data/Anime >/dev/null

sudo -u jellyfin find \
  /data/Movies \
  /data/TV \
  /data/Anime \
  -type f ! -readable -print
```

The directory test succeeded, and the unreadable-file search returned nothing. This proved that Jellyfin retained read access after the ownership changes without granting it responsibility for managing the download workflow.

---

# **4. Preparing Persistent Container State**

Containers should be replaceable, but their configuration and databases must survive recreation. I created dedicated host directories beneath `/opt/appdata` for each planned service:

```text
/opt/appdata
├── jellyseerr/
├── minio/
├── postgres/
├── prowlarr/
├── qbittorrent/
├── radarr/
└── sonarr/
```

The high-level storage responsibilities are now:

| Location | Responsibility |
|---|---|
| `/data` | Large media and torrent payloads |
| `/opt/appdata` | Application configuration, databases, and internal state |
| Container writable layer | Temporary, replaceable runtime files |

I used bind mounts because they make the host locations explicit and easy to inspect while I am learning the system. The trade-off is that I must deliberately manage host ownership and permissions.

## **4.1. Secret Substitution**

PostgreSQL and MinIO credentials belong in a local `.env` file rather than as literal values in `docker-compose.yml`. A review caught that my Compose file still contained literal placeholder passwords. PostgreSQL and MinIO have not been started, so the next correction is to replace those placeholders with variable references that fail validation when the required values are absent:

```yaml
- POSTGRES_PASSWORD=${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
```

```yaml
- MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD:?Set MINIO_ROOT_PASSWORD in .env}
```

The `.env` file is restricted to the host user and excluded from version control. The Compose structure passed an initial quiet validation:

```bash
docker compose config --quiet
```

This command returned exit code `0`. After replacing the literal placeholders, I will run it again to validate the required variable substitution. Even then, it will prove only that the Compose model parses and the required variables resolve; it cannot prove that the applications themselves will start or behave correctly.

---

# **5. Defining the Network Exposure Boundary**

## **5.1. The Motivation**

A short Compose mapping such as:

```yaml
ports:
  - 8080:8080
```

publishes the port on every host interface by default. That is convenient, but a management interface should not automatically become reachable from every attached network.

I already use Tailscale for private remote access, so I bound the qBittorrent WebUI to the mini PC's Tailscale address:

```yaml
ports:
  - ${TAILSCALE_IP}:8080:8080
```

The actual address is stored locally and omitted here. qBittorrent's TCP and UDP peer ports remain separately published because peer traffic and administrative traffic have different purposes.

Before deployment, I used `ss` to verify that none of the planned host ports were already occupied. This avoids discovering a port collision only after Docker attempts to start the service.

## **5.2. Technology Introduction: Tailscale**

Tailscale creates an encrypted peer-to-peer overlay network based on WireGuard. Each authorized device receives a private tailnet address, allowing services to be reached remotely without publishing their management interfaces directly to the public internet.

Docker Compose also creates a private network for its services. Future containers will communicate using service DNS names such as `qbittorrent:8080`. Host port publishing is only required for clients outside that Compose network.

---

# **6. First Controlled Deployment: qBittorrent**

## **6.1. Why Start with One Service?**

Launching the entire stack would mix failures from image downloads, permissions, secrets, ports, storage, and application configuration. I started only qBittorrent so that the first deployment had a small failure domain:

```bash
docker compose up -d qbittorrent
```

Docker pulled the LinuxServer.io image, created the default Compose network, and started the container. The service status showed:

```text
qBittorrent peer traffic → host TCP/UDP port 6881
qBittorrent WebUI        → Tailscale interface only, port 8080
```

LinuxServer.io packages qBittorrent with a consistent container initialization layer. The `PUID` and `PGID` settings map the application process to the host identity that owns the media directories, avoiding files unexpectedly owned by root.

## **6.2. Verifying Mounts**

I inspected the running container rather than assuming the Compose declaration had been applied:

```bash
docker inspect qbittorrent \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

The result confirmed both important mappings:

```text
/opt/appdata/qbittorrent -> /config
/data                    -> /data
```

The first preserves qBittorrent's settings. The second maintains the unified path required for the later qBittorrent-to-Servarr workflow.

## **6.3. Avoiding a False-Positive Permission Test**

My first `docker exec` test ran as root. It could write to the torrent directories, but that did not prove the qBittorrent application user could do the same. `docker exec` uses the container's configured execution identity unless another user is specified, and the LinuxServer initialization process begins as root.

I repeated the check explicitly as UID/GID `1000:1000`:

```bash
docker exec --user 1000:1000 qbittorrent sh -c \
  'id; test -w /data/torrents/movies && echo "movies writable"; test -w /data/torrents/tv && echo "tv writable"'
```

The output identified the unprivileged user and confirmed that both category directories were writable. This is stronger evidence because the test reproduced the identity intended for the application process.

## **6.4. Credential Hygiene**

On first startup, qBittorrent generated a temporary administrator password in its logs. It also appeared in a terminal transcript, so it must be treated as exposed and replaced during the next WebUI configuration step. The permanent credential will not be included in this post.

This was a useful reminder that secrets can appear outside configuration files. Logs, terminal transcripts, screenshots, shell history, and diagnostic bundles all need the same care when they contain credentials.

---

# **7. Technical Interview Application**

**Q: A container starts successfully but sees an empty data directory. What would you investigate?**
> **A:** I would inspect the resolved bind-mount source, distinguish absolute from relative paths, and compare the declared mounts with `docker inspect`. A valid but incorrect host directory can allow the container to start while exposing the wrong data.

**Q: What conditions are required for a hardlink-based import?**
> **A:** The source and destination must be on the same filesystem, the application must see them through a mount topology that preserves that relationship, and its effective UID/GID must have permission to create the destination entry.

**Q: Why can a filesystem test as root be misleading?**
> **A:** Root may bypass restrictions that apply to the application's unprivileged user. A meaningful diagnostic reproduces the process's effective UID, GID, mounts, and paths.

**Q: What is the difference between a Docker image, container, and bind mount?**
> **A:** An image is the packaged, immutable application template. A container is a running instance of that image. A bind mount exposes a specific host path inside the container so data can persist independently of the container lifecycle.

**Q: Why bind an administrative port to a Tailscale address?**
> **A:** It narrows the listening interface to the authenticated private overlay instead of accepting connections on every host network. This reduces exposure while preserving remote management access.

**Q: Does `docker compose config --quiet` prove that the stack is healthy?**
> **A:** No. It proves that the configuration model parses and required variables resolve. Runtime health still depends on images, ports, permissions, mounts, dependencies, and application-level configuration.

---

# **8. Current Status and Next Step**

Today completed the storage and deployment preflight and started the first media automation component.

**Completed:**

* Corrected the `/home/alan/data` versus `/data` path mismatch.
* Preserved the existing Jellyfin library.
* Created the real torrent directory structure.
* Proved hardlinking using matching inode numbers.
* Verified Jellyfin's read access.
* Created persistent application-state directories.
* Validated the Compose structure and caught literal secret placeholders before starting PostgreSQL or MinIO.
* Restricted management interfaces to the Tailscale boundary.
* Deployed qBittorrent and verified its mounts and unprivileged write access.

**Next:**

* Set and verify qBittorrent's permanent WebUI credential.
* Configure the `movies` and `tv` categories and their save paths.
* Restart qBittorrent and confirm that its configuration persists.
* Deploy Prowlarr as the next independently validated service.

PostgreSQL and MinIO remain intentionally stopped. Phase 2 data ingestion will begin only after the complete request, download, import, hardlink, and Jellyfin discovery workflow is proven end to end.
