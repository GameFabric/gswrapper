# gswrapper

The game server wrapper, also known as `gswrapper` or `gsw`, can be used to run a game server executable
from within the container image that runs your GameFabric Armadas or Vessels.

The following features are provided:

- [Parameter templating](#command-line-arguments),
- [Configuration file templating](#configuration-files),
- [Shutdown handling](#shutdown-handling),
- [Log tailing](#log-tailing), and
- [Crash reporting](#crash-reporting).

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

#### Example:

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

#### Example:

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

#### Example:

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

#### Example:

```shell
gsw --shutdown.ready=1h --shutdown.allocated=24h -- /app/gameserver
```

### Crash reporting

Lastly, the crash handler can be configured to run an executable automatically in the event of a server crash.

| Command-line argument               | Environment variable              | Description                                                                                                                        |
|-------------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| `--crashhandler.exec`               | `CRASHHANDLER_EXEC`               | Path to the crash handler executable.                                                                                              |
| `--crashhandler.args`               | `CRASHHANDLER_ARGS`               | Crash handler arguments. Can be passed multiple times.                                                                             |
| `--crashhandler.max-execution-time` | `CRASHHANDLER_MAX_EXECUTION_TIME` | Timeout after which the crash handler should be aborted (default: `30m`). Please add the time unit, as the default is nanoseconds. |

#### Example:

```shell
gsw --crashhandler.exec=crash.sh --crashhandler.args="{{ .GameServerIP }}" --crashhandler.args="{{ .GameServerPort }}" --crashhandler.max-execution-time=5m
```
