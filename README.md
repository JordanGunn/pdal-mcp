# pdal-mcp

PDAL MCP is an open-source server that exposes common **PDAL** command-line utilities through the **Model Context Protocol (MCP)**.  PDAL is a C++ library for reading and processing point‑cloud data such as LiDAR, analogous to GDAL for raster and vector data【213350811818443†L146-L162】.  MCP standardises how AI agents call external tools via JSON‑RPC, emphasising secure, discoverable and context‑preserving operations【922541277255947†L51-L66】【922541277255947†L132-L152】.  This project wraps PDAL’s CLI commands as MCP tools so AI assistants can inspect, translate and process point‑cloud data in a controlled environment.

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
- **PDAL** installed on the host system (ensure the `pdal` CLI is available in your `PATH`).
  You can install PDAL via conda:

  ```sh
  conda install -c conda-forge pdal
  ```

- Clone this repository:

  ```sh
  git clone https://github.com/JordanGunn/pdal-mcp.git
  cd pdal-mcp
  ```

- Create a virtual environment and install dependencies:

  ```sh
  python3 -m venv venv
  source venv/bin/activate
  pip install -r requirements.txt
  ```

### Running the server

This project uses **FastAPI** with Uvicorn to implement the MCP server.  Start the server after setting up a configuration file (see below):

```sh
# Copy the example config and edit as needed
cp config.example.json config.json

# Set the environment variable to point to your config
export PDAL_MCP_CONFIG=$(pwd)/config.json

# Start the MCP server
uvicorn src.main:app --host 0.0.0.0 --port 3001
```

> **Note:** Some MCP servers distributed by others use `npx` or [uv](https://github.com/astrocorp/uv) as packaging tools.  This project targets Python packaging directly; you do not need to run `npx` or `uv`.

## Configuration

The server reads configuration from a JSON file.  Set the path to the file in the `PDAL_MCP_CONFIG` environment variable before starting the server.  A typical configuration looks like:

```json
{
  "workdir": "/srv/pdal_mcp/workdir",
  "allowedCommands": ["info","translate","pipeline","merge","split","tile","tindex"],
  "maxTimeout": 60000,
  "port": 3001
}
```

- **workdir** – Directory where input and output files are stored.  The server will refuse paths outside this folder.
- **allowedCommands** – List of PDAL commands enabled on the server.
- **maxTimeout** – Maximum run time (ms) for any command.
- **port** – Port to bind when launching the server via Uvicorn.

## MCP tools

| Tool name | Description |
| --- | --- |
| **pdal_info** | Wraps `pdal info` to display dataset information such as extents, point count, coordinate system, statistics and metadata【872216820604572†L304-L341】. |
| **pdal_translate** | Wraps `pdal translate` to convert point‑cloud files or run simple filter pipelines; supports JSON pipeline files or `--filter` arguments【962225138174121†L308-L356】. |
| **pdal_pipeline** | Executes full PDAL pipelines described in JSON using `pdal pipeline`, with options for streaming and metadata output【556921230097715†L305-L327】【556921230097715†L330-L390】. |
| **pdal_merge** | Merges multiple input files into one without filtering or reprojection【301070071328877†L307-L314】. |
| **pdal_split** | Splits a point‑cloud file into multiple outputs based on tile size or point capacity; outputs are named using a template【697690024905749†L305-L331】【697690024905749†L339-L341】. |
| **pdal_tile** | Generates square tiles from an input file with optional buffer and output spatial reference; output filenames must contain a `#` placeholder【644268424861177†L304-L345】【644268424861177†L346-L350】. |
| **pdal_tindex** | Creates or merges a GDAL‑style tile index for point‑cloud files【795595144191429†L308-L354】【795595144191429†L375-L378】. |

See the design document for each tool’s input and output JSON schema.

## Example usage

After starting the server on port 3001, you can call MCP methods via JSON‑RPC:

List available tools:

```sh
curl -X POST http://localhost:3001/mcp/tools/list \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

Call `pdal_info` on a file in the working directory:

```sh
curl -X POST http://localhost:3001/mcp/tools/call \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "id": 2,
  "params": {
    "name": "pdal_info",
    "input": {
      "inputFile": "samples/lidar.las",
      "stats": true,
      "schema": true
    }
  }
}
EOF
```

The server returns JSON containing `stdout` (statistics and schema) and `stderr`.  For tools that create files, the response includes a `resourceUri` that you can fetch via the resources API.

## Contributing

Contributions are welcome!  Please read [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on setting up a development environment, coding standards, adding new tools and reporting issues.

## Code of Conduct

We adhere to the [Contributor Covenant](CODE_OF_CONDUCT.md).  By participating in this project, you agree to follow its terms.

## License

This project is licensed under the MIT License – see [LICENSE](LICENSE) for details.

## Acknowledgments

- [PDAL](https://pdal.io) – The powerful open‑source library that this server wraps.
- The [Model Context Protocol](https://example.com) – A universal standard for connecting AI systems to external tools【922541277255947†L51-L66】【922541277255947†L132-L152】.
