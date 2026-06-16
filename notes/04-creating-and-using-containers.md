# Section 4 — Creating and Using Containers Like a Boss

Personal notes tracking the hands-on exercises while following the course.

---

## Assignment: Manage Multiple Containers

**Goal:** Run an `nginx`, a `mysql`, and an `httpd` (apache) server, all detached and named,
inspect them, then clean everything up.

Rules from the assignment slide:

- `docs.docker.com` and `--help` are your friends.
- Run all of them with `--detach` (or `-d`) and name them with `--name`.
- `nginx` listens on `80:80`, `httpd` on `8080:80`, `mysql` on `3306:3306`.
- For `mysql`, pass `--env MYSQL_RANDOM_ROOT_PASSWORD=yes` (or `-e`).
- Use `docker container logs` on mysql to find the random root password created on startup.
- Clean up with `docker container stop` and `docker container rm`
  (both accept multiple names or IDs).
- Use `docker container ls` to verify before and after cleanup.

### Solution

```bash
# Run the three servers detached + named, with the required port mappings
docker container run --publish 80:80   --detach --name nginx  nginx
docker container run --publish 8080:80 --detach --name apache httpd
docker container run --publish 3306:3306 --detach --name mysql \
  --env MYSQL_RANDOM_ROOT_PASSWORD=yes mysql

# Verify all three are running
docker container ls

# Grab the random root password mysql generated on startup
docker container logs mysql        # look for "GENERATED ROOT PASSWORD"

# Clean up (stop, then remove — multiple names at once)
docker container stop nginx apache mysql
docker container rm   nginx apache mysql

# Confirm everything is gone
docker container ls -a
```

> **Note on flag order:** the image name comes *last*. Flags placed *after* the image
> name are passed to the container's command, not to `docker run`, so
> `docker run nginx --name nginx -d` silently fails to do what you want.
> Correct: `docker run --name nginx -d nginx`.

---

## Lecture commands explored (container lifecycle & process monitoring)

These were run while working through the lessons leading up to the assignment.

### Basics — run, publish, detach, list

```bash
docker version
docker container run --publish 80:80 nginx            # foreground
docker container run --publish 80:80 --detach nginx   # background
docker container ls          # running containers
docker container ls -a       # all containers (incl. stopped)
docker container stop 478    # stop by (short) ID
```

### Naming and logs

```bash
docker container run --publish 8080:80 --detach --name webhost nginx
docker container logs webhost     # view container output
docker container top  webhost     # processes running inside the container
docker container rm 957 478 b78   # remove multiple (running ones need -f or stop first)
```

### Process monitoring — mongo example

```bash
docker run --name mongo -d mongo
docker ps                  # = docker container ls
docker top mongo           # processes inside the container
ps aux | grep mongo        # same process visible on the host

docker stop mongo
docker ps                  # mongo no longer listed
docker start mongo         # restart the existing (stopped) container
docker top mongo
```

### Handy shortcuts / equivalents

| Long form                  | Short form         |
| -------------------------- | ------------------ |
| `docker container ls`      | `docker ps`        |
| `docker container run ...` | `docker run ...`   |
| `docker container rm`      | `docker rm`        |
| `docker container logs`    | `docker logs`      |
| `docker container top`     | `docker top`       |
| `--detach`                 | `-d`               |
| `--publish 80:80`          | `-p 80:80`         |
| `--env KEY=val`            | `-e KEY=val`       |

> `--help` works on every level: `docker --help`, `docker run --help`,
> `docker logs --help`, etc.

---

## Inspecting containers: metadata vs. live stats

Two complementary commands for "what's going on inside a container":

### `docker container inspect` — static config / metadata

Returns a JSON array with the container's full configuration: image, env vars,
mounts, network settings, ports, command, restart policy, etc. This is a
*point-in-time snapshot of how the container was set up*, not live usage.

```bash
docker container inspect mysql
docker inspect mysql            # short form
```

Tip: pull out a single value with `--format` (Go template):

```bash
docker container inspect --format '{{ .NetworkSettings.IPAddress }}' mysql
docker container inspect --format '{{ .State.Status }}' mysql
```

### `docker stats` — live resource usage

Streams a live performance view (CPU %, memory usage/limit, network I/O,
block I/O, PIDs) for running containers. Like `top`, but per container.

```bash
docker stats              # live stream for ALL running containers
docker stats mysql        # just one container
docker stats --no-stream  # print one snapshot and exit (good for scripts)
```

| Command                     | Shows                                        | Live? |
| --------------------------- | -------------------------------------------- | ----- |
| `docker container top`      | processes running *inside* the container     | no    |
| `docker container inspect`  | static config / metadata (JSON)              | no    |
| `docker stats`              | CPU / mem / net / IO resource usage          | yes   |

> **Gotcha from the history:** `docker run -d -name nginx` fails — `-name` is not a
> valid flag. Use the double-dash long form `--name` (or there is no short flag for it):
> `docker run -d --name nginx nginx`.
