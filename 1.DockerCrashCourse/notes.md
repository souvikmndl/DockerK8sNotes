# Docker & Kubernetes — Notes

KodeKloud Crash Course: https://www.youtube.com/watch?v=zJ6WbK9zFpI&t=2s 

> **About these notes:** The core concepts below (containers, namespaces, images, pods, services, volumes) are still fully accurate. A few tool/platform specifics have changed since this material was written — those are flagged inline as **🆕 Update** callouts so the original notes stay intact and you can see what moved.
>
> Quick summary of what changed:
> - `docker-compose` (hyphen, Python v1) is **end-of-life** → use `docker compose` (space, Go plugin, "V2"; now versioned v5.x).
> - The Compose `version:` top-level field is **obsolete** — drop it.
> - AWS Elastic Beanstalk's *Multi-container Docker on Amazon Linux AMI* platform was **retired (Jul 2022)**; Travis CI has largely given way to GitHub Actions.
> - Kubernetes renamed **"master" → "control plane"**.
> - A 4th PV access mode, **`ReadWriteOncePod`**, is now GA (K8s 1.29).
> - BuildKit is the default builder; the Ingress API is stable (`networking.k8s.io/v1`), with the **Gateway API** emerging as its successor.

## 1. What is a container?

This is what an OS looks like: whenever a running process (like Chrome, NodeJS, or Spotify) needs to access some resource, it makes a **system call**, which interacts with the **Kernel**, which in turn interacts with the **hardware** (CPU, Memory, Hard disk, etc.).

```
 Chrome        NodeJS         Spotify        }  processes running
   |             |               |              on the computer
sys call      sys call        sys call
   |             |               |
   v             v               v
        ------------------------------
                  Kernel
        ------------------------------
        |             |               |
        v             v               v
      CPU          Memory        Hard disk (e.g. Python v2, Python v3)
```

## 2. Namespace

The process of separating **system resources** based on which process is calling it is called **namespacing**.

> Example: Say Chrome needs Python 2 and NodeJS needs Python 3. So on the hard disk, we make a separate space for Py2 and another for Py3.

## 3. Control Group (cgroup)

Limits the amount of resources that a particular process can use.

## 4. Container

A **container** is a process (or group of processes) that has some resources allocated to it, using **namespaces** and **control groups**.

## 5. Image

An **image** contains two basic things:
1. A **file system snapshot** (of resources/namespaces)
2. A **start-up command**

So when we turn an image into a container, basically:
- The namespace is created in the hard disk, and
- Then the start-up command is run

This process is called the image being **instantiated**.

## 6. Running Docker

```
docker run <image-name>
```

**Overriding the default cmd:**

```
docker run <image-name> <cmd>
```

The `cmd` is a Linux command. If we pass `ls` as the `cmd`, it will refer to the Linux file system snapshot inside the image/container.

```
docker ps
```

Lists all running containers in your system.

## 7. Container Lifecycle

The `docker run` command is actually a **compound command** consisting of:
`docker create` + `docker start`

**Create a container:**
```
docker create <image name>
```
(ref. docker client — tries to create the container; `<image name>` is the name of the image to use for this container)

**Start a container:**
```
docker start <container id>
```
(tries to create the container; `<container id>` is the ID of the container to start)

- **Creating** a container is about the file system snapshot being prepped for use.
- **Starting** a container is running the startup cmd.
- `docker create <img>` returns an id → `docker start -a <id>`

The container that has been stopped earlier can be restarted as well, using `docker start <id>`. We can get all these ids (including stopped containers) by running `docker ps -a` (or `docker ps --all`).

Now, if we had started the container with another cmd (like `echo hi`), then while restarting that cmd, it will start/initiate automatically with that cmd — and if we try to pass a cmd instead of just doing `docker start <id>`, it will throw an error.

## 8. Deleting stopped containers

```
docker system prune
```

This will remove:
- all stopped containers
- all networks not used by at least one container
- all dangling images
- all build cache

This will return ids of deleted containers.

> **🆕 Update:** By default `docker system prune` does **not** delete volumes (named or anonymous) — add `--volumes` to include them, and `-a` to also remove all unused images (not just dangling ones). For a narrower cleanup, `docker container prune` removes only stopped containers.

## 9. Retrieving logs

```
docker logs <container-id>
```

This will return all output logs.

## 10. Stopping a container

**`docker stop <cont-id>`**: This sends a `SIGTERM` (an OS terminate signal). This gives the container time to shut down and do cleanup.

**`docker kill <id>`**: This sends a `KILL` signal (`SIGKILL`), and the container terminates automatically without doing anything else.

> **Note:** If `docker stop` doesn't stop a container in 10 seconds, docker itself will send the `KILL` signal, except for ping — most software listen for the `SIGTERM` signal.

## 11. Executing commands in running containers

Say we have the `redis` docker image. We can get it up and running with the `docker run` cmd. But how do we use the `redis-cli`?

The redis container is a **multi-command container**. It also contains the `redis-cli` cmd. However, to run `redis-cli` you need to get **INSIDE** the container:

```
docker exec -it <cont-id> redis-cli
```

This will run `redis-cli` inside the container and allow us to interact with it as if it was running normally. The `-it` flag allows us to send our keyboard cmds into the container.

- Every docker container is also a Linux process (or Windows, or whatever we use).
- **Every Linux process has:** `STDIN`, `STDOUT` & `STDERR`.
- The `-i` flag basically redirects our cmds to the `STDIN` of the running container.
- The `-t` flag makes it a little "beautiful" (formatted).

## 12. Getting a cmd prompt in a container

```
docker exec -it <container-id> sh
```
(`sh` opens a shell inside the docker container)

**General cmd:**
```
docker exec -it <cont-id> <cmd>
```

Exit with `Ctrl+D` or `exit`.

> **Note:** Two containers created from the same image won't share any data, unless we set them up to do it.

## 13. Creating a Dockerfile

This has some basic steps:
1. Specify a base image
2. Run some commands to install some additional programs (dependencies)
3. Specify a cmd to run on container start up

## 14. Building a Dockerfile

- Go to the dir where the image is supposed to be.
- Create a file named: `Dockerfile` (exact name, no extension)

Inside the file:

```dockerfile
# This is how we write comments (with #)

# Use an existing docker image as base
FROM alpine

# Download & install a dependency
RUN apk add --update redis

# Tell the img what to do when it starts as a container
CMD ["redis-server"]
```

**Explanations:**

| Command | Argument | Explanation |
|---|---|---|
| `FROM` | `alpine` | Use pre-existing base image from docker: alpine |
| `RUN` | `apk add --update redis` | Run this once while prepping the image |
| `CMD` | `["redis-server"]` | Specifies the command to run when starting our container created from this image |

> **🆕 Update:** In modern Alpine images the idiomatic install is `RUN apk add --no-cache redis` — `--no-cache` avoids writing the package index into the image layer, so you don't need a separate cleanup step. `apk add --update` still works but leaves the index behind.

## 15. What is a base image & why use it?

Writing a Dockerfile is the same as being given a computer with no OS and being told to install Chrome. So you first set up an OS, and that is exactly what a base image does → install an OS.

## 16. Build Process

Walking through the Dockerfile step by step:

```dockerfile
FROM alpine
   ↓
Brings/Downloads base image

RUN apk add --update redis
```

1. This gets the image from the previous step
2. Create a container out of it
3. Run the `apk add` cmd in it & create a modified file system inside the above container with redis installed in it
4. Now docker takes a snapshot of the modified container, shuts it down & creates a new image, ready for the next instruction

Basically, we create an image from the alpine image with redis installed, and pass it to the next cmd.

```dockerfile
CMD ["redis-server"]
```

1. Get the image from the previous step & create a container
2. Tell the container it should run `"redis-server"` when started → container created with modified primary cmd
3. Shut down the temp container
4. This is the final image

## 17. Tagging an image

**Syntax:**
```
docker build -t stephengrider/redis:latest .
```
(`.` specifies the dir of files/folders to use for the build)

**Tag breakdown:**
```
stephengrider  /  redis           :  latest
your docker id    project/repo name  version
```

## 18. Docker Commit

We can also convert a running Docker container into an image using the `commit` command. (We don't convert the container itself, we just create a new image from it.)

```
docker commit -c 'CMD ["redis-server"]' some-container-id
```
(`CMD [...]` is the new startup cmd for the container that will be created from this image)

## 19. Port Mapping

```
docker run -p 8080:8080 <img-name>
```

If any request comes to our local machine on this port, redirect it to this port inside the container.

> **Note:** The software inside the container can use the internet of the main computer freely, but incoming access is blocked. It has to be allowed explicitly with port mapping.

## 20. A Simple Dockerfile for a Node Server

```dockerfile
FROM node:alpine       # specify base image: version
WORKDIR /usr/app       # specify root dir, will create if not exist
COPY ./package.json ./ # copies file from local to WORKDIR
RUN npm install
COPY ./ ./             # copy source code
CMD ["npm", "start"]   # default startup cmd
```

**Points to note:**

- Always copy `package.json` (dependencies list) and install, before copying the rest of the files to the docker image. That way, every time code changes during dev work, `docker build` will use the cached image from after `RUN npm install`, and we will not need to install so many dependencies every time.
- `WORKDIR` specifies the root dir with respect to any and all docker cmds. If we use `docker exec` to open a shell into a running container, it will open the shell in the `WORKDIR` by default. Any cmds in the Dockerfile after this statement will be executed relative to this path.
- `COPY . .` → the first `.` is the location where we are running `docker build`. cmd

### Same idea, in Go

A minimal HTTP server (`main.go`):

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello from Go!")
	})
	http.ListenAndServe(":8080", nil)
}
```

A naive Go Dockerfile that mirrors the Node one:

```dockerfile
FROM golang:alpine          # base image with the Go toolchain
WORKDIR /usr/app            # working dir, created if missing
COPY go.mod go.sum ./       # copy dependency manifests first (cache layer)
RUN go mod download         # download deps — cached until go.mod/go.sum change
COPY . .                    # copy the rest of the source
RUN go build -o server .    # compile to a single binary named "server"
CMD ["./server"]            # run the compiled binary
```

**Mapping the concepts across the two:**

| Node | Go | Purpose |
|---|---|---|
| `package.json` / `package-lock.json` | `go.mod` / `go.sum` | dependency manifests — copy these first for caching |
| `npm install` | `go mod download` | fetch dependencies |
| (no build step; runs source) | `go build -o server .` | compile source into one binary |
| `CMD ["npm", "start"]` | `CMD ["./server"]` | start the app |

The same caching lesson applies: copy `go.mod` + `go.sum` and run `go mod download` **before** copying the rest of the source, so the dependency layer stays cached when only your app code changes.

> The big difference: Node ships its source + `node_modules` and runs it with an interpreter, so the image carries the whole toolchain. Go compiles to a **single static binary**, which means we can throw away the toolchain afterwards — see the multistage build in §26, where this makes a dramatic size difference.

## 21. Docker Compose

A YAML file that can be used to network between different running containers, among other things.

> **🆕 Update — Compose V1 is dead, use `docker compose`:** There are two things called Compose and mixing them up is the #1 trap:
> - `docker-compose` (with a hyphen) = the old standalone Python program, **"Compose V1"**. It reached end-of-life and was removed from Docker Desktop and CI runners (~2023–2024). Don't use it.
> - `docker compose` (a space) = the Go-based plugin built into the Docker CLI, **"Compose V2"** (now versioned v5.x, "Mont Blanc"). This is the one to use everywhere — locally and in CI.
>
> So every command below reads the same, just drop the hyphen: `docker compose up`, `docker compose down`, `docker compose ps`, etc.

**Example:** We have our own NodeJS server running on a container that uses a redis server, which itself is a container built from a ready-made docker image.

**`docker-compose.yml`** — "Here are the containers I want created":
- **redis-server**: Make it using the redis image
- **node-app**: Make it using the Dockerfile in current directory; map port 8081 to 8081

### Commands to run & build

Without compose:
```
docker run myimage
```
```
docker build .
docker run myimage
```

With compose (modern V2 syntax — space, not hyphen):
```
docker compose up
```

```
docker compose up --build
```
(build the images and then run the containers)

### `docker-compose.yml` (as written in the course)

```yaml
version: "3"
services:                    # which images to build into containers
  redis-server:               # name of the container
    image: "redis"            # name of the image from which to build
  node-app:
    restart: always
    build: .                  # docker build, use Dockerfile in cwd
    ports:                    # port mapping to expose to outside traffic
      - "4001:8081"
```

> **🆕 Update — modern Compose file:** Two changes in Compose V2:
> 1. The top-level `version:` field is **obsolete** — it's ignored and prints a warning, so just delete it. The schema is now unified.
> 2. The conventional filename is now `compose.yaml` (the older `docker-compose.yml` still works).
>
> Same file, cleaned up:
> ```yaml
> services:
>   redis-server:
>     image: "redis"
>   node-app:
>     restart: always
>     build: .
>     ports:
>       - "4001:8081"
> ```

### Same Compose setup with a Go app

The Compose file is essentially identical — Compose doesn't care what language is inside the image, only how to build and wire the containers. Just swap the app service to build the Go Dockerfile and expose the Go port:

```yaml
services:
  redis-server:
    image: "redis"
  go-app:
    restart: always
    build: .                 # builds the Go Dockerfile in the cwd
    ports:
      - "4001:8080"          # host 4001 → container 8080 (Go server port)
    environment:
      - REDIS_HOST=redis-server   # reach redis by its service name
      - REDIS_PORT=6379
```

Inside the Go code, you'd dial redis at the **service name** as hostname (e.g. `redis-server:6379`), exactly like the Node app does — Compose puts both containers on the same network and resolves `redis-server` for you.

> **Note:** Inside docker, all the containers created by a Compose project can freely communicate with each other by container/service name as hostname.

`docker compose up` will build a new network for the containers to communicate with each other.

### Running in background

```
docker run -d <img-name>       # starts a container in background
docker compose up -d           # starts compose containers in background
```

### Other compose commands

```
docker compose down            # stops & removes containers, networks
docker compose ps              # status of running containers in this compose project
```
(Run this where you have the `compose.yaml` / `docker-compose.yml` file, as it refers only to that file.)

## 22. Container Restarts

- If any process exits with code `0`, it means everything is OK. Any other code means something went wrong.

**Restart Policies:**

| Policy | Meaning |
|---|---|
| `"no"` | Never try to restart (must always be in quotes, unlike others — could have stopped for any reason) |
| `always` | Always try to restart |
| `on-failure` | Only restart if container stops with an error code |
| `unless-stopped` | Always restart unless we forcibly stop it |

## 23. Production Grade Workflow

- We can have separate Docker files for prod & dev — we can name the dev one differently, like `Dockerfile.dev`.
- `docker build -f Dockerfile.dev .` → to run the dev file, else it will look for the default `Dockerfile`.

## 24. Docker Volumes

- A mechanism to allow us to set up a reference of folders inside the container to the local dev env.
- Allows us to selectively control which files/folders to be replaced during the build.

### Volume mapping example

```
Local Folder (Frontend)     Docker Container
  /src         ←──────────►    /app
  /public      ←──────────►    reference
                                reference
```

```
docker run -p 3000:3000 -v /app/node_modules -v $(pwd):/app <img-id>
```

**`-v $(pwd):/app`** → This says that the folder (`$(pwd)`) before the `:` is to be mapped to the `/app` folder inside the container. Every time something changes inside `$pwd` (in my local), the entire folder is copied over to the container's `/app` folder.

Everything inside `/app` is overwritten without running anything else from the Dockerfile. This means the `node_modules` folder will also get overwritten (or rather, deleted) inside the `/app` folder, as we don't have that in our local.

This is where **`-v /app/node_modules`** comes in. The `-v <folder/file>` without the `:` means this folder is set in stone — don't delete/overwrite this when copying `$(pwd)`.

- When we run docker like this, any change in our local react code will get reflected inside the container, if it is running the `npm start` script.

### Simplifying with docker-compose

We can simplify this long `docker build` cmd using a docker-compose file. We need to mention the volumes in addition to the other details.

```yaml
volumes:
  - /app/node_modules
  - .:/app
```

(The `.` in `.:/app` means map the pwd to `/app` inside the container.)

### Note: specifying build context & Dockerfile in compose

We will need to mention in the docker-compose file which Dockerfile to pick. We can add options in the `build` cmd in docker-compose file like this:

```yaml
build:
  context: .          # mentions the address/folder from where we
                       # want to pull the files & folders to create this image
  dockerfile: Dockerfile.dev
```

### Running a container with a different start cmd

```
docker run <img-name> <cmd>
docker run bfcb npm run test
```

## 25. Running the React Unit Tests

We have 2 options, both have shortcomings:

**1) Run the container for the react web app, and:**
```
docker exec -it <cont-id> sh
```
to gain terminal access and run the test cmds.

**Problem:** if we update the test suite, we will have to build again or set up volumes. But most importantly, it is not automated.

**2) Using docker-compose we create another identical container**, but with a start cmd of `npm run test`.

- The issue here is that if the test cli is interactive, we wouldn't be able to access it. Why?
- We have 2 containers running — one is the web app itself and the other is the test app. Both have their own `stdin`, `stdout` & `stderr`. We can interact with it using `docker attach <cont-id>`. `attach` will connect our local terminal's stdin, stdout & stderr to that container's main running process, which is the start-up cmd → `npm`. `npm` in turn spawns another process called `start` or `test`. Both will have their own process id, separate from the process id of `npm`. Our local terminal will only attach to the start-up cmd of the container, which is `npm`, and we won't be able to interact with other processes using `docker attach` (have to fall back to `exec -it`).

## 26. Multistage Builds

- We can have multistage builds where we use one base image for a build, and use the output of that in another base image.
- We tag the builds with the `AS` keyword:

```dockerfile
FROM node:alpine AS builder    # "builder" is the tag name
```

- When the next phase starts, it will start with another `FROM`, which acts as a termination of the previous block.

**Full example** — build the React app in one stage, then copy just the built static files into a tiny nginx image:

```dockerfile
# Stage 1: build
FROM node:alpine AS builder
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
RUN npm run build

# Stage 2: run (final image contains only nginx + built files)
FROM nginx
COPY --from=builder /app/build /usr/share/nginx/html
```

The key is `COPY --from=builder <src> <dest>` — it pulls files out of the earlier stage. The final image doesn't carry Node, npm, or `node_modules`, so it's much smaller.

### Go multistage build — the textbook use case

Multistage is even more compelling for Go, because Go compiles to a **single self-contained binary**. We build in a stage that has the whole Go toolchain, then copy just the binary into a nearly-empty final image:

```dockerfile
# Stage 1: build
FROM golang:alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# CGO_ENABLED=0 → fully static binary with no libc dependency,
# so it runs even in an empty "scratch" image
RUN CGO_ENABLED=0 go build -o server .

# Stage 2: run (final image is just the binary)
FROM scratch
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

- `FROM scratch` is the empty base image — literally nothing in it. Because the Go binary is statically linked, that's all it needs.
- The `golang:alpine` builder is ~350 MB+; the final `scratch` image is basically just the size of your binary (often 10–20 MB). That's the payoff of the compile-then-discard-the-toolchain pattern.
- If your app needs TLS root certs or timezone data, use `FROM alpine` instead of `scratch` (it includes `ca-certificates`), or copy the certs across explicitly.

**Node vs Go multistage, side by side:**

| | Node/React | Go |
|---|---|---|
| Build stage produces | static files in `/app/build` | a compiled binary `server` |
| Final base image | `nginx` (serves the files) | `scratch` or `alpine` (just runs the binary) |
| What's discarded | Node, npm, `node_modules` | the entire Go toolchain |
| Typical final size | small (nginx + assets) | tiny (just the binary)|

> **🆕 Update:** BuildKit is the default build engine in modern Docker (Engine 23.0+), so multistage builds also skip stages you don't need and build independent stages in parallel automatically. For advanced builds use `docker buildx` (multi-platform images, better caching).

## 27. Environment Variables

**Two ways:**

- `variableName=value` → sets a variable name & value in the container at runtime. These values are predefined & always the same.
- `variableName` → sets a variable in the container at runtime. The value is picked up from the computer or env the container runs in. → used for secrets or API keys.

### Inside `docker-compose.yml` file

We use the `environment` array to populate env variables. Note: these values are not baked into the image, they are supplied and populated at runtime.

```yaml
services:
  server:
    build: .
    environment:
      - REDIS_HOST=<name>
      - REDIS_PORT=6379
```

## 28. Architecture of a React/Express Fibonacci App (Multicontainer)

**Dev Env:**

```
[Browser] ←→ [NGINX] ──┬──→ [React Server]
                        │         ↓
                        └──→ [Express Server] ──┬──→ [Redis] ←→ [Worker]
                                                 └──→ [Postgres]
```

**Flow:**

- Browser sends `/index.html` & `/main.js` requests to NGINX, which routes it to the React server. After the browser gets the react files, our react app is now running in the browser.
- The react app has requests to our Express server, in this format: `/api/values/*`. The react app running on the browser sends this request to NGINX, which routes it to the Express server.
- Express server stores a value in Postgres. It also looks in the cache, which in turn interacts with our worker for these values. (This is for educational purposes only — don't look for logic here for existence or role of worker.)
- In the Express server, the routes are defined without the `/api` part — only `/values/*`. When NGINX gets a request with only `/`, it will route that to react. If it gets a request with `/api`, it will truncate the `/api` part and call the Express server with `/values/*`. We will have to set up this logic inside NGINX.

> **Note:** We are not using port numbers for routing, as that might change depending on the deployed env.

We will need to create our own `.conf` file for NGINX to set up all this logic. Then, inside compose, we will need to create an NGINX container, and inside it we'll have to copy our own `.conf` file to `/etc/nginx/nginx.conf`. (Look into the docker-multicontainer repo.)

> **Go equivalent of the API tier:** The whole architecture is language-agnostic — NGINX, Redis, and Postgres don't care what the "Express Server" box is written in. Swapping the Express (Node) API for a **Go** API changes nothing about the diagram or the routing; only the API container's Dockerfile and code differ. The Go service just needs to serve the same `/values/*` routes. A sketch of the equivalent handler:
>
> ```go
> // Express: app.get('/values/all', ...)  →  Go:
> http.HandleFunc("/values/all", func(w http.ResponseWriter, r *http.Request) {
> 	// query Postgres for all submitted indices, return JSON
> })
> http.HandleFunc("/values/current", func(w http.ResponseWriter, r *http.Request) {
> 	// read the computed values out of the Redis cache, return JSON
> })
> http.ListenAndServe(":5000", nil)
> ```
>
> It would connect to Redis and Postgres by their Compose **service names** (e.g. `redis-server:6379`, `postgres:5432`), just like the Node version — the container is built from the Go Dockerfile in §20/§26 instead of the Node one.

## 29. Multicontainer Deployment with Travis/GHA & AWS EB

> **🆕 Update — CI has moved to GitHub Actions:** Travis CI (especially its free tier for open-source) has largely faded; **GitHub Actions (GHA)** is now the default choice for this kind of pipeline. The 7 steps below are identical in spirit — just implemented as a GHA workflow (`.github/workflows/*.yml`) instead of `.travis.yml`. Modern runners ship `docker compose` (V2) preinstalled.

1. Push code to Github
2. Travis/GHA automatically pulls repo
3. Travis builds a test image & tests code
4. Travis builds prod images (not from `Dockerfile.dev`)
5. Travis pushes built prod images to Dockerhub
6. Travis pushes project to AWS EB (Elastic Beanstalk)
7. EB pulls images from Dockerhub and deploys

**Prod env architecture:**

```
[BROWSER] ←→ [PORT 80: NGINX ROUTING] ──┬──→ [PORT 3000: NGINX with prod React files]
                (Elastic Beanstalk)      └──→ [PORT 5000: Express Server]
```

### `Dockerrun.aws.json`

In a multicontainer env, we need to tell EB to pull images from Dockerhub and then how to run them. Much like a docker-compose file which has instructions on how to build and run images, this file will tell EB how to run the containers.

These instructions are called **task definitions**. In truth, EB doesn't really know how to run containers — EB is one layer abstraction above EC2. EB will set up an env that can contain a number of EC2 instances, an optional DB, as well as a few other components like ELB, ASG, Security Group, etc. Then EB will manage those items for you. EB doesn't add any cost on top of other compute resources. Hence, we need to find these task definitions from EC2 services (AWS ECS Task Definition) and add them to this `Dockerrun.aws.json` file.

> **🆕 Update — the EB Multi-container platform is retired:** The *"Multi-container Docker running on 64bit Amazon Linux (AL1)"* platform this section relies on was **retired by AWS on 18 Jul 2022**. Two supported paths now:
> - **Docker running on Amazon Linux 2 / AL2023** — define containers with a plain `docker-compose.yml` (EB runs Compose for you; it can also build images at deploy time, so no need to pre-push every image).
> - **ECS running on AL2 / AL2023** — still uses `Dockerrun.aws.json v2` (unchanged format) and coordinates containers via ECS, for anyone migrating the old setup with minimal changes.
>
> More broadly, this whole flow is often done today with **ECS/Fargate directly** or Kubernetes (EKS) rather than Beanstalk.

---

# Kubernetes (K8s)

## 1. What is K8s?

It is a system for running many different containers over multiple different machines.

## 2. Why use K8s?

When we need to run many different containers with different images.

## 3. Architecture

```
[Request] → [Load Balancer] → [NODE] ←→ [NODE] ←→ [NODE]
                                  ↑          ↑          ↑
                                  └──────────┴──────────┘
                                            ↕
                                        [Master] → controls what each node does
```

- A **Node** = a VM or a physical computer.
- **Nodes + Master** form a **cluster**.

> **🆕 Update — "master" is now "control plane":** Kubernetes renamed **master → control plane** across docs, source, and config (the node label/taint moved from `node-role.kubernetes.io/master` to `node-role.kubernetes.io/control-plane`). Wherever these notes say "master," read **control plane**. The control plane still runs the components that manage the cluster: `kube-apiserver`, `etcd`, `kube-scheduler`, and `kube-controller-manager`.

## 4. Use Case Example

In our Fibonacci multicontainer app, the worker does the most computing. Let's say we get a lot of traffic — we will need to spin up X number of workers & Y number of nodes/servers. This is where we would need K8s.

## 5. Working with K8s

- On prod, we have managed systems like EKS, GKE, etc.
- Locally, we will use **Minikube** (only locally).
- Minikube will be used only to manage the VM we set up locally (which will have our containers).
- Then we will use **kubectl** to manage the containers running on the VM. `kubectl` will be used on prod as well.

## 6. kubectl commands

```
kubectl apply -f <filename>
```
Changes the current config state of our cluster.

```
kubectl get <service-type>
```
Prints status of all running instances of a svc type.

## 7. Entire Docker/K8s Deployment Flow

```
[DockerHub]              Fetch images         [DOCKER] → [POD: WORKER CONTAINER]  ⎫
  img-worker    ◄─────────────────────────────  img-worker                        ⎬ K8s NODE
  img-api                                                                          ⎭
  img-client

[Deployment File]     [KubeAPI Server]         [DOCKER] → [POD: WORKER CONTAINER]
  I want to run 4        Responsibilities:       img-worker
  copies of worker        1) Run 4 instances
                              of worker
                        (MASTER)

                                                 [DOCKER] → [POD: WORKER CONT. / WORKER CONT.]
                                                   img-worker
```

### Flow

- We must have images ready in DockerHub (DH). K8s is not involved in image creation at all.
- Our obj config files that we run using `kubectl` will affect and interact with the K8s master.
- The config files provide responsibilities to the master, like: run 4 instances of our worker image across the 3 nodes that we have.
- If, at any given time, one of our worker containers goes down, the master will restart it.
- When we run `kubectl apply -f <filename>` and the master gets updated, the Nodes will fetch the image from DockerHub, and with help of obj config file instructions, spin up containers inside pods.
- The master will always monitor the pods & containers.

## 8. Important Points

- K8s is a system to deploy containerised apps.
- Nodes are individual machines (VMs) that run containers.
- Masters are VMs with a set of programs to manage nodes.
- K8s doesn't build our images. It gets them from elsewhere.
- K8s master decides where to run each container — each node can have a dissimilar number of containers.
- To deploy something, we have to update the desired state of the master with a K8s obj config file.
- Master works constantly to meet our desired state.

## 9. Imperative vs Declarative Deployments

- In **imperative** deployments, we define the exact config, every step, etc. — like which Nodes should run which containers and how many. We will have a lot of manual, fine-grained control & a lot of responsibilities as well, including updating each container to the latest versions.
- In **Declarative** Deployment, we just tell what we want our desired state to be (like how many containers of worker, but not which node should have how many). K8s handles the rest, including updates & migrations.

**Ex: updating containers to use new images:**

| Imperative | Declarative |
|---|---|
| 1. Run a cmd to list out current running pods | 1. Update our cfg file that originally created the pod |
| 2. Run a cmd to update the current pod to use a new image | 2. Throw the updated cfg file into `kubectl` |

> **Note:** `name` & `kind` fields in a cfg file act as an identifier (name & kind combo). If `kubectl` gets a cfg file and it sees we already have containers running with the same name & kind combo, it will update the node/container; else create new ones.

## 10. Get Detailed Info About an Object

```
kubectl describe <obj-type> <obj-name>
```

If we don't provide `obj-name`, it will return detailed info of all objects of that type.

## 11. Deployment Object Type

- Using `kubectl`, we can only update the image name of a pod. We can't update num of containers, name, or port. That is where the obj type of **deployments** comes in.
- Deployment maintains a set of identical pods, ensuring they have the correct config & that the right number exists.
- Also monitors state of each pod, updating as necessary.
- Good for both dev & prod.
- Pods are used only in dev, running a single set of containers.
- Deployment contains a **pod template** that describes what kind of pods the deployment will be running.

## 12. Remove Existing Object

(This is an imperative update, no other option.)

```
kubectl delete -f <obj-cfg-file>
```

## 13. Services

```
[Browser] → [K8s Proxy] → [Port 31515: Service] → [POD: Port 3000, Container]
                (inside MiniKube VM / the node)
```

Every pod inside our node is given a random & unique IP address — every time it starts/restarts, a new one is assigned. That is why we need a **Service** object, which uses labels/selectors to route traffic to the correct pods.

## 14. Updating a Deployment's Image

- Whenever we update our deployment & run `kubectl apply`, the old pods will get deleted & new ones will be created.
- In deployments, we don't really have a straightforward way to recreate containers from the same image, but updated version.
  - We can delete running & rerun deployment — **VV Bad**.
  - We can always tag the latest images with some tag name, update our `deployment.yaml` file to use the image with that tag → cumbersome to pick version number from CI/CD, paste in deployment file, and then again run deployment.
  - **Solution:** use an imperative update.
    - Tag image with version number, push to Dockerhub.
    - Run a `kubectl` cmd forcing the deployment to use the new image version.

```
kubectl set image <obj-type>/<obj-name> <container-name>=<new-img-to-use>
```

- **set** — we want to change a property.
- **image** — name of the property we want to change. It can be something else as well.
- **obj-type** — type of obj, like pod, service, deployment, etc.
- **obj-name** — name of obj (`metadata.name`).
- **container-name** — name of container we are updating, get this from config file.

**Eg:**
```
kubectl set image deployment/client-deployment client=svk/multiclient:v5
```

Updates image value inside a deployment named `client-deployment`, which has a container named `client` (inside Pod template section).

## 15. NodePort vs ClusterIP

- **NodePort (NP)** allows outside traffic to come in and access our exposed ports.
- **ClusterIP (C-IP)** allows pods/containers inside our Node to connect via a stable IP & port. NP automatically creates a C-IP.

## 16. More kubectl commands

```
kubectl apply -f <folder-name>
```
Runs apply on all cfg files inside folder.

```
kubectl logs <pod-name>
```
Fetches logs from pod.

## 17. Postgres PVC → Persistent Volume Claims

**Need:** The Postgres DB will be running inside a container, inside a pod, inside a deployment in our Node. Typically, the data will be stored inside a file system inside Postgres. But what if the container crashes? Entire data will be lost. That is why we need to set up a Persistent Volume Claim on the host machine.

```
[Postgres Deployment]
        Pod
  [Postgres Container]
          ↓
  [Data] [Data] [Data]   ← Volume on host machine
```

### Volume in generic container terminology

Some type of mechanism that allows a container to access a filesystem outside of itself.

### Volume in K8s

An object, like Service or Deployment, that allows a container to store data at the pod level.

- In K8s we have **Volume**, **Persistent Volume**, and **Persistent Volume Claims** — 3 different things.
- Think of **Volume** as a packet of data inside a Pod. It is tightly connected to the containers inside the Pod. If the container crashes, the Volume will retain the data. But if the Pod crashes, then data is lost. Not ideal for a DB.
- **Persistent Volume:** In a PV, the data packet is stored outside the lifecycle of the pod, hence not connected to the pod. Even if the pod crashes, data will be retained.

```
[Postgres Deployment]                [Postgres Deployment]
  Pod   [Postgres Cont.]                [POD]  [Postgres Container]
  [Data][Data][Data]  ← Volume                    ↓
                                        [Data][Data][Data]  ← Persistent Volume
```

## 18. PVC (Persistent Volume Claim)

It is a config that allows us to set up PVs, but as an advertisement. We will say that "let's say 500GB & 1TB of storage is available." Based on requirement, the pod will ask for either 500GB or 1TB, & K8s will give it. If it is not readily available, it will be created on the fly.

- When we associate a PVC cfg file to a pod, K8s must find an instance of storage that meets the requirements in the spec.

### Access Modes

- **ReadWriteOnce (RWO)** — mounted read-write by a single **node** (multiple pods on that same node can still share it).
- **ReadOnlyMany (ROX)** — multiple nodes can read from this.
- **ReadWriteMany (RWX)** — can be read from & written to by many nodes.
- **🆕 ReadWriteOncePod (RWOP)** — mounted read-write by exactly **one pod** in the whole cluster. Added in K8s 1.22, **GA in 1.29**. This is the one you actually want for a single-writer DB like Postgres, since plain RWO only restricts to a node, not a pod. (CSI volumes only.)

### Where is this space created?

```
kubectl get storageclass       # shows all available options
kubectl describe storageclass
```

In local, there is only one option: our HDD. But on cloud, we will have multiple options — check StorageClass on the K8s official docs. We can specify what to choose, but every cloud has a default storage, and usually that is best.

## 19. Storing Secrets (like DB passwords)

We will need to create an obj of type `secret` to store passwords, keys, etc. However, we will not be using a cfg file to create it, like we do for other objs. Instead, we will use an imperative cmd to create this secret, because we will have to pass the value of the password to it. We will need to run this cmd in every env that we need the secret.

```
kubectl create secret generic <secret-name> --from-literal key=value
```
- `create secret` — imperative cmd to create obj
- `generic` — type of secret (type of obj to create)
- `<secret-name>` — name of secret for later reference in pod cfg
- `--from-literal key=value` — key-val pair of the secret info (we are going to add the secret info into this cmd, as opposed to file form)

### Types of secret

- **generic** — means the secret is arbitrary key-val pair
- **tls** — we are going to store TLS keys
- **docker-registry** — for auth with some custom docker registry

**Cmd example:**
```
kubectl create secret generic pgpassword --from-literal PGPASSWORD=password
```

```
kubectl get secrets
```

## 20. Ingress

Exposes selected containers/pods in our VM or cluster to the outside world. Earlier we used to have LoadBalancer (another service in K8s), but now we have Ingress. There are many types of Ingress; the one we will be using is called `ingress-nginx`, led by the K8s community. It is very different from `kubernetes-ingress`, led by nginxinc.

The setup of this `ingress-nginx` varies a lot depending upon the env — local, AWS, Azure, GCP, etc.

> **🆕 Update — Ingress API & the Gateway API:** The Ingress resource is now stable under `networking.k8s.io/v1` (the old `extensions/v1beta1` was removed in K8s 1.22), so cfg files use `apiVersion: networking.k8s.io/v1`. `ingress-nginx` is still widely used. Going forward, Kubernetes is steering people toward the **Gateway API** (`gateway.networking.k8s.io`) as the more expressive successor to Ingress — worth knowing the name, though Ingress is still perfectly fine for the routing described here.

## 21. Ingress Behind the Scenes

1. We create an obj cfg file for `ingress-nginx`.
2. We feed this file to `kubectl`, which creates a **controller**.
3. This controller then creates a pod running nginx that handles routing.
4. The controller constantly works to make sure our pod is up & running, to maintain our desired state.
5. This is the same flow for each & every obj, even of other types.

> **Note:** Practically speaking, however, the controller & pod are part of the same thing. It's not like we will see 2 separate objs being created.

## 22. Ingress-Nginx on GCP

```
Traffic → [Google Cloud Load Balancer] → NODE: [Load Balancer Service] → [Deployment: nginx-controller + nginx pod] → client
                                                        ↑                              ├→ [ClusterIP Service] → [client pods]
                                                   Ingress Cfg                          ├→ server
                                                                                         └→ default backend pod
```

In GCP, we get a **GC Load Balancer** that handles outside traffic. It is GCP's own LB & will be provided for other services as well. This GCLB itself will create a Load Balancer **Service** obj & attach it to our Deployment obj containing the `nginx-ingress` controller & pod.

The default backend pod is a health check service. We can also replace it with our own server.

> **Why use `ingress-nginx`, going through all these hoops, when we can easily set up a LoadBalancer & nginx router ourself?**
> Ingress comes with a lot of built-in code that makes our life easier. One small example — it can bypass the ClusterIP service & make sure that a request hits the same client pod twice, for **sticky sessions**.