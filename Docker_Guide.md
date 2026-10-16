# Docker: a complete guide

Docker packages an application and everything it needs to run — the language runtime, libraries, config, and start script — into an **image**. A running copy of that image is a **container**. The same image starts the same way on a laptop, a build server, and a production machine.

This guide covers the engine, Docker Desktop, Docker Hub, Docker Compose, Dockerfiles, entrypoint scripts, and how those pieces fit into CI/CD. Examples use a small Python API so each file is complete enough to copy. The last section maps the same ideas onto this repository (SAF).

---

## Contents

1. [What problem Docker solves](#1-what-problem-docker-solves)
2. [Container and virtual machine](#2-container-and-virtual-machine)
3. [Words you will see everywhere](#3-words-you-will-see-everywhere)
4. [How Docker is built](#4-how-docker-is-built)
5. [Install](#5-install)
6. [Docker Desktop](#6-docker-desktop)
7. [Your first container](#7-your-first-container)
8. [Images, names, tags, and digests](#8-images-names-tags-and-digests)
9. [Everyday commands](#9-everyday-commands)
10. [The Dockerfile](#10-the-dockerfile)
11. [The entrypoint script](#11-the-entrypoint-script)
12. [The build context and .dockerignore](#12-the-build-context-and-dockerignore)
13. [Build cache, BuildKit, multi-stage, and multi-arch](#13-build-cache-buildkit-multi-stage-and-multi-arch)
14. [Running a container well](#14-running-a-container-well)
15. [Networking](#15-networking)
16. [Storage](#16-storage)
17. [Docker Hub](#17-docker-hub)
18. [Other registries](#18-other-registries)
19. [Docker Compose](#19-docker-compose)
20. [Worked example: API, database, and worker](#20-worked-example-api-database-and-worker)
21. [Use cases](#21-use-cases)
22. [CI/CD](#22-cicd)
23. [Security](#23-security)
24. [Disk, cleanup, and debugging](#24-disk-cleanup-and-debugging)
25. [How SAF uses Docker](#25-how-saf-uses-docker)
26. [Concept index](#26-concept-index)

---

## 1. What problem Docker solves

Without containers, “it works on my machine” usually means the machine has a particular Python, a particular system library, and a start command nobody wrote down. A new laptop, a colleague, and the server each drift.

Docker makes four promises:

| Promise | What it means in practice |
|---------|---------------------------|
| **Repeatable build** | A `Dockerfile` is the recipe. Anyone with the source can build the same image. |
| **Isolated run** | The process sees its own filesystem, its own network stack, and limits you set. It does not install packages onto the host. |
| **One start command** | `docker run` or `docker compose up` replaces a page of setup notes. |
| **Portable artifact** | You push the image to a registry. The server pulls that image. It does not need your compiler or your virtualenv. |

Docker does not replace the application. It wraps it. Django is still Django. Postgres is still Postgres. Docker is how you build, ship, and start them.

---

## 2. Container and virtual machine

A **virtual machine** boots a full operating system on virtual hardware. Each VM has its own kernel. That is strong isolation and a heavy start: minutes, gigabytes, a hypervisor.

A **container** is a process (or a small process tree) on the **host kernel**. Linux features keep it apart from other processes:

- **Namespaces** give it its own view of process IDs, network, mounts, hostnames, and users.
- **cgroups** cap its CPU, memory, and process count.
- **A layered filesystem** shows it the image contents plus a writable layer for that one container.

Containers start in seconds and share the kernel, so they are the usual unit for application processes. VMs remain the right tool when you need a different kernel, a different OS, or a hard boundary between tenants. Docker Desktop on macOS and Windows runs a small Linux VM, because containers need a Linux kernel, and then runs containers inside that VM.

```text
Virtual machine                         Container
-----------------------------           -----------------------------
App                                     App
Guest libraries                         Image libraries
Guest kernel                            (host kernel, isolated view)
Hypervisor                              containerd + runc
Host kernel                             Host kernel
```

---

## 3. Words you will see everywhere

| Word | Plain meaning |
|------|----------------|
| **Image** | A read-only package: filesystem layers plus a default command and metadata. Built from a Dockerfile or pulled from a registry. |
| **Container** | A running (or stopped) instance of an image, with a thin writable layer on top. |
| **Dockerfile** | The text recipe that builds an image. |
| **Layer** | One filesystem change from a Dockerfile instruction (`RUN`, `COPY`, …). Layers stack. Unchanged layers are reused from cache and shared between images. |
| **Build context** | The folder sent to the engine when you build. Usually the directory that contains the Dockerfile. |
| **Tag** | A human name for an image, such as `myapp:1.4.0` or `myapp:latest`. A tag can move. |
| **Digest** | The immutable id of an image, `sha256:…`. A digest never moves. |
| **Registry** | A server that stores images. Docker Hub is the public default. |
| **Repository** | A named collection of tags for one application, such as `library/postgres` or `yourname/notes-api`. |
| **Volume** | Docker-managed disk for data that must survive container replacement. |
| **Bind mount** | A host folder attached into the container. Typical for source code during development. |
| **Network** | A virtual network containers join so they can reach each other by name. |
| **Dockerfile `ENTRYPOINT`** | The program that always runs when the container starts. |
| **Dockerfile `CMD`** | Default arguments, or the default program if there is no entrypoint. Replaced when you pass a command on `docker run` or in Compose. |
| **Entrypoint script** | A shell script used as the entrypoint: wait for dependencies, migrate, then `exec` the real server. |
| **Compose** | A YAML file that describes several containers, their networks, and their volumes, and starts them together. |
| **Engine** | The Linux daemon (`dockerd`) plus the `docker` CLI. |
| **Docker Desktop** | The desktop application that installs the engine (inside a VM on Mac and Windows), Compose, Buildx, and a GUI. |
| **BuildKit / buildx** | The modern image builder. Parallel stages, cache mounts, secrets during build, multi-platform images. |
| **OCI** | The open standard for image layout and the container runtime. Docker images are OCI images. |

---

## 4. How Docker is built

When you type `docker run`, several programs cooperate. You rarely touch the lower ones, but their names show up in errors and in CI.

```text
docker CLI  --->  dockerd (Engine API)  --->  containerd  --->  runc  --->  your process
   you              builds, networks,            supervises              creates the        PID 1 in
                    volumes, images              containers              Linux container    the container
```

| Piece | Role |
|-------|------|
| **Docker CLI** | What you type. It talks to the engine over a socket (`/var/run/docker.sock` on Linux). |
| **Docker Engine (`dockerd`)** | Builds images, creates networks and volumes, and asks containerd to run containers. |
| **containerd** | Industry-standard supervisor. Kubernetes uses it too, often without the Docker Engine. |
| **runc** | The low-level OCI runtime. It creates the namespaces and starts the process. |
| **containerd-shim** | Keeps the container alive if the daemon restarts, and reaps the process when it exits. |
| **overlayfs (overlay2)** | The storage driver that stacks image layers. This is the default on modern Linux. |
| **BuildKit** | Builds images. The CLI plugin is `docker buildx`. |
| **Compose plugin** | `docker compose` (a space, not a hyphen). It is a CLI plugin, not a separate Python tool. |

**Context.** The CLI can point at different engines. `docker context ls` shows them. Docker Desktop creates a `desktop-linux` context. A remote server can be a context too (`docker context use`).

**Rootless mode.** The engine can run as a normal user instead of root. It is safer and slightly less capable (some network modes and low ports need extra setup). The installer is `dockerd-rootless-setuptool.sh`.

**Swarm.** Docker Swarm is the engine’s older built-in cluster mode (`docker swarm init`, services, overlay networks across machines). New clusters are usually Kubernetes. Compose on one machine, or Kubernetes for many machines, covers almost every current project. The Compose `deploy:` key is a leftover of Swarm stacks; on a single host, Compose ignores most of it unless you use Swarm.

---

## 5. Install

Pick **one** of these for a machine. Docker Desktop includes the engine. On a Linux server you usually install the engine only, with no GUI.

### 5.1 Linux server or laptop: Docker Engine

These steps match Docker’s apt repository for Ubuntu. For Debian, Fedora, or RHEL, use the same package names from Docker’s docs for that distro (`docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`).

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Allow your user to talk to the engine without `sudo` (log out and back in afterwards):

```bash
sudo usermod -aG docker "$USER"
```

The engine starts on boot via systemd:

```bash
sudo systemctl enable --now docker
docker version
docker compose version
```

You should see a **Client** section and a **Server** section. If the server section is missing, the daemon is not running or your user cannot access the socket.

### 5.2 Windows: Docker Desktop

1. Install **WSL 2** (Windows Subsystem for Linux). Docker Desktop’s backend is a Linux VM; WSL 2 is how Windows provides it.
2. Download Docker Desktop from [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/).
3. During setup, leave **Use WSL 2 based engine** enabled.
4. Start Docker Desktop and wait until it says the engine is running.
5. In a terminal: `docker version` and `docker compose version`.

Files you bind-mount from the Windows filesystem (`C:\...`) are slower than files inside the WSL filesystem (`\\wsl$\Ubuntu\home\...`). Keep source code inside WSL when containers need to read it on every request.

### 5.3 macOS: Docker Desktop

1. Download the Apple Silicon or Intel build of Docker Desktop.
2. Open the disk image and drag Docker to Applications.
3. Start it and accept the privileged helper (it needs this to run the Linux VM).
4. Check `docker version`.

On a Mac there is no Linux kernel. Every container runs inside Desktop’s VM. Bind-mounted macOS folders cross that boundary, so large projects and databases feel faster with named volumes than with bind mounts of `node_modules` or Postgres data.

### 5.4 Linux desktop: Docker Desktop or Engine

Both work. Desktop adds the GUI, a Kubernetes option, and automatic updates. The engine from section 5.1 is enough for Compose-based development and is what servers use.

### 5.5 Check that install worked

```bash
docker run --rm hello-world
```

That pulls a tiny image, starts a container, prints a short message, and deletes the container (`--rm`). If this works, the client, daemon, registry pull, and runtime are all fine.

---

## 6. Docker Desktop

Docker Desktop is a product wrapped around the engine. Use it on a developer machine. Use the engine alone on a server.

### What it includes

| Part | What you use it for |
|------|---------------------|
| **Engine** | Same `docker` commands as on Linux. On Mac and Windows the daemon runs in a Linux VM. |
| **Compose v2** | `docker compose` is already installed. |
| **buildx** | Multi-platform builds and the modern builder. |
| **Credential helper** | Stores `docker login` tokens in the OS keychain. |
| **GUI dashboard** | Lists containers, images, volumes, and logs. You can start, stop, delete, and open a shell. |
| **Kubernetes** | Optional single-node cluster for learning or for manifests you will later run in a real cluster. Turn it on in Settings only when you need it; it uses extra RAM. |
| **Extensions** | Optional GUI add-ons (disk usage, logs). The engine does not need them. |
| **Dev Environments / Compose Watch** | Rebuild or sync files into containers when you save code. |
| **Resource limits** | Settings → Resources: CPU, memory, swap, and disk for the Linux VM. Give the VM enough RAM for your databases. A Compose stack with Postgres, Redis, and an app often wants 4 GB or more. |
| **File sharing** | On Mac, only shared folders can be bind-mounted. On Windows with WSL 2, share through the WSL distro. |

### Settings that matter

- **Start Docker Desktop when you sign in** if you use containers daily.
- **Use containerd image store** (the default on current Desktop) so images sit in the same store BuildKit expects.
- **Expose daemon on tcp without TLS** stays off. The socket is enough. An open TCP daemon is an unauthenticated root shell.
- **Docker Scout** in Desktop shows known vulnerabilities in local images. Treat it as a hint, and scan again in CI.

### Licensing

Docker Engine and the Compose plugin are open source. Docker Desktop is free for personal use, education, non-commercial open source, and many small businesses, and requires a paid subscription above Docker’s current employee and revenue thresholds. Check the current subscription page before rolling Desktop out to a company. Servers do not need Desktop.

### Desktop versus Engine, in one picture

```text
Your laptop (Mac / Windows)                 Linux server
-----------------------------               -----------------------------
Docker Desktop GUI                          (no GUI)
   Linux VM                                    host kernel
      dockerd + containerd                      dockerd + containerd
         containers                                containers
```

Commands in the rest of this guide are the same on both.

---

## 7. Your first container

```bash
docker run --rm -it ubuntu:24.04 bash
```

| Flag | Meaning |
|------|---------|
| `run` | Create a container from an image and start it. Pull the image if it is not local. |
| `--rm` | Delete the container when it exits. |
| `-it` | Interactive terminal. `-i` keeps stdin open, `-t` allocates a TTY. Use both for a shell. |
| `ubuntu:24.04` | Image name and tag. |
| `bash` | Command inside the container. It replaces the image’s default command. |

`exit` stops that container. With `--rm`, it is gone. The `ubuntu:24.04` image stays cached.

A long-running process, published on a host port:

```bash
docker run -d --name web -p 8080:80 nginx:1.27
```

| Flag | Meaning |
|------|---------|
| `-d` | Detached. The CLI returns, the container keeps running. |
| `--name web` | A name you can use instead of the random id. |
| `-p 8080:80` | Host port 8080 forwards to container port 80. |
| `nginx:1.27` | Official Nginx image. |

Open `http://127.0.0.1:8080`. Then:

```bash
docker logs web
docker exec -it web sh
docker stop web
docker rm web
```

`docker exec` starts a **second** process in an already running container. It is how you inspect a live app. It is not how the app itself starts.

---

## 8. Images, names, tags, and digests

An image name has four parts. Most of them have defaults.

```text
[registry-host[:port]/][namespace/]repository[:tag]

docker.io / library / nginx : 1.27
registry      namespace  repo   tag
```

| Example | Meaning |
|---------|---------|
| `nginx` | `docker.io/library/nginx:latest` |
| `nginx:1.27` | Official image, tag `1.27`. |
| `yourname/notes-api:1.4.0` | Your Docker Hub repository. |
| `ghcr.io/acme/notes-api:1.4.0` | GitHub Container Registry. |
| `localhost:5000/notes-api:dev` | A registry running on your machine. |

**`latest` is a tag like any other.** It is not “the newest version” unless whoever publishes the image moves it. Pin production to a version tag and, when you can, to a digest:

```bash
docker pull nginx:1.27
docker image inspect nginx:1.27 --format '{{index .RepoDigests 0}}'
docker run --rm nginx@sha256:the-digest-here
```

A tag can be retargeted. A digest cannot. CI should record the digest it deployed.

**Official images** live under `library/` on Docker Hub (`python`, `postgres`, `redis`, `nginx`). Prefer a specific major or minor tag (`python:3.12-slim`) over `python:latest`, so a rebuild next month does not jump a major version.

**Where images live on disk.** Layers sit in the engine’s data root, usually `/var/lib/docker` on Linux, and inside Desktop’s VM on Mac and Windows. You manage them with `docker image` and `docker system`, not by copying those folders.

---

## 9. Everyday commands

Commands are grouped by object. `docker container ls` and `docker ps` are the same list. The long form (`docker container …`) matches the docs; the short form is what people type.

### Containers

| Command | What it does |
|---------|----------------|
| `docker ps` | Running containers. |
| `docker ps -a` | Running and stopped. |
| `docker run` | Create and start. |
| `docker create` | Create without starting. |
| `docker start` / `stop` / `restart` | Control an existing container. |
| `docker kill` | SIGKILL immediately. `stop` sends SIGTERM, waits, then SIGKILL. |
| `docker pause` / `unpause` | Freeze the processes with the freezer cgroup. Rare. |
| `docker rm` | Remove a stopped container. `-f` forces a running one. |
| `docker logs -f --tail 100 name` | Stdout and stderr. `-f` follows. |
| `docker exec -it name sh` | Extra shell in a running container. |
| `docker cp name:/app/file ./file` | Copy a file out. The reverse path copies in. |
| `docker inspect name` | JSON: IP, mounts, env, state. |
| `docker stats` | Live CPU and memory. |
| `docker top name` | Processes inside. |
| `docker port name` | Which host ports are published. |
| `docker diff name` | Files changed in the writable layer. |
| `docker rename old new` | Rename. |
| `docker update --memory 512m name` | Change limits on a running container. |
| `docker wait name` | Block until it exits; print the exit code. |
| `docker events` | Stream engine events (create, start, die). |

### Images

| Command | What it does |
|---------|----------------|
| `docker images` | Local images. |
| `docker pull nginx:1.27` | Download. |
| `docker build -t notes-api:dev .` | Build from the Dockerfile in `.` |
| `docker tag notes-api:dev yourname/notes-api:1.4.0` | Add another name to the same image id. |
| `docker push yourname/notes-api:1.4.0` | Upload. |
| `docker rmi notes-api:dev` | Remove a local tag. The layers go away when no tag or container references them. |
| `docker history notes-api:dev` | Layers and the command that created each. |
| `docker image inspect notes-api:dev` | Full metadata. |
| `docker save` / `docker load` | Export and import images as a tar, for machines with no registry. |
| `docker image prune` | Remove dangling images (layers with no tag). |

### Builder, volumes, networks, system

| Command | What it does |
|---------|----------------|
| `docker buildx build` | Build with BuildKit, including multi-platform. |
| `docker buildx ls` | Builders. |
| `docker volume ls` / `create` / `inspect` / `rm` | Named volumes. |
| `docker network ls` / `inspect` | Networks. |
| `docker system df` | Disk used by images, containers, volumes. |
| `docker system prune` | Remove stopped containers, unused networks, dangling images. |
| `docker login` / `logout` | Registry credentials. |
| `docker context ls` / `use` | Which engine the CLI talks to. |

`docker compose` commands are in [section 19](#19-docker-compose).

---

## 10. The Dockerfile

A Dockerfile is a list of steps. Each instruction creates a layer or sets metadata for the containers you will run later.

### 10.1 A complete production Dockerfile

This is the pattern to copy: small base, dependencies before source (so code edits do not reinstall packages), a non-root user, an entrypoint script, and an exec-form command.

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.12-slim AS base

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq5 \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY docker/entrypoint.sh /app/docker/entrypoint.sh
COPY src /app/src

RUN chmod +x /app/docker/entrypoint.sh \
    && useradd --create-home --shell /bin/bash app \
    && chown -R app:app /app

USER app

ENV PORT=8080
EXPOSE 8080

ENTRYPOINT ["/app/docker/entrypoint.sh"]
CMD ["gunicorn", "src.wsgi:application", "--bind", "0.0.0.0:8080"]
```

`# syntax=docker/dockerfile:1` tells BuildKit to use the current Dockerfile frontend, which unlocks cache mounts, heredocs, and `COPY --chmod`.

### 10.2 Every instruction

| Instruction | What it does | Notes |
|-------------|--------------|-------|
| `FROM` | Starts a stage from a base image. | The first instruction (after a parser directive or comments). `FROM scratch` is an empty image for a static binary. |
| `RUN` | Runs a command **during the build** and commits the filesystem change. | Chain commands in one `RUN` and clean package lists in that same `RUN`, or the deleted files stay in an earlier layer. |
| `COPY` | Copies files from the build context into the image. | Preferred over `ADD`. Use `COPY --from=stage` to take files from another stage. |
| `ADD` | Like `COPY`, plus it can unpack a local tar and fetch a URL. | The extra behavior surprises people. Use `COPY` unless you need the tar unpack. |
| `WORKDIR` | Sets the working directory for later instructions and for the container. | Creates the directory. Use absolute paths. |
| `ENV` | Sets an environment variable in the image. Later `RUN` steps and the running container see it. | Baked into the image. Fine for non-secrets (`PORT`, `PYTHONUNBUFFERED`). |
| `ARG` | A build-time variable. | Available only during the build, unless you assign it to `ENV`. Changing an `ARG` invalidates cache from that line down. Do not pass secrets as `ARG`; they remain in history. |
| `EXPOSE` | Documents which port the process listens on. | Does **not** publish the port. Publishing is `-p` or Compose `ports`. |
| `USER` | The user later `RUN`, `CMD`, and `ENTRYPOINT` use. | Create the user first. Running as root in production is avoidable for most apps. |
| `CMD` | Default command or default arguments. | Overridden by `docker run … command` and by Compose `command:`. |
| `ENTRYPOINT` | The executable that starts. | Not replaced by extra `docker run` arguments; those become arguments to the entrypoint. Override with `--entrypoint`. |
| `VOLUME` | Marks a path as a mount point. | Creates an anonymous volume if you do not mount one. Prefer declaring volumes in Compose so names and lifetimes are obvious. |
| `LABEL` | Metadata (`org.opencontainers.image.source`, version, maintainer). | OCI labels are the standard set. |
| `HEALTHCHECK` | A command the engine runs to decide healthy or unhealthy. | Compose and orchestrators use this. |
| `SHELL` | The shell used by shell-form `RUN`, `CMD`, and `ENTRYPOINT`. | Default is `["/bin/sh", "-c"]`. |
| `STOPSIGNAL` | Signal sent on `docker stop`. | Default TERM. Some apps want QUIT. |
| `ONBUILD` | Instructions saved for images that later use this image as `FROM`. | Rare. Easy to surprise downstream builds. |
| `MAINTAINER` | Old author field. | Replaced by `LABEL`. |

**Parser directives** sit at the very top: `syntax` and `escape` (Windows Dockerfiles sometimes set `escape=\``).

### 10.3 `ENTRYPOINT` and `CMD` together

Docker builds the final command like this:

```text
ENTRYPOINT  +  CMD
fixed program   default arguments
```

| Dockerfile | `docker run image` | `docker run image extra` |
|------------|--------------------|--------------------------|
| `CMD ["nginx", "-g", "daemon off;"]` | `nginx -g daemon off;` | `extra` **replaces** the whole command |
| `ENTRYPOINT ["entrypoint.sh"]` and `CMD ["gunicorn", …]` | `entrypoint.sh gunicorn …` | `entrypoint.sh extra` |
| shell form `CMD gunicorn …` | `/bin/sh -c 'gunicorn …'` | replaced the same way |

**Exec form** is the JSON array: `CMD ["gunicorn", "app:app"]`. The process is PID 1 and receives signals.

**Shell form** is a bare string: `CMD gunicorn app:app`. Docker wraps it in `/bin/sh -c`. The shell is PID 1, and the app often never sees SIGTERM, so `docker stop` waits the full timeout and then kills it. Use exec form.

A useful entrypoint script **does** use a shell, and its last line is `exec` so the app replaces the shell and becomes PID 1. That is the whole point of [section 11](#11-the-entrypoint-script).

### 10.4 `ARG` versus `ENV`

```dockerfile
ARG INCLUDE_DEV_TOOLS=false
RUN if [ "$INCLUDE_DEV_TOOLS" != "true" ]; then rm -rf /app/dev_tools; fi

ENV PORT=8080
```

- `ARG` exists while the image is built. `docker build --build-arg INCLUDE_DEV_TOOLS=true` sets it. It is not in the running container unless you copy it: `ENV INCLUDE_DEV_TOOLS=$INCLUDE_DEV_TOOLS`.
- `ENV` is in the image and in every container. Compose `environment:` overrides it at run time.

Build-arg values are visible in `docker history`. Never put passwords, tokens, or API keys in `ARG` or `ENV`.

### 10.5 Layer rules that keep images small and rebuilds fast

1. Put instructions that change rarely **above** instructions that change often. System packages, then `requirements.txt` and `pip install`, then application source.
2. One `RUN` for “install and clean” so the cleanup is in the same layer as the install.
3. `--no-install-recommends` on apt, and `rm -rf /var/lib/apt/lists/*` in that same `RUN`.
4. `pip install --no-cache-dir` so pip’s download cache is not stored in the layer.
5. A `.dockerignore` so the context does not contain `.git`, virtualenvs, or secrets.
6. Multi-stage builds leave compilers in an earlier stage ([section 13](#13-build-cache-buildkit-multi-stage-and-multi-arch)).

### 10.6 `HEALTHCHECK` in the Dockerfile

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health/')"
```

States you will see in `docker ps`: `starting` (during `start-period`), `healthy`, `unhealthy`. Compose `depends_on` can wait for `service_healthy`.

---

## 11. The entrypoint script

The image’s entrypoint is the place for **startup work that is not the server itself**: wait until the database accepts connections, run migrations, then hand the process slot to the server.

Put the script in the repo (for example `docker/entrypoint.sh`) and copy it in the Dockerfile. A script created only inside a `RUN echo` is harder to review.

### 11.1 `docker/entrypoint.sh`

```sh
#!/bin/sh
# Wait for Postgres, migrate, then replace this shell with the server.
set -eu

echo "[entrypoint] Waiting for database..."
python - <<'PY'
import os, time
import psycopg

host = os.environ.get("DB_HOST", "db")
port = int(os.environ.get("DB_PORT", "5432"))
user = os.environ["DB_USER"]
password = os.environ["DB_PASSWORD"]
name = os.environ["DB_NAME"]

for attempt in range(60):
    try:
        psycopg.connect(host=host, port=port, user=user, password=password, dbname=name).close()
        print("[entrypoint] Database is ready.")
        break
    except psycopg.OperationalError:
        time.sleep(1)
else:
    raise SystemExit("[entrypoint] Database not reachable after 60s.")
PY

echo "[entrypoint] Running migrations..."
python manage.py migrate --noinput

if [ "${RUN_BOOTSTRAP:-false}" = "true" ]; then
  echo "[entrypoint] Seeding data..."
  python manage.py bootstrap
fi

echo "[entrypoint] Starting: $*"
exec "$@"
```

Why each part is there:

| Line | Why |
|------|-----|
| `#!/bin/sh` | Alpine and Debian both have `sh`. Do not require bash unless you used bash features. |
| `set -eu` | Exit on error and on unset variables. A failed migration must not continue into the server. |
| Wait loop | `depends_on` starts the database container. Postgres inside it may still be starting. The app waits until a connection works. |
| `migrate` | Schema updates run before traffic. One replica should do this, or migrations must be safe to run concurrently. |
| `exec "$@"` | Replaces the shell with the command from `CMD` (or Compose `command`). That process becomes PID 1 and receives SIGTERM on `docker stop`. Without `exec`, the shell stays PID 1 and the app becomes a child that can be orphaned. |

`CMD` in the Dockerfile supplies `"$@"`. Compose can replace `command` and the same script still wraps it.

Make it executable in git (`git update-index --chmod=+x docker/entrypoint.sh`) **and** `chmod +x` in the Dockerfile. Windows checkouts sometimes drop the bit; the Dockerfile line is the reliable one.

### 11.2 What does not belong in the entrypoint

- Installing packages. That belongs in the Dockerfile so it is cached.
- Secrets generation you then lose on the next container. Generate them outside and pass them in.
- An infinite retry with no failure. After a bounded wait, exit non-zero so Compose restarts or the deploy fails visibly.
- Starting two servers in the background. One container, one main process. Extra processes (a worker, a scheduler) are extra services in Compose.

### 11.3 `init` and zombie processes

If the app starts child processes and does not reap them, zombies accumulate. `docker run --init` (Compose `init: true`) puts a tiny init such as `tini` at PID 1. The entrypoint’s `exec "$@"` is still right: tini becomes PID 1, then the app. Use `--init` when the app is a shell script that backgrounds children, or a runtime that does not reap.

---

## 12. The build context and .dockerignore

`docker build -t notes-api:dev .` sends the directory `.` to the engine as the **context**. `COPY` can only see files in that context. A `COPY` of a secret, a `.env`, or a 2 GB data dump makes the image large and the secret hard to remove.

`.dockerignore` uses gitignore-style patterns. It is the standard file to commit next to the Dockerfile:

```gitignore
.git
.venv
**/__pycache__
**/*.pyc
.env
.env.*
!.env.example
*.md
tests
.pytest_cache
.mypy_cache
.idea
.vscode
docker-compose*.yml
```

The Dockerfile itself is not required inside the image. Leaving `.env` out is mandatory: a secret in a layer is still in the layer after a later `RUN rm`.

---

## 13. Build cache, BuildKit, multi-stage, and multi-arch

### 13.1 Cache

Docker rebuilds from the first instruction whose input changed, and reuses every layer above it. That is why `COPY requirements.txt` and `RUN pip install` come **before** `COPY` of the source. An edit to a Python file then reuses the pip layer.

`docker build --no-cache` ignores the cache. Use it when you suspect a stale layer, not on every build.

Cache is local to the builder. CI machines start empty unless you export a cache (see [section 22](#22-cicd)).

### 13.2 BuildKit extras

BuildKit is the default builder. A few features the classic builder does not have:

**Cache mount** — keep apt or pip downloads in a cache that is not stored in the image:

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

**Secret mount** — a secret visible to one `RUN` and absent from the image:

```bash
docker build --secret id=pypi,src=./.pypi-token -t notes-api:dev .
```

```dockerfile
RUN --mount=type=secret,id=pypi \
    pip install --index-url "https://user:$(cat /run/secrets/pypi)@private.example/simple" -r requirements.txt
```

**SSH mount** — `RUN --mount=type=ssh` for private git clones during the build, without copying a key into the image.

**Bind mount** — `RUN --mount=type=bind` reads a file during the build without `COPY`-ing it into a layer.

**Heredoc** — a readable multi-line script:

```dockerfile
RUN <<'EOF'
set -eux
apt-get update
apt-get install -y --no-install-recommends curl
rm -rf /var/lib/apt/lists/*
EOF
```

### 13.3 Multi-stage build

The image you ship should not contain compilers, test tools, or the package index cache. A later stage copies only the files it needs.

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.12-slim AS build
WORKDIR /app
RUN pip install --upgrade pip
COPY requirements.txt .
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt

FROM python:3.12-slim AS runtime
WORKDIR /app
RUN useradd --create-home app
COPY --from=build /install /usr/local
COPY docker/entrypoint.sh /app/docker/entrypoint.sh
COPY src /app/src
RUN chmod +x /app/docker/entrypoint.sh
USER app
ENTRYPOINT ["/app/docker/entrypoint.sh"]
CMD ["gunicorn", "src.wsgi:application", "--bind", "0.0.0.0:8080"]
```

`docker build --target build` stops at the build stage when you want to debug it. The final image is the last stage.

### 13.4 Multi-platform images

A Mac with Apple Silicon builds `linux/arm64` by default. Many servers are `linux/amd64`. Buildx can build both and push a **manifest list** (one tag that points at both):

```bash
docker buildx create --name multi --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourname/notes-api:1.4.0 \
  --push \
  .
```

`--push` is required for a multi-platform build because the local image store holds one platform at a time. To load a single platform into the local engine for a test, use `--load` and one `--platform`.

---

## 14. Running a container well

### 14.1 Lifecycle

```text
docker create  -->  created
docker start   -->  running  -->  paused (docker pause)
                      |  ^
                      |  +---- docker restart
                      v
                   exited   (docker stop / the process ended)
docker rm      -->  gone
```

`docker run` is create + start. `--rm` adds “remove on exit”.

A stopped container still has its writable layer. `docker start` brings the same filesystem back. Replacing a container (`docker rm` then `docker run`, or Compose up with a new image) **discards** that layer. Anything you need to keep belongs on a volume.

### 14.2 Restart policies

| Policy | When the engine starts the container again |
|--------|-----------------------------------------------|
| `no` | Never. The default for `docker run`. |
| `on-failure[:n]` | When the process exits non-zero, up to n times. |
| `always` | When it exits for any reason, and when the daemon starts. |
| `unless-stopped` | Like `always`, except a container you stopped yourself stays stopped across daemon restarts. |

Compose services that should survive a reboot use `restart: unless-stopped`.

### 14.3 Publishing ports

```bash
docker run -d -p 8000:8080 notes-api:dev
```

Host port 8000 → container port 8080. Inside the container the app must listen on `0.0.0.0`, not `127.0.0.1`. A process bound to `127.0.0.1` is reachable only from inside that container, so the published port connects and then refuses.

Bind to localhost on the host when the service must not be public:

```bash
docker run -d -p 127.0.0.1:5432:5432 postgres:16
```

### 14.4 Environment

```bash
docker run --env PORT=8080 --env-file .env notes-api:dev
```

`--env-file` does not expand shell variables. Compose interpolates `$VARS` in the YAML file; that is a different step, done by Compose before the container starts.

The container’s environment is visible to anyone who can `docker inspect` the container. Do not treat env vars as hidden from the host’s administrators. Do treat them as the normal way to pass config and secrets at **run** time (not bake them into the image).

### 14.5 Limits

```bash
docker run -d \
  --memory 512m \
  --memory-swap 512m \
  --cpus 1.0 \
  --pids-limit 256 \
  notes-api:dev
```

`--memory-swap` equal to `--memory` disables extra swap. Without a memory limit, one container can consume the host until the kernel kills processes.

### 14.6 Logs

By default the process’s stdout and stderr are the logs (`docker logs`). The `json-file` driver stores them on disk and will fill the disk if you never rotate them:

```bash
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  notes-api:dev
```

Other drivers (`local`, `journald`, `syslog`, `fluentd`, `gelf`) ship logs off the machine. Applications should log to stdout, not to a file inside the container, unless you have a volume and a reason.

### 14.7 Stop, signals, and PID 1

`docker stop` sends SIGTERM, waits 10 seconds (`--time` / Compose `stop_grace_period`), then SIGKILL. Servers need to catch SIGTERM, stop accepting connections, and exit. That only works if the server is the process that receives the signal: exec form, and `exec` at the end of the entrypoint.

---

## 15. Networking

Every container has its own network namespace: its own `localhost` and its own interfaces.

### Drivers

| Driver | Behaviour | When to use it |
|--------|-----------|----------------|
| **bridge** | Default. Containers on the same user-defined bridge resolve each other by name. | Almost all Compose projects and `docker run` on one host. |
| **host** | The container shares the host network namespace. No `-p` mapping; the process binds host ports directly. | Linux only, when you need raw performance or a fixed port with no NAT. Not on Docker Desktop’s VM the way people expect. |
| **none** | No network. | Locked-down jobs that should not dial out. |
| **overlay** | A bridge stretched across a Swarm. | Swarm clusters. |
| **macvlan / ipvlan** | The container gets an address on the physical LAN. | Appliances that must appear as a device on the network. Easy to get wrong. |
| **container:** | Share another container’s namespace (`--network container:name`). | Sidecars that must use the same `localhost`. |

### User-defined bridge and DNS

```bash
docker network create appnet
docker run -d --name db --network appnet postgres:16
docker run -d --name api --network appnet notes-api:dev
```

Inside `api`, the hostname `db` resolves to the database container. The default bridge (`docker0`) does **not** do this automatic DNS. Compose creates a user-defined network for the project, which is why service names work in Compose and “it worked in Compose but not in two plain `docker run`s on the default bridge”.

### Publish versus expose

- `EXPOSE` in the Dockerfile and `expose:` in Compose are documentation and, for Compose, a port open **only on the Compose network**.
- `ports:` and `-p` publish on the host.

Two containers on the same network reach each other on the container port with no `ports:` entry. Publishing is for humans and for processes on the host.

### `host.docker.internal`

Inside a container, `host.docker.internal` is the host machine. Docker Desktop provides it. On Linux Engine you add it yourself:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Use it when a container must call a service that is running on the host and not in Compose (a database you installed with apt, or a dev server). Prefer putting that dependency in Compose when you can, so the name stays a service name.

---

## 16. Storage

The container’s writable layer disappears when the container is removed. Databases, uploads, and caches that must survive go on a mount.

| Kind | Syntax | Lifetime | Typical use |
|------|--------|----------|-------------|
| **Named volume** | `-v pgdata:/var/lib/postgresql/data` | Until `docker volume rm` | Database files, anything you want Docker to manage. |
| **Anonymous volume** | `-v /var/lib/postgresql/data` or a Dockerfile `VOLUME` | Until prune, easy to lose track of | Avoid for real data. |
| **Bind mount** | `-v /home/me/src:/app` | The host directory | Source code in development. |
| **tmpfs** | `--tmpfs /tmp` | RAM, gone on stop | Secrets files, scratch that must not hit disk. |

```bash
docker volume create pgdata
docker run -d --name db \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
```

You can `docker rm -f db` and run a new container on `pgdata`. The files are still there.

Inspect and back up:

```bash
docker volume inspect pgdata
docker run --rm -v pgdata:/data -v "$PWD":/backup alpine \
  tar czf /backup/pgdata.tgz -C /data .
```

**Permissions.** The process user inside the container must be able to write the mount. Official database images fix this in their own entrypoint. For your app, either run as the user who owns the files or `chown` the volume once.

**Bind mount hides image content.** If the image has `/app` populated by `COPY`, and you bind-mount a host folder onto `/app`, the container sees the host folder, not the image. That is what you want in development and a frequent surprise in production.

---

## 17. Docker Hub

Docker Hub ([https://hub.docker.com](https://hub.docker.com)) is the default registry. `docker pull nginx` means `docker pull docker.io/library/nginx`.

### Accounts, organizations, repositories

| Concept | Meaning |
|---------|---------|
| **Account** | A person. The namespace is the username: `yourname/notes-api`. |
| **Organization** | A shared namespace (`acme/notes-api`) with teams and permissions. |
| **Repository** | One app’s tags. Public or private. |
| **Visibility** | Public images are pullable by anyone (within rate limits). Private images need a login that has access. |
| **Description and README** | The repository overview on the website. A README in the repo can be the full description. |
| **Official Images** | Docker-curated images (`library/postgres`). Reviewed Dockerfiles, versioned tags, and a documented contract. Prefer them over random personal images for base layers. |
| **Verified Publisher / Sponsored OSS** | Programs for vendors. A badge, not a technical feature. |

### Login, tag, push, pull

```bash
docker login
# username and a personal access token (Hub no longer accepts the account password for docker login)

docker tag notes-api:dev yourname/notes-api:1.4.0
docker push yourname/notes-api:1.4.0

docker pull yourname/notes-api:1.4.0
```

Create the repository on the website first, or let the first push create it (depending on your plan and settings). Private is the right default for application images.

A **personal access token** is created in Hub → Account Settings → Security. Give it the smallest scope that works (`Read` for CI that only pulls, `Read & Write` for CI that pushes). Store it as a CI secret, not in the Dockerfile and not in git.

### Tags people expect

| Tag | Convention |
|-----|------------|
| `1.4.0` | Immutable release. Do not move it once published. |
| `1.4` | Moving tag for the latest patch of 1.4. |
| `1` | Moving tag for the latest minor of major 1. |
| `sha-a1b2c3d` | The git commit. CI uses this so a deploy points at one build. |
| `latest` | Whatever the publisher last tagged as latest. Fine for a demo. Weak for production. |

### Rate limits

Anonymous pulls from Hub are limited per IP. Logged-in pulls have a higher limit that depends on the plan (Personal, Pro, Team, Business). CI jobs that pull base images all day should `docker login` or mirror base images to your own registry. Exact numbers change; the Hub pricing and fair-use pages are the source of truth.

### Automated builds, webhooks, Scout

Hub used to build images from a linked GitHub repo (“automated builds”). New pipelines usually build in GitHub Actions or GitLab CI and push the result. That keeps test, scan, and push in one place.

**Webhooks** on a repository fire an HTTP call on push. A deploy server can listen and run `docker compose pull && docker compose up -d`.

**Docker Scout** (and Hub’s vulnerability view) scans images you push and reports CVEs in base layers and packages. It complements, and does not replace, a scanner you run in CI before you trust the image.

### What not to put on Hub

- Secrets in the image.
- Large data dumps. Use object storage.
- One-off images you will never pull. They count toward retention and clutter.

---

## 18. Other registries

The commands are the same. Only the hostname in the image name changes.

| Registry | Image name | Typical login |
|----------|------------|----------------|
| Docker Hub | `yourname/app:1` | `docker login` |
| GitHub Container Registry | `ghcr.io/acme/app:1` | `docker login ghcr.io -u USER --password-stdin` with a `GITHUB_TOKEN` or PAT |
| GitLab Container Registry | `registry.gitlab.com/acme/app:1` | `docker login registry.gitlab.com` |
| Google Artifact Registry | `REGION-docker.pkg.dev/PROJECT/REPO/app:1` | `gcloud auth configure-docker REGION-docker.pkg.dev` |
| Amazon ECR | `ACCOUNT.dkr.ecr.REGION.amazonaws.com/app:1` | `aws ecr get-login-password` piped to `docker login` |
| Private, self-hosted | `registry.internal:5000/app:1` | Your own auth, TLS certificate trusted by the daemon |

A local registry for experiments:

```bash
docker run -d -p 5000:5000 --name registry registry:2
docker tag notes-api:dev localhost:5000/notes-api:dev
docker push localhost:5000/notes-api:dev
```

Production registries need TLS. An insecure HTTP registry requires an explicit daemon exception and should stay on a lab network.

**Mirroring.** A pull-through cache in front of Docker Hub keeps CI working when Hub rate-limits you and keeps a copy of base images you already trust.

---

## 19. Docker Compose

Compose runs a **project**: several services, one network, named volumes, from a YAML file. The command is:

```bash
docker compose up --build
```

There is a space. The older `docker-compose` (hyphen) was a separate Python program. Current installs use the Go plugin shipped with Docker Desktop and with `docker-compose-plugin`.

### 19.1 A complete Compose file

```yaml
name: notes

services:
  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME:-notes}
      POSTGRES_USER: ${DB_USER:-notes}
      POSTGRES_PASSWORD: ${DB_PASSWORD:?Set DB_PASSWORD in .env}
    ports:
      - "127.0.0.1:${DB_HOST_PORT:-5432}:5432"
    volumes:
      - notes_pg:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-notes} -d ${DB_NAME:-notes}"]
      interval: 5s
      timeout: 5s
      retries: 20
      start_period: 10s

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - notes_redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 20

  api:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        INCLUDE_DEV_TOOLS: "false"
    image: notes-api:local
    restart: unless-stopped
    env_file:
      - .env
    environment:
      DB_HOST: db
      DB_PORT: "5432"
      DB_NAME: ${DB_NAME:-notes}
      DB_USER: ${DB_USER:-notes}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_URL: redis://redis:6379/0
      PORT: "8080"
    ports:
      - "${API_HOST_PORT:-8000}:8080"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test:
        [
          "CMD",
          "python",
          "-c",
          "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health/', timeout=5)",
        ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
    command: ["/bin/sh", "/app/docker/entrypoint.sh"]

  worker:
    image: notes-api:local
    restart: unless-stopped
    env_file:
      - .env
    environment:
      DB_HOST: db
      REDIS_URL: redis://redis:6379/0
    depends_on:
      api:
        condition: service_healthy
    command: ["python", "-m", "src.worker"]

volumes:
  notes_pg:
  notes_redis:
```

`DB_HOST: db` is the **service name**. Compose’s DNS resolves `db` to the database container. Do not point it at `127.0.0.1` inside the API container; that would be the API container itself.

`${DB_PASSWORD:?Set DB_PASSWORD in .env}` refuses to start if the variable is missing. `${DB_NAME:-notes}` uses `notes` when the variable is unset.

### 19.2 Commands

| Command | What it does |
|---------|----------------|
| `docker compose up` | Create and start. Attaches logs. Ctrl-C stops. |
| `docker compose up -d` | Detached. |
| `docker compose up --build` | Build images first. |
| `docker compose up --force-recreate` | Replace containers even if the config looks unchanged. |
| `docker compose down` | Stop and remove containers and the project network. |
| `docker compose down -v` | Also remove **named volumes**. This deletes database data. |
| `docker compose ps` | Services in this project. |
| `docker compose logs -f api` | Follow one service. |
| `docker compose exec api sh` | Shell in the running service. |
| `docker compose run --rm api python manage.py migrate` | One-off container, same image and network, removed after. |
| `docker compose build` | Build only. |
| `docker compose pull` | Pull images that use `image:` without rebuilding. |
| `docker compose stop` / `start` / `restart` | Keep containers, change their state. |
| `docker compose config` | Print the resolved YAML after interpolation. The best check before a deploy. |
| `docker compose top` | Processes. |
| `docker compose events` | Event stream. |
| `docker compose cp` | Copy files. |
| `docker compose pause` / `unpause` | Freeze. |

Compose finds `compose.yaml` or `docker-compose.yml` in the current directory. `-f` selects a file. `--env-file` selects the variable file (the default is `.env` next to the project).

The **project name** prefixes containers, networks, and volumes (`notes_api_1`, network `notes_default`). It comes from `name:` in the file, or the directory name, or `-p`.

### 19.3 Service keys you will actually use

| Key | Role |
|-----|------|
| `image` | Image to run, or the tag to give a built image. |
| `build` | `context`, `dockerfile`, `args`, `target`, `cache_from`, `additional_contexts`. |
| `command` | Replaces Dockerfile `CMD`. |
| `entrypoint` | Replaces Dockerfile `ENTRYPOINT`. |
| `env_file` / `environment` | Run-time config. `environment` wins over `env_file` on the same key. |
| `ports` | `"host:container"` or just `"8080"` to publish on a random host port. |
| `expose` | Port on the Compose network only. |
| `volumes` | Named volumes, bind mounts (`./src:/app/src`), or tmpfs. Long syntax adds `:ro`. |
| `depends_on` | Start order. With `condition: service_healthy`, wait for the healthcheck. |
| `healthcheck` | Same fields as the Dockerfile. `disable: true` turns off an inherited check. |
| `restart` | `no`, `always`, `on-failure`, `unless-stopped`. |
| `networks` | Attach to networks beyond the default. |
| `profiles` | Services that start only when that profile is requested. |
| `pull_policy` | `always`, `missing`, `never`, `build`. |
| `init` | Run a tiny init as PID 1. |
| `user` | Override `USER`. |
| `working_dir` | Override `WORKDIR`. |
| `extra_hosts` | Extra `/etc/hosts` lines. |
| `logging` | Driver and options (rotation). |
| `labels` | Metadata for filtering. |
| `container_name` | Fixed name. Avoid it when you want to scale the service; names must be unique. |
| `hostname` | Hostname inside the container. |
| `read_only` | Read-only root filesystem. Pair with tmpfs for `/tmp`. |
| `cap_drop` / `cap_add` | Linux capabilities. Dropping `ALL` and adding back the few you need is the tight setting. |
| `security_opt` | Seccomp and AppArmor profiles. |
| `mem_limit` / `cpus` | Resource caps on a single host. |
| `stop_grace_period` | How long Compose waits after SIGTERM. |
| `stop_signal` | Signal to send. |
| `develop.watch` | Compose Watch rules (sync files or rebuild on change). |
| `deploy` | Replicas, resources, and placement for Swarm. Ignored by plain `docker compose up` except for a few fields newer Compose versions read (`deploy.resources` limits on non-Swarm in recent releases — check `docker compose config` if you rely on them). |

### 19.4 Volumes, networks, configs, secrets

```yaml
volumes:
  notes_pg:
    name: notes_pg_prod    # explicit volume name, stable across directory renames

networks:
  frontend:
  backend:
    internal: true         # no route to the outside world

configs:
  app_config:
    file: ./config/app.toml

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

A service then mounts them:

```yaml
services:
  api:
    networks: [frontend, backend]
    configs:
      - source: app_config
        target: /app/config/app.toml
    secrets:
      - db_password
```

Compose secrets appear as files under `/run/secrets/`. They are not encrypted at rest by Compose itself; they are a way to avoid putting the value in `environment` and in `docker inspect`’s env list. The file on the host still needs normal filesystem permissions.

`internal: true` on a network is a good place for a database: the API joins both networks, and the database joins only the internal one, so the database has no default route off the host.

### 19.5 Profiles

```yaml
services:
  api:
    image: notes-api:local
  mailpit:
    image: axllent/mailpit
    profiles: ["dev"]
    ports:
      - "8025:8025"
```

`docker compose up` starts `api` only. `docker compose --profile dev up` also starts Mailpit. Use profiles for debug tools, one-off admin UIs, and simulators that must not run in production.

### 19.6 Several files

Development and production should not be one file full of comments. Split them.

```bash
# development
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build

# or, if the dev file is a full definition of its own:
docker compose -f docker-compose.dev.yml --env-file .env up --build
```

When several `-f` files are given, Compose **merges** them. Later files override the same keys. A typical override for development bind-mounts source and turns on the debugger:

```yaml
# docker-compose.dev.yml
services:
  api:
    build:
      args:
        INCLUDE_DEV_TOOLS: "true"
    volumes:
      - ./src:/app/src
    environment:
      DJANGO_ENV: development
    ports:
      - "8000:8080"
```

`docker compose -f docker-compose.yml -f docker-compose.dev.yml config` shows the merge.

There is also a conventional automatic override: if both `compose.yaml` and `compose.override.yaml` exist, `docker compose up` merges the override without `-f`. Production servers should call an explicit `-f` so a forgotten override file cannot change the deploy.

`include` pulls another Compose file in by path:

```yaml
include:
  - path: ./ops/observability.compose.yaml
```

### 19.7 `depends_on` is not readiness

```yaml
depends_on:
  db:
    condition: service_started
```

This waits until the database **process has started**, not until Postgres accepts connections. Always pair it with a healthcheck (`condition: service_healthy`) or with a wait loop in the entrypoint. Doing both is normal: Compose starts in a sane order, and the entrypoint survives a slow recovery.

### 19.8 One image, several services

The API, the worker, and the scheduler are the same image with different `command` values. Build once (`image: notes-api:local` on the service that has `build:`), and let the others use that tag. Rebuilding four times from the same Dockerfile wastes cache and drifts if args differ. SAF does this: `api`, `ingest-consumer`, `celery-worker`, and `celery-beat` share one image.

### 19.9 Variable substitution

Compose reads a `.env` file for **interpolation in the YAML** (`${DB_NAME}`). That file is not automatically the container environment. `env_file:` is what loads variables into the container. Many projects pass the same file to both:

```bash
docker compose --env-file .env up -d
```

```yaml
env_file:
  - .env
```

Quotes matter. `PASSWORD: "pa$$word"` in `.env` can be eaten by interpolation because `$` starts a variable. Use `$$` in the YAML when you want a literal `$`.

### 19.10 Compose Watch

For development, Watch syncs files into the container or rebuilds when files change:

```yaml
services:
  api:
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: requirements.txt
```

```bash
docker compose watch
```

`sync` copies files. `rebuild` runs a new image build. This is optional; a bind mount of `./src` is the older equivalent for interpreted languages.

---

## 20. Worked example: API, database, and worker

A minimal tree:

```text
notes-api/
  Dockerfile
  .dockerignore
  docker-compose.yml
  .env.example
  requirements.txt
  docker/entrypoint.sh
  src/wsgi.py
  src/worker.py
```

**.env.example** (commit this; copy to `.env`, which is gitignored):

```text
DB_NAME=notes
DB_USER=notes
DB_PASSWORD=change-me
```

**requirements.txt**

```text
gunicorn==23.0.0
psycopg[binary]==3.2.3
```

**Dockerfile** — the file in [section 10.1](#101-a-complete-production-dockerfile).

**docker/entrypoint.sh** — the file in [section 11.1](#111-dockerentrypointsh).

**docker-compose.yml** — the file in [section 19.1](#191-a-complete-compose-file).

Day-to-day:

```bash
cp .env.example .env
docker compose up --build
```

In another terminal:

```bash
docker compose ps
docker compose logs -f api
docker compose exec api sh
curl -fsS http://127.0.0.1:8000/health/
```

When the schema changes:

```bash
docker compose exec api python manage.py migrate
```

If migrations already run from the entrypoint, recreating the API container runs them again:

```bash
docker compose up -d --build api
```

Ship it:

```bash
docker login
docker build -t yourname/notes-api:1.4.0 .
docker push yourname/notes-api:1.4.0
```

On the server, the Compose file uses `image: yourname/notes-api:1.4.0` and **no** `build:` key, so the server pulls a known image instead of compiling:

```bash
docker compose pull
docker compose up -d
```

---

## 21. Use cases

| Use case | What you containerize | What you do not |
|----------|----------------------|-----------------|
| **Local development** | App, database, cache, queue | Your editor. Source is bind-mounted or synced so you edit on the host. |
| **One server in production** | The same stack, with published ports only where needed, secrets from an env file, `restart: unless-stopped` | Desktop. The engine and Compose are enough. |
| **Background work** | Worker and scheduler as extra services of the same image | A second process started with `&` inside the API container. |
| **CI** | The app under test, plus disposable Postgres and Redis | The CI runner itself, unless you are building a custom runner image. |
| **Try a tool once** | `docker run --rm -it postgres:16` | A permanent install on your laptop for a five-minute test. |
| **Reproducible jobs** | A batch image with a pinned digest | “Whatever Python the server happens to have.” |
| **Edge or plant machine** | A collector image that only needs `docker compose up` | A hand-built virtualenv that nobody can rebuild. |
| **Many machines, rolling deploys, autoscaling** | Images you already built | Compose as the cluster scheduler. That is Kubernetes, Nomad, or a cloud run service. Compose still builds the image they run. |

A container is a poor fit for a desktop GUI app, for software that must load a host kernel module, and for data you care about stored only in the writable layer.

**One process family per container.** The API container runs the API. The worker container runs the worker. The database container runs the database. Logs, restarts, and healthchecks stay understandable. A supervisor inside the container that starts five daemons hides which one failed.

---

## 22. CI/CD

CI/CD is the pipeline that builds the image, proves it works, and delivers it. Docker’s job in that pipeline is the **artifact**: an image identified by a digest.

```text
git push
   |
   v
lint Dockerfile and unit tests
   |
   v
docker build  (tag = git sha)
   |
   v
scan the image
   |
   v
push to the registry  ----->  record the digest
   |
   v
deploy: pull that digest and recreate containers
   |
   v
smoke-check /health
```

### 22.1 What each stage is for

| Stage | Purpose | Typical tool |
|-------|---------|--------------|
| **Lint** | Catch Dockerfile mistakes (running as root, `latest`, missing `.dockerignore` habits). | `hadolint` |
| **Test** | Unit tests on the runner, or integration tests with `docker compose up` and a test command. | `pytest`, Compose |
| **Build** | Produce the image once. | `docker buildx` |
| **Scan** | Fail the pipeline on known critical CVEs. | Trivy, Grype, Docker Scout |
| **Push** | Put the image where production can pull it. | Registry of your CI host, or Hub |
| **Sign / attest** | Record who built it and from which commit. | `buildx` provenance and SBOM, or cosign |
| **Deploy** | Pull the same digest you scanned and start it. | Compose on a VM, Kubernetes, Cloud Run, ECS |
| **Smoke** | Hit `/health` before you call the deploy done. | `curl` |

### 22.2 Tagging rules that keep rollbacks possible

- Tag every build with the **git SHA**: `notes-api:sha-a1b2c3d`.
- Also tag releases with the **version**: `notes-api:1.4.0`.
- Do not retag `1.4.0` onto a different build. Cut `1.4.1`.
- Deploy by digest when the platform allows it: `yourname/notes-api@sha256:…`.
- `latest` may move for convenience. Production manifests should not say `latest`.

### 22.3 GitHub Actions

```yaml
name: image

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  packages: write

env:
  IMAGE: ghcr.io/acme/notes-api

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile

      - name: Set up buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        if: github.event_name == 'push'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          sbom: true
          provenance: true

      - name: Scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ${{ env.IMAGE }}:sha-${{ github.sha }}
          exit-code: "1"
          severity: CRITICAL,HIGH
          ignore-unfixed: true
```

On a pull request the image is built and not pushed (`push: false`). On `main` it is pushed, with an SBOM and provenance attestation attached by BuildKit. `cache-from: type=gha` stores layers in GitHub Actions cache so the next run does not reinstall every package.

The scan step as written expects the image to exist in the registry, so keep it on the push path or scan the local image id Buildx exports. Failing on `CRITICAL` and `HIGH` is a policy choice; unfixed issues in a base image need a base-image bump, not an ignored failure that grows forever.

### 22.4 GitLab CI

```yaml
stages: [test, build, scan, deploy]

variables:
  IMAGE: $CI_REGISTRY_IMAGE

test:
  stage: test
  image: docker:27
  services:
    - docker:27-dind
  script:
    - docker compose -f docker-compose.yml -f docker-compose.ci.yml up -d --build --wait
    - docker compose exec -T api pytest
    - docker compose down -v

build:
  stage: build
  image: docker:27
  services:
    - docker:27-dind
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker build -t "$IMAGE:$CI_COMMIT_SHA" .
    - docker push "$IMAGE:$CI_COMMIT_SHA"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH "$IMAGE:$CI_COMMIT_SHA"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

deploy:
  stage: deploy
  image: alpine:3.20
  before_script:
    - apk add --no-cache openssh-client
  script:
    - ssh "$DEPLOY_USER@$DEPLOY_HOST" "cd /opt/notes && IMAGE_TAG=$CI_COMMIT_SHA docker compose pull && docker compose up -d"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```

`docker:dind` is Docker-in-Docker: the CI job runs a daemon so `docker build` works inside the job. The runner can instead mount the host socket. DinD is more isolated; the host socket is faster and shares the host’s images. Either is common. Do not mount the host Docker socket into untrusted pull-request builds: access to the socket is root on that host.

`when: manual` on deploy keeps a human in the path until you trust the pipeline.

### 22.5 Integration tests with Compose

A CI override publishes nothing on the public interface and uses a throwaway project name:

```bash
docker compose -p notes-ci -f docker-compose.yml up -d --build --wait
docker compose -p notes-ci exec -T api pytest
code=$?
docker compose -p notes-ci down -v
exit $code
```

`--wait` blocks until healthchecks pass (Compose v2). `-v` on `down` deletes the test database so the next run starts clean.

### 22.6 Deploying with Compose on a server

The server does not need the source tree’s compilers. It needs:

- Docker Engine and the Compose plugin
- A Compose file that references `image:` tags
- An env file with secrets, mode `600`, owned by the deploy user
- Firewall rules so only the API port is public

```bash
cd /opt/notes
docker login ghcr.io
export IMAGE_TAG=sha-a1b2c3d
docker compose pull
docker compose up -d
docker compose exec -T api python manage.py migrate   # if the entrypoint does not migrate
curl -fsS http://127.0.0.1:8000/health/
```

Rollback is the previous tag or digest:

```bash
export IMAGE_TAG=sha-old
docker compose up -d
```

Database migrations that delete columns need a plan (expand, then contract) because you cannot roll the code back onto a schema it does not understand. That is an application concern Compose will not solve.

### 22.7 Promotion between environments

Build **once**. Promote the same digest:

```text
CI builds sha-a1b2c3d
   -> deploy that digest to staging
   -> tests against staging
   -> retag the same digest as prod (or deploy the same digest; do not rebuild)
```

Rebuilding “the same” Dockerfile for production can pull newer base layers and ship a different image than the one you tested.

### 22.8 Supply-chain extras

| Idea | What it gives you |
|------|-------------------|
| **SBOM** | A list of packages inside the image (`--sbom=true` or `syft`). |
| **Provenance** | A statement of which repo, commit, and builder produced the image (`--provenance=true`). |
| **Signature** | A cryptographic signature (cosign, Notation) so the cluster can refuse unsigned images. |
| **Pin base images** | `FROM python:3.12-slim@sha256:…` so a rebuild does not silently take a new base. |

These sit next to scanning. Scanning asks “are there known bugs?”. Provenance asks “did our CI build this?”.

### 22.9 Secrets in the pipeline

- Registry passwords and deploy SSH keys live in the CI secret store.
- The Dockerfile never contains them.
- Build-time secrets use BuildKit secret mounts.
- Runtime secrets are injected by the deploy environment (env file on the server, or the platform’s secret manager).
- Pull-request workflows from forks should not receive secrets.

---

## 23. Security

Containers are process isolation, not a full security boundary against a hostile process running as root on the same kernel. Treat them as one layer.

| Practice | Why |
|----------|-----|
| Small base (`-slim`, distroless, or `scratch` for a static binary) | Fewer packages, fewer CVEs. |
| Pin tags, record digests | You know what you deployed. |
| Non-root `USER` | A bug in the app is not automatically root in the container. |
| No secrets in layers | `docker history` and anyone who pulls the image can read them. |
| Read-only root filesystem where the app allows it | The process cannot rewrite its own binaries. |
| Drop capabilities | The default set is already smaller than a normal root shell; drop further if you can. |
| Do not use `--privileged` | It removes most isolation. Device access can be granted narrowly with `devices:`. |
| Do not mount the Docker socket into a container | That container can start a privileged sibling and control the host. |
| Scan in CI | Base images age. Rebuild on a schedule even when your code did not change. |
| Limit published ports | Databases on `127.0.0.1` or on an internal network only. |
| Rotate registry tokens | A pushed token in a log is a leaked token. |
| Log rotation | A full disk takes the host down. |

**User namespaces and rootless** reduce the impact of a container escape. They are worth enabling on multi-tenant build machines.

**Image trust.** Pull official images or images you build. A Dockerfile that starts `FROM some-random-user/ubuntu` is an unaudited root filesystem.

---

## 24. Disk, cleanup, and debugging

### Disk

```bash
docker system df
docker system df -v
```

Images and volumes grow until you delete them.

| Command | Removes |
|---------|---------|
| `docker container prune` | Stopped containers. |
| `docker image prune` | Dangling images. |
| `docker image prune -a` | Every image not used by a container. |
| `docker volume prune` | Volumes not attached to a container. **Can delete data.** |
| `docker network prune` | Unused networks. |
| `docker builder prune` | Build cache. |
| `docker system prune` | Stopped containers, unused networks, dangling images. |
| `docker system prune -a --volumes` | Almost everything unused, including volumes. Do not run this on a server casually. |

### A container exits immediately

```bash
docker ps -a
docker logs name
docker inspect name --format '{{.State.ExitCode}} {{.State.Error}}'
```

Common causes: the command is missing, the script is not executable, `set -e` tripped on a failed migration, or the process daemonized and the container thought PID 1 had exited. Foreground processes only: Nginx needs `daemon off;`, and your entrypoint needs `exec`.

### Build fails on `COPY`

The file is not in the context, or `.dockerignore` excludes it. `docker build` prints the context transfer size; a context of a few kilobytes when you expected a whole repo means the ignore file is too broad or you ran the command in the wrong directory.

### “Port is already allocated”

Another container or a host process holds the port.

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

### Permission denied on the socket

Your user is not in the `docker` group, or you have not logged in again since `usermod`.

### Database files owned by the wrong user

The volume was first written by another image user. Prefer a fresh volume over a recursive `chown` you do not understand, unless the data matters, in which case `chown` to the uid the official image documents (Postgres in the official image is uid 999).

### Changes in the container vanished

They were in the writable layer and the container was recreated. Put persistent data on a volume. Put permanent software in the Dockerfile.

### Compose config is not what you think

```bash
docker compose config
```

This prints the merged, interpolated file. It catches a missing variable and a mistaken override.

---

## 25. How SAF uses Docker

The ideas above are already the layout of this repository.

| Piece | Where it lives |
|-------|----------------|
| Backend image | `backend/Dockerfile` — `python:3.12-slim`, non-root user `saf`, build arg `INCLUDE_DEV_TOOLS`, entrypoint `docker/entrypoint.sh` |
| Entrypoint | `backend/docker/entrypoint.sh` — wait for the database, `migrate`, optional `bootstrap_saf`, optional `collectstatic`, then `exec` Gunicorn with Uvicorn workers |
| Ignore file | `backend/.dockerignore` — keeps `.venv`, `.env`, tests, and Compose files out of the image |
| Production stack | `backend/docker-compose.yml` — TimescaleDB, Redis, NATS JetStream, API, ingest consumer, Celery worker, Celery beat |
| Development stack | `backend/docker-compose.dev.yml` — same services, published database port, `INCLUDE_DEV_TOOLS=true`, separate volumes so dev data and prod data never mix |
| Edge image | `saf_edge/Dockerfile` — collector image |
| Edge stack example | `saf_edge/docker-compose.yml.example` |

Production and development are two Compose files on purpose, matching [section 19.6](#196-several-files). One image is reused for the API, the NATS consumer, and both Celery processes, matching [section 19.8](#198-one-image-several-services). Databases bind to `127.0.0.1` in production so they are not published on the public internet. Healthchecks gate `depends_on`, and the entrypoint still waits for the database itself.

From `backend/`:

```bash
# development
cp .env.example .env
docker compose -f docker-compose.dev.yml --env-file .env up --build

# production on the server
cp .env.production.example .env
docker compose --env-file .env up --build -d
```

---

## 26. Concept index

A checklist of the concepts this guide covers, so you can find a name when you see it in an error or a job description.

| Concept | Section |
|---------|---------|
| Image, container, layer | 3, 8, 13 |
| Virtual machine versus container | 2 |
| Namespaces, cgroups, overlayfs | 2, 4 |
| Engine, CLI, containerd, runc, shim | 4 |
| OCI image | 3, 4 |
| Docker Desktop, WSL2, Linux VM | 5, 6 |
| Context (`docker context`) | 4, 6 |
| Rootless mode | 4 |
| Swarm | 4, 19 |
| `docker run` flags | 7, 14 |
| Tag, digest, `latest`, official images | 8, 17 |
| Dockerfile instructions | 10 |
| Exec form and shell form | 10, 14 |
| `ENTRYPOINT` versus `CMD` | 10, 11 |
| `ARG` versus `ENV` | 10 |
| Entrypoint script and `exec` | 11 |
| PID 1, SIGTERM, `--init` / tini | 11, 14 |
| `.dockerignore`, build context | 12 |
| Build cache | 13 |
| BuildKit: cache mounts, secret mounts, SSH, heredoc | 13 |
| Multi-stage build | 13 |
| Multi-arch, manifest list, buildx | 13 |
| Restart policies | 14 |
| Port publishing, listen address | 14, 15 |
| Resource limits | 14 |
| Logging drivers and rotation | 14 |
| Bridge, host, none, overlay, macvlan | 15 |
| DNS by container or service name | 15, 19 |
| `host.docker.internal` | 15 |
| Named volume, bind mount, tmpfs | 16 |
| Docker Hub: org, repo, token, rate limit, Scout | 17 |
| Other registries, local registry | 18 |
| Compose services, project name, interpolation | 19 |
| `depends_on` and healthchecks | 10, 19 |
| Profiles, override files, `include` | 19 |
| Compose secrets and configs | 19 |
| Compose Watch | 19 |
| One image, many commands | 19, 25 |
| CI build, scan, push, deploy, rollback | 22 |
| Image digest promotion | 22 |
| SBOM, provenance, signing | 22 |
| Non-root, capabilities, privileged, socket safety | 23 |
| Prune and disk | 24 |

---

## Further reading

- Docker’s own manuals: Dockerfile reference, Compose specification, and Engine API. They are the authority when a flag in this guide gains a new option.
- OCI image and runtime specifications, for what “standard container” means outside Docker’s CLI.
- This repository’s `backend/README.md` for the SAF-specific ports, env files, and services.
