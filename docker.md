# Docker

## Stateless vs Stateful

Before containers make sense, split applications into two kinds:

| Type | Holds data? | Examples |
|------|-------------|----------|
| **Stateless** | No — no data lives inside the app | frontend, API/backend services, web servers |
| **Stateful** | Yes — data lives with the app | databases (MySQL, MongoDB), caches, message queues |

Stateless apps are easy to containerise and throw away — if one dies, spin up another and nothing is lost. Stateful apps need their data to survive restarts, so they need extra care (volumes, managed database services). This is why the first thing teams containerise is usually the stateless tier.

---

## How We Got Here — The Evolution

Two evolutions happened in parallel: **how applications are built** and **where they run**.

```
Application architecture:   Legacy  →  Monolithic  →  Microservices
Infrastructure:             Bare metal → Virtualisation (VM) → Containerisation
```

### Legacy / Monolithic — enterprise Java era

Early enterprise applications were one giant unit — frontend and backend packaged together and shipped as a single `.ear` (Enterprise Archive) file.

- Built with servlets, JSPs, JDBC, Struts, Seam, etc.
- Ran on heavy application servers — WebSphere, WebLogic, JBoss
- Deployed on **bare metal servers**: a physical Linux/Unix box + application server + Java runtime + the enterprise app

**The problems:**
- Provisioning a bare metal server took ~1 month
- A small change to one part meant rebuilding and redeploying **everything**
- One physical machine = one workload; if it sat idle, the hardware was wasted

The first improvement was to split frontend and backend and connect them through **REST/API services** — the beginning of the move away from a single monolith.

### Virtualisation — the VM era

**Virtualisation** slices one big bare metal server into multiple **logical machines (VMs)** using a hypervisor.

```
Bare metal: 128 GB RAM, 2 TB HDD
   └── VM1: 1 GB RAM, 8 GB HDD   ← reserved up front
   └── VM2: 1 GB RAM, 8 GB HDD   ← reserved up front
   └── ...
```

Better utilisation than bare metal — one physical box now runs many workloads (e.g. an Apache Tomcat server per VM).

**The catch — resource blocking.** When you give a VM 1 GB RAM and 8 GB disk, those resources are **reserved** whether the app uses them or not. Idle VMs still hold onto their slice, so capacity is wasted.

### Containerisation — the container era

Then came **digitalisation** — everything moving to online services — which pushed apps toward **microservices** (many small independently-deployable services) and infrastructure toward **containers**.

A container packages just the app and its dependencies and shares the host OS kernel. No full guest OS per workload.

**Boot time** — measured from "start it" to "application is up" — drops from minutes (VM) to **seconds** (container), and resources are used **dynamically** instead of being reserved up front.

```
Bare metal  →  VM  →  Container
   heavy         medium    light
   ~1 month      minutes   seconds
```

These build on each other — containers usually run **on** VMs, which run **on** bare metal.

---

## A Family Analogy

A nice way to feel the trade-offs — think of where people live:

| Living arrangement | Maps to | Freedom / Control | Cost & Maintenance |
|--------------------|---------|-------------------|--------------------|
| **Joint family** — big individual house, 30–40 members | **Bare metal** | Full control, total privacy, design it your way | High cost, you maintain everything, slow to build |
| **Small family** — one couple + kids, a flat | **VM** | Less privacy, shared space, follow building rules | Lower cost, low maintenance, quicker |
| **Individual** — single person in a PG / shared room | **Container** | No privacy, everything shared, zero control | Very low cost, no maintenance, no responsibilities, instant |

As you move from your own house → flat → shared room, you trade **control and privacy** for **cost, speed, and low maintenance**. Containers sit at the far end: cheapest and fastest, but the most shared. (The privacy/isolation concern is real — and it's something we can address with namespaces, cgroups, and proper configuration.)

---

## Why Containers — The Benefits

1. **Dynamic resource allocation** — no resource blocking; use what you need, when you need it
2. **Less boot time** — seconds, not minutes
3. **Less cost** — pack many containers onto one host
4. **Less size** — image contains only what the app needs, not a whole OS
5. **Portable** — same image runs on any machine with a container runtime ("works on my machine" → "works everywhere")
6. **Immutable** — the image never changes; to update, you build a new image and replace the container
7. **Scalable and reliable** — spin up more copies instantly; if one dies, replace it
8. **Isolation** — the "everything is shared" downside is mitigated by kernel isolation (namespaces + cgroups)

---

## Installing Docker (RHEL / Amazon Linux)

```bash
# add the docker plugin tooling and repo
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# install engine, CLI, containerd, and plugins
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# start the service
sudo systemctl enable --now docker
```

**Running docker without `sudo`:** only **root** or users in the **`docker` group** can talk to the Docker daemon. Add your user to the group:

```bash
sudo usermod -aG docker ec2-user
exit            # log out and back in for the new group to take effect
```

> Adding a user to the `docker` group effectively grants root-equivalent access on the host — do it deliberately.

---

## Image vs Container

This is the single most important idea in Docker.

- **Image** — a read-only template: bare-minimum OS + required packages + app runtime + app code + app libraries. Built once, never changes.
- **Container** — a **running instance** of an image. You can run many containers from one image.

The relationship is exactly like AWS:

```
AMI    →  Instance     (AWS)
Image  →  Container     (Docker)
```

**What goes into each:**

```
AMI    = OS + system packages + app runtime + app code + app libraries + service files
Image  = bare-min OS + required packages + app runtime + app code + app libraries
```

An image is leaner than an AMI — it carries only what the app needs and relies on the host kernel, so there's no full OS or service-manager layer.

---

## Core Commands

```bash
docker images                     # list images on this host
docker ps                         # running containers
docker ps -a                      # all containers (including stopped)

docker pull nginx:latest          # download an image (name:version/tag)
docker create nginx:latest        # create a container from an image (not started)
docker start <container-id>       # start a created/stopped container

docker rm <container-id>          # remove a container
docker rmi nginx                  # remove an image
```

### `docker run` = pull + create + start

`docker run` is a shortcut that does three things at once — pulls the image if it's missing, creates a container, and starts it:

```
docker run  =  docker pull  +  docker create  +  docker start
```

```bash
docker run nginx:latest                 # foreground
docker run -d nginx:latest              # -d = detached (background)
docker run -d -p 8080:80 nginx:latest   # -p host:container  → map ports
docker run -d --name web nginx:latest   # give the container a friendly name
```

> `:latest` is the default **tag** if you don't specify one. Pin a real version in production — `latest` moves and can surprise you.

---

## The Typical Lifecycle

```
pull image  →  create container  →  start  →  (running)  →  stop  →  rm
                                                             rmi (remove image)
```

Or collapsed: `docker run` to get going, `docker stop` + `docker rm` to clean up, `docker rmi` to reclaim the image.

---

## Quick Reference

| Concept | One-liner |
|---------|-----------|
| **Stateless** | App holds no data — safe to kill and recreate (frontend, APIs) |
| **Stateful** | App holds data — needs volumes / managed DB (databases, queues) |
| Legacy → Monolithic → Microservices | Evolution of application architecture |
| Bare metal → VM → Container | Evolution of infrastructure; each runs on the one before |
| **Bare metal** | One physical box per workload; full control, high cost, slow (~1 month) |
| **Virtualisation / VM** | Split one box into logical machines; resources are **reserved** (blocked) |
| **Containerisation** | Share the host kernel; dynamic resources, boot in seconds |
| Resource blocking | VM reserves RAM/disk up front even when idle — containers don't |
| Boot time | Time from start to app-up — seconds for containers, minutes for VMs |
| **Image** | Read-only template (bare-min OS + runtime + code + libs); never changes |
| **Container** | Running instance of an image; many containers per image |
| Image → Container | Same relationship as AMI → Instance in AWS |
| `docker images` | List images on the host |
| `docker ps` / `docker ps -a` | Running containers / all containers |
| `docker pull name:tag` | Download an image |
| `docker create` | Make a container from an image (not started) |
| `docker start` / `docker stop` | Start / stop a container |
| `docker rm` / `docker rmi` | Remove a container / remove an image |
| `docker run` | pull + create + start in one command |
| `-d` | Detached — run the container in the background |
| `-p host:container` | Publish/map a port from host to container |
| `--name` | Give a container a friendly name |
| `docker` group | Members can run docker without `sudo` (≈ root on host) |

---
