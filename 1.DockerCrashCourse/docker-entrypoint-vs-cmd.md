# Docker `CMD` vs `ENTRYPOINT`

Both `CMD` and `ENTRYPOINT` define what runs when a container starts, but they play different roles. The confusion usually comes from the fact that they interact with each other, and each has two syntax forms that behave differently.

## The core distinction

`ENTRYPOINT` defines the executable — the thing that always runs. `CMD` defines the default arguments to that executable, or the default command if no entrypoint is set. The key difference is what happens when you pass arguments on the command line:

- Arguments to `docker run` **override** `CMD` entirely.
- Arguments to `docker run` get **appended** to `ENTRYPOINT`.

That single behavioral difference drives almost every decision about which to use.

## Two syntax forms (this matters a lot)

Each instruction can be written in *exec form* or *shell form*, and they behave very differently.

**Exec form** (JSON array):

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
ENTRYPOINT ["python", "app.py"]
```

This runs the binary directly as PID 1, no shell involved. This is the form you almost always want.

**Shell form** (plain string):

```dockerfile
CMD nginx -g "daemon off;"
ENTRYPOINT python app.py
```

This wraps the command in `/bin/sh -c "..."`. So the actual PID 1 becomes `sh`, and your process is a child of it.

The shell-form gotcha bites people in production: `sh -c` doesn't forward signals like `SIGTERM` to child processes by default. So when Kubernetes or `docker stop` sends `SIGTERM`, your app may never receive it and gets `SIGKILL`ed after the grace period instead. For a Go service doing graceful shutdown (draining SQS consumers, finishing in-flight requests), this is exactly the bug you don't want. Exec form makes your process PID 1 directly, so signals reach it.

## How they combine

This table is the mental model worth memorizing. Given a Dockerfile with some combination, here's the final command executed:

| ENTRYPOINT     | CMD             | `docker run img` executes | `docker run img arg1` executes |
| -------------- | --------------- | ------------------------- | ------------------------------ |
| (none)         | `["echo","hi"]` | `echo hi`                 | `arg1` (CMD replaced)          |
| `["echo"]`     | (none)          | `echo`                    | `echo arg1` (appended)         |
| `["echo"]`     | `["hi"]`        | `echo hi`                 | `echo arg1`                    |

So the common, idiomatic pattern is: put the fixed executable in `ENTRYPOINT`, and put the default arguments in `CMD`. That way the container has sensible default behavior but stays flexible.

```dockerfile
ENTRYPOINT ["myserver"]
CMD ["--port=8080"]
```

- `docker run img` → `myserver --port=8080`
- `docker run img --port=9090` → `myserver --port=9090`

The user overrides just the args, never has to remember the binary name.

## When to use which

**Use only `CMD`** when your image is a general-purpose environment and you want the default command to be trivially replaceable. A classic example is a language base image:

```dockerfile
CMD ["python3"]
```

`docker run python` drops you into a REPL, but `docker run python my_script.py` runs your script instead. If they'd used `ENTRYPOINT ["python3"]`, then `docker run python my_script.py` would try to run `python3 my_script.py` — which happens to work here, but `docker run python bash` would fail because it'd become `python3 bash`.

**Use `ENTRYPOINT` (+ `CMD` for defaults)** when the container is meant to *be* a specific application — a microservice, a CLI tool. You want the container to always run your binary, and treat any `docker run` arguments as flags to that binary. This is the right choice for Go services.

## The entrypoint-script pattern

A very common real-world use of `ENTRYPOINT` is a wrapper shell script that does setup (waits for a DB, runs migrations, substitutes env vars) and then execs the real process:

```dockerfile
ENTRYPOINT ["/entrypoint.sh"]
CMD ["myserver"]
```

```bash
#!/bin/sh
# entrypoint.sh
run_migrations
exec "$@"   # exec replaces the shell with CMD, so PID 1 becomes myserver
```

The `exec "$@"` is the important line — it hands off PID 1 to your actual process (using whatever `CMD` or `docker run` args were passed), preserving correct signal handling. Without `exec`, the shell stays PID 1 and you're back to the signal problem.

## Overriding at runtime

For completeness: you can override the entrypoint too, but it takes a flag rather than positional args:

```bash
docker run --entrypoint /bin/sh myimage    # replace ENTRYPOINT
docker run myimage --some-flag             # append to ENTRYPOINT / replace CMD
```

This is handy for debugging — dropping into a shell in an image whose entrypoint is your server.

## Quick summary

Think of it as `ENTRYPOINT` = the verb (what this container does), `CMD` = the default object (the arguments). Prefer exec form for both so signals propagate correctly. For microservices, `ENTRYPOINT ["binary"]` + `CMD ["default-flags"]`, or an entrypoint script ending in `exec "$@"`, is the pattern that behaves well under Kubernetes lifecycle management.