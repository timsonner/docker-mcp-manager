# Docker Container Manager MCP Server

This repository is a local [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server that lets an agent manage one long-running Docker container. The server speaks MCP over stdin and stdout. Its tools shell out to the `docker` CLI against a container named `managed-container`.

File tools keep data on a Docker named volume mounted at `/data` inside that container, so files survive container restarts. `execute_command` runs a shell as root in that privileged container. The repo wires the server up for OpenCode (`opencode.json`) and VS Code (`.vscode/mcp.json`).

## Prerequisites

- Python 3.10 or newer, available as `python3`. The `mcp` 1.x release that provides `FastMCP` requires Python >= 3.10.
- Docker Engine, with the `docker` CLI on `PATH`. Every tool calls `docker`.
- Docker Compose v2 (`docker compose`) to start the service in `docker-mcp-manager/docker-compose.yml`.

## Install, run, and test

Run these from the root of the git repository (the directory that contains `docker-mcp-manager/`).

### Install

```bash
python3 -m pip install 'mcp[cli]>=1.6.0,<2'
```

`docker-mcp-manager/pyproject.toml` depends on `mcp[cli]>=1.6.0`. `docker_mcp_server.py` imports `FastMCP` from `mcp.server.fastmcp`. That class is in mcp 1.x (verified with 1.30.0). mcp 2 removes it, so the upper bound is required for this code to import.

There is no build step. Start the server with `python3` as shown below.

### Run the container

```bash
docker compose -f docker-mcp-manager/docker-compose.yml up -d
docker compose -f docker-mcp-manager/docker-compose.yml ps
```

`docker-mcp-manager/docker-compose.yml` defines one service:

- image `ubuntu:24.04`
- container name `managed-container`
- `privileged: true`
- named volume `persistent-data` mounted at `/data`
- process `tail -f /dev/null`, so the container stays up

Logs and the privileged flag:

```bash
docker compose -f docker-mcp-manager/docker-compose.yml logs managed-container
docker inspect managed-container --format '{{.HostConfig.Privileged}}'
```

Stop the container and keep the named volume:

```bash
docker compose -f docker-mcp-manager/docker-compose.yml down
```

### Run the MCP server

```bash
python3 docker-mcp-manager/docker_mcp_server.py
```

The process stays in the foreground. MCP messages use stdin and stdout. Logs go to stderr.

### Test

The repository has no automated test suite. `pyproject.toml` lists optional dev dependencies `pytest`, `black`, and `ruff`, and it does not include test files or config for those tools.

`docker-mcp-manager/mcp_client.py` is a manual client. It starts `python3 docker_mcp_server.py` in the current working directory, lists tools, and calls each one. Run it from `docker-mcp-manager/` after the container is up:

```bash
cd docker-mcp-manager
python3 mcp_client.py
```

## Usage

The server name passed to FastMCP is `docker-mcp-manager`. Tool arguments below match the Python functions in `docker_mcp_server.py`.

File paths are joined onto `/data`. A leading `/` is removed first, so `notes.txt` and `/notes.txt` both resolve to `/data/notes.txt`. `.` and an empty path resolve to `/data`. A path that would leave `/data` is rejected. `execute_command` is not limited to `/data`; it runs the shell string in the container.

`mcp_client.py` passes file paths such as `/data/test.txt`. After the leading slash is stripped, those resolve under `/data/data/`.

### `execute_command`

Run a shell command as root in `managed-container` (`docker exec --user root --privileged managed-container sh -c <command>`).

- `command` (string): shell command
- Returns the combined stdout and stderr. On a non-zero exit, the string starts with `Command failed with exit code`.

### `read_file`

- `file_path` (string): path under `/data`, using the join rules above
- Returns the file contents, or an error string

### `write_file`

- `file_path` (string): path under `/data`
- `content` (string): bytes to write, decoded as text
- Returns `Successfully wrote to <path>` or an error string

### `list_directory`

- `dir_path` (string): directory under `/data`
- Returns a list of names parsed from `ls -la` (this includes `.` and `..` when `ls` prints them). On failure, returns a one-element list whose item is an error string.

### `check_file_exists`

- `file_path` (string): path under `/data`
- Returns `true` or `false`. A rejected path or a failed `docker` call returns `false`.

### `get_container_info`

No parameters. When `managed-container` is running, returns an object with:

- `id`, `name`, `status`, `image`, `ports`
- `host_config.privileged`
- `host_config.binds`

If the container is missing or not running, returns `{"error": "<message>"}`.

### OpenCode

`opencode.json` at the repository root already registers the server. The command is relative to that root:

```json
{
  "mcp": {
    "docker-mcp-manager": {
      "type": "local",
      "command": ["python3", "docker-mcp-manager/docker_mcp_server.py"],
      "enabled": true
    }
  }
}
```

### VS Code

`.vscode/mcp.json` already registers the server for the MCP client in VS Code. Paths are relative to the repository root:

```json
{
  "servers": {
    "docker-container-manager": {
      "type": "stdio",
      "command": "python3",
      "args": [
        "docker-mcp-manager/docker_mcp_server.py"
      ],
      "env": {}
    }
  }
}
```

## Layout

```text
.
├── opencode.json
├── .vscode/mcp.json
├── .opencode/skills/docker-mcp-manager/SKILL.md
└── docker-mcp-manager/
    ├── README.md
    ├── docker_mcp_server.py
    ├── mcp_client.py
    ├── docker-compose.yml
    └── pyproject.toml
```

## License

This repository does not include a LICENSE file.
