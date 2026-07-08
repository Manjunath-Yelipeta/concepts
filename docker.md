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

**Bulk cleanup** — remove all containers at once (`-q` prints just the IDs, `-f` forces removal even if running):

```bash
docker rm -f `docker ps -a -q`
```

---

## Where Docker Stores Everything

Everything Docker downloads and creates lives under one directory on the host:

```
DOCKER_HOME = /var/lib/docker
```

Downloaded images, containers, and volumes all pile up here. It grows fast — an install that starts at ~2 GB can jump to 30 GB+ as you pull images and run containers. On a server, mount a dedicated/large disk at `/var/lib/docker` so it doesn't fill the root filesystem.

---

## Docker Hub & Registries

When you `docker pull nginx`, where does the image come from? A **registry** — a store of images. The default public one is **Docker Hub** (`docker.io`).

```
docker pull nginx
   └── image not found locally?  → pull it from Docker Hub
   └── image already local?      → use the local copy
```

An image's full name has four parts:

```
docker.io / joindevops / from : v1
  registry    username   image  tag
```

If you don't type the registry and username, Docker assumes `docker.io` and an official library image — so `nginx` really means `docker.io/library/nginx:latest`.

---

## Working With Running Containers

The container should run in the **foreground** (that's what keeps it alive), but you push it to the **background** with `-d` so it doesn't lock up your terminal. Once it's running detached, these commands let you look inside and manage it:

```bash
docker exec -it <container-id> bash   # open an interactive shell inside a running container
docker logs <container-id/name>       # view the container's stdout/stderr logs
docker stats                          # live resource usage (CPU, memory, network) per container
docker inspect <container-id/name>    # full low-level details as JSON (IP, mounts, env, config)
```

- `docker exec -it` — `-i` keeps input open, `-t` gives a terminal; together they drop you into a shell **inside** the container to debug it
- `docker logs` — first place to look when a container misbehaves or exits
- `docker stats` — the container equivalent of `top`
- `docker inspect` — everything Docker knows about the object, in JSON

---

## Ports: Host ↔ Container

A container has its own full port range (0–65,535), isolated from the host. To reach a service inside a container, the request must enter through a **host port**, which Docker forwards to a **container port**.

```
client → host-port → container-port → app inside container
```

```bash
docker run -d -p host-port:container-port nginx
docker run -d -p 80:80 --name nginx nginx:latest
```

`-p 80:80` means "traffic hitting port 80 on the host is forwarded to port 80 in the container." The two numbers don't have to match — `-p 8080:80` maps host 8080 → container 80.

---

## Dockerfile

A **Dockerfile** is a set of instructions used to build a **custom image**. Instead of manually installing things in a container and saving it, you declare the steps and Docker builds a repeatable image from them.

Each instruction is written in UPPERCASE and (mostly) adds a layer to the image.

### FROM — base image

`FROM` **must be the first instruction**. It sets the base image everything else builds on.

```dockerfile
# dockerfiles/FROM/Dockerfile — almalinux is a clone of RHEL
FROM almalinux:9
```

```
FROM base-os:version
```

### RUN — build-time commands

`RUN` executes a command **while building the image** — typically to install packages. A Dockerfile can have **many** `RUN` instructions.

```dockerfile
# dockerfiles/RUN/Dockerfile
FROM almalinux:9
RUN dnf install nginx -y
```

### CMD — the container's start command

An image is a **static file**. To turn it into a container, you need something that **runs indefinitely** — otherwise the container starts and immediately exits.

`systemctl` won't work here: it needs kernel/init access that containers don't have. So instead of `systemctl start nginx`, you run the binary directly in the **foreground**.

```dockerfile
# dockerfiles/CMD/Dockerfile
FROM almalinux:9
RUN dnf install nginx -y
# Exec form — daemon off keeps nginx in the foreground so the container stays alive
CMD ["nginx", "-g", "daemon off;"]
```

`CMD` runs **when a container is created** from the image (not during build). `daemon off;` stops nginx from forking to the background — the process stays in the foreground and keeps the container running.

### RUN vs CMD

| | `RUN` | `CMD` |
|---|-------|-------|
| When it runs | While **building** the image | While **starting** the container |
| How many allowed | Many | Only **one** takes effect (last one wins if you write several) |
| Purpose | Install packages, set up the image | Define the process that keeps the container alive |

### EXPOSE — document the port

`EXPOSE` documents which port the containerised app listens on. It's **informational only** — it does **not** actually open or publish the port. You still need `-p` at run time to reach it.

```dockerfile
# dockerfiles/EXPOSE/Dockerfile
FROM almalinux:9
RUN dnf install nginx -y
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### ENV — environment variables

`ENV` sets key-value pairs available as environment variables inside the container. Programs running in the container read these for their configuration.

```dockerfile
# dockerfiles/ENV/Dockerfile
FROM almalinux:9
ENV course="docker" \
    trainer="sivakumar" \
    duration="10hrs"
CMD ["sleep", "500"]
```

### LABEL — image metadata

`LABEL` attaches metadata (author, purpose, version) to the image. Visible via `docker inspect`.

```dockerfile
# dockerfiles/LABEL/Dockerfile
FROM almalinux:9
LABEL author="sivakumar" \
      course="docker" \
      duration="10hrs"
```

### COPY — copy local files into the image

`COPY` copies files from your local build context into the image. Common use — put a website's files into nginx's web root.

```dockerfile
# dockerfiles/COPY/Dockerfile
FROM nginx
RUN rm -rf /usr/share/nginx/html/*    # clear the default nginx page
COPY qi/ /usr/share/nginx/html/       # copy our site in
```

### ADD — copy, plus two extras

`ADD` does everything `COPY` does, with two extra abilities:

1. It can **fetch content directly from a URL**
2. It can **automatically extract a tar archive** into the destination

```dockerfile
# dockerfiles/ADD/Dockerfile
FROM almalinux:9
ADD https://raw.githubusercontent.com/daws-90s/notes/refs/heads/main/session-01.txt /tmp/
ADD sample-1.tar /tmp/                # auto-extracted into /tmp
CMD ["sleep", "100"]
```

> Best practice: prefer `COPY` for plain file copies (it's explicit and predictable); reach for `ADD` only when you actually need the URL download or auto-extract behaviour.

### ENTRYPOINT — the un-overridable start command

`ENTRYPOINT` also defines what runs when the container starts, but it behaves differently from `CMD`:

1. A `CMD` command **can be overridden** at run time (`docker run image <new-command>`)
2. An `ENTRYPOINT` **cannot be overridden** — anything you pass at run time is **appended** to it
3. `CMD` can supply **default arguments** to `ENTRYPOINT` — and those defaults *can* be overridden at run time

Used together, `ENTRYPOINT` fixes the command and `CMD` provides a default argument:

```dockerfile
# dockerfiles/ENTRYPOINT/Dockerfile
FROM almalinux:9
ENTRYPOINT ["ping"]     # always runs ping — can't be changed
CMD ["yahoo.com"]       # default target — overridable
```

- `docker run img` → runs `ping yahoo.com`
- `docker run img facebook.com` → runs `ping facebook.com` (the `CMD` default is replaced, `ENTRYPOINT` stays)

### CMD vs ENTRYPOINT

| | `CMD` | `ENTRYPOINT` |
|---|-------|--------------|
| Overridable at run time | Yes — fully replaced | No — run-time args are appended |
| Typical role | The whole command, or default args for ENTRYPOINT | The fixed command that always runs |
| Best combo | Provide default arguments | Lock the executable; let CMD fill the args |

---

## Building, Tagging & Pushing Images

### Build

`docker build` reads a Dockerfile and produces an image. `-t` tags it (`name:version`); the `.` is the **build context** (the current directory).

```bash
docker build -t from:v1 .
```

### Tag for a registry

`from:v1` is just a **local** image. To push it, tag it with the full registry path (`registry/username/image:version`):

```bash
docker tag from:v1 docker.io/joindevops/from:v1
```

You can also build with the full name directly:

```bash
docker build -t joindevops/from:v1 .
```

### Login & push

```bash
docker login -u <username>
docker push joindevops/from:v1
```

After pushing, anyone can `docker pull joindevops/from:v1` from Docker Hub.

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
| `docker exec -it <c> bash` | Open an interactive shell inside a running container |
| `docker logs <c>` | View a container's stdout/stderr — first stop for debugging |
| `docker stats` | Live CPU/memory/network usage per container (like `top`) |
| `docker inspect <c>` | Full low-level details of an object as JSON |
| `docker rm -f \`docker ps -a -q\`` | Force-remove **all** containers |
| `/var/lib/docker` | `DOCKER_HOME` — where images, containers, volumes are stored; grows fast |
| **Registry / Docker Hub** | Store of images; `docker.io` is the default public one |
| Image full name | `registry/username/image:tag` (e.g. `docker.io/joindevops/from:v1`) |
| **Dockerfile** | Set of instructions to build a custom image |
| `FROM` | First instruction — sets the base image |
| `RUN` | Runs a command at **build** time (install packages); many allowed |
| `CMD` | Command that runs when the **container starts**; only one takes effect; overridable |
| `EXPOSE` | Documents the listening port — informational only, doesn't publish it |
| `ENV` | Set environment variables inside the container |
| `LABEL` | Attach metadata (author, version) to the image |
| `COPY` | Copy local files into the image |
| `ADD` | Like COPY, plus fetch from URL and auto-extract tar archives |
| `ENTRYPOINT` | Fixed start command — can't be overridden; run-time args are appended |
| ENTRYPOINT + CMD | ENTRYPOINT locks the executable; CMD supplies overridable default args |
| `daemon off;` | Keeps a service (e.g. nginx) in the foreground so the container stays alive |
| `docker build -t name:tag .` | Build an image from a Dockerfile (`.` = build context) |
| `docker tag <img> <full-name>` | Retag a local image for a registry |
| `docker login -u <user>` / `docker push` | Authenticate to and upload an image to a registry |

---
