# gswrapper

The game server wrapper, also known as `gswrapper` or `gsw`, can be used to run a game server executable
from within the container image that runs your GameFabric Armadas or Vessels.

The following features are provided:

- [Parameter templating](#command-line-arguments),
- [Configuration file templating](#configuration-files),
- [Shutdown handling](#shutdown-handling),
- [Log tailing](#log-tailing), and
- [Post-stop hook](#post-stop-hook) (e.g. crash reporting).

## Features

### Templating

The GSW uses the standard Go templating syntax and its basic functionality as
described [here](https://pkg.go.dev/text/template#section-documentation).

#### Command-line arguments

The GSW can be configured to generate a templated command to execute the game server with.
The available templating variables are:

| Placeholder           | Type                | Description                                                                                                                                                                                                 |
|-----------------------|:--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `{{.GameServerIP}}`   | `string`            | The IP given by Agones to the game server.                                                                                                                                                                  |
| `{{.GameServerPort}}` | `uint16`            | The port given by Agones to the game server. Defaults to the port named `game`, or if there is no port with that name, to the first port found. A different name can be configured using `--ports.primary`. |
| `{{.Ports}}`          | `map[string]uint16` | The known game server ports.                                                                                                                                                                                |
| `{{.Env}}`            | `map[string]string` | The environment variables.                                                                                                                                                                                  |

Example:

```shell
gsw -- /app/gameserver --port={{ .GameServerPort }} --query-port={{ .Ports.query }} --servername={{ .Env.POD_NAME }}
```

#### Configuration Files

The GSW can be configured to generate a templated configuration file.
It needs a template file as an input, which must be available within the gswrapper's container, and an output file path.
The available templating variables are the same as the ones for command-line arguments.

| Command-line argument    | Environment variable   | Description                                                |
|--------------------------|------------------------|------------------------------------------------------------|
| `--config.template-path` | `CONFIG_TEMPLATE_PATH` | Path at which to write the game server configuration file. |
| `--config.output-path`   | `CONFIG_OUTPUT_PATH`   | Path to the configuration file template.                   |

Example:

```yaml
# template.yaml
gameserver:
  ip: "{{ .GameServerIP }}"
  port: "{{ .GameServerPort }}"
  {{- if .Env.POD_NAME }}
  servername: "{{ .Env.POD_NAME }}"
  {{- end }}
```

```shell
gsw --config.template-path=template.yaml --config.output-path=config.yaml -- /app/gameserver --config=config.yaml
```

### Log tailing

The GSW also supports tailing log files and printing them on stdout using the wrapper's logger.

| Command-line argument | Environment variable | Description                                                       |
|-----------------------|----------------------|-------------------------------------------------------------------|
| `--tail-log.paths`    | `TAIL_LOG_PATHS`     | Paths from which to tail log files. Can be passed multiple times. |

Example:

```shell
gsw --tail-log.paths=gameserver.log --tail-log.paths=error.log -- /app/gameserver
```

### Shutdown handling

The GSW can also manage the lifetime of the game server by initiating the shutdown for different phases and after a configured
duration.

This can be useful to force the shutdown of stuck game servers or allow compaction of the fleet.

| Command-line argument  | Environment variable | Description                                                                                          |
|------------------------|----------------------|------------------------------------------------------------------------------------------------------|
| `--shutdown.scheduled` | `SHUTDOWN_SCHEDULED` | Shutdown when the game server has been `Scheduled` for the given duration (default: `0s`, disabled). |
| `--shutdown.ready`     | `SHUTDOWN_READY`     | Shutdown when the game server has been `Ready` for the given duration (default: `0s`, disabled).     |
| `--shutdown.allocated` | `SHUTDOWN_ALLOCATED` | Shutdown when the game server has been `Allocated` for the given duration (default: `0s`, disabled). |

Example:

```shell
gsw --shutdown.scheduled=24h --shutdown.ready=1h --shutdown.allocated=24h -- /app/gameserver
```

### Post-stop hook

Lastly, the post-stop hook allows the configuration of an executable, provided by the container image, that runs after the game server has stopped.
One use-case could be a crash handler, uploading the core dump for further processing.

| Command-line argument                 | Environment variable                | Description                                                                                                                                                                                                                                             |
|---------------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `--post-stop-hook.path`               | `POST_STOP_HOOK_PATH`               | Path to the post-stop hook.                                                                                                                                                                                                                             |
| `--post-stop-hook.args`               | `POST_STOP_HOOK_ARGS`               | The post-stop hooks arguments. Can be used multiple times.                                                                                                                                                                                              |
| `--post-stop-hook.max-execution-time` | `POST_STOP_HOOK_MAX_EXECUTION_TIME` | Maximum execution time for the post-stop hook (default: `30m`). Warning: The maximum execution time cannot exceed the termination grace period, which is set to 30s. This can be configured on GameFabric-Armadas/Formations-Settings-Advanced section. |
| `--post-stop-hook.on-error`           | `POST_STOP_HOOK_ON_ERROR`           | Determines if the post-start hook should run, when the game server exited with a non-zero exit code. Core dump crashes always cause the post-start hook to run.                                                                                         |
| `--post-stop-hook.on-success`         | `POST_STOP_HOOK_ON_SUCCESS`         | Determines if the post-start hook should run, when the game server exited with exit code 0.                                                                                                                                                             |


Example:

```shell
gsw \
  --post-stop-hook.path=hook.sh \
  --post-stop-hook.args="{{ .GameServerIP }}" \
  --post-stop-hook.args="{{ .GameServerPort }}" \
  --post-stop-hook.max-execution-time=5m \
  --post-stop-hook.on-error=true \
  --post-stop-hook.on-success=false
```

The GSW provides access to the detected game server exit code and signal (if any)
by setting the the environment variables `GAMESERVER_EXITCODE` (`int`) and `GAMESERVER_SIGNAL` (`string`) accordingly before calling the hook.

Example `hook.sh`:

```/bin/sh
# Game server success (hook called only if --post-stop-hook.on-success=true)
# GAMESERVER_EXITCODE=0
# GAMESERVER_EXITSIGNAL="-1"

# Game server error (hook called only if --post-stop-hook.on-error=true)
# GAMESERVER_EXITCODE=1
# GAMESERVER_EXITSIGNAL="-1"

# Game server crash (e.g. SIGSEGV)
# GAMESERVER_EXITCODE="-1"
# GAMESERVER_SIGNAL=11

echo "Post-stop hook here. Game server exited with code $GAMESERVER_EXITCODE / signal $GAMESERVER_EXITSIGNAL."
```

The `stdout` and `stderr` output of the hook is caught and each attached as one-line message to the GSW logger.

Example output:

```
lvl=info msg="Game server exit code: \"0\"\nGame server exit signal: \"-1\"\nIndex: 1\nIndex: 2\nIndex: 3\nIndex: 4\nIndex: 5\nIndex: 6\nIndex: 7\nIndex: 8\nIndex: 9\nIndex: 10\n" svc=gswrapper output=stdout
lvl=eror msg= svc=gswrapper output=stderr
```

## Deprecated Features

- `--crashhandler.exec` / `CRASHHANDLER_EXEC`: Use `--post-stop-hook.path` /  `POST_STOP_HOOK_PATH` instead.
- `--crashhandler.args` / `CRASHHANDLER_ARGS`: Use `--post-stop-hook.args` /  `POST_STOP_HOOK_ARGS` instead.
- `--crashhandler.max-execution-time` / `CRASHHANDLER_MAX_EXECUTION_TIME`: Use `--post-stop-hook.max-execution-time` /  `POST_STOP_HOOK_MAX_EXECUTION_TIME`
  instead.

## Exit codes

The game server wrapper exits with code `0` (success), if:

- the game server exited with code `0`, and
- there either is
    - no post-stop hook set, or
    - no post-stop hook condition applies, or
    - post-stop hook exited with code `0`.

For any other situation, the exit code is `1` (error).

Examples:
```
lvl=info msg="Starting game server" svc=gswrapper
lvl=info msg="Game server stopped" svc=gswrapper runtimeSeconds=10.007 exitCode=0 exitSignal=-1

lvl=info msg="Starting post-stop hook" svc=gswrapper
lvl=info msg="Post-stop hook stopped" svc=gswrapper runtimeSeconds=10.042 exitCode=0 exitSignal=-1
```

```
lvl=info msg="Starting game server" svc=gswrapper
lvl=eror msg="Game server stopped with error" svc=gswrapper runtimeSeconds=10.015 exitCode=1 exitSignal=-1 error="gsemu exited: exit status 1"

lvl=info msg="Starting post-stop hook" svc=gswrapper
lvl=eror msg="Post-stop hook stopped with error" svc=gswrapper runtimeSeconds=3.014 exitCode=-1 exitSignal=-1 error="post-stop hook timed out: context deadline exceeded"
```
