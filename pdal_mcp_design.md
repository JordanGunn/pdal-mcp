# PDAL MCP Design and Implementation Plan

## Introduction and context

The **Point Data Abstraction Library (PDAL)** is an open‑source C++ library designed for translating and manipulating point‑cloud data such as LiDAR.  PDAL plays a similar role for point clouds as GDAL does for raster and vector data【213350811818443†L146-L162】.  It provides a suite of command‑line utilities for inspecting, converting, merging, splitting and indexing point‑cloud files.

The **Model Context Protocol (MCP)** defines a standard for connecting AI agents to external data sources and tools.  MCP uses JSON‑RPC 2.0 messages to describe resources, prompt templates and tools exposed by a server.  The protocol emphasises **universal interfaces**, **security and access control**, **bidirectional communication**, **context preservation** and capability discovery【922541277255947†L51-L66】【922541277255947†L132-L152】.  Under MCP, each tool is described with a name, title, description and JSON schema for its inputs and outputs, allowing clients to introspect available operations and safely execute them.

This document outlines the design of a **PDAL MCP server** that exposes selected PDAL command‑line utilities as MCP tools.  The goal is to create an open‑source server that follows best practices for security, modularity, and usability while facilitating community contributions.

## Objectives

* **Expose core PDAL utilities as MCP tools** — Provide wrappers for key PDAL commands (`info`, `translate`, `pipeline`, `merge`, `split`, `tile`, `tindex`), enabling AI clients to inspect and manipulate point‑cloud datasets.
* **Conform to MCP design principles** — Adhere to the protocol’s requirements for security and user consent, including validating inputs, sandboxing file operations, and requiring confirmation for actions that write or delete data.
* **Plan for extensibility** — Provide a clear framework so contributors can easily add new PDAL tools or extend existing ones.
* **Open‑source and community‑friendly** — Build a repository with standard documentation (README, CONTRIBUTING, CODE_OF_CONDUCT, CHANGELOG) and a permissive license so others can use and improve the project.

## Overview of PDAL commands

The PDAL utilities targeted for MCP wrapping include:

| Command | Purpose and key options | Sources |
| --- | --- | --- |
| **`pdal info`** | Displays information about a point‑cloud file, including extents, number of points, coordinate reference system, metadata and summary statistics【872216820604572†L304-L341】.  Options include `--input` (file path), `--all`, `--stats`, `--point` (specific point index), `--query` (spatial query), `--schema` (dimensions), `--metadata`, etc.  If no specific option is provided, PDAL prints statistics by default. | PDAL docs【872216820604572†L304-L341】 |
| **`pdal translate`** | Converts files from one format to another or executes simple pipelines.  It requires an input and output file and can accept a filter list (`--filter`), a full pipeline specification via `--json` or `--pipeline`, or options like `--metadata` and `--dims`.  It runs in stream mode if possible【962225138174121†L308-L356】. | PDAL docs【962225138174121†L308-L356】 |
| **`pdal pipeline`** | Executes a PDAL pipeline described in JSON.  It supports reading the pipeline from a file or STDIN (`--input`), validating pipelines, streaming processing, producing metadata, and overriding stage parameters via substitutions【556921230097715†L305-L327】【556921230097715†L330-L390】. | PDAL docs【556921230097715†L305-L327】【556921230097715†L330-L390】 |
| **`pdal merge`** | Merges multiple files without filtering or reprojection; the last file specified is the output.  It simply concatenates the inputs, allowing heterogenous input formats【301070071328877†L307-L314】. | PDAL docs【301070071328877†L307-L314】 |
| **`pdal split`** | Splits a single input file into multiple output files.  The output filename acts as a template and may contain a sequence placeholder; if a directory is given, PDAL writes files named `file_#.ext`.  Options include `--length` (tile side length), `--capacity` (number of points per chip) and origin coordinates; by default, capacity is 100 000 points【697690024905749†L305-L331】【697690024905749†L339-L341】. | PDAL docs【697690024905749†L305-L331】【697690024905749†L339-L341】 |
| **`pdal tile`** | Generates square tiles of points in streaming mode.  Requires an output filename with a `#` placeholder and optional tile `--length`, `--buffer` width, origin coordinates and output spatial reference; tiles cover all points and do not create directories【644268424861177†L304-L350】. | PDAL docs【644268424861177†L304-L345】【644268424861177†L346-L350】 |
| **`pdal tindex`** | Creates a GDAL‑style tile index for point clouds or merges points from an existing tile index.  It accepts a target index file (`--tindex`), a file specification pattern (`--filespec`), and options for layer naming, coordinate systems, output driver, writing absolute paths and skipping files with different SRIDs【795595144191429†L308-L354】.  In merge mode it reads an index and produces merged output【795595144191429†L375-L378】. | PDAL docs【795595144191429†L308-L354】【795595144191429†L375-L378】 |

The MCP server will wrap these commands to provide AI clients with structured, safe access to PDAL functionality.  Additional PDAL tools can be added in future releases.

## Architecture and design

### High‑level architecture

1. **Server implementation** — Use Python with FastAPI to implement a JSON‑RPC server.  The server will expose MCP endpoints (`/mcp/resources/list`, `/mcp/tools/list`, `/mcp/tools/call`) and run under an ASGI server such as Uvicorn.  It will read configuration from a JSON file (path provided via `PDAL_MCP_CONFIG` environment variable) specifying allowed working directory, allowed commands and maximum timeout.

2. **Tool wrappers** — For each PDAL command, create a Python function that:
   * Parses and validates JSON‑RPC input parameters against a defined JSON schema.
   * Constructs the appropriate PDAL CLI command with validated arguments.
   * Executes the command in a subprocess within a sandboxed working directory and time limit.
   * Captures standard output and error streams.  For operations that produce files, return a resource URI pointing to the output file.  For `info` and `pipeline` commands, return parsed JSON when possible.

3. **Tool definitions** — Use MCP tool definitions including `name`, `title`, `description`, `inputSchema`, `outputSchema` and optional `annotations`.  For example, the `pdal_info` tool will accept `inputFile` and optional flags (e.g., `all`, `stats`, `schema`) and return the PDAL info output as JSON.

4. **Resource management** — Provide a `resources` endpoint that lists files within the server’s working directory.  When tools produce new files, return URIs referencing these resources so clients can fetch them via GET requests.

5. **Security and consent** — The server will implement strict validation of file paths to prevent directory traversal or access outside the configured working directory.  For commands that write data or modify files (`translate`, `merge`, `split`, `tile`, `tindex`), the server will require explicit confirmation by including a flag `confirmWrite: true` in the input.  The server will log all tool invocations with timestamps for auditing.  No network access or external commands outside PDAL will be allowed.

### Example tool definition

Below is a schematic JSON definition for the `pdal_info` tool.  Actual implementation may include more options and output schemas.

```json
{
  "name": "pdal_info",
  "title": "PDAL info",
  "description": "Inspect point‑cloud files with PDAL’s info utility and return metadata, statistics and schema information.",
  "inputSchema": {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "required": ["inputFile"],
    "properties": {
      "inputFile": { "type": "string", "description": "Path to the input point‑cloud file." },
      "all": { "type": "boolean", "description": "Return all info fields (stats, schema, metadata)." },
      "stats": { "type": "boolean", "description": "Return summary statistics about the dataset." },
      "schema": { "type": "boolean", "description": "Return dimension schema information." },
      "metadata": { "type": "boolean", "description": "Return metadata content." }
    },
    "additionalProperties": false
  },
  "outputSchema": {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "properties": {
      "stdout": { "type": "string" },
      "stderr": { "type": "string" }
    }
  }
}
```

Other tools will have analogous schemas.  For example, `pdal_translate` will accept `inputFile`, `outputFile`, optional `filters` (array of strings), `jsonPipeline`, etc., and will return an output file resource.  `pdal_split` and `pdal_tile` will require `length` or `capacity` and produce multiple output files.

### Configuration file

The server reads configuration from a JSON file, for example:

```json
{
  "workdir": "/srv/pdal_mcp/workdir",
  "allowedCommands": ["info", "translate", "pipeline", "merge", "split", "tile", "tindex"],
  "maxTimeout": 60000,
  "port": 3001
}
```

* `workdir` – Directory where input and output files reside.  Tools will reject paths outside this directory.
* `allowedCommands` – List of PDAL commands that the server can execute.
* `maxTimeout` – Maximum execution time (in milliseconds) for any tool invocation.
* `port` – Port on which the MCP server listens.

Set the environment variable `PDAL_MCP_CONFIG` to the path of this configuration file when starting the server.  This design mirrors the configuration for the GDAL MCP server to maintain consistency.

### Implementation plan

1. **Repository setup** — Initialise a new Git repository `pdal-mcp` with an MIT license.  Include the design document (`pdal_mcp_design.md`) and a comprehensive README describing the project, installation, configuration, and usage.

2. **Dependencies** — The server will depend on Python 3.9+, [FastAPI](https://fastapi.tiangolo.com/), `pydantic` for schema validation, and PDAL installed on the host system.  A `requirements.txt` file will list these dependencies.  PDAL can be installed via package managers or conda; ensure the server environment has PDAL CLI accessible.

3. **Core server** — Create a `src/` directory containing:
   * `main.py` – Launches FastAPI, reads configuration and registers MCP routes.
   * `tools/` – Contains Python modules for each PDAL command wrapper (`info.py`, `translate.py`, etc.).  Each module exports a function to build and execute the PDAL command and returns outputs in MCP‑compatible format.
   * `schemas/` – Holds JSON schema definitions for tool inputs and outputs to separate logic from definitions.
   * `utils/` – Helper functions for path validation, subprocess execution, timeout handling and resource management.

4. **Resource endpoint** — Implement an endpoint under `/mcp/resources/list` to list available files in `workdir` and `/mcp/resources/get` to serve file contents.  Use streaming responses for large files.

5. **Tool registry** — Build a registry of available tool definitions.  The `/mcp/tools/list` endpoint returns this registry to clients.  The `/mcp/tools/call` endpoint validates input against the appropriate schema and invokes the wrapper function.

6. **Testing** — Provide unit tests under `tests/` to verify individual wrappers, schema validation and error handling.  Use sample LAS/LAZ files in `tests/data/` for functional tests.

7. **Documentation and examples** — Extend the README with instructions to install PDAL (e.g., via `conda install -c conda-forge pdal`), configure the server, start it (e.g., `uvicorn src.main:app --reload`), and call MCP tools with `curl` or an example client.  Provide an example JSON pipeline file and sample command usage.

8. **Continuous integration** — Configure GitHub Actions to run tests on push and pull requests.  Add a `CODEOWNERS` file to ensure appropriate reviewers for PDAL code.  Enable Dependabot and secret scanning to maintain security.

## Repository structure

The repository will follow best practices for open‑source projects:

```
pdal-mcp/
├── LICENSE
├── README.md
├── pdal_mcp_design.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── CHANGELOG.md
├── SECURITY.md
├── SUPPORT.md
├── requirements.txt
├── config.example.json
├── src/
│   ├── main.py
│   ├── tools/
│   │   ├── info.py
│   │   ├── translate.py
│   │   ├── pipeline.py
│   │   ├── merge.py
│   │   ├── split.py
│   │   ├── tile.py
│   │   └── tindex.py
│   ├── schemas/
│   │   ├── info_schema.json
│   │   └── ...
│   └── utils/
│       ├── paths.py
│       └── subprocess.py
├── tests/
│   ├── test_info.py
│   ├── data/
│   │   └── sample.laz
└── .github/
    ├── workflows/
    │   └── python.yml
    ├── ISSUE_TEMPLATE/
    │   └── bug_report.md
    └── PULL_REQUEST_TEMPLATE.md
```

Additional documents include:

* **CONTRIBUTING.md** — Guidelines for contributors on setting up a development environment, following coding conventions, adding new tools, reporting issues, and submitting pull requests.
* **CODE_OF_CONDUCT.md** — Adopt the Contributor Covenant to ensure a welcoming environment【534066316430593†L76-L90】.
* **CHANGELOG.md** — Records notable changes across releases, adhering to Keep a Changelog guidelines.
* **SECURITY.md** — Outlines supported versions and how to report vulnerabilities.
* **SUPPORT.md** — Provides instructions for seeking help or contacting maintainers.

## Conclusion

This design document provides a blueprint for building a PDAL MCP server that exposes PDAL’s powerful point‑cloud utilities through the Model Context Protocol.  By following MCP’s principles of standardisation, security and extensibility【922541277255947†L51-L66】【922541277255947†L132-L152】, and by adopting best practices for open‑source development, the PDAL MCP project aims to serve both AI agents and the wider geospatial community.
