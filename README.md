# pdal-mcp

PDAL MCP is an open-source server that exposes common **PDAL** command-line utilities through the **Model Context Protocol (MCP)**. PDAL is a C++ library for reading and processing point-cloud data such as LiDAR, analogous to GDAL for raster and vector data【213350811818443†L146-L162】. MCP standardises how AI agents call external tools via JSON‑RPC, emphasising secure, discoverable and context-preserving operations【922541277255947†L51-L66】【922541277255947†L132-L152】. This project wraps PDAL’s CLI commands as MCP tools so AI assistants can inspect, translate and process point‑cloud data in a controlled environment.

## Features

- Provides MCP tool definitions for key PDAL commands: `info`, `translate`, `pipeline`, `merge`, `split`, `tile` and `tindex`.
- Validates inputs via JSON schema and prevents access outside of a configured working directory.
- Produces structured outputs (JSON or file resources) for easy consumption by AI agents.
- Configurable through a JSON file and environment variable.
- Designed to be open-source and extensible—contributors can add more PDAL tools.

For architectural details and JSON schemas for each tool, see [pdal_mcp_design.md](pdal_mcp_design.md).

## Installation

### Prerequisites

- **Python 3.9+**.
- **PDAL** installed on the host system (ensure the `pdal` CLI is available in your `PATH`). You can install PDAL via conda:

```sh
conda install -c conda-forge pdal
```

- Clone this repository:

```sh
git clone https://github.com/JordanGunn/pdal-mcp.git
cd pdal-mcp
```

### Install dependencies

- **Recommended**: use [`uv`](https://github.com/astronautcorp/uv) to manage a local Python environment and install the MCP Python SDK and PDAL bindings:

```sh
uv init  # initialise a uv project (skip if already in a uv-managed directory)
uv add "mcp[cli]" pdal
```

- **Alternatively**, install via pip:

```sh
python3 -m pip install "mcp[cli]" pdal
```

## Running the server

This project uses the MCP Python SDK’s **FastMCP** server. After creating a configuration file (see below), you can run the server with `uv`:

```sh
# Copy the example config and edit as needed
cp config.example.json config.json

# Point the environment variable to your config
export PDAL_MCP_CONFIG=$(pwd)/config.json

# Start the MCP server in development mode with hot‑reloading
uv run mcp dev src/server.py

# Start the MCP server via stdio (useful for production)
uv run mcp src/server.py stdio
```

> **Note:** Some MCP servers are distributed via Node (`npx`) or Rust (`uvx`), but this project uses the official Python SDK and `uv` for dependency management and execution.

## Configuration

The server reads configuration from a JSON file. Set the path to the file in the `PDAL_MCP_CONFIG` environment variable before starting the server. A typical configuration looks like:

```json
{
  "workdir": "/srv/pdal_mcp/workdir",
  "allowedCommands": ["info","translate","pipeline","merge","split","tile","tindex"],
  "maxTimeout": 60000,
  "port": 3001
}
```

- **workdir** – Directory where input and output files are stored. The server refuses paths outside this folder.
- **allowedCommands** – List of PDAL commands enabled on the server.
- **maxTimeout** – Maximum run time (ms) for any command.
- **port** – Port to bind when launching the server via uv.

## MCP tools

| Tool name | Description |
| --- | --- |
| **pdal_info** | Wraps `pdal info` to display dataset information such as extents, point count, coordinate system, statistics and metadata【872216820604572†L304-L341】. |
| **pdal_translate** | Wraps `pdal translate` to convert point-cloud files or run simple filter pipelines; supports JSON pipeline files or `--filter` arguments【962225138174121†L308-L356】. |
| **pdal_pipeline** | Executes full PDAL pipelines described in JSON using `pdal pipeline`, with options for streaming and metadata output【556921230097715†L305-L327】【556921230097715†L330-L390】. |
| **pdal_merge** | Merges multiple input files into one without filtering or reprojection【301070071328877†L307-L314】. |
| **pdal_split** | Splits a point-cloud file into multiple outputs based on tile size or point capacity; outputs are named using a template【697690024905749†L305-L331】【697690024905749†L339-L341】. |
| **pdal_tile** | Generates square tiles from an input file while handling an optional buffer and output spatial reference; output filenames must contain a `#` placeholder【644268424861177†L304-L345】【644268424861177†L346-L350】. |
| **pdal_tindex** | Creates or merges a GDAL-style tile index for PDAL‑readable point-cloud files【79559144191429†L308-L354】【79559144191429†L375-L378】. |

See the design document for each tool’s input and output JSON schema.

## Example usage

After starting the server on port `3001`, you can call MCP methods via JSON‑RPC:

List available tools:

```sh
curl -X POST http://localhost:3001/mcp/tools/list \
 -H "Content-Type: application/json" \
 -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

Call `pdal_info` on a file in the working directory:

```sh
curl -X POST http://localhost:3001/mcp/calls \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "jsonrpc": "2.0",
  "method": "pdal_info",
  "params": {
    "inputFile": "data.las",
    "all": true
  },
  "id": 1
}
EOF
```

## Contributing

CContributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on how to report issues, propose new features and add new PDAL tools to this project.tools to this project.
## Code of Conduct

This project adheres to a [Code of Conduct](CODE_OF_CONDUCT.md). By participating, you are expected to uphold this code.

## License

This project is licensed under the [MIT License](LICENSE).

## Acknowledgments

- [PDAL](https://pdal.io/) and its developers.
- [Model Context Protocol](https://modelcontext.org) community for defining the MCP specification.
