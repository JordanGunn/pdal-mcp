# Contributing to PDAL MCP

Thank you for your interest in contributing to PDAL MCP! We welcome contributions to improve the server and add new tools.

## Getting started

1. **Fork** this repository and clone your fork locally.
2. **Create a branch** for your feature or bug fix:

   ```bash
   git checkout -b my-feature
   ```

3. **Set up a Python virtual environment** and install dependencies:

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

   Ensure that PDAL is installed on your system and that the `pdal` CLI is available in your `PATH`. You can install PDAL via conda:

   ```bash
   conda install -c conda-forge pdal
   ```

4. **Write your code**, following PEP 8 coding conventions and using type hints where appropriate.
5. **Write unit tests** for your changes using `pytest`.
6. **Commit** your changes with descriptive messages and push to your fork.
7. **Open a pull request** against the `master` branch describing your changes and linking to any relevant issues.

## Adding a new MCP tool

To expose a new PDAL command as an MCP tool:

- Create a Python wrapper function in the `src/tools/` directory that invokes the appropriate PDAL CLI command using Python's `subprocess` module.
- Define the tool's metadata (name, title, description, `inputSchema`, `outputSchema` and annotations) so that AI agents understand the tool's purpose and parameters. See the design document (`pdal_mcp_design.md`) for examples.
- Validate all inputs, ensuring file paths are within the configured working directory and parameters are safe.
- Return structured outputs (e.g. JSON summaries or file resources) to clients.
- Add unit tests for the new tool and update the documentation (README and design doc) accordingly.
- Ensure the new tool is exported by the server so that it appears in responses to `tools/list` requests.

## Reporting issues

If you encounter a bug, please open an issue with a clear and concise description of the problem, the steps to reproduce, and any relevant logs or sample data. For feature requests, explain your use case and why the feature would be valuable.

## Code review and merging

All pull requests are subject to review by the maintainers. We may suggest changes or ask questions before merging. Please be responsive and incorporate feedback. Once your pull request is approved and all tests pass, it will be merged into the `master` branch.
