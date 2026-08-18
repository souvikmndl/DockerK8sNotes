# Docker & Ingress — Notes

Compiled from the projects in this repo. Every snippet here is the **correct,
current version** — outdated APIs, EOL base images and Compose v1 syntax have been
folded out, so what you read is what you should write.

| Folder | What it teaches |
|---|---|
| `redis-image/` | The absolute minimum Dockerfile |
| `simpleweb/` | Node app image, `WORKDIR`, layer caching |
| `visits/` | First Compose file, container-to-container networking |
| `docker-react/` | Dev vs prod images, multi-stage builds, test container, CI |
| `prod_grade/frontend/` | Multi-stage build for a static frontend |
| `docker-multi-container/` | 5-service architecture, nginx reverse proxy, CI |
| `complex-k8s/` | Same app on Kubernetes + **Ingress** |

---

## 1. The mental model

- An **image** is a filesystem snapshot + a default startup command.
- A **container** is a running instance of that image, with its own isolated
  process namespace, network namespace and filesystem view.
- A **Dockerfile** is a recipe. Every instruction produces a new **layer**, and
  layers are cached.

Build lifecycle of a single instruction:

```
FROM alpine          → download base image FS snapshot
RUN apk add redis    → 1. create temp container from previous FS snapshot
                       2. run the command inside it
                       3. take a new FS snapshot
                       4. throw the temp container away
CMD ["redis-server"] → no FS change, only sets the default startup command
```

`CMD` does **not** run at build time. It is only metadata that says what to run
when the container starts.

---

## 2. Minimal Dockerfile — `redis-image/`

```dockerfile
# Use an existing docker image as a base
FROM alpine:3.21
# Download and install a dependency
RUN apk add --no-cache redis
# Tell the image what to do when it starts as a container
CMD ["redis-server"]
```

```bash
docker build -t <yourname>/redis-image .
docker run <yourname>/redis-image
```

**Pin the base image.** `FROM alpine` tracks a moving tag — a rebuild six months
later silently gives you a different OS. Always pin at least the major version.

**`--no-cache`** tells apk not to write its package index to disk, which keeps the
layer small. It replaces the older `apk add --update` + a manual
`rm -rf /var/cache/apk/*`.

---

## 3. Node app image — `simpleweb/`

```dockerfile
# Specify a base image
FROM node:22-alpine

WORKDIR /usr/app

# Copy the manifest first, on its own, so the install layer caches
COPY package*.json ./

# Install dependencies
RUN npm ci --omit=dev

# Now copy the rest of the source
COPY . .

EXPOSE 8080
USER node

# Default command
CMD ["npm", "start"]
```

The app it wraps (`simpleweb/index.js`):

```js
const express = require("express");
const app = express();

app.get("/", (req, res) => {
	res.send("Hi there");
});

app.listen(8080, () => {
	console.log("Listening on port 8080");
});
```

### What this file is really teaching

**a) `WORKDIR`** — sets the cwd for every following `COPY`/`RUN`/`CMD`, and creates
the directory if missing. Without it you dump your app into `/`, colliding with
`/bin`, `/etc`, `/lib`.

**b) Layer caching — why `package.json` is copied on its own line first.**

If you write `COPY . .` before `RUN npm ci`, then *any* source change invalidates
the `COPY` layer, which invalidates the install layer, and you reinstall every
dependency on every build. Copying only the manifest first means the install
layer is reused as long as your dependencies haven't changed.

**c) `npm ci`, not `npm install`.** `npm ci` installs exactly what the lockfile
says and errors if `package.json` and `package-lock.json` disagree. `npm install`
will happily resolve new versions and mutate the lockfile, so builds drift.

**d) `USER node`.** The official Node images ship a non-root `node` user. Switch
to it after the last step that needs root, so a container breakout doesn't hand
over root.

**e) Pin your dependencies too.** `"express": "*"` in `package.json` means every
rebuild can pull a different major version. Use a real range: `"^4.19.2"`.

### Port mapping

Container ports are **not** reachable from the host by default. `EXPOSE` is
documentation only — it opens nothing.

```bash
docker run -p 5000:8080 <image>
#           │     └── port inside the container (what index.js listens on)
#           └──────── port on your machine
```

---

## 4. Compose & container networking — `visits/`

### `visits/Dockerfile`

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

CMD ["npm", "start"]
```

### `visits/docker-compose.yml`

```yaml
# There is no top-level `version:` key. It is obsolete in the Compose Spec —
# Docker Compose v2 ignores it and warns if present.
services:
    redis-server:
        image: "redis:7-alpine"
    node-app:
        restart: always
        build: .
        ports:
            - "4001:8081"
        depends_on:
            - redis-server
```

### The key idea: service name == hostname

Compose puts every service on a shared user-defined bridge network and registers
each **service name** in the network's DNS. So the app connects to the literal
string `redis-server` — no IPs, no links, no config:

```js
const express = require("express");
const { createClient } = require("redis");

const app = express();

// "redis-server" is the compose service name; docker's embedded DNS resolves it
// to that container's IP on the shared network
const client = createClient({ url: "redis://redis-server:6379" });
client.on("error", (err) => console.error("Redis error", err));

app.get("/", async (req, res) => {
	const visits = await client.get("visits");
	res.send("Number of visits is " + visits);
	await client.set("visits", parseInt(visits) + 1);
});

(async () => {
	await client.connect();
	await client.set("visits", 0);
	app.listen(8081, () => console.log("listening on port 8081"));
})();
```

Note the modern `node-redis` shape: a single `url`, an explicit `await
client.connect()`, and promises rather than callbacks. There is no top-level
`host`/`port` option, and hash commands are camelCase (`hSet`, `hGetAll`).

### `restart` policies

| Policy | Behaviour |
|---|---|
| `no` | (default) never restart |
| `always` | always restart, including after a clean exit and after daemon restart |
| `on-failure` | restart only on a non-zero exit code |
| `unless-stopped` | like `always`, but stays stopped if *you* stopped it |

### Compose CLI

```bash
docker compose up            # create + start  (v2 — two words, no hyphen)
docker compose up -d         # detached
docker compose up --build    # force a rebuild of `build:` services
docker compose down          # stop and delete containers + network
docker compose ps            # status of services in this compose file
docker compose logs -f api   # follow one service's logs
```

`docker-compose` (hyphenated, the Python v1 tool) is end-of-life. Everything is
now `docker compose`, a Go plugin bundled with Docker Desktop.

---

## 5. Dev image + bind mounts — `docker-react/`

### `docker-react/Dockerfile.dev`

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

CMD ["npm", "run", "start"]
```

Note `npm ci` without `--omit=dev` here — the dev image *needs* devDependencies
(the dev server, the test runner).

Because the filename isn't `Dockerfile`, you have to name it:

```bash
docker build -f Dockerfile.dev -t react-dev .
```

### `docker-react/docker-compose.yml`

```yaml
services:
    web:
        build:
            context: .
            dockerfile: Dockerfile.dev
        ports:
            - "3000:3000"
        volumes:
            - /app/node_modules
            - .:/app
        environment:
            - WDS_SOCKET_PORT=0
    tests: # a test container. It doesn't need ports exposed, but needs a separate run cmd
        build:
            context: .
            dockerfile: Dockerfile.dev
        volumes:
            - /app/node_modules
            - .:/app
        command: ["npm", "run", "test"]
```

### The two-volume trick

```yaml
volumes:
    - /app/node_modules   # anonymous volume — "don't map this path, leave it alone"
    - .:/app              # bind mount — host cwd shadows /app inside the container
```

The *more specific* path wins. Without the first line, the bind mount would hide
`/app/node_modules` (your host has no `node_modules` — see `.dockerignore`) and
the app would crash on startup. The anonymous volume carves out an exception so
the container keeps the `node_modules` that the install step produced at build
time.

With the bind mount in place, editing a file on your host is instantly visible
inside the container and the dev server hot-reloads. **You no longer need to
rebuild the image for source changes** — only for dependency changes.

### `context` vs `dockerfile`

- `context:` — the directory sent to the daemon as the build context; all `COPY`
  paths are relative to it.
- `dockerfile:` — path to the Dockerfile, **relative to the context**.

### `.dockerignore`

```
node_modules
.git
build
npm-debug.log
```

`node_modules` is the important one: copying a host `node_modules` into the image
sends thousands of files over the build context and can carry platform-specific
native binaries that break inside Linux.

**Do not ignore `package-lock.json`.** It has to reach the image — `npm ci`
errors without it, and without it your builds stop being reproducible.

### CRA hot-reload inside Docker

```yaml
environment:
    - WDS_SOCKET_PORT=0
```

Needed with react-scripts 5 / webpack-dev-server 4: the HMR client otherwise tries
to open a websocket back to the *container's* port instead of the published one,
and live reload silently dies. If file changes still aren't detected on some
bind-mount setups, add `CHOKIDAR_USEPOLLING=true`.

---

## 6. Multi-stage builds — `docker-react/Dockerfile`

```dockerfile
# Build phase --> AS builder names this phase 'builder' so that we can refer to it in successive builds
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# FROM cmds also act as terminators for prev phases/blocks, essentially saying the prev phase is over
FROM nginx:1.27-alpine
EXPOSE 80
COPY --from=builder /app/build /usr/share/nginx/html
# /app/build contains the build output
# this copies the build files over from the builder phase
# nginx img's start cmd automatically runs the nginx server, so no other start code
# EXPOSE 80 is documentation — it opens nothing on its own.
# But AWS Beanstalk reads this instruction and opens the port for incoming traffic after deployment

# the dev server is powerful, has a lot of code for interacting with our dev changes and all
# the prod server's only task is to serve up the static build files (which also contain the bundled deps)
# that is why we use nginx, whose sole purpose here is to serve static files, so not power hungry
# /usr/share/nginx/html is the default location nginx serves static files from

# docker run -p 8080:80 <img_id>  --> 80 is the default port for nginx
```

### Why this matters

| | Build stage | Run stage |
|---|---|---|
| Base | `node:22-alpine` | `nginx:1.27-alpine` |
| Needs | node, npm, every devDependency, all source | one folder of static files |
| Ends up in final image | ❌ nothing | ✅ only `/app/build` |

The final image is **only** the last `FROM` onwards. Everything from the builder
stage is discarded except what `COPY --from=builder` explicitly pulls across.
Result: ~1.5 GB of toolchain collapses to a ~50 MB nginx image, and none of your
source code or build tooling ships to production.

Use the `-alpine` nginx variant — it's a fraction of the size of the Debian-based
default. If you migrate the app off Create React App to Vite, the output folder
becomes `dist`, not `build`; adjust the `COPY --from` path.

---

## 7. nginx as a static file server — `client/nginx/default.conf`

Used by the client image in both `docker-multi-container/` and `complex-k8s/`:

```nginx
server {
    # nginx will listen on port 3000 instead of 80
    listen 3000;

    location / {
        # setting root directory to look for files when a hostname/ request comes in
        root /usr/share/nginx/html;
        index index.html index.htm;
        # this means the index file will be index.html, or index.htm if .html is absent
        try_files $uri $uri/ /index.html;
        # try_files says: Try these files in order, and serve the first one that exists.
        # $uri is the exact requested path (like GET /static/js/main.js), nginx
        # will check /usr/share/nginx/html/static/js/main.js
        # $uri/ is a directory. like /static from above, if it exists nginx will apply index rules inside it
        # /index.html is the fallback: if neither file nor directory exists, serve index.html
    }
}
```

`try_files $uri $uri/ /index.html` is the **SPA fallback**. A React app with
client-side routing has no `/otherpage.html` on disk; a hard refresh on
`/otherpage` must still return `index.html` so the JS router can take over.
Without this line you get a 404 on every deep link.

The point of `listen 3000` instead of the default 80 is that the outer reverse
proxy (and the Kubernetes Service) both address this container on 3000.

### Wiring it in — `docker-multi-container/client/Dockerfile`

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine
EXPOSE 3000
COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/build /usr/share/nginx/html
```

Anything dropped into `/etc/nginx/conf.d/*.conf` gets `include`d by the stock
`nginx.conf`. Overwriting `default.conf` **replaces** the built-in port-80 server
block rather than adding a second one alongside it.

---

## 8. nginx as a reverse proxy — `docker-multi-container/nginx/default.conf`

This is the front door of the multi-container app.

```nginx
upstream client {
    server client:3000;
}

upstream api {
    server api:5000;
}
# upstream means sitting behind nginx when seen from outside
# `api` and `client` refer to the docker compose services we defined
# since nginx is also on that network, it shares the same DNS
# that is why we can use service names as hostnames

# `server` is an nginx keyword, that is why we renamed our backend service to `api`

server {
    # we expose port 80 to the outside world
    listen 80;

    # whenever we get a request to "/", route it to http://client in our docker network
    # proxy_pass means route "/" to "http://client"
    location / {
        proxy_pass http://client;
    }

    # for websocket requests (used by the dev server to hot-reload files we change locally)
    location /ws {
        proxy_pass http://client;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }

    # rewrite any request matching the regex /api/(.*) to /$1
    # $1 refers to whatever was matched by (.*) — the first capture group
    # break means stop rewrites here, don't match any other rules
    location /api {
        rewrite /api/(.*) /$1 break;
        proxy_pass http://api;
    }
}
```

### Why a reverse proxy at all

Without it the React app must know the backend's host and port, and you get CORS
problems and hardcoded URLs baked into the build. With it, the browser only ever
talks to **one** origin (`localhost:3000` in dev), and nginx fans requests out
internally. The frontend just calls `/api/values/all` — a relative path.

### The `/api` rewrite

The Express server defines routes as `/values/all`, not `/api/values/all`. The
`rewrite` strips the prefix before proxying:

```
browser:  GET /api/values/all
          ↓ rewrite /api/(.*) → /$1
upstream: GET /values/all      → api:5000
```

`break` stops further rewrite processing in this block; without it nginx could
loop back through the rules.

### The nginx image (`nginx/Dockerfile`)

```dockerfile
FROM nginx:1.27-alpine
COPY ./default.conf /etc/nginx/conf.d/default.conf
```

The `/ws` block exists purely for local development — it's the CRA dev server's
hot-reload websocket. In production, where the client is a static nginx bundle,
nothing hits `/ws`; it's harmless to leave in.

---

## 9. The full multi-container app

```
                         ┌──────── nginx :80 ────────┐
   browser ──:3000──────▶│  /      → client:3000     │
                         │  /api/* → api:5000        │
                         └───────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
              client (React)                  api (Express)
                                                    │
                                    ┌───────────────┴──────────────┐
                                    ▼                              ▼
                              postgres :5432                 redis :6379
                              (permanent values)             (cache + pub/sub)
                                                                   │
                                                                   ▼
                                                          worker (subscriber)
                                                          computes fib(), writes
                                                          results back to redis
```

The **worker** demonstrates a background job: the API publishes an index on the
`insert` channel, the worker subscribes, computes the (deliberately slow,
recursive) Fibonacci value and stores it in a Redis hash.

### `docker-compose-dev.yml`

```yaml
services:
    postgres:
        image: "postgres:16-alpine"
        environment:
            - POSTGRES_PASSWORD=postgres_password
            - POSTGRES_USER=postgres
        volumes:
            # without a named volume, `docker compose down` destroys the database
            - pgdata:/var/lib/postgresql/data
        healthcheck:
            test: ["CMD-SHELL", "pg_isready -U postgres"]
            interval: 5s
            retries: 5
    redis:
        image: "redis:7-alpine"
    nginx:
        restart: always
        build:
            dockerfile: Dockerfile.dev
            context: ./nginx
        ports:
            - "3000:80"
            # map port 3000 on our local machine to nginx port 80
        depends_on:
            - api
            - client
    api:
        build:
            context: ./server
            dockerfile: Dockerfile.dev
        volumes:
            - /app/node_modules
            - ./server:/app
        depends_on:
            postgres:
                condition: service_healthy
        environment:
            - REDIS_HOST=redis # redis service from above (inside the docker network, hostname == service name)
            - REDIS_PORT=6379
            - PGUSER=postgres
            - PGHOST=postgres
            - PGDATABASE=postgres
            - PGPASSWORD=postgres_password
            - PGPORT=5432
    client:
        build:
            context: ./client
            dockerfile: Dockerfile.dev
        volumes:
            - /app/node_modules
            - ./client:/app
        environment:
            - WDS_SOCKET_PORT=0
    worker:
        build:
            context: ./worker
            dockerfile: Dockerfile.dev
        volumes:
            - /app/node_modules
            - ./worker:/app
        environment:
            - REDIS_HOST=redis
            - REDIS_PORT=6379

volumes:
    pgdata:
```

### `depends_on` orders startup, not readiness

Plain `depends_on: [postgres]` only guarantees the postgres *container* was
created — not that Postgres is accepting connections. Pair it with a
`healthcheck` and `condition: service_healthy`, as above, when the dependent
service can't tolerate a cold start. For services that reconnect on their own
(the Redis clients here use a reconnect strategy), plain `depends_on` is fine.

### Config via environment variables

Never hardcode connection details. Each service reads them at runtime
(`server/keys.js`):

```js
module.exports = {
  redisHost: process.env.REDIS_HOST,
  redisPort: process.env.REDIS_PORT,
  pgUser: process.env.PGUSER,
  pgHost: process.env.PGHOST,
  pgDatabase: process.env.PGDATABASE,
  pgPassword: process.env.PGPASSWORD,
  pgPort: process.env.PGPORT,
};
```

The exact same image then runs unchanged under Compose, under Kubernetes, or on
Elastic Beanstalk — only the env vars differ.

The Redis clients built from those keys (`server/index.js`, `worker/index.js`):

```js
const { createClient } = require('redis');

const redisClient = createClient({
  url: `redis://${keys.redisHost}:${keys.redisPort}`,
  socket: { reconnectStrategy: () => 1000 }, // retry every 1s instead of giving up
});
await redisClient.connect();

// pub/sub needs its own connection — a subscribed client can't run other commands
const subscriber = redisClient.duplicate();
await subscriber.connect();
await subscriber.subscribe('insert', (message) => {
  redisClient.hSet('values', message, fib(parseInt(message)));
});
```

### `docker-compose.yml` (production / Elastic Beanstalk)

```yaml
services:
    client:
        image: "svk72/multi-client"
        mem_limit: 128m
        hostname: client
    server:
        image: "svk72/multi-server"
        mem_limit: 128m
        hostname: api
        environment:
            - REDIS_HOST=$REDIS_HOST
            - REDIS_PORT=$REDIS_PORT
            - PGUSER=$PGUSER
            - PGHOST=$PGHOST
            - PGDATABASE=$PGDATABASE
            - PGPASSWORD=$PGPASSWORD
            - PGPORT=$PGPORT
    worker:
        image: "svk72/multi-worker"
        mem_limit: 128m
        hostname: worker
        environment:
            - REDIS_HOST=$REDIS_HOST
            - REDIS_PORT=$REDIS_PORT
    nginx:
        image: "svk72/multi-nginx"
        mem_limit: 128m
        hostname: nginx
        ports:
            - "80:80"
```

Differences from the dev file:

- `image:` (pull prebuilt) instead of `build:` — nothing is compiled on the server.
  These are the tags the CI pipeline pushes, so they must match your own Docker
  Hub namespace.
- No `postgres`/`redis` services — in production those are managed AWS services
  (RDS / ElastiCache), addressed via `$PGHOST` / `$REDIS_HOST` set in the Elastic
  Beanstalk environment. That's why the api service sets `hostname: api`
  explicitly: nginx's `upstream api { server api:5000; }` needs that name to
  resolve.
- `mem_limit: 128m` keeps five containers inside a `t3.micro`.

---

## 10. CI/CD with GitHub Actions

### Single container — `docker-react/.github/workflows/deploy.yaml`

```yaml
name: Deploy Frontend
on:
    push:
        branches:
            - master

permissions:
    contents: read

jobs:
    build:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v4

            - uses: docker/login-action@v3
              with:
                  username: ${{ secrets.DOCKER_USERNAME }}
                  password: ${{ secrets.DOCKER_TOKEN }}

            - run: docker build -t svk72/react-test -f Dockerfile.dev .
            - run: docker run -e CI=true svk72/react-test npm test

            - name: Generate deployment package
              run: zip -r deploy.zip . -x '*.git*'

            - name: Deploy to EB
              uses: einaregilsson/beanstalk-deploy@v18
              with:
                  aws_access_key: ${{ secrets.AWS_ACCESS_KEY }}
                  aws_secret_key: ${{ secrets.AWS_SECRET_KEY }}
                  application_name: docker-gh
                  environment_name: Dockergh-env
                  existing_bucket_name: elasticbeanstalk-us-east-1-923445559289
                  region: us-east-1
                  version_label: ${{ github.sha }}
                  deployment_package: deploy.zip
```

**`-e CI=true` is the critical flag.** `npm test` for CRA defaults to interactive
watch mode, which never exits — CI would hang forever. `CI=true` makes it run
once and exit with a proper status code. Docker propagates the container's exit
code, so a failing test fails the workflow step.

**Authenticate with `docker/login-action`, not `docker login -p`.** Passing a
password on the command line leaks it into the run log and emits a warning. Use a
Docker Hub *access token* rather than your account password.

### Multi container — `docker-multi-container/.github/workflows/deploy.yaml`

```yaml
name: Deploy MultiDocker
on:
    push:
        branches:
            - master # check your repo — your default branch might be main

permissions:
    contents: read

jobs:
    build:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v4

            - uses: docker/login-action@v3
              with:
                  username: ${{ secrets.DOCKER_USERNAME }}
                  password: ${{ secrets.DOCKER_TOKEN }}

            # build the dev image and run the test suite in it — if this fails,
            # nothing below runs and broken code never reaches Docker Hub
            - run: docker build -t svk72/react-test -f ./client/Dockerfile.dev ./client
            - run: docker run -e CI=true svk72/react-test npm test

            # tag with the commit SHA as well as latest, so deploys are traceable
            - run: docker build -t svk72/multi-client:${{ github.sha }} -t svk72/multi-client:latest ./client
            - run: docker build -t svk72/multi-nginx:${{ github.sha }}  -t svk72/multi-nginx:latest  ./nginx
            - run: docker build -t svk72/multi-server:${{ github.sha }} -t svk72/multi-server:latest ./server
            - run: docker build -t svk72/multi-worker:${{ github.sha }} -t svk72/multi-worker:latest ./worker

            - run: docker push --all-tags svk72/multi-client
            - run: docker push --all-tags svk72/multi-nginx
            - run: docker push --all-tags svk72/multi-server
            - run: docker push --all-tags svk72/multi-worker
```

Shape of the pipeline: **build the dev image → run tests in it → build the four
production images → push them.**

Note `docker build -t <tag> ./client` — the trailing path is the build context,
and the Dockerfile is found at `./client/Dockerfile` by default.

The image names here must match the ones your production `docker-compose.yml`
pulls, or you'll deploy someone else's images.

---

## 11. Ingress (Kubernetes)

Ingress is the production replacement for `NodePort`. The four Service types:

```
- ClusterIP    → reachable only from inside the cluster
- NodePort     → exposes a port to the outside world (dev only)
- LoadBalancer → one cloud LB per service (expensive, one IP each)
- Ingress      → one entry point, path/host-based routing to many services
```

**Ingress is not a Service type in the same sense.** It's a separate object kind
that only describes *routing rules*; something has to implement them. That
something is an **Ingress Controller** — a pod running in your cluster that
watches Ingress objects and reconfigures itself. This repo uses
**ingress-nginx**, which is literally nginx driven by the Kubernetes API.

> There are two different projects with confusingly similar names:
> **`ingress-nginx`** (Kubernetes community, the one used here) and
> **`nginx-ingress`** (F5/NGINX Inc.). Their annotations are *not* interchangeable.
> Everything below is `ingress-nginx`.

### Installing the controller

```bash
# minikube — simplest
minikube addons enable ingress

# any cluster — check the current release tag at
# https://github.com/kubernetes/ingress-nginx/releases
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-vX.Y.Z/deploy/static/provider/cloud/deploy.yaml

# or via Helm (recommended for real clusters)
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

Verify:

```bash
kubectl get pods -n ingress-nginx
kubectl get ingressclass          # should list `nginx`
```

### `complex-k8s/k8s/ingress-service.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: ingress-service
    annotations:
        # treat the `path` fields below as regular expressions
        nginx.ingress.kubernetes.io/use-regex: "true"
        # rewrite the proxied path to capture group 1 (this is what strips /api)
        nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
    # selects which controller implements this Ingress
    ingressClassName: nginx
    rules:
        - http:
              paths:
                  - path: /?(.*)
                    pathType: ImplementationSpecific
                    backend:
                        service:
                            name: client-cluster-ip-service
                            port:
                                number: 3000
                  - path: /api/?(.*)
                    pathType: ImplementationSpecific
                    backend:
                        service:
                            name: server-cluster-ip-service
                            port:
                                number: 5000
```

Most Ingress tutorials you'll find online are written against the old API. What
changed:

| Old (`extensions/v1beta1`, removed in k8s 1.22) | Current |
|---|---|
| `apiVersion: extensions/v1beta1` | `apiVersion: networking.k8s.io/v1` |
| `annotations: kubernetes.io/ingress.class: nginx` | `spec.ingressClassName: nginx` |
| `backend: { serviceName: x, servicePort: 3000 }` | `backend.service.name` + `backend.service.port.number` |
| no `pathType` | `pathType` is **required** |

### How the rewrite works

The `/api` prefix has to be stripped before the request reaches Express, exactly
like the nginx `rewrite` in the Compose version:

```
nginx.ingress.kubernetes.io/use-regex: "true"      ← treat `path` as a regex
nginx.ingress.kubernetes.io/rewrite-target: /$1    ← rewrite to capture group 1

path: /api/?(.*)
      └─────┬──┘
         $1 = everything after /api/

  GET /api/values/all  →  $1 = "values/all"  →  upstream gets  /values/all
  GET /                →  $1 = ""            →  upstream gets  /
```

`pathType: ImplementationSpecific` is what *allows* the regex. The other two
values ignore regex entirely:

| `pathType` | Meaning |
|---|---|
| `Exact` | URL must match the path character-for-character |
| `Prefix` | matched by whole path *segments* — `/api` matches `/api/foo`, not `/apifoo` |
| `ImplementationSpecific` | up to the controller — for ingress-nginx, enables regex when `use-regex: "true"` |

**Precedence:** `/?(.*)` would also match `/api/values/all`. ingress-nginx sorts
the generated nginx `location` blocks by descending path length, so the longer
`/api/?(.*)` wins for `/api/*` requests. This works, but it is implicit — if you
add a third route, verify the generated config:

```bash
kubectl exec -n ingress-nginx <controller-pod> -- cat /etc/nginx/nginx.conf
```

### What it routes to

The Ingress points at ClusterIP Services, which point at Deployments by label:

```yaml
# client-cluster-ip-service.yaml
apiVersion: v1
kind: Service
metadata:
    name: client-cluster-ip-service
spec:
    type: ClusterIP
    selector:
        component: web       # must match the pod template's labels
    ports:
        - port: 3000         # port exposed to other pods in the cluster
          targetPort: 3000   # port the container actually listens on
```

```yaml
# client-deployment.yaml (abridged)
spec:
    replicas: 3
    selector:
        matchLabels:
            component: web
    template:
        metadata:
            labels:
                component: web
        spec:
            containers:
                - name: client
                  image: svk72/multi-client   # the image your CI pipeline pushed
                  ports:
                      - containerPort: 3000   # nginx inside multi-client listens here
```

Full chain:

```
browser
   │
   ▼
ingress-nginx controller pod  (the only thing exposed outside)
   │  path /?(.*)      → client-cluster-ip-service:3000 → pods labelled component=web
   │  path /api/?(.*)  → server-cluster-ip-service:5000 → pods labelled component=server
   ▼
```

The port numbers `3000` and `5000` are not arbitrary — `3000` is the `listen 3000`
in the client's `nginx/default.conf`, and `5000` is `app.listen(5000)` in
`server/index.js`. Change one and you must change all three (container, Service,
Ingress).

### Useful commands

```bash
kubectl apply -f k8s/                     # apply every config in the folder
kubectl get ingress
kubectl describe ingress ingress-service  # shows rules + backend endpoint health
minikube ip                               # the address to hit in your browser
minikube service list
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller -f
```

### Adding TLS

The natural next step — with cert-manager installed:

```yaml
metadata:
    annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
    ingressClassName: nginx
    tls:
        - hosts:
              - myapp.example.com
          secretName: myapp-tls
    rules:
        - host: myapp.example.com
          http:
              paths: ...
```

Adding a `host:` also removes the reliance on path-length precedence, since rules
are then selected by Host header first.

---

## 12. Command cheat sheet

```bash
# ---- images ----
docker build .                          # build from ./Dockerfile
docker build -t <user>/<name>:<tag> .   # tag it (defaults to :latest)
docker build -f Dockerfile.dev .        # non-default Dockerfile name
docker images
docker rmi <image>
docker image prune -a                   # reclaim disk

# ---- containers ----
docker run <image>                      # create + start
docker run -it <image> sh               # override CMD, get a shell
docker run -p 8080:80 <image>           # host:container port map
docker run -e CI=true <image> npm test  # inject an env var, override the command
docker ps                               # running
docker ps -a                            # including exited
docker exec -it <container> sh          # shell into a *running* container
docker logs -f <container>
docker stop <container>                 # SIGTERM, 10s grace
docker kill <container>                 # SIGKILL, immediate
docker system prune                     # nuke stopped containers, unused networks, dangling images

# ---- registry ----
docker login -u <user>                  # prompts for the token; never pass -p inline
docker push <user>/<name>
docker pull <user>/<name>

# ---- compose (v2 — two words) ----
docker compose up -d --build
docker compose down -v                  # also removes named volumes
docker compose logs -f <service>
docker compose exec <service> sh
```

`-it` is two flags: `-i` keeps stdin open, `-t` allocates a TTY so you get a
proper prompt with line editing.

---

## 13. Conventions worth keeping

- **Pin every base image** — `node:22-alpine`, `nginx:1.27-alpine`,
  `postgres:16-alpine`, `redis:7-alpine`, `alpine:3.21`. Never a bare `latest`.
- **Copy the manifest, install, then copy the source** — that ordering is the
  whole layer-caching strategy.
- **`npm ci`, and let `package-lock.json` into the image.**
- **`USER node`** after the last root-requiring step.
- **No `version:` key** in Compose files.
- **Config through environment variables**, so one image runs everywhere.
- **Name your volumes** for anything stateful, or you lose the data on `down`.
- **Tag CI images with the commit SHA**, not just `latest`.
- **Never `docker login -p <password>`** in CI — it lands in the log.
