# systemctl-alpine 
β _(beta)_

[![Go Version](https://img.shields.io/github/go-mod/go-version/mobydeck/systemctl-alpine)](https://golang.org/doc/devel/release.html)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

A CLI tool that helps you manage services on Alpine Linux by translating systemd commands to their OpenRC equivalents.

## Overview

`systemctl-alpine` bridges the gap between systemd and OpenRC service management by providing a familiar systemd-like interface for Alpine Linux. It allows you to:

- Convert systemd service files to OpenRC init scripts
- Enable and disable services
- Start, stop, restart, and check the status of services
- List all available services and their status
- Edit OpenRC service scripts with modification tracking
- Work with template services (e.g., `nginx@.service`)

This tool is particularly useful for:
- Docker container images transitioning from systemd-based distributions to Alpine Linux
- Developers familiar with systemd who need to work with Alpine Linux
- Automation scripts that use systemd commands but need to run on Alpine

## Installation

## Prerequisites

- Go 1.24 or higher
- Alpine Linux (tested on version 3.20 and above)
- Root/sudo access for service management

### From Source

Clone the repository

```bash
git clone https://github.com/mobydeck/systemctl-alpine.git
cd systemctl-alpine
```

Build the binary

```bash
go build -o systemctl-alpine
```

Install it to your PATH

```bash
sudo mv systemctl-alpine /bin/systemctl
```

### Usage

```
systemctl is a CLI tool that helps you manage services on Alpine Linux
by translating systemctl commands to their OpenRC equivalents.

For example, you can use 'systemctl enable some-service' to convert a systemd
service file to an OpenRC init script and enable it to start at boot.

Usage:
  systemctl [command]

Available Commands:
  completion    Generate the autocompletion script for the specified shell
  daemon-reload Reload systemd manager configuration (not needed in OpenRC)
  disable       Disable one or more services from starting at boot
  edit          Edit an OpenRC service script
  enable        Enable one or more services to start at boot
  help          Help about any command
  is-enabled    Check if a service is enabled to start at boot
  list          List all systemd services and their OpenRC status
  reload        Reload a service
  restart       Restart a service
  start         Start a service
  status        Check the status of a service
  stop          Stop a service

Flags:
  -h, --help   help for systemctl

Use "systemctl [command] --help" for more information about a command.
```

Enable a service (converts systemd service file to OpenRC and enables it)

```bash
systemctl enable nginx
```

Enable a template service with an instance name

```bash
systemctl enable nginx@user1
```

Enable and start a service

```bash
systemctl enable --now nginx
```

Force enable a service (overwrite manually modified service files)

```bash
systemctl enable --force nginx
```

Start a service

```bash
systemctl start nginx
```

Stop a service

```bash
systemctl stop nginx
```

Restart a service

```bash
systemctl restart nginx
```

Check service status

```bash
systemctl status nginx
```

Reload service configuration

```bash
systemctl reload nginx
```

Disable a service

```bash
systemctl disable nginx
```

Disable and stop a service

```bash
systemctl disable --now nginx
```

Check if a service is enabled

```bash
systemctl is-enabled nginx
```

Edit a service script

```bash
systemctl edit nginx
```

List services

```bash
# List enabled OpenRC services and all systemd services
systemctl list

# List all services including disabled OpenRC services
systemctl list --all
systemctl ls -a

# The output shows service status and origin for converted services
SERVICE                        STATUS
nginx                          enabled
redis                          enabled (from /lib/systemd/system/redis.service)
mysql                          disabled
apache2                        disabled (not converted)
```

### Working with Multiple Services

You can enable or disable multiple services at once:

Enable multiple services

```bash
systemctl enable nginx mysql redis
```

Enable and start multiple services

```bash
systemctl enable --now nginx mysql redis
```

Disable multiple services

```bash
systemctl disable nginx mysql redis
```

Disable and stop multiple services

```bash
systemctl disable --now nginx mysql redis
```

### Service File Locations

The tool looks for systemd service files in these locations:
- `/lib/systemd/system/`
- `/etc/systemd/system/`

## How It Works

When you run `systemctl enable some-service`:

1. The tool looks for `some-service.service` in the standard systemd locations
2. If a systemd service file is found:
   - It parses the systemd service file and extracts key configuration
   - It generates an equivalent OpenRC init script
   - It installs the script to `/etc/init.d/some-service`
3. If no systemd service file is found but an OpenRC service exists:
   - It skips the conversion step and just enables the existing OpenRC service
4. It enables the service using `rc-update add some-service default`
5. If the `--now` flag is used, it also starts the service

For template services like `nginx@.service`, when you run `systemctl enable nginx@user1`:

1. The tool looks for the template service file `nginx@.service`
2. It processes the template, substituting variables like `%i` with the instance name `user1`
3. It creates an OpenRC service script named `nginx@user1` with the instance name available as an environment variable

For other commands like `start`, `stop`, etc., it translates them to the appropriate `rc-service` commands.

### Editing and Modification Protection

When you run `systemctl edit some-service`:

1. The tool opens the OpenRC service script in an editor (using `$EDITOR`, `vi`, `nano`, or `ed`)
2. After editing, it adds a modification comment with a timestamp to track changes
3. If you edit the file again, it updates the timestamp to reflect the most recent edit

When you run `systemctl enable some-service` on a service that has been manually edited:

1. The tool detects the modification comment and avoids overwriting your changes
2. It notifies you that the service has been manually modified
3. It enables the existing service without converting it again
4. If you want to force a conversion, you can use the `--force` flag

This protection ensures that your manual customizations to service scripts are preserved.

## Features

- **Service Conversion**: Converts systemd service files to OpenRC init scripts
- **ExecStop Support**: Properly handles custom stop commands from systemd services
- **Reload Support**: Implements service reloading via SIGHUP
- **Multiple Service Management**: Enable or disable multiple services with a single command
- **Smart Listing**: Shows all systemd services and enabled OpenRC services by default, with an option to show all services
- **Existing Service Detection**: Works with existing OpenRC services even without systemd service files
- **Service Type Support**: Handles different systemd service types (simple, forking, notify)
- **Capabilities Support**: Converts systemd AmbientCapabilities to OpenRC capabilities
- **Edit Command**: Edit OpenRC service scripts with your preferred editor
- **Modification Protection**: Prevents automatic overwriting of manually edited service scripts
- **Template Services**: Supports systemd template services with instance names and variable substitution

### Supported Systemd Service Directives

The following systemd service directives are supported in the conversion process:

| Systemd Directive | OpenRC Equivalent | Notes |
|-------------------|-------------------|-------|
| Description | description | Service description |
| User | command_user | User to run the service as |
| Group | command_user | Group to run the service as (combined with User) |
| WorkingDirectory | directory | Directory to run the service in |
| EnvironmentFile | Sourced in script | Environment file for the service |
| Environment | export statements | Environment variables |
| ExecStartPre | start_pre() | Commands to run before starting the service |
| ExecStart | command/command_args | Main service command |
| ExecStop | stop() | Custom stop command |
| Type | command_background | Affects whether service runs in background |
| AmbientCapabilities | capabilities | Linux capabilities for the service |

#### Service Type Handling

- `Type=simple` or `Type=notify` (or no Type): Sets `command_background=true` in OpenRC
- `Type=forking`: Omits `command_background` as the service handles its own daemonization

#### Capabilities Handling

Systemd's space-separated capabilities list is converted to OpenRC's comma-separated format with the `^` prefix:

```
# Systemd
AmbientCapabilities=CAP_NET_BIND_SERVICE CAP_SYS_TIME

# Converted to OpenRC
capabilities="^cap_net_bind_service,^cap_sys_time"
```

#### Template Service Substitutions

When processing template services, the following systemd specifiers are supported:

| Specifier | Description | Example |
|-----------|-------------|---------|
| `%i` | Instance name | For `nginx@user1`, this is `user1` |
| `%I` | Unescaped instance name | Same as `%i` but with systemd escaping undone |
| `%p` | Prefix name | For `nginx@user1`, this is `nginx` |
| `%P` | Unescaped prefix name | Same as `%p` but with systemd escaping undone |
| `%n` | Full unit name | For `nginx@user1`, this is `nginx@user1.service` |
| `%N` | Unit name without type suffix | For `nginx@user1.service`, this is `nginx@user1` |
| `%a` | Architecture | `x86_64`, `aarch64`, etc. |
| `%l` | Short hostname | Hostname without domain part |
| `%m` | Machine ID | Contents of `/etc/machine-id` |
| `%o` | Operating system ID | From `/etc/os-release`, e.g., `alpine` |
| `%%` | Percent sign | Literal `%` character |

The instance name is also made available as the `INSTANCE` environment variable in the OpenRC script.

## Limitations

- Not all systemd features are supported in the conversion process
- Some complex systemd unit files may require manual adjustment after conversion
- Socket activation is not supported
- Timer units are not supported

## Getting Help

- Open an issue on GitHub with:
  - The command you ran
  - The complete error message
  - Your Alpine Linux version
  - The content of your systemd service file

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

[MIT License](LICENSE)
